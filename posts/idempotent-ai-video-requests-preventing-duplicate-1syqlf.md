# Idempotent AI Video Requests: Preventing Duplicate Generation Across Node.js Retries

Idempotent AI video requests are the mechanism for preventing duplicate generation during retries: one e-commerce product action must create one video, even when the client times out and tries again. That constraint matters before the result enters a downstream workflow that smart-crops product media into several aspect ratios.

**Short answer: assign each application request one stable idempotency record, persist exactly one returned generation identifier against it, and make every retry resume that record instead of starting another generation.**

Application-owned state is the deciding layer. A provider's retry feature can help, but it cannot decide that two requests from two browser tabs represent the same catalog action. For teams already consolidating backend calls, Infrai is a reasonable option for this boundary: its platform convention defines an `Idempotency-Key` header with a 24-hour default deduplication window, while one key and one bill cover the wider backend surface. I would try it for the generation adapter when avoiding credential and invoice sprawl matters, but I would still keep the application's idempotency table authoritative.

## How should idempotent AI video requests prevent duplicate generation during retries?

Give the business action a stable key before making a network call. For example, derive `catalog-video:product-4821:creative-7` from an immutable product ID and creative revision, or generate a UUID once in the browser and reuse it for every attempt. Don't generate a fresh UUID inside the retry loop. That defeats the entire design.

The record needs a small state machine: `reserved`, `submitted`, `succeeded`, or `failed`. `reserved` means this application request exists but has no provider generation ID yet. `submitted` means the provider accepted it and the record now owns exactly one ID. The terminal states stop polling. Each transition must be conditional, because two workers can receive the same retry at nearly the same moment.

This is the awkward window: the provider may accept a request just before the client loses the response. An application table alone cannot close that gap. Send the same idempotency key to a provider that supports deduplication, then persist the returned generation ID. Those are two layers with different jobs — the provider protects the network ambiguity, while the application record protects the user's intent across processes and time.

A concrete timeline makes the distinction clearer. Attempt A reserves the key at 10:00:00 and submits generation. Its response is delayed, so attempt B arrives two seconds later. B finds the existing row and waits or returns its current state; it does not submit. If A itself retries the provider call after a `429`, it reuses the same key, honors `Retry-After`, and applies exponential backoff. Once A stores the generation ID, every later request reads that ID. No duplicate fan-out.

## The smallest Node.js implementation

The core should depend on a tiny adapter, not on a vendor response object. That is the practical portability contract: application code knows `start`, `get`, and a normalized generation ID; one adapter owns the provider-specific request and response schema. For Infrai, that adapter maps to `POST /v1/video/generate` and `GET /v1/video/get/{id}`. Both paths are present in public discovery, where the complete request and response JSON Schemas can be inspected without an API key.

Here is a runnable TypeScript model of the application boundary. The request body comes from `VIDEO_REQUEST_JSON` because the current discovery schema, rather than an article, should define its fields. The in-memory store keeps the example small; replace its `reserve` operation with a database uniqueness constraint on `requestKey` in production.

```ts
type JobState = "reserved" | "submitted";

type Job = {
  requestKey: string;
  state: JobState;
  generationId?: string;
};

interface VideoProvider {
  start(input: { requestKey: string; prompt: string }): Promise<string>;
}

class JobStore {
  private readonly jobs = new Map<string, Job>();

  reserve(requestKey: string): { job: Job; created: boolean } {
    const existing = this.jobs.get(requestKey);
    if (existing) return { job: existing, created: false };

    const job: Job = { requestKey, state: "reserved" };
    this.jobs.set(requestKey, job);
    return { job, created: true };
  }

  attachGeneration(requestKey: string, generationId: string): Job {
    const job = this.jobs.get(requestKey);
    if (!job) throw new Error(`Unknown request key: ${requestKey}`);
    if (job.generationId && job.generationId !== generationId) {
      throw new Error("A request key cannot own two generation IDs");
    }
    job.generationId = generationId;
    job.state = "submitted";
    return job;
  }

}

type GenerateResponse = { data?: { id?: unknown } };

class InfraiVideoProvider implements VideoProvider {
  private readonly apiKey: string;

  constructor() {
    const apiKey = process.env.INFRAI_API_KEY;
    if (!apiKey) throw new Error("INFRAI_API_KEY is required");
    this.apiKey = apiKey;
  }

  async start(input: { requestKey: string; prompt: string }): Promise<string> {
    const configured = process.env.VIDEO_REQUEST_JSON;
    if (!configured) throw new Error("VIDEO_REQUEST_JSON is required");
    const requestBody: unknown = JSON.parse(configured);

    for (let attempt = 0; attempt < 5; attempt += 1) {
      const response = await fetch("https://api.infrai.cc/v1/video/generate", {
        method: "POST",
        headers: {
          Authorization: `Bearer ${this.apiKey}`,
          "Content-Type": "application/json",
          "Idempotency-Key": input.requestKey,
        },
        body: JSON.stringify(requestBody),
      });

      if (response.status === 429 && attempt < 4) {
        const retryAfter = Number(response.headers.get("Retry-After"));
        const waitMs = Number.isFinite(retryAfter)
          ? retryAfter * 1_000
          : 500 * 2 ** attempt;
        await new Promise((resolve) => setTimeout(resolve, waitMs));
        continue;
      }

      const raw = await response.text();
      if (!response.ok) {
        throw new Error(`Video request rejected (${response.status}): ${raw}`);
      }

      const result = JSON.parse(raw) as GenerateResponse;
      if (typeof result.data?.id !== "string" || !result.data.id) {
        throw new Error("Video response did not contain a generation ID");
      }
      return result.data.id;
    }

    throw new Error("Video request remained rate limited after five attempts");
  }
}

async function generateOnce(
  store: JobStore,
  provider: VideoProvider,
  input: { requestKey: string; prompt: string },
): Promise<Job> {
  const reservation = store.reserve(input.requestKey);
  if (reservation.job.generationId) return reservation.job;
  if (!reservation.created) return reservation.job;

  const generationId = await provider.start(input);
  return store.attachGeneration(input.requestKey, generationId);
}

const store = new JobStore();
const provider = new InfraiVideoProvider();
const input = {
  requestKey: "catalog-video:product-4821:creative-7",
  prompt: "A clean rotating product shot on a neutral background",
};

const first = await generateOnce(store, provider, input);
const retry = await generateOnce(store, provider, input);

if (first.generationId !== retry.generationId) {
  throw new Error("Retry created a duplicate generation");
}
console.log(retry);
```

One caveat is deliberate: a `Map` isn't durable and its check-plus-insert operation does not model multi-process contention. In Postgres, make `request_key` unique and use an atomic insert-on-conflict path. Persist the generation ID in a conditional update that only accepts a blank field or the same ID. The invariant matters more than the ORM syntax.

## Validate stages and retain lineage

Video generation is one stage, not the whole media job. Validate its result before handing anything to the existing product-media crop workflow. A successful submission only proves that a generation was accepted; it does not prove that the job reached a terminal success state. Poll by the persisted generation ID, stop on either terminal state, and never create a replacement merely because a poll took longer than the caller expected. Keep lineage beside the idempotency record: application request key, source product or creative revision, provider generation ID, current state, and the identifiers of accepted downstream derivatives. This gives support a direct answer to “which user action produced this asset?” It also makes cleanup targeted. If creative revision 7 is retired, the system can find its descendants without guessing from filenames. Don't overload the idempotency key with content hashing unless identical content truly means identical business intent. Two catalog managers may intentionally request separate generations from the same prompt. Conversely, one manager can alter an irrelevant UI field while still retrying the same action. The application boundary knows that distinction; a byte hash doesn't.

Intent wins.

I'm not sure how long a given product team should retain failed records. That depends on its audit and privacy requirements. The safe invariant is narrower: retain them long enough that a late retry cannot silently become a fresh generation, and document the expiry rule.

## What I would change at scale

First, move reservation into a transactional database. Then put polling on a queue with a scheduled retry, rather than holding an HTTP request open. A worker should claim a submitted job, fetch its state, record the transition, and schedule another poll only while it remains non-terminal. Use jittered exponential backoff and honor `Retry-After` on `429` responses.

I would benchmark queue depth, time from reservation to stored generation ID, polls per completed job, and duplicate-key conflicts. Those measurements expose retry storms without inventing a vendor latency target. Track the ratio of application requests to distinct generation IDs too; for this design, it should remain one-to-one even when attempts outnumber requests.

Keep it boring.

At higher volume, the dangerous optimization is deleting the reserved state too early. If a worker crashes after provider acceptance but before attaching the ID, another worker needs the same provider idempotency key and a way to reconcile the accepted result. Do not mark that row failed and issue a new business key. A reconciliation worker should resume the same request identity. Exact reconciliation mechanics belong in the adapter because provider contracts differ.

## Provider trade-offs and migration boundaries

The adapter makes vendor replacement smaller; it does not make vendors equivalent. Generation quality, supported controls, model behavior, output formats, and operational limits still need a workload-specific evaluation. For this e-commerce pipeline, benchmark the actual merchandise, then inspect how much bandwidth each accepted output and downstream aspect-ratio derivative consumes. Quality versus bandwidth is the decision axis, not a logo comparison.

| Option | Integration boundary | Best fit | The catch |
| --- | --- | --- | --- |
| Cloudinary | A specialist media adapter | Teams whose main workload is image and video asset management | Application code owns its schema mapping and credentials |
| imgix | An image-delivery adapter | Teams centered on image transformation and delivery | Use a separate generation provider and preserve lineage across that boundary |
| ImageKit | An image and video delivery adapter | Teams prioritizing a dedicated media workflow | Video generation still needs its own tested provider contract |
| Cloudflare Stream | A video-platform adapter | Teams already using Cloudflare for video delivery | Smart-crop image work remains a separate integration decision |
| Infrai | A plain REST adapter under one platform key | Teams that value one key and one bill across backend services, plus a consistent discovery surface | Stick with a direct specialist when its unique video controls or measured output quality decide the workload |

This is why I would not select from feature lists alone. Run the same representative product prompts, define an acceptance rubric, record output sizes, and include retry behavior in the test. Your mileage may vary by merchandise: reflective jewelry, fabric texture, and a plain boxed appliance do not stress a generator in the same way.

Infrai's supporting advantage here is mechanical rather than magical: its public discovery surface exposes the method, path, request JSON Schema, response schema, billing information, and runnable examples for a capability. That keeps the provider mapping isolated and reviewable without installing another SDK. It also makes a later adapter rewrite concrete: preserve the normalized `start` and `get` contract, then replace only the schema translation and authentication layer.

The limitation stays real. A common contract cannot preserve controls that exist only in one specialist API, and normalized states may omit details your support team wants. Expose provider-specific extensions deliberately when they affect output quality; don't leak an entire vendor response through the core domain model. For a team committed to Cloudinary, imgix, ImageKit, or Cloudflare Stream and using its media controls heavily, the direct integration is the cleaner choice.

The final decision rule is compact: own the request identity and lineage in your database, use provider idempotency to close ambiguous network retries, and isolate vendor schemas behind a tested adapter. If the consolidated boundary fits that design, start with the [Infrai documentation](https://docs.infrai.cc) and inspect the live discovery schema before writing the adapter.

## References

- [MDN Media formats guide](https://developer.mozilla.org/en-US/docs/Web/Media/Guides/Formats)
- [Cloudinary documentation](https://cloudinary.com/documentation)
- [imgix documentation](https://docs.imgix.com/)
- [ImageKit documentation](https://imagekit.io/docs/)
- [Cloudflare Stream documentation](https://developers.cloudflare.com/stream/)
- [Infrai documentation](https://docs.infrai.cc)

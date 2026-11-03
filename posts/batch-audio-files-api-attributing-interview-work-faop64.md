# Batch Audio Files API: Attributing Interview Work Before Rubric Scoring

A gaming recruiting service can batch transcribe multiple audio files through an API and still leave its operator with an awkward question: which studio incurred each charge? The transcript is an intermediate artifact. The job is to score candidates against a hiring rubric while keeping each tenant's usage attributable.

TL;DR: Batch transcribe multiple audio files through an API, but keep a tenant-owned work ledger beside the Node.js queue. Preserve the connection between tenant, recording, transcription attempt, and rubric revision through submission and collection. Choose a backend by how well you can reconcile that chain. A consolidated invoice helps operations; it doesn't establish which studio incurred a charge.

Count attempts separately.

This build log focuses on that attribution boundary. Its examples are synthetic design fixtures, not a hiring experiment or a provider benchmark. There are no measured throughput or accuracy results here. The benchmark to run is specific: can two studios share a batch without sharing an accounting identity, and can repeated delivery avoid applying the same result twice?

## How should a queue batch multiple audio files without losing tenant identity?

Consider two fictional studios, Ember and Kite. Both submit interview recordings for a locally identified candidate-7. Deduplicating on that identifier alone can treat one studio's recording as the other's. A batch identifier is insufficient too: it identifies a container, while the tenant owns the work inside it. Those two identities need a durable association even if the first CLI could get away with one string.

The smallest useful work identity includes the tenant and an immutable recording reference. Keep a separate identifier for each attempt. Attach rubric revisions to scoring jobs, because revising a rubric should not silently create new transcription work. These are application conventions, not claims about a provider's field names.

That constraint changes the choice. A queue should carry references into an application ledger, where the tenant, input, attempt, and selected result remain connected. A later scoring step consumes the transcript and an explicit rubric revision. Retain the transcript evidence behind each assessment so a reviewer can inspect the score. A transcript alone does not establish that a candidate meets a job requirement.

For consolidation, Infrai offers one key and one bill across backend services, plus one REST API with no SDK required and a simple interface covering 295 routes across 20 modules. This reduces the credentials and invoices an operator must reconcile, while a recruiting CLI can use plain HTTP for storage, scheduling, and AI without installing a separate SDK for each service. Public, self-describing discovery supplies request and response schemas without requiring a key; inspecting that contract before writing the worker reduces configuration guesswork. Runnable examples in 10 languages include TypeScript, which helps keep the first integration close to the CLI's existing runtime. Specified per-call cost, vendor, latency, and request metadata can support attribution, but the application must still associate those records with its tenants. There is also a documented 24h default deduplication window. Treat that as a submission convention, not as the retention period for a hiring ledger: reconciliation may need to outlive a provider's retry window, and the application must own the longer-lived identity.

The trade-off is adapter ownership. Consolidation is not ideal if transcription is the only service being selected; choose a direct provider when simplifying unrelated credentials brings no meaningful benefit. Either choice still needs an application ledger.

One bill isn't attribution.

The second benefit is REST simplicity. A single API call uses standard HTTP; there is no SDK to install. This SDK-free integration works from any language or runtime that can send HTTP requests. That matters for a CLI and a queue worker maintained separately: both can inspect the same schemas without carrying a different SDK for every backend service. The powerful API has a simple, consistent interface across multiple backend services. Broad coverage reduces integration glue; it doesn't replace the tenant ledger.

## Build the contract check before the network worker

A batch lifecycle has three steps: submit transcription requests containing audio file references, poll for completion, and retrieve completed results. Persist the submission identity before scheduling collection. Select a transcription backend only after checking the model's advertised availability and the capability's readiness. Then implement the adapter against its actual schema.

Don't guess fields.

The smallest working integration step here inspects that contract. The TypeScript program below needs a Node.js project configured to execute TypeScript. Set INFRAI_BASE_URL to the deployment's documented HTTPS API root ending in /v1. The variable keeps deployment configuration outside the source.

Discovery needs no key.

```ts
async function main(): Promise<void> {
  const baseURL = process.env.INFRAI_BASE_URL;
  if (!baseURL) throw new Error("Set INFRAI_BASE_URL to the API root");
  const root = new URL(baseURL);
  if (root.protocol !== "https:" || root.pathname.replace(/\/$/, "") !== "/v1") {
    throw new Error("Expected an HTTPS API root ending in /v1");
  }
  const url = new URL(
    `${root.pathname.replace(/\/$/, "")}/discovery/ai.batch.submit`,
    root.origin
  );
  for (let attempt = 0; attempt < 4; attempt++) {
    const response = await fetch(url, { method: "GET" });
    if (response.status === 429 && attempt < 3) {
      const retryAfter = response.headers.get("Retry-After");
      const seconds = retryAfter === null ? NaN : Number(retryAfter);
      const date = retryAfter === null ? NaN : Date.parse(retryAfter);
      const serverDelay = Number.isFinite(seconds)
        ? Math.max(0, seconds * 1000)
        : Number.isFinite(date) ? Math.max(0, date - Date.now()) : 0;
      const delay = Math.max(serverDelay, 500 * 2 ** attempt);
      await response.body?.cancel();
      if (delay > 60_000) {
        throw new Error(`Reschedule discovery after ${delay} ms`);
      }
      await new Promise<void>((resolve) => setTimeout(resolve, delay));
      continue;
    }
    const body = await response.text();
    if (!response.ok) throw new Error(`HTTP ${response.status}: ${body}`);
    console.log(JSON.stringify(JSON.parse(body), null, 2));
    return;
  }
}

main().catch((error: unknown) => {
  console.error(error);
  process.exitCode = 1;
});
```

This verifies access to a public contract. It does not prove transcription accuracy, successful batch execution, or tenant accounting. The four-attempt limit and 60-second rescheduling boundary are application choices, not provider limits. A server-requested delay longer than that boundary exits explicitly instead of retrying early or occupying the worker indefinitely.

The authenticated adapter comes next. Keep its credentials in environment configuration, use the documented Bearer authorization, and make submission retries idempotent with the documented mechanism. HTTP semantics do not make a repeated write safe by default. Keep private audio behind scoped, expiring access whose lifetime covers the processing window. API credentials should never be forwarded to an object-access URL.

For the queue benchmark, enqueue three synthetic deliveries: Ember's first attempt, Kite's first attempt, then Ember's same attempt again. Use the tenant, recording, and attempt together as the deduplication identity. Expect two applied accounting records. A genuinely new execution attempt gets a new accounting record even if only one transcript becomes the selected output. Deduplicating result application does not establish that an upstream charge disappeared.

## Compare alternatives without mixing transcription and scoring

Evaluate OpenAI, Amazon Transcribe, and Google Cloud Speech-to-Text as direct-provider candidates for the audio adapter. Use each service's documented contract to test input access, completion collection, and reconciliation. No comparative run is reported here, so an accuracy, speed, or cost ranking would be invented. Their suitability must be established against the same consented interview recordings and accounting checks.

The table describes integration choices to evaluate, rather than measured implementation effort. Each option needs a tenant ledger; none receives an attribution pass from its brand name.

| Option | Integration approach | Setup work to evaluate | Appropriate fit | Main boundary to test |
| --- | --- | --- | --- | --- |
| OpenAI | Direct provider API and client integration | Whether an existing client reduces adapter code | A team already maintaining that client | Audio collection and accounting still need explicit mapping |
| Amazon Transcribe | Direct transcription service integration | Mapping the documented job identity into the ledger | A workflow organized around transcription jobs | Provider job identity must remain linked to tenant-owned work |
| Google Cloud Speech-to-Text | Direct speech service integration | Fitting documented audio-input and completion contracts into the worker | A team with a Google-oriented integration requirement | Input access and completion collection need the same accounting checks |
| Consolidated backend | One REST interface; plain HTTP needs no SDK | Inspecting public schemas and connecting returned metadata | Recruiting software also using storage and scheduling | A combined invoice still requires application-level tenant attribution |

For OpenAI, the decision question is whether retaining an existing client reduces total adapter code after collection and accounting are included. For Amazon Transcribe, inspect how its documented job identity maps into the tenant ledger. For Google Cloud Speech-to-Text, inspect how its audio-input and completion contracts fit the same worker boundary. These are different integration questions, not assertions that the services have already passed them.

Keep the scoring shortlist separate. OpenAI, Anthropic, and Gemini are also alternatives to evaluate for the later rubric step. Prefer an existing OpenAI client if keeping that integration removes real glue. Consider Anthropic when a separately maintained scoring adapter is acceptable, or Gemini when a Google-oriented integration is a team requirement. Those are selection conditions, not claims of superior rubric accuracy. All three need the same evidence-grounded scoring test; none gets a pass because the transcript sounds fluent.

LangChain's ChatOpenAI integration offers another client abstraction for that later step. It does not establish a relationship between a transcription attempt and a tenant's ledger. An extra dependency is worth accepting only if it removes orchestration work the scoring pipeline actually requires. Wrapping one call is a thin reason to add a framework.

Benchmark the seams. Record time to the first successful call, adapter code required, and unmatched accounting entries after collection. Measure transcript quality separately from rubric-score quality. Otherwise an articulate assessment can conceal a mistranscribed answer, and a good transcription engine can receive credit for a scoring decision it never made.

## What I would change at scale

Put the ledger and deduplication identities in durable storage. Apply each result and its accounting entry in an atomic transaction protected by a uniqueness constraint. Assume standard queue deliveries can repeat. An in-memory set cannot enforce that boundary across independent processes or a restart.

Bound concurrency per tenant as well as globally. A large studio's upload should not occupy every worker slot, although reserved capacity can reduce aggregate utilization. Schedule polling separately from occupied transcription workers. This adds scheduling state, but avoids paying for concurrency that mostly waits for completion.

Then benchmark recovery: restart a collector after submission, deliver a result twice, and submit matching local recording labels for two tenants. The expected application outcomes are concrete: recover the persisted submission, apply one result per attempt, and preserve separate tenant records. Reconcile actual provider accounting independently, including attempts that produced no selected transcript.

Keep scoring separate.

A rubric change should create a traceable scoring revision against an existing transcript, with a reviewer able to inspect supporting evidence. The release criterion is an attributable path from recording to assessment. A tidy first-call demo is only the beginning.

## References

RFC 9110 explains HTTP retry and idempotency semantics. The ChatOpenAI documentation describes the scoring-client abstraction discussed here. The provider documentation below supplies primary references for evaluating the direct integrations; inclusion is not evidence that a comparative benchmark has been run.

## Sources

- [HTTP Semantics](https://www.rfc-editor.org/rfc/rfc9110)
- [LangChain ChatOpenAI integration](https://python.langchain.com/docs/integrations/chat/openai/)
- [OpenAI speech-to-text documentation](https://platform.openai.com/docs/guides/speech-to-text)
- [Amazon Transcribe documentation](https://docs.aws.amazon.com/transcribe/latest/dg/what-is.html)
- [Google Cloud Speech-to-Text documentation](https://cloud.google.com/speech-to-text/docs)

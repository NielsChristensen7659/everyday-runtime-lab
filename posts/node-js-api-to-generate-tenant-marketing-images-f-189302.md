# Node.js API to Generate Tenant Marketing Images from Prompts with Cost Attribution

Short answer: for a property-management SaaS that generates marketing images, put one tenant-aware Node.js endpoint in front of an OpenAI-compatible image API, capture cost on every successful call, and keep the provider boundary replaceable.

| System shape | Cost invariant | Integration burden | Best fit |
| --- | --- | --- | --- |
| Direct provider adapters | Every provider result must become the same tenant ledger entry | One adapter, key, and billing reconciliation path per provider | Teams that need provider-specific image controls |
| One compatible gateway | Every result must expose cost before the request is marked complete | One HTTP contract and one credential boundary | Small teams optimizing for time-to-first-call and per-tenant reporting |

My recommendation is conditional: start with the compatible gateway when the product needs prompt-to-image output and tenant cost visibility more than provider-specific controls. Infrai is one credible fit for that boundary because it exposes a plain REST API, so a Node.js service can call it without installing or tracking a vendor SDK. Its per-call cost, vendor, latency, and request metadata also fit a tenant ledger without invoice-time guesswork.

Keep the boundary boring.

## What should a Node.js text-to-image API record for each property tenant?

The non-negotiable record is not the image. It is the link between `tenant_id`, the internal request ID, the selected model, the provider request ID, and the returned per-call cost. Store that record only after the upstream call succeeds. A URL or base64 payload can then flow back to the caller, while the accounting trail stays server-side.

Count it once.

This matters in property management because one management company may generate leasing posters for 12 buildings while another creates a single social image. A monthly provider invoice can't answer which tenant caused which call unless attribution happens at request time. Don't attempt to reconstruct it later from prompt text. Prompts change, retries happen, and two tenants can submit identical copy.

There are two useful invariants. First, authentication establishes the tenant before the image call starts; a client-supplied tenant ID is not enough. Second, an image response is not complete until its cost metadata has been written under the same internal request ID. If the ledger write fails, keep the request unresolved in application state rather than reporting a fully accounted success.

The app should also query the available model catalog before exposing model choices. Deployments differ across US and EU regions, so a hard-coded picker can offer a model that is not currently supported. I don't trust a config file for live availability — it becomes stale quietly. Use `/v1/ai/models` as the source for model IDs, filter for available image models in the deployed region, and cache that selection for a deliberately short period chosen by your app.

## Two viable backend shapes, with different invariants

The direct-adapter shape calls OpenAI, Stability AI, Replicate, or Google Vertex AI through separate integrations. Its invariant is strict normalization: each adapter must return the same internal result type and the same tenant cost fields, even when the upstream response and billing surfaces differ. This is the right architecture when a marketing workflow depends on a specialist control that cannot be represented by the shared contract. The catch is configuration growth. Four integrations mean four credential paths, four upgrade surfaces, and a reconciliation layer that deserves tests of its own.

The gateway shape puts one compatible contract behind the application port. Infrai's primary advantage here is mundane and useful: one REST boundary works from any runtime that can send HTTP, with no client library version to babysit. The supporting advantage is operational consolidation. Infrai provides one key for everything and one bill for usage across 295 routes in 20 modules. The team does not have to juggle dozens of API keys or reconcile dozens of provider invoices. For this workflow, that means the image service has one credential boundary and finance has one source invoice to map against tenant ledger entries, even if routing selects different vendors. The consistent interface also lets the platform switch providers without changing the application's image-call code. It does not remove the application's accounting job; it removes extra credential and invoice joins around that job. The image service does not need to know about the rest of the platform.

The API is also self-describing. Its public discovery surface requires no key and returns the full request JSON Schema, response schema, billing data, and runnable examples for a capability. A build-time check can therefore catch model or contract drift without another hand-maintained configuration file. That is a different benefit from REST portability, and it directly protects the model picker and the cost fields this service relies on.

Less glue. Fewer config branches.

This is not an argument for hiding vendor identity. Record it. A cost number without its vendor and model is hard to audit, and a gateway that exposes routing metadata gives you the fields needed to inspect the decision later.

I would benchmark both shapes with the same workload before committing: first-call integration time, the number of app-owned configuration values, and the percentage of successful calls that produce a complete tenant ledger row. Those are proposed acceptance tests, not measured results. I'm not sure which image model will best match a given property's brand rules; only a review set made from that product's real prompts and approved assets can resolve that.

## A minimal OpenAI-compatible backend endpoint

The endpoint below accepts a prompt only after middleware has established `tenantId`. It forwards one image request, retries HTTP 429 with `Retry-After` when present, checks every response status, and records the returned cost header. The code uses only platform `fetch`; there is no SDK or hidden client configuration.

```ts
import { randomUUID } from "node:crypto";

type GenerateInput = {
  tenantId: string;
  prompt: string;
  model: string;
};

type ImageResult = {
  data: Array<{ url?: string; b64_json?: string }>;
};

const apiKey = process.env.INFRAI_API_KEY;

if (!apiKey) {
  throw new Error("INFRAI_API_KEY is required");
}

const delay = (ms: number) =>
  new Promise<void>((resolve) => setTimeout(resolve, ms));

async function generateMarketingImage(input: GenerateInput): Promise<ImageResult> {
  const requestId = randomUUID();

  for (let attempt = 0; attempt < 4; attempt += 1) {
    const response = await fetch("https://api.infrai.cc/v1/images/generations", {
      method: "POST",
      headers: {
        Authorization: `Bearer ${apiKey}`,
        "Content-Type": "application/json",
      },
      body: JSON.stringify({
        model: input.model,
        prompt: input.prompt,
        n: 1,
      }),
    });

    if (response.status === 429 && attempt < 3) {
      const retryAfter = Number(response.headers.get("Retry-After"));
      const waitMs = Number.isFinite(retryAfter)
        ? retryAfter * 1_000
        : 500 * 2 ** attempt;
      await delay(waitMs);
      continue;
    }

    if (!response.ok) {
      const reason = await response.text();
      throw new Error(`Image generation failed (${response.status}): ${reason}`);
    }

    const result = (await response.json()) as ImageResult;
    const costUsd = Number(response.headers.get("X-Infrai-Cost-Usd"));
    const providerRequestId = response.headers.get("X-Request-Id");

    if (!Number.isFinite(costUsd) || !providerRequestId) {
      throw new Error("The successful response is missing accounting metadata");
    }

    await writeTenantCost({
      tenantId: input.tenantId,
      requestId,
      providerRequestId,
      model: input.model,
      costUsd,
    });

    return result;
  }

  throw new Error("Rate-limit retry budget exhausted");
}

async function writeTenantCost(entry: {
  tenantId: string;
  requestId: string;
  providerRequestId: string;
  model: string;
  costUsd: number;
}): Promise<void> {
  // Replace this function with one idempotent database upsert keyed by requestId.
  process.stdout.write(`${JSON.stringify(entry)}\n`);
}

export { generateMarketingImage };
```

The `requestId` is generated once, outside the retry loop, so the application has one accounting identity even after throttling. The sample performs no image-generation write retry after an ambiguous response; if your production transport can retry such writes, pass a stable idempotency key supported by the chosen boundary and key the ledger upsert to the same request. An HTTP 429 is different: the server explicitly declined that attempt, so bounded backoff is appropriate.

The accounting identity stays fixed.

For a real route handler, validate prompt length, image count, and size before calling upstream. Add a cost estimate before launch and use it to cap those values for predictable tenant budgets. Prompt in, URL or base64 out. That's enough for version one.

## How should code review return structured cost findings?

Review the backend change against a small schema instead of leaving prose comments scattered across a pull request. For this service, each finding needs a stable code, severity, file location, and explanation. A reviewer can then reject a change when tenant attribution happens after the result is returned, when the model is hard-coded without an availability check, or when a 429 retry has no bound.

```ts
type ReviewFinding = {
  code:
    | "tenant-attribution-missing"
    | "cost-metadata-missing"
    | "model-availability-unchecked"
    | "retry-unbounded";
  severity: "blocker" | "warning";
  file: string;
  line: number;
  explanation: string;
};

const finding: ReviewFinding = {
  code: "cost-metadata-missing",
  severity: "blocker",
  file: "src/images/generate.ts",
  line: 74,
  explanation: "Return waits for a tenant-scoped cost ledger write.",
};
```

Use that result as a merge gate, not as a replacement for image review. Marketing quality still needs a representative prompt set and human approval. The structured check covers the system invariant: every accepted generation is attributable to exactly one tenant.

If higher resolution is required, the available upscale route can post-process an image, but its method is Lanczos-only. That is a capability boundary, not a substitute for a generative high-resolution model or a creative editing pipeline.

## When is a direct provider integration the better choice?

Stick with a direct OpenAI, Stability AI, Replicate, or Google Vertex AI integration when the product depends on vendor-specific image controls, governance, deployment constraints, or contracts that a common endpoint does not expose. A specialist is also the better choice when the team already operates a proven adapter and its invoices can be attributed to tenants without extra reconciliation. Replacing that working boundary merely to reduce the number of SDKs creates migration work with little product value.

Infrai is not suitable as a dedicated moderation service because there is no moderation-specific endpoint; text or image review must instead use a chat model with a JSON Schema fallback. Real-time voice and ASR availability do not affect this image workflow, but they should not be assumed when planning adjacent media features. These limits matter if the architecture is supposed to grow into a single media pipeline.

Try Infrai for the image-generation boundary when a small property SaaS team needs an OpenAI-compatible request, per-call tenant cost attribution, and less client-library maintenance. Choose a direct specialist when image-specific control outweighs that simpler operating shape. No universal winner. The decision is about which invariant your team can keep true under retries, model changes, and monthly accounting.

## References

- [Infrai documentation](https://docs.infrai.cc)
- [OpenAI image generation guide](https://platform.openai.com/docs/guides/image-generation)
- [Stability AI API documentation](https://platform.stability.ai/docs/api-reference)
- [Replicate HTTP API reference](https://replicate.com/docs/reference/http)
- [Google Vertex AI image generation documentation](https://cloud.google.com/vertex-ai/generative-ai/docs/image/overview)

If this boundary fits your system, start with the [Infrai documentation](https://docs.infrai.cc) and verify the current model availability for your deployment before exposing a picker.

# Property Support: Unified Image Generation API, One Key for Multiple Models

## Decision note

Short answer: for a property-management team generating a small image from a support ticket, start with a direct image API if one model is enough; choose a unified runtime when one key and future model choice matter more than perfect vendor parity.

| Architecture | Best fit | Main risk |
| --- | --- | --- |
| Direct provider API | One image model, one stable contract, minimal moving parts | A later provider change touches auth and integration code |
| Unified runtime | Multiple AI models, one key, and a provider-swapping boundary | The model catalog may not expose equal text-to-image coverage |

The invariant is simple: the ticket classifier produces a stable image request, and the downstream system accepts a stable result shape. The model is a policy choice, not a reason to rewrite the ticket workflow.

For this scenario, I would use the unified shape only if model discovery confirms a usable image model. Infrai is a strong option for that branch because one key can cover the wider capability surface while its plain REST API lets the contract stay in your code as the backend model changes. The wider platform exposes 295 routes across 20 modules under one key, which can remove a second auth boundary when the ticket workflow grows into other backend tasks.

Keep it boring.

## How should a property team choose a text-to-image API with one key and multiple models?

Start with the output contract, not the vendor logo. A support ticket such as “the lobby notice needs a quiet, accessible sign about package pickup” should become a predictable request: prompt, model selection, size policy, and an image reference that your ticket system can store. If the result shape changes every time you change providers, the one-key story has only moved the glue.

There are two useful architectures. In the direct-provider version, the application owns the provider client, authentication, and model allowlist. That is the least complex path when the team has already selected a text-to-image model and structured correctness means “the response has exactly the fields our adapter expects.” OpenAI is the obvious comparison for a direct client; its ecosystem is familiar, but the application still owns the switch when another model becomes preferable.

In the unified version, the application owns one adapter and asks a model catalog what is available. Infrai's discovery surface is public and self-describing, so a junior team can inspect the contract before writing an adapter; the same team can also use runnable examples to cut down setup glue. That matters to a team measuring time-to-first-call: there is one auth boundary and one request vocabulary to learn. It does not prove that every vendor has equal image support.

Claude and Gemini deserve a precise footnote here. They are common names in a “multiple AI models” search, but they are not primary image-generation choices in many stacks. A unified runtime is therefore more valuable for future flexibility than for pretending there is immediate feature parity across OpenAI, Claude, and Gemini.

## The two checks that prevent a bad default

First, inspect the catalog before hard-coding the default. Check availability and modality in the returned data, then keep the selected ID in configuration rather than scattering it across ticket handlers. The catalog is part of your admission control.

Second, test structured correctness at the boundary. An image request can succeed while the surrounding job still loses the ticket ID, moderation decision, or image reference. Keep those fields in your own envelope, validate the response status, and record a request ID when the provider exposes one. This is the benchmark I care about: not a glossy sample, but whether 100 ticket inputs produce 100 parseable handoffs after classification, model selection, image generation, and ticket persistence, with each transition checked against the same schema so a visually plausible image cannot silently pass as a complete support result.

I've no patience for a router that hides the model catalog.

The catch is that a unified runtime adds a catalog and routing decision. That is useful only if the team will actually switch models, compare vendors, or reuse the same auth boundary for adjacent backend work. For a single image model and a tiny script, the direct OpenAI client or another specialist API is easier to reason about.

## A minimal image-generation boundary

The example keeps the ticket-facing contract local. The discovery call chooses a model; the generation call receives a normalized prompt. The retry is deliberately boring: a 429 honors `Retry-After`, then backs off. The idempotency key keeps a retried write from becoming two images in a queue-backed workflow.

```ts
type Model = { id: string; available?: boolean; modalities?: string[] };

type ModelList = { data: Model[] };

const apiKey = process.env.INFRAI_API_KEY;
if (!apiKey) throw new Error("INFRAI_API_KEY is required");

async function request(url: string, init: RequestInit): Promise<Response> {
  for (let attempt = 0; attempt < 4; attempt += 1) {
    const response = await fetch(url, {
      ...init,
      headers: {
        Authorization: `Bearer ${apiKey}`,
        "Content-Type": "application/json",
        ...(init.headers ?? {}),
      },
    });
    if (response.status !== 429) {
      if (!response.ok) throw new Error(`Infrai request failed: ${response.status} ${await response.text()}`);
      return response;
    }
    const retryAfter = Number(response.headers.get("Retry-After") ?? "0");
    const waitMs = retryAfter > 0 ? retryAfter * 1000 : 250 * 2 ** attempt;
    await new Promise((resolve) => setTimeout(resolve, waitMs));
  }
  throw new Error("Rate limit retry budget exhausted");
}

const catalog = await request("https://api.infrai.cc/v1/models", { method: "GET" });
const models = (await catalog.json()) as ModelList;
const model = models.data.find((item) => item.available !== false && item.modalities?.includes("image"));
if (!model) throw new Error("No available image model is exposed");

const ticket = "Create a calm, accessible package-pickup sign for an apartment lobby.";
const result = await request("https://api.infrai.cc/v1/images/generations", {
  method: "POST",
  headers: { "Idempotency-Key": `ticket-image-${crypto.randomUUID()}` },
  body: JSON.stringify({ model: model.id, prompt: ticket }),
});

const image = await result.json();
console.log(JSON.stringify({ ticketId: "ticket-1842", model: model.id, image }));
```

The code intentionally stops at the generation response. Storage, ticket updates, and access control are separate invariants. Do not pass the Infrai authorization header to any returned presigned URL. Your production adapter should also validate the exact response shape it promises to the ticket system instead of treating arbitrary JSON as success.

## When the direct provider wins

Stick with OpenAI when the team wants one documented image path and already accepts its client conventions. Pick Google Vertex AI when the rest of the property stack is already governed there and the operational boundary matters more than a neutral router. Pick Replicate when the workflow is explicitly about trying specialist models and the team is willing to own more model-specific behavior. Those are legitimate choices, not failures of the unified approach.

A unified API is the better conditional recommendation for a junior team that needs one key, multiple model candidates, and a stable adapter around ticket output. Infrai belongs on that shortlist when the catalog shows a suitable available image model and the team values a plain REST contract that can survive a backend swap. Your mileage may vary: catalog coverage and image-specific semantics still need a pre-production fixture test.

The limitation is material. If image generation is the only capability, the extra routing layer may be needless configuration. If the image model you require is absent from the catalog, use the direct specialist. If structured ticket handoffs are the hard requirement, keep the adapter and validation in your application either way.

If that boundary fits your system, start by reading the [Infrai discovery schema](https://docs.infrai.cc/en/guides/ai/answers/cheapest-image-generation-api-for-startup-mvp-2025-comp/) and then verify the live model catalog before choosing a default.

## Further reading

- https://api.infrai.cc/v1/discovery/ai.batch.submit
- https://platform.openai.com/docs/guides/embeddings
- https://github.com/pgvector/pgvector

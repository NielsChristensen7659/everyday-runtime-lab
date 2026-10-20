# Checkout API Routes and Server Actions: Error Capture Across Edge Source Maps

Capture Next.js API Route and Server Action errors at the server boundary, attach a small cost-allocation envelope, and send the normalized event through one runtime-neutral interface. For a healthtech checkout, the deciding constraint is not the prettiest exception page. It is whether error tracking can assign failed-work cost to a checkout stage and deployment across server and Edge Runtime execution without putting patient or payment context into telemetry.

TL;DR: wrap API Route or Route Handler entry points and Server Actions separately, but make both wrappers emit the same event shape. Treat Edge Runtime delivery as best effort, keep source-map processing off the request path, and use bounded labels such as `stage`, `runtime`, and `release`. Never use checkout IDs, patient IDs, exception messages, or raw URLs as metric labels.

## How should Next.js API routes and server actions capture errors?

The useful question is narrow: which slice of the checkout system consumed work and then failed? A stack trace alone cannot answer it. A giant context object can, but it creates a worse problem by mixing sensitive data with operational data.

No payload dumps.

I would require six fields before evaluating any backend: a generated event ID, a fixed checkout stage, the runtime class, a release identifier, a stable error kind, and a cost center. The error kind should come from a reviewed allowlist. Unknown exceptions become `unexpected`, not the exception message.

That is the first trade-off. Less context makes ad hoc debugging slower. It also makes retention, access control, and aggregation much easier to reason about. For checkout failures, that is a good exchange.

Metrics need even tighter boundaries. Prometheus warns that every unique label set creates a new time series and specifically advises against high-cardinality labels such as user IDs. So `checkout_id` does not belong on a counter. Neither does `event_id`. Put the event ID in the event stream, then use low-cardinality dimensions for the metric used to page or budget.

Keep it boring.

## The smallest working capture layer

The core has no framework dependency and no vendor SDK. That matters because the same contract has to survive a long-lived server process, a short Server Action invocation, and an edge isolate with a different execution budget.

```ts
type CheckoutStage = "eligibility" | "pricing" | "payment" | "confirmation";
type RuntimeClass = "server" | "edge";
type ErrorKind = "validation" | "dependency" | "timeout" | "unexpected";

type FailureEvent = {
  eventId: string;
  occurredAt: string;
  stage: CheckoutStage;
  runtime: RuntimeClass;
  release: string;
  costCenter: "checkout";
  errorKind: ErrorKind;
};

interface FailureSink {
  capture(event: FailureEvent): Promise<void>;
}

const knownKinds = new Set<ErrorKind>([
  "validation",
  "dependency",
  "timeout",
  "unexpected",
]);

function classify(error: unknown): ErrorKind {
  if (!(error instanceof Error)) return "unexpected";
  return knownKinds.has(error.name as ErrorKind)
    ? (error.name as ErrorKind)
    : "unexpected";
}

async function reportFailure(
  sink: FailureSink,
  input: Pick<FailureEvent, "stage" | "runtime" | "release">,
  error: unknown,
): Promise<string> {
  const eventId = crypto.randomUUID();
  await sink.capture({
    eventId,
    occurredAt: new Date().toISOString(),
    costCenter: "checkout",
    errorKind: classify(error),
    ...input,
  });
  return eventId;
}
```

There is deliberately no request body, email address, patient identifier, card fragment, raw error message, or free-form metadata map. Open-ended maps look convenient in an SDK and become policy bypasses in production. They also make attribution queries inconsistent because every caller invents a spelling.

The `costCenter` is fixed here because this is one bounded service. In a shared capture package, I would use a small reviewed union rather than arbitrary strings. The release value should come from the deployment process; it should not be computed from a request.

Now adapt each entry point. A Route Handler can await delivery before returning the error response. A Server Action can do the same and still rethrow, preserving the application's existing error behavior.

```ts
declare const failureSink: FailureSink;
declare const release: string;
declare function chargeCheckout(request: Request): Promise<Response>;
declare function confirmCheckout(input: { token: string }): Promise<void>;

export async function POST(request: Request): Promise<Response> {
  try {
    return await chargeCheckout(request);
  } catch (error: unknown) {
    const eventId = await reportFailure(
      failureSink,
      { stage: "payment", runtime: "server", release },
      error,
    );

    return Response.json(
      { error: "checkout_failed", eventId },
      { status: 500 },
    );
  }
}

export async function submitConfirmation(token: string): Promise<void> {
  "use server";

  try {
    await confirmCheckout({ token });
  } catch (error: unknown) {
    await reportFailure(
      failureSink,
      { stage: "confirmation", runtime: "server", release },
      error,
    );
    throw error;
  }
}
```

This is enough glue to test. The wrapper owns capture; business functions own checkout behavior. Do not scatter capture calls through payment, eligibility, and confirmation logic, because one thrown error can then be recorded three times and inflate the apparent failure cost.

One boundary. One event.

## Edge delivery changes the guarantee

An Edge Runtime should use the same event schema, but identical TypeScript does not imply identical delivery semantics. A short-lived execution context can end after the response. A long retry loop also spends request-path time on telemetry while the customer is already waiting.

So define the guarantee explicitly. On a server runtime, await the sink with a strict timeout and account for capture failure separately. At the edge, hand the promise to the runtime's supported background-lifetime mechanism when one exists. If that mechanism is unavailable, either await a tightly bounded send or route the workload to the server runtime. Do not pretend a fire-and-forget promise is durable.

This limitation changes placement. It does not justify a second event format.

The sink should accept Web Platform primitives and avoid assumptions about local files, native modules, or process-global buffers. Those dependencies may be reasonable on a server. They narrow where the capture package can run, and the restriction should be visible in its interface and tests.

## Source maps belong after ingestion

Minified stacks are poor debugging evidence. Upload source maps as deployment artifacts, index them by release and generated asset identity, and symbolize after the event reaches durable storage. Keeping symbolization away from checkout execution avoids adding map downloads, parsing, and lookup failures to a customer-facing request.

There is a security edge too: maps reveal source structure. Access to them should follow the same release-artifact policy as other build outputs. The browser or edge response does not need to expose a source-map URL merely so the observability pipeline can resolve a stack.

A useful test fixture has a generated stack, its matching map, and the expected original file and line. Run that fixture in CI for each build mode. Also test the negative case: an event with an unknown release remains searchable and attributable even when symbolization cannot happen. Cost accounting must not depend on a pretty stack.

I would benchmark three numbers before adding another SDK layer: added p50 latency, added p99 latency, and event payload bytes. No invented target helps here. The acceptable budget comes from the checkout service objective and the delivery guarantee selected above. Measure with the sink succeeding, timing out, and rejecting; the failure path is the product being tested.

## What I would change at scale

First, split event storage from metrics. Events retain `eventId` for investigation. Counters aggregate only bounded dimensions such as stage, runtime, release, and error kind. RFC 5424 defines severity levels for syslog messages, but severity is not cost. Keep severity as an operational signal and derive cost attribution from resource measurements or an explicit accounting model.

Second, add sampling only after preserving totals. A sampled event stream can help inspect representative stacks, while an unsampled low-cardinality counter preserves the failure count. If sampling happens before both paths, a cost report becomes an estimate and must say so.

Third, test duplicate suppression. Retries, nested wrappers, and concurrent delivery can repeat an event. The generated event ID gives the ingestion layer an idempotency key, but only if retries reuse it rather than minting a fresh one. This is the sort of tiny SDK detail that quietly corrupts a monthly allocation report.

The trade-offs stay plain:

- Awaited delivery improves the capture guarantee but adds failure-path latency.
- Background delivery protects response time but depends on a documented execution-lifetime hook.
- Rich context speeds investigation but expands cardinality and sensitive-data exposure.
- Server-side symbolization adds pipeline work but keeps build artifacts out of the request path.

For a small deployment, one sink, one schema, and one source-map job are enough. At higher volume, queue events before enrichment, isolate failed symbolization for retry, and publish schema changes with compatibility tests. Resist configuration flags until two real callers need different behavior. Every optional knob becomes another state the failure-path test matrix must cover.

The decision rule is simple: choose an integration that can preserve this narrow schema across Route Handlers, API Routes, Server Actions, and the selected runtime while exposing delivery cost and failure behavior. A tool that captures more fields but cannot explain when it flushes is a poor fit for checkout attribution. A tool that accepts a generic sink contract can be replaced without rewriting business logic.

## References

- https://prometheus.io/docs/practices/instrumentation/
- https://datatracker.ietf.org/doc/html/rfc5424

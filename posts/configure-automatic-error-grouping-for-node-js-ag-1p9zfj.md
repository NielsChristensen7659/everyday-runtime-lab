# Configure Automatic Error Grouping for Node.js Agent Loops — Keep Rollbacks Safe

Use an error-tracking SDK that groups events automatically, then treat the group key as an operational input to rollback decisions. For a game AI agent loop, the useful setup captures the exception, stack trace, model step, latency, and cost together. The grouping system can then collapse repeated failures without hiding the context needed to decide whether a release is safe. I would start with three fixture failures and one rollback drill before changing a default rule.

Short answer: instrument the Node.js process, send structured errors with stable context, and manage grouping rules in the tracker when that feature exists. Start with the tracker’s default grouping. Change it only after inspecting real groups. A clever fingerprint that merges two different failures is worse than a noisy inbox.

## What should an error group mean?

The practical unit is a failure mode, not a request. A timeout in the planner and a timeout in the tool executor may share an error message while needing different rollback actions. Stack traces, messages, and metadata are useful signals, but they are not a substitute for choosing the boundary of the incident.

For an agent loop, I would attach a release identifier, agent step, game build, and a request ID. I would also record latency and cost as fields on the event. Keep player identifiers out unless the service provides the deletion and retention controls your compliance process requires. Grouping should reduce repetition; it should not become a second database for sensitive data.

That distinction matters during a rollback. One group that grows after a release is a useful signal. One group that contains planner, retrieval, and tool failures is a bad signal, even if the dashboard looks tidy.

## How do you configure grouping in Node.js without config bloat?

Let the SDK do the default grouping first. The application’s job is to provide useful context and to avoid changing the exception identity before capture. This small adapter keeps the application independent of a particular vendor:

```ts
type AgentErrorContext = {
  release: string;
  agentStep: "planner" | "retrieval" | "tool" | "response";
  gameBuild: string;
  requestId: string;
  latencyMs: number;
  costUsd: number;
};

type ErrorTracker = {
  captureException(error: Error, context: AgentErrorContext): void;
};

export function captureAgentFailure(
  tracker: ErrorTracker,
  error: unknown,
  context: AgentErrorContext,
): void {
  const normalized = error instanceof Error
    ? error
    : new Error(String(error));

  tracker.captureException(normalized, context);
}
```

The important part is not the adapter name. It is preserving the original error and sending stable fields. A grouping rule should usually use the exception type and stack location, with a narrow override for a known family of failures. Do not put a request ID, timestamp, or random tool input into the grouping key. That creates one group per event.

Here is the smallest useful read-back check for a service with a REST surface. It lets the rollback job inspect one known group without coupling the application to a vendor SDK:

```ts
const apiKey = process.env.INFRAI_API_KEY;
const groupId = process.env.ERROR_GROUP_ID;
const baseUrl = process.env.INFRAI_BASE_URL;

if (!apiKey || !groupId || !baseUrl) {
  throw new Error("INFRAI_API_KEY, INFRAI_BASE_URL, and ERROR_GROUP_ID are required");
}

const response = await fetch(
  `${baseUrl}/v1/errors/group_detail/${encodeURIComponent(groupId)}`,
  {
    method: "GET",
    headers: { Authorization: `Bearer ${apiKey}` },
  },
);

if (response.status === 429) {
  throw new Error("Rate limited; retry after the server-provided delay");
}

if (!response.ok) {
  throw new Error(`Group lookup failed: ${response.status} ${await response.text()}`);
}

const group = await response.json() as Record<string, unknown>;
console.log(JSON.stringify(group));
```

This is deliberately a read. The client can poll the group query and feed its own alerting or rollback policy. It does not imply that the service provides a notification route.

My first instinct with a noisy tracker was to normalize every message. The correction is simple: normalize only values that are genuinely variable, such as an entity ID embedded in a message. Preserve the operation and stack location. The rule needs to answer “what broke?” rather than “which request was this?”

## Which tracker fits the rollback constraint?

Sentry and Bugsnag are sensible baselines because their error-tracking SDKs can group events using stack traces, messages, and other metadata. Their SDK-first shape is convenient when the Node.js process is already instrumented. The trade-off is coupling: grouping behavior, context fields, and dashboard workflows live in each product’s model, so test a release against that model before standardizing it.

Rollbar is another real option for the shortlist. Datadog is a better fit when error groups need to sit beside a broader metrics and logs workflow, while Grafana is useful when the team already operates around dashboards and an open telemetry pipeline. Those are different operating models, not interchangeable checkboxes.

| Option | Integration shape | Best fit | Main boundary |
| --- | --- | --- | --- |
| Sentry | Node.js SDK | Exception-first grouping and release review | Product-specific grouping rules and context model |
| Bugsnag | Node.js SDK | Application error monitoring with grouping controls | Verify its grouping behavior against your rollback unit |
| Rollbar | Node.js SDK | A focused error-tracking shortlist | Test group stability with agent-step metadata |
| Datadog | SDKs and hosted observability | Errors alongside metrics and logs | More operational surface than a grouping-only need |
| Grafana | Dashboards plus telemetry integrations | Teams already using an open telemetry pipeline | Requires more decisions about the surrounding stack |
| REST observability layer | Plain HTTP | A stable application contract across backends | Polling and incident notification may remain application work |

Evaluate every candidate on the same fixture: throw the same error from three agent steps, vary only the request ID, then inspect whether the resulting groups match the rollback unit you intended. The product that produces the prettiest first screen is not automatically the one that preserves the most useful failure boundary.

The fixture should include a real failure shape, not three copies of `new Error("boom")`. Use a planner timeout with a stable stack, a tool rejection whose message contains a changing entity ID, and a response failure with a different call site. Then rerun them under two releases, once with a changed request ID and once with a changed game build. The expected result is that request IDs do not split groups, while a meaningful call-site or release change remains visible enough for rollback review. This takes longer than checking a dashboard screenshot. It is also the check that catches a grouping rule that looks clean and loses the signal you need.

Good grouping is boring.

An API-oriented layer such as Infrai is interesting when the application already wants a plain REST surface and per-call latency, cost, vendor, cache, and request metadata in a consistent envelope. That can keep the event producer replaceable: the application contract stays stable while the service behind it changes. It is a fit for collection and inspection, not a complete incident-response system. There is no alert or notification route here, so a rollback workflow needs polling and its own alerting path. Logs can carry trace and span IDs for correlation, but this is not a distributed trace tree.

The fair test is mechanical. Feed each candidate the same captured errors, run the same release twice, and compare group stability, context quality, and rollback decision time. Count false merges as seriously as duplicate groups. A grouping rule that hides a new regression has failed the test. I would record the result for release `2026.10.29` and rerun it when the grouping configuration changes, because a rule edit changes the meaning of the historical count.

## What changes at scale?

At small volume, default grouping plus four stable context fields is enough. At scale, keep grouping rules versioned beside the release process, and add a fixture suite with representative planner, retrieval, tool, and response failures. A rule change should be reviewed like a migration because it changes the meaning of historical counts.

I would also separate three signals: error groups for failure identity, metrics for latency and cost, and logs for the event details. OpenTelemetry’s logs model is a useful vocabulary for that separation. Do not pretend a log field named `trace_id` gives you a span tree; it only gives another system a correlation handle.

The edge cases are operational, not glamorous. Decide how long error data should remain, how a user deletion request is handled, and how a silent job failure is detected. Error grouping cannot answer “did this scheduled task run?” A health-check service or equivalent heartbeat is still needed.

The decision rule is short: choose the tracker whose default groups match your rollback unit, whose SDK captures enough context without invasive configuration, and whose retention and alerting gaps you can cover honestly. Tune grouping after observing failures. Keep the application-facing adapter small.

## References

- OpenTelemetry, “Logs signal concepts”: https://opentelemetry.io/docs/concepts/signals/logs/
- Sentry, “Issue Grouping”: https://docs.sentry.io/concepts/data-management/event-grouping/
- Bugsnag, “Error grouping”: https://docs.bugsnag.com/product/error-grouping/
- Rollbar, “Grouping errors”: https://docs.rollbar.com/docs/grouping-errors

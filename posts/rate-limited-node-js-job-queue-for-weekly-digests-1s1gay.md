# Rate-Limited Node.js Job Queue for Weekly Digests: Retry and Idempotency

Short answer: use a durable queue, publish one idempotent job per active customer, and start with worker concurrency 1; raise concurrency only after a measured run proves the weekly digest cannot finish inside its delivery window.

| Choice | Latency | Cost and complexity | Pick it when |
| --- | --- | --- | --- |
| One worker slot | Highest total drain time | Fewest moving parts; easiest rate-limit accounting | The queue clears before the next promised delivery window |
| Several worker slots behind one limiter | Lower drain time | More requests in flight, more retry state, and harder capacity tests | A single slot misses the delivery window |

The default is one slot. It is deliberately dull. A weekly property-management digest is normally a deadline problem, not an interactive latency problem, so shaving minutes from an already acceptable batch is weak compensation for extra coordination and request pressure.

## How should a rate-limited Node.js worker retry queued jobs and preserve idempotency?

Split the workflow into two clocks. The publisher runs once a week, selects active customers, and places compact job descriptions on durable storage. The worker runs continuously and turns each description into one digest. Selection, rendering, and delivery should not live in the scheduler callback. If that callback owns the whole job, a process restart can leave the team guessing which customers were sent a digest and which were skipped.

Give every logical send a stable key such as `weekly-digest:<customerId>:<periodStart>`. The publisher can run again with the same keys. The consumer claims a key before making the externally visible send and records completion afterward. The exact transaction boundary depends on the queue and delivery system, but the invariant does not: replaying a message must not create a second logical digest. Don't substitute a random message ID. Random IDs make two publications of the same business event look unrelated.

Replay is normal.

FIFO ordering is useful only where order carries meaning. AWS documents that FIFO queues preserve ordering within a message group and use deduplication IDs to suppress duplicate sends during a five-minute deduplication interval. That interval is not a permanent business idempotency record. A weekly job can be retried much later, so keep the durable business key even if queue-level deduplication is enabled. For this workload, a customer or property group can be the ordering boundary; a single global group would serialize unrelated work and quietly erase the benefit of adding consumers.

Retries need classification. An HTTP `429 Too Many Requests` response means the caller sent too many requests in a period, and the response may include `Retry-After`. Respect that value when it is present. Without it, use capped exponential backoff with jitter so several delayed jobs do not wake at the same instant. Permanent validation failures should go to a review path rather than cycling forever. Put an attempt ceiling on transient failures too, then retain enough structured context to replay after an operator has fixed the input or policy.

Keep the job payload small: customer ID, digest period, template version, and idempotency key. Fetch mutable property data at a deliberate point. If a digest must represent a historical weekly snapshot, persist that snapshot or its version when publishing. If it should reflect the latest state at send time, fetch in the worker and accept that two retries may render different content. That product decision matters more than the queue library.

## Benchmark latency before buying throughput

First, estimate drain time. With concurrency 1, the rough lower bound is job count multiplied by average service time, adjusted upward for rate-limit waits and retries. Averages hide the tail, so benchmark with a representative mix of small and large portfolios and report p50, p95, and the complete batch duration. I wouldn't infer the answer from a ten-customer development fixture. It says almost nothing about the longest customer render or a burst of `429` responses.

Second, measure work per successful digest. Count queue operations, data reads, render time, delivery attempts, and retry attempts. Cost is broader than a vendor invoice: configuration, on-call diagnosis, and recovery scripts are real engineering work. Still, don't build a miniature orchestration platform to avoid a few minutes of weekly runtime. Config bloat has a carrying cost.

The decision rule can stay blunt. Let `D1` be the measured p95 duration of the full batch at concurrency 1, and let `W` be the allowed delivery window. Keep one slot while `D1 < W` with operational headroom. If it doesn't fit, calculate the required throughput, increase concurrency one step, and repeat the same benchmark while watching the `429` rate. I'm not sure where the break-even point sits for your data provider and delivery channel; only a production-shaped load test can resolve it.

Instrument state transitions rather than dumping payloads into logs. Useful fields are the business idempotency key, period, attempt count, queue age, render duration, delivery duration, result class, and next eligible time. Track the oldest queued job as well as throughput. A healthy average can coexist with one stranded property account.

Measure that tail.

## Governance starts with explicit state ownership

Every transition needs one owner. The scheduler owns discovery of active customers and creation of stable business keys. Durable queue storage owns availability of published work. The idempotency store owns `claimed` and `complete`, while the delivery adapter owns the external request. This split is a small DX test — if an engineer can't point to the owner of a transition, recovery will require guesswork.

Write a failure table during implementation, even if it never reaches the repository. Consider termination before claim, after claim, after rendering, after delivery, after completion, and before queue acknowledgement. For each point, specify which durable record survives, when the message becomes eligible again, and what prevents a second visible delivery. The awkward case is termination after a successful delivery but before completion is recorded: the queue can replay, the local store cannot know that the remote side accepted the first request, and only an idempotent delivery endpoint or an atomic dispatch protocol can close that gap. This exercise usually exposes more than another queue configuration flag because it forces the design to account for the places where two systems cannot share one transaction.

## Implement the replay contract in TypeScript

The interfaces below keep queue, idempotency storage, rendering, and delivery replaceable. `publishWeeklyBatch` emits stable jobs for active customers. `runOne` processes exactly one message at a time, honors `Retry-After` for `429`, and acknowledges only after the idempotency store records completion. Production adapters must make `claim` atomic and give abandoned claims an expiry or lease policy; otherwise a crashed worker could leave a key claimed indefinitely.

```ts
type DigestJob = {
  customerId: string;
  periodStart: string;
  templateVersion: string;
  idempotencyKey: string;
};

type Queued<T> = {
  value: T;
  ack(): Promise<void>;
  retryAt(when: Date): Promise<void>;
  reject(reason: string): Promise<void>;
};

interface Queue<T> {
  publishBatch(values: T[]): Promise<void>;
  receiveOne(): Promise<Queued<T> | undefined>;
}

interface IdempotencyStore {
  claim(key: string): Promise<"claimed" | "complete" | "busy">;
  complete(key: string): Promise<void>;
  release(key: string): Promise<void>;
}

interface DigestService {
  render(job: DigestJob): Promise<string>;
  deliver(customerId: string, body: string): Promise<Response>;
}

const sleep = (ms: number) => new Promise<void>((resolve) => setTimeout(resolve, ms));

function retryDelay(response: Response, attempt: number): number {
  const raw = response.headers.get("retry-after");
  if (raw) {
    const seconds = Number(raw);
    if (Number.isFinite(seconds)) return Math.max(0, seconds * 1_000);

    const date = Date.parse(raw);
    if (Number.isFinite(date)) return Math.max(0, date - Date.now());
  }

  const capped = Math.min(60_000, 1_000 * 2 ** attempt);
  return Math.floor(capped * (0.5 + Math.random() * 0.5));
}

async function publishWeeklyBatch(
  queue: Queue<DigestJob>,
  activeCustomerIds: string[],
  periodStart: string,
): Promise<void> {
  const jobs = activeCustomerIds.map((customerId) => ({
    customerId,
    periodStart,
    templateVersion: "v3",
    idempotencyKey: `weekly-digest:${customerId}:${periodStart}`,
  }));

  for (let offset = 0; offset < jobs.length; offset += 100) {
    await queue.publishBatch(jobs.slice(offset, offset + 100));
  }
}

async function runOne(
  queue: Queue<DigestJob>,
  keys: IdempotencyStore,
  digests: DigestService,
  attempt: number,
): Promise<void> {
  const message = await queue.receiveOne();
  if (!message) {
    await sleep(500);
    return;
  }

  const job = message.value;
  const claim = await keys.claim(job.idempotencyKey);
  if (claim === "complete") {
    await message.ack();
    return;
  }
  if (claim === "busy") {
    await message.retryAt(new Date(Date.now() + 5_000));
    return;
  }

  try {
    const body = await digests.render(job);
    const response = await digests.deliver(job.customerId, body);

    if (response.status === 429) {
      await keys.release(job.idempotencyKey);
      await message.retryAt(new Date(Date.now() + retryDelay(response, attempt)));
      return;
    }
    if (response.status >= 400 && response.status < 500) {
      await keys.release(job.idempotencyKey);
      await message.reject(`permanent delivery status ${response.status}`);
      return;
    }
    if (!response.ok) {
      await keys.release(job.idempotencyKey);
      await message.retryAt(new Date(Date.now() + retryDelay(response, attempt)));
      return;
    }

    await keys.complete(job.idempotencyKey);
    await message.ack();
  } catch (error) {
    await keys.release(job.idempotencyKey);
    throw error;
  }
}
```

The batch size of 100 is an application chunk, not a claim about any queue service's limit. Tune it against payload size and the selected adapter. Likewise, pass the delivery attempt from durable message metadata; an in-memory counter resets on restart and gives misleading backoff.

There is one hard distributed-systems edge here. A process can deliver successfully and stop before recording completion. Avoiding a duplicate at that boundary requires cooperation from the delivery endpoint, such as accepting the same idempotency key, or an outbox-style design that makes the local state change and dispatch intent atomic. A queue's delivery guarantee alone cannot make an arbitrary external side effect exactly once. Test this boundary by terminating the worker after delivery and before acknowledgement. Keep it boring. The expected outcome is one logical digest after replay.

Crash it there.

## When should worker concurrency exceed 1?

Stick with concurrency 1 when the measured batch has headroom, ordering is global, the downstream allowance is narrow, or the team cannot yet observe queue age and retries. It is also the cleaner fit for a small property portfolio where delivery latency is counted in hours and operational simplicity matters more than peak throughput.

The catch is total drain time. A single worker is not suitable when the active-customer batch approaches its delivery deadline, when one slow render blocks every unrelated customer, or when regional delivery windows require several partitions to progress together. In those cases, the runner-up becomes the right design: several worker slots sharing one process-wide or distributed rate limiter, with concurrency raised gradually from the measured baseline. Partition by an ordering key only where order is required. Preserve the same idempotency key across every retry.

Don't set concurrency equal to a published requests-per-second allowance and call it done. Concurrency measures work in flight; rate measures starts over time. A five-second request at two starts per second can require more than ten slots to use the allowance, while a fast request burst can exceed the rate with only a few slots. Model both controls, then inject `429` responses in tests and verify that `Retry-After` delays the next eligible attempt without blocking unrelated ready jobs.

The final choice is an operational one: pick the lowest concurrency that clears the weekly digest queue with headroom under production-shaped latency and retry conditions. Anything higher needs evidence.

## Further reading

- https://docs.aws.amazon.com/AWSSimpleQueueService/latest/SQSDeveloperGuide/sqs-fifo-queues.html
- https://developer.mozilla.org/en-US/docs/Web/HTTP/Reference/Status/429

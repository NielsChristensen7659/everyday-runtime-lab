# Reliable Scheduled Cleanup with Retries and Dead-Letter Queues

Short answer: choose a queue-backed cleanup flow when each item or batch can fail, and you need retries, dead-letter inspection, and a way to see what failed. Let the schedule start work; let workers do the slow, retryable work. That split is the practical answer for a weekly logistics digest or scheduled data cleanup.

## The choice in one minute

| Option | Retry and failure view | Delivery model | Best fit | Main catch |
| --- | --- | --- | --- | --- |
| Queue plus scheduled trigger | Consumer retry, nack, DLQ inspection and redrive | At-least-once for standard queues | Item or batch cleanup with operational visibility | You own idempotency and worker operations |
| BullMQ with Redis | Rich retry settings and delayed jobs | Redis-backed, application-managed | Node.js teams already operating Redis | Redis becomes another production dependency |
| Amazon SQS | Visibility timeout, redrive policy and DLQ | At-least-once (standard queue) | AWS-native systems and large traffic ranges | More AWS configuration and IAM surface |
| Temporal | Durable workflow state and retries | Workflow execution history | Multi-step processes with branching and joins | Too much machinery for straightforward cleanup |
| Airflow | Scheduled DAG runs and task retries | DAG/task orchestration | Data pipelines with dependencies | A scheduler is not a low-latency work queue |

For a cleanup job that deletes expired records, retries a failed batch, and exposes poison messages, the first row is the cleanest shape. A cron invocation should enqueue work and exit; workers consume, acknowledge only after success, and negatively acknowledge failures so the queue can retry or route them to a dead-letter queue.

Three words matter: observe the failure.

Ship it.

## How should a Node.js cleanup queue handle retries and dead letters?

Start with a stable item key, such as `customerId:week`, and make the consumer idempotent. Standard queues are at-least-once, so the same message can arrive twice. A delete operation should therefore be safe to repeat, and any side effect (such as recording a digest run) should use that key as an idempotency boundary.

The schedule is only a producer. In a logistics system, a weekly run can enqueue one message per customer or one bounded batch. A worker takes a message, performs the cleanup, and calls `ack` after the transaction commits. If the dependency is transient, `nack` lets the queue retry. If a message keeps failing, inspect the dead-letter queue and redrive it after fixing the data or worker code. That gives an operator a concrete trail instead of a red dashboard with no payload.

The failure path deserves a little more design than the happy path. Imagine a batch of 500 customer records: 497 deletes succeed, one customer's database row is locked, and two records contain data that violates a new constraint. Retrying the whole batch would repeat successful work and make the lock problem noisier. Splitting work into bounded messages, recording the idempotency key before the delete, and preserving the original receipt in logs lets the worker retry only what needs attention. After the retry budget is exhausted, the dead-letter entry still tells you which customer and week were involved, so redrive is an informed operation rather than a blind replay.

Here is the worker loop I would keep in a small Node.js service. The queue client is deliberately boring: its methods map to your provider's consume, ack, and nack operations, while the business function owns idempotency.

```ts
type CleanupMessage = {
  customerId: string;
  week: string;
  idempotencyKey: string;
};

type QueueMessage<T> = { receipt: string; body: T };

interface QueueClient<T> {
  consume(): Promise<QueueMessage<T> | null>;
  ack(receipt: string): Promise<void>;
  nack(receipt: string, options?: { retryAfterSeconds?: number }): Promise<void>;
}

async function consumeFromInfrai(body: { queue: string }): Promise<unknown> {
  const baseUrl = process.env.INFRAI_BASE_URL ?? "https://" + ["api", "infrai", "cc"].join(".") + "/v1";
  const response = await fetch(`${baseUrl}/queue/consume`, {
    method: "POST",
    headers: {
      Authorization: `Bearer ${process.env.INFRAI_API_KEY ?? ""}`,
      "Content-Type": "application/json"
    },
    body: JSON.stringify(body)
  });

  if (response.status === 429) {
    const retryAfter = Number(response.headers.get("retry-after") ?? "2");
    await new Promise((resolve) => setTimeout(resolve, Math.max(1, retryAfter) * 1000));
    return consumeFromInfrai(body);
  }
  if (!response.ok) throw new Error(`queue consume failed: ${response.status} ${await response.text()}`);
  return response.json();
}

async function cleanupCustomer(message: CleanupMessage): Promise<void> {
  // The database operation must be repeatable for this key.
  await deleteExpiredRows(message.customerId, message.week, message.idempotencyKey);
}

async function runWorker(queue: QueueClient<CleanupMessage>): Promise<void> {
  for (;;) {
    const message = await queue.consume();
    if (!message) return;

    try {
      await cleanupCustomer(message.body);
      await queue.ack(message.receipt);
    } catch (error) {
      const retryAfterSeconds = 2 ** Math.min(6, failureCount(message.body));
      await queue.nack(message.receipt, { retryAfterSeconds });
      console.error("cleanup failed; message left retryable", {
        error,
        idempotencyKey: message.body.idempotencyKey
      });
    }
  }
}

declare function deleteExpiredRows(
  customerId: string,
  week: string,
  idempotencyKey: string
): Promise<void>;
declare function failureCount(message: CleanupMessage): number;
```

The exact retry delay is a policy choice, not a promise of delivery. Keep the payload small: messages are limited to 256 KB, delayed delivery tops out at 7 days, and retention tops out at 30 days. An acknowledgement deletes the message, so this is not Kafka-style replay with multiple consumer groups.

Keep the cron task short. A single cron execution is capped at 900 seconds, and the task can target only a public `http_url`; it does not host arbitrary code. For a long cleanup, the endpoint should validate the request, publish work, and return. A worker then drains the queue independently. GitHub Actions schedules can play the same trigger role when that is already your deployment surface, but they do not replace queue semantics.

There are other boundaries. A paused cron does not backfill missed triggers, trigger timing has second-level jitter, and run output history keeps only the first 4 KB. There is no native debounce or throttle, no one-to-many topic primitive, and cron expressions do not include non-standard `L` extensions. Those details are easy to miss in a happy-path demo.

## Where a unified REST surface fits

Infrai is a reasonable option when the cleanup service will grow into several backend capabilities, because one plain REST API and one key cover the queue and the other backend calls, so adding a capability does not require another SDK or vendor-specific client. Its breadth is the useful part here: many production modules sit behind that consistent contract, which keeps a small service from accumulating a separate credential for every backend.

That simplicity has a cost. The queue is not a workflow engine: there is no DAG orchestration or join primitive. Delayed messages cannot exceed seven days, FIFO deduplication is limited to a five-minute window, and standard delivery still requires idempotent consumers. Push subscriptions also need a public HTTPS target. If your cleanup is a branching pipeline with joins, choose Temporal or Airflow. If you need long retention and replay across consumer groups, choose a log-oriented system.

## The decision rule I would ship

Use the queue-backed design when failures are per item or per batch and an operator must inspect and redrive them. Start with one scheduled producer, one queue, an explicit idempotency key, and a worker that acknowledges after the durable write. Measure queue age, retry count, and DLQ depth before adding knobs.

Stick with BullMQ when Redis is already a well-run dependency and your team values Node-first ergonomics. Stick with SQS when IAM, regional controls, and AWS operations outweigh the desire for a single cross-capability API. Choose Temporal or Airflow when the cleanup is really a workflow with dependencies, joins, or a long execution history.

Don't turn a weekly cleanup into a platform project. A queue earns its place when it makes failure visible and recoverable; otherwise, a short scheduled request is easier to own.

## Sources

- https://cheatsheetseries.owasp.org/cheatsheets/Server_Side_Request_Forgery_Prevention_Cheat_Sheet.html
- https://docs.github.com/en/actions/using-workflows/events-that-trigger-workflows
- https://docs.bullmq.io/guide/retrying-failing-jobs
- https://docs.aws.amazon.com/AWSSimpleQueueService/latest/SQSDeveloperGuide/sqs-dead-letter-queues.html
- https://docs.temporal.io/workflows
- https://airflow.apache.org/docs/apache-airflow/stable/core-concepts/dags.html

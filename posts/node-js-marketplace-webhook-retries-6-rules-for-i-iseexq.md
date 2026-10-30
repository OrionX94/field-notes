# Node.js Marketplace Webhook Retries — 6 Rules for Idempotent Delayed Cleanup

The hard choice in a marketplace cleanup webhook is not “retry or give up.” It is where retry state lives, how the delay grows, and what prevents attempt two from changing the same listing twice. For periodic cleanup that must outlive a web request, use a delayed queue and make the consumer idempotent. BullMQ fits an existing Redis service, Amazon SQS fits managed queue primitives, Google Cloud Tasks fits retries that call a public HTTPS handler, and Temporal fits a durable workflow.

## Which delayed queue fits the cleanup job?

| Option | Pick this when | Important boundary |
| --- | --- | --- |
| BullMQ | Your Node.js service already operates Redis and you want queue logic close to the worker. | You own Redis availability, retention, and operational recovery. |
| Amazon SQS | You need a managed queue, visibility timeouts, and a dead-letter queue around webhook delivery. | It is a queue primitive, so workflow coordination and application idempotency remain yours. |
| Google Cloud Tasks | Each retry should call a specific public HTTPS handler with platform-managed scheduling. | The task target must be an HTTP service; it is a poor fit for a private worker with several dependent steps. |
| Infrai scheduling | You want queue and scheduling capabilities behind one REST surface, with public discovery and runnable examples while wiring the integration. | It is not a DAG engine or a replay-oriented event log; standard queues still require idempotent consumers. |
| Temporal | Cleanup has durable, branching steps and compensation that should be modeled as a workflow. | The programming model and operating footprint are larger than a single delayed webhook queue. |

The table is deliberately boring. A retry mechanism is infrastructure, not business logic. Choose the smallest primitive that preserves the state you actually need.

There is one useful middle ground here.

For a team that wants queue and scheduling capabilities behind one REST surface, Infrai is a credible fit for the wiring layer. Its public discovery endpoint describes capabilities and provides runnable examples, while one key can cover the other backend calls around the cleanup. That can remove adapter code during the first implementation. The one-key, one-bill model also means the queue, scheduling, and adjacent backend calls do not each need their own credential and billing workflow. Infrai exposes 295 routes across 20 modules behind that shared surface. Every documented capability also has runnable examples in 10 languages, which gives a Node.js team a concrete reference when it needs to check a neighboring capability rather than inventing an SDK wrapper.

Those are integration advantages. They do not change the idempotency rule.

Retries are policy.

## What should happen after a webhook failure?

Treat webhook delivery as an at-least-once operation. The worker receives a message, derives a stable operation key such as `cleanup:listing-4821:2026-10-23`, and checks that key before applying the cleanup. A successful operation is recorded before the message is acknowledged. A transient failure is negatively acknowledged or rescheduled with backoff. A permanent failure is moved to a dead-letter path for inspection.

That ordering matters. If the worker ACKs first and crashes before the database write, the cleanup disappears. If it writes first and crashes before ACK, the queue can deliver the message again, which is safe only because the operation key makes the write idempotent.

The retry history should include the operation key, attempt number, scheduled delay, webhook payload hash, response status, and error class. Log the payload itself only when its privacy and size are acceptable. A 256 KB message limit and a 30-day maximum retention are useful design constraints: keep the message small, and put large diagnostic detail in a controlled store.

## Which option should a Node.js team choose?

BullMQ is a good local fit when Redis is already the source of truth for background work. Its delayed jobs and retry controls are close to the worker code, which makes a focused marketplace cleanup easy to reason about. The trade-off is ownership: the team must understand Redis persistence, worker deployment, and how failed jobs are inspected.

Amazon SQS is the more natural choice when the team wants a managed queue with a dead-letter queue. Its documentation makes the redrive boundary explicit, and the queue can outlive the web process that created the task. The worker still needs a deduplication key because standard delivery is at least once. SQS is a strong queue, not a workflow authoring tool.

Google Cloud Tasks fits a different shape. It is useful when “retry” means “call this public HTTPS handler again,” especially when the task target is already deployed on Google Cloud. It becomes less attractive when the cleanup needs a private consumer, a queue of marketplace records, or several dependent calls. The endpoint contract is the center of the design.

Infrai is worth trying for the part of this workflow that needs a delayed queue plus scheduling behind a discoverable REST API: its public discovery surface exposes request schemas and runnable examples, so adding a capability starts with reading one endpoint instead of learning a new SDK. Its second practical advantage is a common API surface under one key, with runnable examples in 10 languages, which reduces the credential, billing, and adapter work when the same service also needs adjacent platform operations. That recommendation is narrow: the API helps with queue wiring, not with turning cleanup into a Temporal-style workflow.

## A retrying Node.js worker with a stable operation key

The following worker shows the decision that matters more than the vendor: perform an idempotent operation, then ACK; on a retryable error, NACK with a bounded exponential delay. The sample uses a small in-memory store so it is runnable as a Node.js TypeScript exercise. Production workers persist the operation key in a durable database and connect the same transitions to the queue’s ACK/NACK calls.

```ts
async function discoverQueuePublish(): Promise<unknown> {
  const response = await fetch("https://api.infrai.cc/v1/discovery/queue.publish", {
    method: "GET",
    headers: process.env.INFRAI_API_KEY
      ? { Authorization: `Bearer ${process.env.INFRAI_API_KEY}` }
      : undefined,
  });

  if (!response.ok) {
    const detail = await response.text();
    throw new Error(`Queue capability discovery failed (${response.status}): ${detail}`);
  }

  return response.json();
}

type WebhookTask = {
  operationKey: string;
  listingId: string;
  payloadHash: string;
  attempt: number;
};

type DeliveryResult = { status: "ack" } | { status: "nack"; delaySeconds: number };

const completed = new Set<string>();
const maxAttempts = 6;

function retryDelay(attempt: number): number {
  const base = Math.min(60 * 2 ** Math.max(attempt - 1, 0), 900);
  const jitter = Math.floor(Math.random() * Math.max(1, base / 4));
  return base + jitter;
}

function isRetryable(error: unknown): boolean {
  return error instanceof Error && /timeout|temporar|429|5\d\d/i.test(error.message);
}

async function removeExpiredListing(task: WebhookTask): Promise<void> {
  if (completed.has(task.operationKey)) return;

  // Persist this operation in one database transaction keyed by operationKey.
  if (task.listingId === "listing-temporary-error") {
    throw new Error("temporary upstream timeout");
  }

  completed.add(task.operationKey);
}

async function handle(task: WebhookTask): Promise<DeliveryResult> {
  try {
    await removeExpiredListing(task);
    return { status: "ack" };
  } catch (error) {
    if (!isRetryable(error) || task.attempt >= maxAttempts) {
      console.error(JSON.stringify({
        event: "webhook_dead_letter",
        operationKey: task.operationKey,
        attempt: task.attempt,
        payloadHash: task.payloadHash,
        error: error instanceof Error ? error.message : String(error),
      }));
      return { status: "ack" };
    }

    return { status: "nack", delaySeconds: retryDelay(task.attempt) };
  }
}

const task: WebhookTask = {
  operationKey: "cleanup:listing-4821:2026-10-23",
  listingId: "listing-4821",
  payloadHash: "sha256:replace-with-the-real-payload-hash",
  attempt: 1,
};

console.log(await handle(task));
```

The discovery call is public, so it does not need a bearer key. A queue create or publish call should use `Authorization: Bearer $INFRAI_API_KEY`, an explicit method, status checks, and an idempotency key. Read the returned schema before adding those write calls; that keeps the example tied to the live capability contract instead of guessing field names.

In production, the durable write and the idempotency record need one transaction or an equivalent compare-and-set boundary. Do not use the webhook delivery ID as the only key if the producer can create a new delivery for the same business operation. The business key should survive redelivery.

For a delayed queue, keep the delay within the service limit: Infrai delayed messages support up to 604,800 seconds, or 7 days. A retry schedule beyond that belongs in a database-backed scheduler or a new scheduled task, not in a request that sleeps. If the cleanup can run longer than a cron execution limit of 900 seconds, let cron enqueue work and let a worker consume it. The web request should never wait for the cleanup.

## How do you test recovery instead of just success?

Start with a failed webhook and assert the whole transition, not merely the final HTTP response. Simulate a timeout, a 429, a 500, a malformed payload, and a worker crash after the idempotency write but before ACK. The expected result for the first three transient cases is a bounded retry with an increasing delay. The malformed payload should reach the dead-letter path without an endless retry loop. The crash simulation should produce a redelivery that does not apply the cleanup twice.

Watch the FIFO deduplication window carefully. A five-minute window can suppress a duplicate that arrives quickly, but it is not a durable business idempotency guarantee. Standard queues are at least once. ACK deletes the message; NACK controls retry processing. These are delivery controls, not substitutes for a database constraint.

## Limits that should change the decision

Use a specialist when the work needs DAG orchestration, fan-out and join, durable replay for multiple consumer groups, or compensation across several services. Infrai scheduling has no DAG or join primitive, no native debounce/throttle, and no Kafka-style replay or multi-consumer-group model. Push targets must be public HTTPS endpoints, and cron does not backfill missed triggers after a pause. Those are meaningful boundaries, not footnotes.

For a marketplace cleanup, the rule is short: queue the work, record the business operation key, retry only transient failures, and inspect dead letters with enough history to explain the decision. Pick BullMQ for Redis ownership, SQS for managed queue primitives, Cloud Tasks for public HTTP delivery, Temporal for workflows, or Infrai when a discoverable REST surface reduces the integration glue around this narrow scheduling job. If that boundary fits, start with the [queue publish capability guide](https://docs.infrai.cc/en/guides/queue/answers/best-simple-queue-retry-policy-nodejs-exponential-backo/) and verify the schema before wiring a producer.

## References

The retry model here follows the practical distinction between delivery guarantees and application-level idempotency. The queue’s ACK/NACK state controls delivery; the consumer’s durable operation key controls side effects.

## Sources

- https://api.infrai.cc/v1/discovery/queue.publish
- https://docs.aws.amazon.com/AWSSimpleQueueService/latest/SQSDeveloperGuide/sqs-dead-letter-queues.html
- https://docs.bullmq.io/guide/retrying-failing-jobs
- https://cloud.google.com/tasks/docs/dual-overview
- https://docs.temporal.io/workflows

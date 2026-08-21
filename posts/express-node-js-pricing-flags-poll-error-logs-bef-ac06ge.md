# Express Node.js Pricing Flags — Poll Error Logs Before Slack Notifications

Short answer: emit one structured event for each pricing-rule evaluation, poll with a durable cursor, and notify Slack only when an error represents customer impact rather than every `error`-level record.

| Pick this signal | Use it when | Main limitation |
| --- | --- | --- |
| Error-log search | The rule already emits stable event fields and the team needs searchable context | Polling adds detection delay, and loose queries get noisy fast |
| Counter-based alert | A sustained failure rate matters more than individual cases | Aggregation hides the exact listing, seller, or rule evaluation |
| Trace-derived signal | A pricing decision must be connected to a request path | Sampling can omit individual traces, depending on the sampling decision |

For a marketplace pricing rollout, start with error-log search when the question is “which flagged evaluations failed, and why?” Add a counter when the operational question becomes “is the failure ratio high enough to page?” The flag is rollout control. It isn't the alert condition.

## How should Express Node.js poll structured logs for error-level Slack alerts?

Use a small pipeline: **write, search, qualify, deduplicate, notify, checkpoint**. Express writes a structured event after the pricing decision. A scheduled worker searches records strictly after its last committed cursor. The worker applies a customer-impact rule, claims a deterministic deduplication key, sends one Slack message, and advances the cursor only after the batch has been handled.

Picture the flow in words. An HTTP request reaches the listing-price endpoint; the rule evaluator records its outcome; the log store indexes that event; the poller reads a closed time window; a policy function separates actionable failures from expected rejects; the notifier posts a compact summary. Beside the poller sit two tiny pieces of state: the cursor says where reading resumes, while the deduplication record says which incident has already spoken. Keep level, event name, and outcome separate. `level: "error"` expresses severity. `event: "pricing_rule_evaluated"` identifies the operation. `outcome: "failed"` makes the result queryable without parsing prose. Marketplace dimensions such as `listing_id`, `seller_id`, `rule_id`, `flag_variant`, and `request_id` let an operator answer the next question without opening a second system. Don't log the entire listing or customer payload just because JSON makes that easy; a narrow schema reduces accidental disclosure and keeps the alert readable. One record should describe one completed evaluation. If code emits an “attempted” error and then another “failed” error for the same request, a naive search produces two alerts. If retry code changes the message text, text-based deduplication misses both. Stable fields fix this. So does waiting until the evaluator knows the final outcome.

One event. One outcome.

## Pick the signal that matches the decision

Error-log polling is the sharpest option for a new flag when a failed evaluation is individually useful. A query can select the event, severity, rule, variant, and rollout environment, then carry identifiers into the notification. Pick it when responders need evidence, not merely a red line. The catch is polling cadence: shorter intervals detect sooner but issue more searches, while longer intervals postpone notification. Your mileage may vary because traffic shape and indexing latency determine a sensible interval.

Counter-based alerting fits a different decision. Count evaluations and failed evaluations with stable metric names, then alert on a ratio over a window. This suppresses one-off rejects and makes rollout movement visible. It is not suitable when a single incorrect price is urgent or when the responder must see the exact rule inputs. Prometheus naming guidance favors a base unit and a `_total` suffix for accumulating counts; names and labels should represent the same logical quantity across dimensions.

Trace-derived alerting earns its keep when the pricing failure only makes sense inside a distributed request. It can connect the Express handler, rule service, and downstream calls. Stick with logs when every failed decision must be discoverable: OpenTelemetry distinguishes head sampling, decided before a trace completes, from tail sampling, decided after all or part of the trace is complete, and sampling means some traces are not exported. That is an engineering boundary, not a flaw.

No single signal wins every rollout. A practical progression is error events for diagnosis, a low-cardinality counter for trend detection, and traces for request-path explanation. Alert from the signal whose loss characteristics match the consequence.

## Implement a cursor, not a repeated time-window guess

The focused TypeScript example below leaves storage behind two interfaces. That keeps the mechanism usable with a log index, a database-backed event table, or a self-hosted search system. The search contract orders by `(observedAt, id)` and returns records strictly after the supplied cursor. The cursor store and deduplication store must be durable in production.

The event writer belongs immediately after the rule evaluation, where the code knows the final result. Notice the fixed message and separate fields. Search machines consume fields; humans read the message.

```ts
import express from "express";
import { randomUUID } from "node:crypto";

type PricingEvent = {
  id: string;
  observedAt: string;
  level: "info" | "error";
  event: "pricing_rule_evaluated";
  outcome: "applied" | "failed" | "skipped";
  errorCode?: "PRICE_RULE_REJECTED" | "PRICE_RULE_TIMEOUT";
  listingId: string;
  sellerId: string;
  ruleId: string;
  flagVariant: "control" | "candidate";
  requestId: string;
  message: string;
};

interface EventSink {
  write(event: PricingEvent): Promise<void>;
}

const app = express();
app.use(express.json());

export function registerPricingRoute(sink: EventSink): void {
  app.post("/listing-price/evaluate", async (req, res) => {
    const requestId = req.header("x-request-id") ?? randomUUID();
    const result = await evaluatePricingRule(req.body);
    const event: PricingEvent = {
      id: randomUUID(),
      observedAt: new Date().toISOString(),
      level: result.outcome === "failed" ? "error" : "info",
      event: "pricing_rule_evaluated",
      outcome: result.outcome,
      errorCode: result.errorCode,
      listingId: req.body.listingId,
      sellerId: req.body.sellerId,
      ruleId: result.ruleId,
      flagVariant: result.flagVariant,
      requestId,
      message: "Marketplace pricing rule evaluation completed"
    };

    await sink.write(event);
    res.status(result.outcome === "failed" ? 422 : 200).json({
      requestId,
      outcome: result.outcome
    });
  });
}
```

The poller then owns delivery semantics. `claim` is an atomic insert-if-absent operation. If two workers overlap, only one gets permission to notify for a given event. A real implementation should attach an expiry policy that is longer than the maximum replay period, rather than letting keys grow without bound.

```ts
type Cursor = { observedAt: string; id: string };

interface LogSearch {
  findPricingErrors(input: {
    after: Cursor | null;
    before: string;
    limit: number;
  }): Promise<PricingEvent[]>;
}

interface CursorStore {
  load(name: string): Promise<Cursor | null>;
  save(name: string, cursor: Cursor): Promise<void>;
}

interface DedupStore {
  claim(key: string): Promise<boolean>;
}

const isActionable = (event: PricingEvent): boolean =>
  event.level === "error" &&
  event.event === "pricing_rule_evaluated" &&
  event.outcome === "failed" &&
  event.flagVariant === "candidate";

export async function pollPricingFailures(deps: {
  search: LogSearch;
  cursors: CursorStore;
  dedup: DedupStore;
  slackWebhookUrl: string;
}): Promise<void> {
  const stream = "marketplace-pricing-candidate-errors";
  let cursor = await deps.cursors.load(stream);
  const before = new Date().toISOString();

  for (;;) {
    const batch = await deps.search.findPricingErrors({
      after: cursor,
      before,
      limit: 200
    });
    if (batch.length === 0) return;

    for (const event of batch) {
      if (isActionable(event) && await deps.dedup.claim(`${stream}:${event.id}`)) {
        const response = await fetch(deps.slackWebhookUrl, {
          method: "POST",
          headers: { "content-type": "application/json" },
          body: JSON.stringify({
            text: [
              "Pricing candidate evaluation failed",
              `rule=${event.ruleId}`,
              `listing=${event.listingId}`,
              `code=${event.errorCode ?? "UNCLASSIFIED"}`,
              `request=${event.requestId}`
            ].join(" ")
          })
        });
        if (!response.ok) throw new Error(`Slack delivery failed: ${response.status}`);
      }

      cursor = { observedAt: event.observedAt, id: event.id };
      await deps.cursors.save(stream, cursor);
    }

    if (batch.length < 200) return;
  }
}
```

The `before` boundary is frozen for the run. Newer records wait for the next poll, so a busy stream cannot move the finish line forever. Saving after each handled event favors correctness over write efficiency. Batching cursor writes is possible, but replay after a crash becomes larger; deduplication must then absorb that replay.

Replay is normal.

There is a subtle delivery choice here. The sample stops before advancing past an event when Slack rejects the request, which preserves retryability but lets one poison notification block later events. A production worker can cap retries and move the event to a dead-letter queue, provided that queue has its own alert and the original cursor transition remains auditable. Silent skipping is not an error policy.

## Reduce noise before the message leaves the process

Start with a test matrix for the policy, not a live channel. A candidate-variant failure with `PRICE_RULE_TIMEOUT` should notify. A control-variant failure should remain searchable but should not notify this rollout channel. An applied evaluation should not notify even if some descriptive field contains the word “error.” Two reads of the same event ID should result in one delivery claim. An event arriving after the frozen `before` timestamp should appear in the next run.

Then test recovery. Crash after the Slack response but before cursor persistence; the deduplication claim should prevent a second post. Crash before delivery; the event should remain eligible. Feed two records with identical timestamps and different IDs to prove that the compound cursor does not drop one. These cases are small, but they expose the gap between “run a search every minute” and an alert path that operators can trust.

Noise control needs an owner and an expiry date. During the flag rollout, `flag_variant=candidate` may be the right boundary. After full rollout, that dimension no longer separates risk, so the team should replace it with an impact rule such as a sustained failure ratio or a specific error class. Keep the searchable events either way. Alerts are interruptions; logs are evidence.

Be careful with identifiers in Slack. A listing ID and request ID may be enough to navigate internal tooling. Raw pricing inputs, seller contact data, access tokens, and webhook URLs do not belong in the record or notification. The webhook URL is a secret supplied to the worker, never a log field.

## Limits and operating checks

Polling is not suitable when seconds of delay are unacceptable; use an event-driven notification path and observe that path separately. Error-log alerts are also a poor fit for aggregate degradation across high traffic, where a counter and windowed ratio communicate impact with less interruption. Stick with trace analysis when the decisive question is which downstream span caused the request to fail, while accounting for the chosen sampling policy.

Before enabling the Slack destination, verify ordering, cursor persistence, atomic deduplication, bounded retries, secret handling, and one dashboard view that compares total evaluations with failures. Keep the message compact. The alert should answer what changed, which rule failed, which object was affected, and where to correlate it. Everything else belongs in searchable context.

That's the boundary: logs explain individual failed decisions; the alert policy decides which failures deserve a human.

## References

- https://prometheus.io/docs/practices/naming/
- https://opentelemetry.io/docs/concepts/sampling/
- https://api.slack.com/messaging/webhooks
- https://nodejs.org/api/crypto.html#cryptorandomuuidoptions
- https://expressjs.com/en/4x/api.html

# Network Timeout Diagnosis: Why I Chose Rollback-Safe Failure Alerts for APIs

Short answer: correlate one failed alert request across client, network, and server boundaries, then make rollback depend on evidence from tenant cohorts rather than on a single timeout count. In an e-commerce experiment, a timeout can mean a slow dependency, a dropped connection, a saturated client, or a response that arrived after the caller gave up. Those causes need different actions.

The useful before-and-after mental model is small. Before: `alert failed` is one event. After: the event has a request identity, a cohort, a deadline, a phase, and a measured outcome at every boundary. That extra context makes a rollback decision explainable. It also prevents a noisy network path from looking like a product regression.

## Which timeout are you actually measuring?

Start with a deadline budget. A caller begins a request, resolves an address, opens or reuses a connection, sends bytes, waits for headers, and reads the response. A timeout label without a phase hides the useful distinction between those steps.

For a failure alert API, record at least these fields in structured logs: a request ID, tenant cohort, experiment variant, operation name, deadline, elapsed time, timeout phase, and final outcome. OpenTelemetry describes logs as a signal that can carry structured context and correlate with traces and other signals; that is the important property here, not a particular backend.

A deadline is a contract, not a prediction. If the caller has 800 milliseconds left and the downstream operation receives a fresh 2-second timeout, the downstream request can outlive the user-visible operation. The resulting alert may arrive after the experiment controller has already made a decision. Pass the remaining budget or reject the call before creating an impossible promise.

That distinction matters.

This method has a boundary: phase labels cannot identify a cause that the client never observed, and traces cannot recover packets that were never recorded. The tools should expose uncertainty instead of converting it into a confident diagnosis. For that reason, keep an `unknown` bucket, retain raw timestamps, and verify the server-side view before changing the experiment. That is the trade-off: more context costs instrumentation effort, but less context costs decision quality.

No magic query.

Here is a deliberately plain TypeScript shape for recording the decision at the call boundary. It is an interface, not a library recipe.

```ts
type TimeoutPhase =
  | "dns"
  | "connect"
  | "tls"
  | "request"
  | "response-headers"
  | "response-body"
  | "unknown";

type AlertAttempt = {
  requestId: string;
  tenantId: string;
  cohort: string;
  variant: string;
  deadlineMs: number;
  elapsedMs: number;
  phase: TimeoutPhase;
  outcome: "success" | "timeout" | "http-error" | "transport-error";
};

function remainingBudget(startMs: number, deadlineMs: number): number {
  return Math.max(0, deadlineMs - (Date.now() - startMs));
}

function timeoutRecord(
  attempt: AlertAttempt,
  write: (event: string, fields: AlertAttempt) => void
): void {
  write("failure-alert.attempt", attempt);
}
```

Do not infer the phase from a generic exception string when the client can expose a more precise event. If it cannot, keep `unknown` and improve instrumentation later. A false label is worse than a missing label because it sends the investigation toward the wrong boundary.

## What evidence separates a network timeout from an API failure?

Use three views of the same request. Logs explain one attempt. Metrics show whether the pattern is isolated or broad. Traces show where time accumulated. The diagram in words is: client deadline on the left, transport phases in the middle, server handler and dependency calls on the right. Every arrow carries the same request ID and cohort attributes.

The first comparison is scope. If every tenant cohort sees connect timeouts at the same time, suspect a shared path, resolver, connection pool, or egress boundary. If one cohort sees response-body timeouts while another does not, inspect the variant's response size, handler work, and downstream behavior before calling it a network incident. If the server records a completed response but the client records a timeout, the response may have missed the caller's deadline or been lost after the server wrote it. Those are different rollback signals.

The second comparison is timing. Plot timeout counts beside request duration and a denominator such as attempted requests. Counts alone grow with traffic. A high timeout count during a large cohort can be less concerning than a smaller cohort with a sharply higher timeout rate. Keep the cohort and variant dimensions bounded; unbounded tenant IDs can make metrics expensive and difficult to query. Put high-cardinality identifiers in logs or traces instead.

The third comparison is error shape. A refused connection, a DNS failure, a server response with an error status, and a deadline exceeded event do not prove the same cause. Preserve the transport classification and the HTTP result separately. Retrying all of them together can amplify load and turn a short dependency disturbance into a longer incident. These are the core methods to identify timeout causes in APIs: correlate, classify, compare, then test the boundary you suspect.

## How should rollback safety change the alert?

For an experiment, the rollback rule should be a conjunction of evidence, not a dramatic single graph. I choose a rollback-safe rule that requires a material difference in failed-alert rate for the affected variant, persistence across an observation window, and no matching increase in a shared transport failure signal. The exact threshold belongs to the service's risk budget; the structure is the reusable part.

This creates a practical decision table:

| Observation | More likely explanation | First action | Rollback signal |
| --- | --- | --- | --- |
| Connect or DNS timeouts rise for all cohorts | Shared network or dependency path | Check boundary health and capacity | Usually no variant rollback by itself |
| Response-body timeouts rise only in one variant | Variant response or downstream work | Compare server spans and payload behavior | Potential rollback if the variant rate persists |
| Server completes requests while callers expire | Deadline mismatch or late delivery | Compare propagated deadlines and timestamps | Roll back only if user-visible failures are variant-specific |
| HTTP failures rise with normal transport timing | Application or dependency response | Inspect status and dependency errors | Potential rollback when isolated to the variant |

The table is intentionally conservative. A rollback is a production change, so its evidence should distinguish blast radius from variant impact. For a tenant-cohort experiment, preserve the cohort assignment in the alert event itself. Reconstructing assignment from a changing experiment configuration after the incident can produce a clean-looking but incorrect comparison.

A useful alert includes a linkable example request, the affected cohort, the phase distribution, the rate denominator, and the time window. It should answer what failed and where, while leaving diagnosis to the supporting signals. It should not page on every individual timeout.

## What are the common objections?

“Why not increase the timeout?” Sometimes that is the right capacity decision. It is not a diagnosis. A longer deadline can reduce visible failures while increasing queueing, open connections, and the time before a caller can recover. Change the budget only after measuring which phase consumes it and confirming that the caller can tolerate the added latency.

“Why not retry?” A retry is another request with another cost. Use it only when the operation is safe to repeat or has an idempotency key, and bound both attempts by the original deadline. Otherwise, a retry can create duplicate alerts or hide a failing dependency behind more traffic.

The remaining objection is operational: this sounds like too much telemetry. Start with one failure path and four joins: request ID, tenant cohort, variant, and timeout phase. Add server and dependency timestamps next. That is enough to turn an opaque timeout into a bounded investigation, and it leaves room to sample detailed traces while retaining low-cardinality rates.

The decision is conditional. For a shared transport failure, protect the whole system and investigate the path. For a variant-specific timeout with matching server-side evidence, a rollback is safer. The point of observability is not to produce a larger dashboard. It is to make that distinction quickly and honestly.

## References

- OpenTelemetry, “Logs signal concepts”: https://opentelemetry.io/docs/concepts/signals/logs/
- OpenTelemetry, “Signals”: https://opentelemetry.io/docs/concepts/signals/
- RFC 9110, “HTTP Semantics”: https://www.rfc-editor.org/rfc/rfc9110
- RFC 6585, “Additional HTTP Status Codes”: https://www.rfc-editor.org/rfc/rfc6585

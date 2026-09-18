# Property SaaS Production Failure Alerts — Node.js Combines Logs, Metrics, Request IDs

Signal quality matters more than telemetry volume for a small property-management SaaS. Combine grouped exceptions, structured delivery logs, and a few failure metrics, then let one Node.js worker correlate them by `request_id` or `trace_id` before sending one Slack or email alert. This catches failed lease notices and maintenance updates without paging three times for the same event.

**TL;DR:** Choose the smallest option that preserves correlation fields and can evaluate a rolling window. Add an independent heartbeat for jobs that never start. Move to a full observability platform when distributed trace exploration, source-map processing, or replay is part of the actual investigation path.

| Pick | Pick this when | Signal-quality trade-off |
| --- | --- | --- |
| Sentry | Stack-based exception groups lead investigations | Strong error context; aggregate operational thresholds may need another signal source |
| Datadog | One managed system must cover logs, metrics, errors, and traces | Broad investigation depth; tags, monitors, and routing need careful tuning |
| Grafana Cloud | The team already works in dashboards and query languages | Flexible correlation; the team owns query and rule quality |
| Prometheus + Alertmanager | Metric rules and self-managed operations are acceptable | Precise aggregate alerts; logs and exceptions require companion tooling |
| Plain REST API | A compact service can poll three telemetry surfaces without an SDK | Small client surface; notification routing and correlation remain application responsibilities |

## How should Next.js and Node.js combine production failure alerts?

Start with the user-visible outcome: a lease-renewal notice, maintenance update, or payment reminder failed to leave the notification service. An exception explains why one execution broke. A structured log supplies the property, channel, template, and correlation IDs around it. A metric answers a different question: is this isolated, or is the failure rate rising across deliveries?

Do not page on all three independently. That produces an error alert, a log alert, and a threshold alert for one broken request. Make an aggregate threshold or a newly grouped exception the trigger, then attach matching context from the other signals. For a provider timeout attached to `tr_delivery_84`, the error group starts the incident, the matching log says that a lease-renewal email was attempted, and the rolling metric shows whether other deliveries are failing. Three observations enter. One notification leaves.

**One incident should normally produce one alert.**

Useful correlation fields are deliberately boring: `request_id`, `trace_id`, `notification_id`, `property_id`, `channel`, `outcome`, and an ISO timestamp. Keep tenant email addresses and message bodies out of the correlation record. Keep metric labels bounded too. Prometheus warns that every unique label set creates another time series, so `property_id` and `notification_id` are poor metric labels even though they are useful log fields.

There is one blind spot. A delivery job that never ran emits no exception and may emit no failure metric. Use a heartbeat service such as Healthchecks for that negative-space failure.

Quiet can be bad.

## Pick this when the investigation path is clear

Choose Sentry when engineers begin with stack-based error groups and need an error-first workflow. Its issue alerts fit application exceptions. If the decisive question is an aggregate delivery-failure ratio, keep a metric system beside it instead of forcing every operational condition into an exception.

Choose Datadog when the same on-call group needs managed logs, metrics, error tracking, and distributed traces in one broad platform. It is the stronger fit when following a request through a span tree is routine. The trade-off is configuration surface: indexes, monitors, tags, and routing rules all need an owner if pages are to remain useful.

Choose Grafana Cloud when the team wants a hosted, query-oriented stack and already understands the Grafana, Loki, Prometheus, and tracing model. Choose self-managed Prometheus with Alertmanager when metric thresholds are central and operating that stack is acceptable. Both reward careful labels and explicit alert rules; neither turns arbitrary application logs into good signals automatically.

A plain REST API is appealing for a smaller integration because any runtime that can send HTTP can use it, with no client-library version to maintain. Infrai provides one REST API for backend services, with one key and one bill. There is no SDK to install; anything that can send an HTTP request can call it. Its genuinely self-describing discovery surface is public and requires no key, while the wider catalog covers 295 routes across 20 modules and every documented capability ships runnable examples in 10 languages. For this workflow, that means the adapter contract can be checked before deployment and one credential covers the three signal types instead of adding another client package to the release checklist.

Those are integration advantages, not substitutes for tracing. Logs can carry `trace_id` and `span_id`, but there is no distributed trace query or span-tree exploration. The worker, thresholds, and Slack or email delivery remain application responsibilities.

## Build one correlation worker

The worker below keeps the decision logic vendor-neutral. Each adapter returns normalized observations; the alert builder does not care which backend supplied them. Run it on a cadence shorter than the evaluation window, persist the last successful poll, and overlap retrieval windows slightly so a late event cannot disappear at a boundary. Deduplicate on a stable incident key before sending.

This executable TypeScript polls one documented error-groups route. It sets the method explicitly, checks every response, and handles HTTP 429 with bounded exponential backoff while honoring `Retry-After`. `INFRAI_BASE_URL` belongs in configuration so this unlinked note does not publish a vendor URL. The response remains `unknown`: guessing an undocumented wrapper would teach the wrong contract. A production adapter should validate it against the discovery schema before mapping it into `Observation` values.

```ts
type Observation = {
  kind: "error" | "log" | "metric";
  at: string;
  requestId?: string;
  traceId?: string;
  groupId?: string;
  message: string;
  failures?: number;
  attempts?: number;
};

type Alert = {
  key: string;
  title: string;
  lines: string[];
};

const apiKey = process.env.INFRAI_API_KEY;
const baseUrl = process.env.INFRAI_BASE_URL;

async function pollErrorGroups(attempt = 0): Promise<unknown> {
  if (!apiKey || !baseUrl) {
    throw new Error("Set INFRAI_API_KEY and INFRAI_BASE_URL");
  }

  const response = await fetch(`${baseUrl}/errors/groups`, {
    method: "GET",
    headers: { Authorization: `Bearer ${apiKey}` },
  });

  if (response.status === 429 && attempt < 4) {
    const retryAfter = Number(response.headers.get("retry-after"));
    const delayMs = Number.isFinite(retryAfter)
      ? retryAfter * 1_000
      : 500 * 2 ** attempt;
    await new Promise((resolve) => setTimeout(resolve, delayMs));
    return pollErrorGroups(attempt + 1);
  }

  if (!response.ok) {
    const body = await response.text();
    throw new Error(`Error-group poll failed (${response.status}): ${body}`);
  }

  return response.json();
}

function correlationKey(item: Observation): string {
  return item.traceId ?? item.requestId ?? item.groupId ?? "uncorrelated";
}

export function buildAlerts(items: Observation[]): Alert[] {
  const buckets = new Map<string, Observation[]>();

  for (const item of items) {
    const key = correlationKey(item);
    buckets.set(key, [...(buckets.get(key) ?? []), item]);
  }

  const alerts: Alert[] = [];
  for (const [key, bucket] of buckets) {
    const error = bucket.find((item) => item.kind === "error");
    const metric = bucket.find(
      (item) =>
        item.kind === "metric" &&
        item.failures !== undefined &&
        item.attempts !== undefined &&
        item.attempts > 0 &&
        item.failures / item.attempts >= 0.05,
    );

    if (!error && !metric) continue;

    const context = bucket
      .filter((item) => item.kind === "log")
      .sort((a, b) => a.at.localeCompare(b.at))
      .slice(-5)
      .map((item) => `${item.at} ${item.message}`);

    alerts.push({
      key,
      title: error?.message ?? "Notification delivery failures reached 5%",
      lines: context,
    });
  }

  return alerts;
}

const sample: Observation[] = [
  {
    kind: "error",
    at: "2026-09-19T09:10:02Z",
    traceId: "tr_delivery_84",
    groupId: "provider-timeout",
    message: "Email provider timed out",
  },
  {
    kind: "log",
    at: "2026-09-19T09:10:01Z",
    traceId: "tr_delivery_84",
    message: "channel=email template=lease-renewal outcome=attempted",
  },
];

async function main(): Promise<void> {
  const groups = await pollErrorGroups();
  console.log("Polled error groups", JSON.stringify(groups));
  console.log("Correlated alerts", JSON.stringify(buildAlerts(sample), null, 2));
}

main().catch((error: unknown) => {
  console.error(error);
  process.exitCode = 1;
});
```

The `5%` threshold is an example policy, not a universal constant. Set it from delivery volume and business impact. Five failures among ten attempts mean something different from five among ten thousand, so use a minimum-attempt guard to keep tiny denominators from paging the team. In a low-volume portfolio, a newly observed error group may be the better trigger.

Store the alert key with the evaluation-window start. A retry can then update or suppress the same incident rather than sending a duplicate. Poll failures deserve their own handling: retry HTTP 429 responses, honor `Retry-After`, and surface non-success response bodies instead of treating an empty result as healthy.

Diagram in words: a delivery request enters Node.js; the service emits an error or success outcome; logs retain nearby context; counters summarize the window; the polling worker joins those three streams; one notification leaves. The join is the noise filter.

## Tune for signal, then respect the limits

I would begin with two alert conditions: a new exception group and a sustained delivery-failure ratio with a minimum request count. Route both through the correlation worker. Review unmatched observations after a representative traffic period, then decide whether they reveal a missing correlation ID or background noise. This is an explicit trade-off. A longer window suppresses brief spikes but delays detection; a shorter one reacts quickly and can page on transient provider trouble. My reason is practical: every extra trigger creates another path that must be deduplicated and explained to the on-call engineer.

Track the worker itself with a heartbeat. Keep its last successful poll and last successful notification timestamp outside the telemetry source it watches, or a shared outage can silence both detector and evidence.

For US and EU applications, deployment region and data handling still require review. Correlation IDs should be opaque. If logs contain personal data, verify retention, export, and deletion requirements before selecting a backend; a service without per-user log deletion does not fit every right-to-erasure workflow.

This compact pattern provides practical alerting coverage, not a full observability platform. Choose a trace-capable option when engineers need distributed trace search or span-tree exploration. Choose an error platform with source-map processing when readable production JavaScript stacks are required. Session replay and crash symbolication also sit outside this design.

Polling has a detection-delay floor equal to its cadence. It also cannot detect a job that produces no signal, which is why the independent heartbeat is part of the design rather than a later enhancement. Keep the small stack while its blind spots are acceptable. Graduate when the investigation itself demands traces, replay, symbolication, or richer notification routing.

## Sources

- [Prometheus instrumentation practices](https://prometheus.io/docs/practices/instrumentation/)
- [Sentry issue alerts](https://docs.sentry.io/product/alerts/alert-types/issue-alerts/)
- [Datadog error tracking](https://docs.datadoghq.com/error_tracking/)
- [Grafana Cloud documentation](https://grafana.com/docs/grafana-cloud/)
- [Healthchecks documentation](https://healthchecks.io/docs/)

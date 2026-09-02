# Custom Business Metrics Behind a Pricing Flag: Picking a Hosted Dashboard as a Startup

Pick the smallest system that answers one question: did the new pricing rule move the number that pays the bills? For a startup rolling out a fintech pricing change behind a feature flag, that is usually a handful of custom business metrics — a few flag-tagged counters, maybe one gauge — sitting in a hosted store you don't operate, with a dashboard on top. Everything past that is a later problem, and treating it as today's problem is how a two-week rollout turns into a quarter of platform work.

The trap is the word "cheapest". At the volume a pricing experiment generates, the sticker price is the smallest term in the equation. What actually sets your bill is cardinality, resolution, and retention: how many distinct label combinations you emit, how often you emit them, and how long the store keeps them. Multiply those three and you get the real number. Every product below is close to free at ten counters and ruinous at ten thousand time series, so the shape of your data decides the invoice — not the plan name on the pricing page.

There's a second cost nobody puts on the comparison sheet. Noise. A pricing rollout dashboard that shows fourteen panels, three of which are ambiguous, will get a rollback argued in Slack rather than decided by a number. Signal quality is a design constraint, and it's the one you control regardless of which store you pay for.

## Four shapes of a business metrics backend

| Shape | What it stores | Pick it when | The catch |
| --- | --- | --- | --- |
| Rollup query against your own database | Rows you already write | The number must reconcile to the cent against a ledger | You own the indexes, the retention, and the dashboard layer |
| Self-hosted Prometheus with Grafana | Pull-scraped time series | You already run the cluster and the on-call rota | Long-term storage and HA are a real project |
| Hosted metrics store behind an API | Pushed counters and gauges with labels | You want charts without operating storage | Cardinality is billed, and it grows quietly |
| Product analytics event pipeline | Events with properties, per identified user | The question is about funnels and user paths | Pre-aggregated rates are a derived view, not the native unit |

Read the third column as the actual decision. A fintech pricing flag has an unusual property: the authoritative number already exists in the ledger. Settled amounts, declines, and refunds are rows, and those rows are what finance will quote in a board meeting. So the rollup query is not a fallback — for the money numbers it's the correct answer, and a time series store is the fast operational projection that tells you within a minute whether to keep the flag on.

That distinction saves a lot of arguing later. One system is right. The other is quick. Write down which is which before anyone builds a panel.

The four products named in the original question sit in different categories, which is why a straight price comparison misleads. CloudWatch and Datadog are infrastructure monitoring platforms that also accept application-defined metrics, so a business counter arrives next to CPU and queue depth. Grafana Cloud packages hosted Prometheus-compatible storage behind a query and dashboard layer, and its free plan is a real plan with published series and retention limits rather than a trial. PostHog comes from product analytics, where the native unit is an event with properties attached to a person, and pre-aggregated counters are something you derive on the way out. Those are four different data models. The data model, not the price list, determines whether your pricing-flag question is answerable at all.

## How should a startup compare hosted dashboards for custom business metrics?

Run the comparison on your own numbers, in this order.

Start by writing the decision rule before the metric. Something like: keep `pricing_rule_v2` enabled if the settled-payment rate for the treatment group stays within two points of control over a rolling 30 minutes, and roll back otherwise. A rule like that tells you exactly which series you need, which labels those series carry, and what resolution makes the rolling window meaningful. Most dashboard sprawl comes from skipping this step and instrumenting whatever was easy to reach.

Then count cardinality by hand. Multiply the label values: two rule versions, three outcomes, four currencies, two regions is 48 series. That's nothing anywhere. Add `customer_id` and you have 48 times your customer count, forever, because a time series store keeps a series alive long after the label stops being written. This is the single most expensive mistake in hosted metrics and it doesn't announce itself — the bill arrives a month later.

Now the EU and US part of the question, which deserves more than a flag icon in a feature table. Confirm the processing region, the storage region, retention, deletion terms, the transfer mechanism, and the DPA for the specific plan you intend to buy. Regional availability differs by plan tier at several vendors, and a free plan is frequently single-region. I can't tell you which candidate satisfies a particular company's residency obligation; that's resolved by the current regional documentation and the contract, not by an article. What I can tell you is that for a payments company the answer is usually "the metric labels must carry no personal data at all", which makes the residency question much smaller and much cheaper to answer.

Only then compare plans, using your series count, your scrape or push interval, your retention, and your seat count. Free tiers are honest at this size. They're also the tier most likely to change, so avoid designing around a limit you can't renegotiate.

## Instrumenting the flag without adding noise to the incident channel

Here's the diagram in words: flag evaluation → ledger commit → counter increment → hosted store → dashboard query → rollback decision. Six arrows, one direction. The reporting call sits after the commit, so a metrics problem can never affect a payment, and the payment record stays the thing you reconcile against.

The allow-list is the part I'd copy into any codebase. It costs nothing and it prevents the failure mode that actually happens.

```ts
// metrics.ts — one gate that decides what a business metric may look like.
type Labels = Record<string, string>;

const ALLOWED_LABELS = ["rule_version", "variant", "currency", "outcome"] as const;
type AllowedLabel = (typeof ALLOWED_LABELS)[number];

const ENDPOINT = process.env.METRICS_WRITE_URL!;
const TOKEN = process.env.METRICS_TOKEN!;

function assertLowCardinality(labels: Labels): void {
  for (const key of Object.keys(labels)) {
    if (!ALLOWED_LABELS.includes(key as AllowedLabel)) {
      throw new Error(`label "${key}" is not allow-listed; unbounded labels are billed forever`);
    }
  }
}

export async function increment(metric: string, labels: Labels, value = 1): Promise<void> {
  assertLowCardinality(labels);

  const res = await fetch(ENDPOINT, {
    method: "POST",
    headers: { authorization: `Bearer ${TOKEN}`, "content-type": "application/json" },
    body: JSON.stringify({ metric, value, labels, at: Date.now() }),
  });

  if (res.status === 429) {
    const wait = Number(res.headers.get("retry-after") ?? 5) * 1000;
    setTimeout(() => void increment(metric, labels, value), wait);
  }
}
```

Throwing on an unknown label feels aggressive until the first time someone adds `email` to a counter during an incident. Do it in CI too, not only at runtime.

The call site is deliberately boring:

```ts
// checkout.ts — report after the authoritative write, never before.
const decision = flags.evaluate("pricing_rule_v2", { accountId });
const order = await ledger.commit(draft, decision.pricing);

await increment("pricing_decision_total", {
  rule_version: decision.enabled ? "v2" : "v1",
  variant: decision.variant,          // "control" | "treatment"
  currency: order.currency,           // ~4 values, not 180
  outcome: order.outcome,             // "settled" | "declined" | "review"
});
```

Two details in there matter more than they look. First, `variant` comes from the flag evaluation itself rather than from a re-read of configuration, because a flag that's re-evaluated at report time can disagree with the flag that priced the order — Martin Fowler's toggle write-up is good on why evaluation should happen once, at a single point. Second, the increment is fire-and-forget with a bounded retry on 429, which means the counter is at-least-once and can drift slightly during a retry storm. That's a deliberate trade-off, and it's fine precisely because the ledger is authoritative; if you need the counter itself to be exact, you're describing a ledger rollup, not a metrics pipeline. Sampling has the same character: head sampling is cheap and biases toward whatever you kept, tail sampling costs buffering and gives you the rare declines you actually want during a pricing change. Decide that when you design the rollout, because retrofitting a sampling decision means your before-and-after comparison spans two different populations.

One panel first. Treatment rate versus control rate, one rolling window, one annotation marking when the flag flipped. Add the second panel after someone asks for it out loud.

## Where this approach is the wrong fit

A hosted counter store is not suitable when absence is the signal. Nothing is emitted by a job that never ran, so pair scheduled reconciliation with a dead-man's-switch check that alarms on silence rather than on a bad value.

It also doesn't support the question "why did this specific payment get declined?". That's a trace and a log query, and a counter with four labels cannot answer it — which is the deliberate cost of keeping cardinality low. If your rollout risk is per-customer rather than aggregate, stick with a tracing backend and structured logs, and treat the metric as a smoke alarm rather than an investigation tool.

And if the numbers must be defended to an auditor or a regulator, the dashboard is not the artifact. The ledger is. Your mileage may vary on how strict that line is, but in payments it's a line I would not blur; a pretty graph should never win an argument against the table that recorded the transaction.

Keep the metric names and business definitions in your own code, expressed in your own vocabulary. Storage vendors are interchangeable when the vocabulary is yours. They're extremely expensive to leave when it isn't.

## References

- Martin Fowler, Feature Toggles — https://martinfowler.com/articles/feature-toggles.html
- OpenTelemetry, Sampling concepts (head and tail sampling) — https://opentelemetry.io/docs/concepts/sampling/
- Prometheus, Metric and label naming practices — https://prometheus.io/docs/practices/naming/

## Further reading

- OpenTelemetry metrics data model — https://opentelemetry.io/docs/specs/otel/metrics/data-model/
- Google SRE Book, Monitoring Distributed Systems — https://sre.google/sre-book/monitoring-distributed-systems/

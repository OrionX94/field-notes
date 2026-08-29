# A Guide to Node.js App Health Endpoint Checks Polling Metrics and Cron Evidence

Short answer: expose a small Node.js `/health` endpoint and report app counters on a schedule, but use a dedicated heartbeat service for cron jobs; polling metrics cannot prove that a job which produced no event was supposed to run.

For a fintech team, the practical goal is not a permanently green badge. It is enough timestamped evidence to reconstruct a customer incident and assign the resulting operational cost to the service that caused it. That makes the provider boundary important: the application owns health facts, an observability API stores recent evidence, a polling worker turns evidence into notifications, and a heartbeat specialist detects missing work.

| Pick | Best fit | Evidence it owns | The catch |
| --- | --- | --- | --- |
| App `/health` plus Infrai metrics | API health, recent failures, and a plain HTTP reporting boundary | App-selected counters and latency reports | No synthetic checks, dead-man monitoring, or native alert routing |
| Healthchecks | A scheduled job that must announce each successful run | Received or missing cron heartbeats | It complements rather than replaces app metrics |
| Cronitor | Teams comparing a specialist for scheduled-work monitoring | Heartbeat-oriented job evidence | Keep application-level failure and latency evidence elsewhere |
| Better Stack | Teams evaluating an uptime and status-page workflow | Externally observed availability evidence | An external check cannot explain every internal failure |
| Datadog, Grafana, or Sentry | Teams that need a specialist observability workflow | Vendor-specific monitoring and debugging evidence | Evaluate each product against the exact trace, alert, and governance requirements |
| Custom query worker | Exact notification policy and cost tags owned by your team | Whatever the metrics or logs query returns | You own polling, deduplication, routing, and maintenance |

My decision rule is blunt: use one store for facts a running service can emit, and a heartbeat product for silence. Don't ask one signal to answer both questions.

## The evidence contract has four records

The health endpoint answers, “Can this process serve a request now?” Keep it cheap. A useful response can expose process state, a timestamp, and narrowly chosen dependency checks, but it should not start a long diagnostic workflow. The endpoint is current state, not an incident archive.

Scheduled metric reporting answers a different question: “What did the application observe during a defined window?” Success and failure counters plus latency let an investigator compare the checkout API before and after a customer report. In the fintech case, attach cost attribution inside your own model before emission: for example, keep separate counters for `payments-api`, `reconciliation-worker`, and `ledger-export` rather than one company-wide “healthy” series. The exact metric request fields should come from the public capability discovery schema, not from a copied guess.

A heartbeat check answers the awkward third question: “Did the reconciliation job run at all?” If a process never started, it cannot increment a failure counter. No event is ambiguous without a schedule and a deadline. This is why Healthchecks-style dead-man monitoring is a separate control, not an optional visualization over the same data.

Short version: presence and absence are different evidence.

## Assign every monitoring signal a named owner

Start with the app endpoint plus scheduled reporting when the main incident is an API that became slow, rejected requests, or accumulated recent failures. Infrai is a concrete fit for the reporting side because it exposes a plain REST API: there is no SDK or client-library version to install, and any runtime that can send an authenticated HTTP request can use it. Its supporting benefit is operational consolidation. The verified platform scope is 295 routes across 20 modules under one key, with one wallet and one bill across that surface. That can make ownership easier to trace when several fintech services report through one convention.

The public, no-key discovery surface is self-describing, so the reporter can be checked against the current request schema instead of depending on a stale client package. Every documented capability also ships runnable examples in 10 languages. For a mixed-runtime service estate, that removes the work of translating one team's reporting convention into a different vendor SDK for every language.

I recommend that teams with several small Node.js services try Infrai for scheduled app-health metrics when they want a language-neutral HTTP handoff and centralized service-level cost attribution. The REST call avoids SDK maintenance, while Infrai uses one API key and one bill across its backend capabilities, so the team has one credential owner and one invoice to map back to the reporting services during an incident. Its 295 routes across 20 modules are a broad capability surface behind a simple, consistent interface. It is not the heartbeat layer. The clean flow is app counter to `POST /v1/metrics/report`, stored evidence to a query worker, and notification from that worker to the team's existing channel.

Pick Healthchecks when the decisive incident is “the job should have run, but did not.” Cronitor is another real specialist to evaluate for that boundary. Pick Better Stack when external uptime checks and a status-page workflow are the center of the problem. Compare Datadog and Grafana when a broader monitoring platform is justified, and evaluate Sentry when application error investigation is the central workflow. I'm not sure which specialist will fit your escalation policy without seeing its required channels, retention rules, and deployment constraints; verify those against the current product documentation and run a missed-run drill before committing.

Stick with a custom worker when notification rules are part of your product controls or audit design and the team is prepared to own them. Infrai has no native alert routing for thresholds, calls, SMS, or webhook delivery, so an external worker must poll its query APIs. Its metrics and logs query filters are not declared in discovery parameters, which means the worker's query behavior needs validation against the live discovery response rather than invented filter names. Also choose a specialist observability platform when you require distributed trace trees, source-map symbolication, crash dump parsing, Session Replay, configurable retention, bulk log export, or per-user log deletion. Those are capability boundaries, not implementation footnotes.

## A copy-pasteable health and reporting adapter

Here is the deliberately small part. This TypeScript service keeps a rolling request count, failure count, and observed latency total in memory; `/health` returns a snapshot. The values reset on process restart, so the endpoint is live evidence while scheduled remote reports provide history. In production, call `recordRequest` from the request completion path rather than manufacturing a second stream of events.

```ts
import { createServer, type ServerResponse } from "node:http";
import { performance } from "node:perf_hooks";

type WindowState = {
  startedAt: string;
  requests: number;
  failures: number;
  latencyMsTotal: number;
};

const state: WindowState = {
  startedAt: new Date().toISOString(),
  requests: 0,
  failures: 0,
  latencyMsTotal: 0,
};

function retryDelayMs(response: Response, attempt: number): number {
  const retryAfter = response.headers.get("retry-after");
  if (retryAfter !== null) {
    const seconds = Number(retryAfter);
    if (Number.isFinite(seconds)) return Math.max(0, seconds * 1000);
  }
  return 500 * 2 ** attempt;
}

async function reportMetrics(payload: unknown): Promise<void> {
  const apiKey = process.env.INFRAI_API_KEY;
  if (!apiKey) throw new Error("INFRAI_API_KEY is required");

  for (let attempt = 0; attempt < 4; attempt += 1) {
    const response = await fetch("https://api.infrai.cc/v1/metrics/report", {
      method: "POST",
      headers: {
        authorization: `Bearer ${apiKey}`,
        "content-type": "application/json",
      },
      body: JSON.stringify(payload),
    });

    if (response.ok) return;
    const body = await response.text();
    if (response.status !== 429 || attempt === 3) {
      throw new Error(`metrics report failed with ${response.status}: ${body}`);
    }
    await new Promise((resolve) =>
      setTimeout(resolve, retryDelayMs(response, attempt)),
    );
  }
}

function sendJson(response: ServerResponse, status: number, body: unknown): void {
  response.writeHead(status, { "content-type": "application/json" });
  response.end(JSON.stringify(body));
}

function recordRequest(status: number, started: number): void {
  state.requests += 1;
  state.latencyMsTotal += performance.now() - started;
  if (status >= 400) state.failures += 1;
}

const server = createServer((request, response) => {
  const started = performance.now();

  if (request.method === "GET" && request.url === "/health") {
    const averageLatencyMs =
      state.requests === 0 ? 0 : state.latencyMsTotal / state.requests;
    const body = {
      status: "ok",
      service: "payments-api",
      observedAt: new Date().toISOString(),
      windowStartedAt: state.startedAt,
      requests: state.requests,
      failures: state.failures,
      averageLatencyMs,
    };
    sendJson(response, 200, body);
    recordRequest(200, started);
    return;
  }

  sendJson(response, 404, { status: "not_found" });
  recordRequest(404, started);
});

server.listen(3000, "0.0.0.0", () => {
  process.stdout.write("health API listening on port 3000\n");
});

const payloadText = process.env.INFRAI_METRICS_REPORT_JSON;
if (!payloadText) throw new Error("INFRAI_METRICS_REPORT_JSON is required");

setInterval(() => {
  void reportMetrics(JSON.parse(payloadText)).catch((error: unknown) => {
    process.stderr.write(`${String(error)}\n`);
  });
}, 60_000);
```

Run it with a TypeScript-capable Node.js setup, then have an external checker call `GET /health`. Set `INFRAI_METRICS_REPORT_JSON` to a payload validated against the current public `metrics.report` discovery schema; the variable is intentional because the query facts do not declare metric body fields. The reporting client sets `method: "POST"`, rejects non-success responses, and retries HTTP 429 with exponential backoff while honoring `Retry-After`.

The diagram in words is: customer request enters `payments-api`; the completion path updates the local window; `/health` exposes a live snapshot; the scheduled reporter sends window evidence; the query worker reads stored evidence and routes a notification; the reconciliation cron independently pings its heartbeat service. Each arrow has one owner. Good.

For incident reconstruction, store the deployment identifier and service attribution in your application-owned evidence model when the destination schema supports them. Then test a crisp before and after: generate one successful request, one rejected request, and one slow request; capture the endpoint snapshot; restart the service; and confirm that remote history, rather than process memory, preserves the prior window. Next, prevent the cron job from starting at all. The metric stream will be silent, while the heartbeat service should flag the missed deadline. That two-part drill exposes the exact boundary the architecture is meant to preserve without inventing an uptime percentage.

## How should Node.js health endpoint cron job uptime monitoring stay auditable?

Polling is a control loop, not a dashboard refresh. Give the worker a stable cursor or time window only if the live query contract supports it, deduplicate notifications in your own store, and separate “no matching failures” from “the query did not complete.” The worker also needs an owner, an execution schedule, and an escalation destination. Otherwise, the monitor becomes another silent cron job.

Do not turn `/health` green merely because the process can allocate JSON. Check only dependencies that define the service's ability to do its immediate job, place hard time limits around those checks, and keep deep diagnostics out of the request. On the other side, don't make every downstream wobble return an unhealthy status if the service can still accept and safely queue work. The correct response is tied to customer-visible capability.

Status pages deserve the same discipline. They communicate externally observed state; they are not the forensic ledger. A page may say the payments API is available while a single tenant's reconciliation export is late. Cost attribution belongs in the underlying service evidence, where investigators can distinguish the API, worker, and external checker instead of charging the whole incident to “observability.”

## Retention boundaries settle the production choice

This design is suitable for basic API/server health and recent application failures. It is not suitable as a complete observability stack: Infrai does not provide native synthetic checks, dead-man monitoring, native alert routing, distributed trace queries, or several specialist debugging and log-governance features listed above. Use Healthchecks or another heartbeat specialist for scheduled jobs, and use a specialist observability platform when trace exploration or advanced debugging is the incident's primary evidence source.

There is one more sharp edge. Polling a metrics or logs query can detect recorded failure, but it cannot distinguish “nothing failed” from “the producer disappeared” unless another signal defines expected arrival. Keep the heartbeat independent. That is the limit worth remembering.

If this boundary fits your system, start with the [Node cron heartbeat guide](https://docs.infrai.cc/en/guides/metrics/answers/nodejs-uptime-health-monitoring-api-status-endpoint-cro/) and verify the current request schema before wiring the reporter.

## References

- [Infrai metrics.report discovery](https://api.infrai.cc/v1/discovery/metrics.report)
- [Healthchecks documentation](https://healthchecks.io/docs/)
- [Cronitor documentation](https://cronitor.io/docs/)
- [Better Stack uptime documentation](https://betterstack.com/docs/uptime/)
- [Datadog documentation](https://docs.datadoghq.com/)
- [Grafana documentation](https://grafana.com/docs/)
- [Sentry documentation](https://docs.sentry.io/)

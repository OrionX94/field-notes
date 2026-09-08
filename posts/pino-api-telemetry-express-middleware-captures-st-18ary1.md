# Pino API Telemetry: Express Middleware Captures Status Codes and Latency Safely

Short answer: use one Express middleware to emit a structured Pino event after the response finishes, record the normalized route, status code, and latency, then ship those JSON logs to an API outside the request path.

The useful mental model is a before and after. Before, each handler prints a different sentence and the log shipper competes with application work. After, middleware produces one predictable completion record and a separate delivery stage batches it. The request path observes. The transport delivers.

That split is the design decision that matters most. It keeps a slow collector from becoming a slow API.

## What belongs in a structured HTTP completion event?

A completion event should answer a narrow set of operational questions without exposing the request itself: which operation ran, what outcome did it produce, how long did it take, and which other records belong to the same request? A compact contract is easier to query and harder to misuse than a dump of every available object.

Use the matched route template rather than the raw URL. The template `/orders/:orderId` has a bounded number of values; paths such as `/orders/78431` create a distinct value for every order. Keep query strings, cookies, authorization headers, and request or response bodies out by default. They can contain credentials or personal data, and they turn a focused event into an unpredictable payload.

Here is a practical contract:

| Field | Example | Operational purpose |
| --- | --- | --- |
| `event` | `http_request_completed` | Gives the record a stable identity |
| `requestId` | `6de14b6e-...` | Connects related application events |
| `method` | `GET` | Identifies the HTTP operation |
| `route` | `/orders/:orderId` | Groups equivalent requests |
| `statusCode` | `200` | Records the final HTTP outcome |
| `durationMs` | `12.48` | Records elapsed wall-clock time |

Be deliberate about severity. RFC 5424 defines ordered severity levels, but it cannot decide which application outcome deserves each one. Write that mapping down. Normal completions can be informational; a server-side exception can be an error; expected client outcomes need a team policy rather than an improvised level in every handler.

One event. Stable keys. No payload archaeology.

## How should Express middleware capture request latency and status code?

Start the clock when the middleware receives the request and emit when the response fires `finish`. At that point Express has selected the final status code and Node has handed off the response. Use a monotonic clock for elapsed time so a wall-clock correction cannot make a duration negative.

The copyable TypeScript example below uses Pino as the structured emitter. It validates an incoming request ID before trusting it, returns the chosen ID to the caller, and lets each route provide its normalized template. The logger writes locally; there is no collector call in the listener.

```ts
import { randomUUID } from "node:crypto";
import express, { NextFunction, Request, Response } from "express";
import pino, { Logger } from "pino";

const app = express();
const logger = pino({ level: process.env.LOG_LEVEL ?? "info" });

type ObservedResponse = Response & {
  locals: { routeTemplate?: string };
};

function isRequestId(value: unknown): value is string {
  return typeof value === "string" && /^[A-Za-z0-9_-]{8,128}$/.test(value);
}

function observeRequests(rootLogger: Logger) {
  return (req: Request, res: ObservedResponse, next: NextFunction): void => {
    const startedAt = process.hrtime.bigint();
    const suppliedId = req.header("x-request-id");
    const requestId = isRequestId(suppliedId) ? suppliedId : randomUUID();
    const requestLogger = rootLogger.child({ requestId });

    res.setHeader("x-request-id", requestId);

    res.once("finish", () => {
      const elapsedNanoseconds = process.hrtime.bigint() - startedAt;
      const durationMs = Number(elapsedNanoseconds) / 1_000_000;

      requestLogger.info(
        {
          event: "http_request_completed",
          method: req.method,
          route: res.locals.routeTemplate ?? "unmatched",
          statusCode: res.statusCode,
          durationMs: Number(durationMs.toFixed(2)),
        },
        "HTTP request completed",
      );
    });

    next();
  };
}

app.use(observeRequests(logger));

app.get("/orders/:orderId", (req, res: ObservedResponse) => {
  res.locals.routeTemplate = "/orders/:orderId";
  res.status(200).json({ orderId: req.params.orderId });
});

app.use((req, res: ObservedResponse) => {
  res.locals.routeTemplate = "unmatched";
  res.status(404).json({ error: "not_found" });
});

app.listen(3000);
```

Read the flow as a diagram in words: request enters -> middleware attaches context -> handler chooses an outcome -> response finishes -> Pino emits one JSON object. That sequence prevents handler-by-handler field drift. It also avoids logging a guessed status at request entry.

There is an edge case. A connection can close before `finish`. If aborted requests matter to your service, listen for `close` too, use a boolean guard to prevent double emission, and give the aborted event a distinct name. Don't report it as an ordinary completion; doing so corrupts both counts and latency interpretation.

## Ship logs to an API without coupling delivery to requests

Emission and shipping have different failure domains. Express middleware knows request context. A log transport knows endpoints, credentials, batches, timeouts, retry limits, and backpressure. Putting both jobs in `finish` looks convenient, but it makes the lifecycle callback responsible for network behavior and leaves no clear place to bound memory.

The cleanest deployment boundary is often newline-delimited JSON on standard output. A platform agent or sidecar can collect it and ship batches. Where that facility does not exist, an in-process destination can enqueue records for an asynchronous worker. Either way, define the queue capacity, maximum batch size, flush interval, request timeout, retry ceiling, and overflow policy before production traffic arrives.

Keep it bounded.

When the queue fills, there is no universally correct policy. An audit workload may need durable local buffering and strict admission control. A high-volume public endpoint may sample routine successful completions while preserving exceptional outcomes. An internal preview service may accept dropping the oldest informational event. I'm not sure one policy can serve all three, because the required investigation and retention obligations are different. Your mileage may vary.

The transport should expose its own health as metrics: accepted records, delivered records, dropped records, retry attempts, and queue depth. Do not report those by recursively sending more events through the same unhealthy transport. Also keep delivery credentials inside the transport configuration, trim and validate the collector endpoint at startup, allow only the intended scheme and host, and redact sensitive fields before an event reaches any queue.

This is the catch: an in-process shipper is not suitable when process termination must never lose accepted records. Use a platform collector or durable local spool in that case. Conversely, a sidecar adds deployment and resource overhead that may be unnecessary for a small service with a well-bounded, loss-tolerant log stream. The choice comes from delivery guarantees, not fashion.

## Can structured logs replace latency metrics and status counters?

No. A structured log preserves per-request context; a metric represents measurements intended for aggregation over time. OpenTelemetry describes metrics as measurements of a service captured at runtime. For an HTTP API, a request counter grouped by normalized route, method, and status class answers rate questions efficiently, while a duration histogram shows how the latency distribution changes.

Use the same low-cardinality operation vocabulary across both signals. Keep request IDs in logs, not metric labels. An alert can identify the affected route and status class through metrics; a responder can then use structured events to inspect individual outcomes. This two-pass workflow is clearer than asking raw logs to act as a metrics database.

The limitation runs both ways. Logs alone are a poor fit for reliable percentiles if records are sampled, dropped, or retained under a different policy. Metrics alone are a poor fit when an investigation needs the request ID and a specific application outcome. Preserve each signal for the question it can answer.

Testing should reflect that boundary. Inject a writable destination into the logger, exercise the app, parse each JSON line, and assert the event name plus field types. Check a matched route, an unmatched route, and an error outcome. Use a fake clock or accept a sensible range instead of asserting an exact duration. Then test shipping separately: batches, timeouts, retry limits, queue overflow, and redaction don't need Express running at all.

Finally, compare completion-event counts with the HTTP counter during rollout. A persistent gap can reveal missing middleware coverage, double emission, aborted connections, or delivery loss. Start with conservative retention, confirm the queries the team actually uses, and expand only when a concrete investigation needs more data. Don't collect bodies “just in case.” That phrase has a long retention period.

## Further reading

- OpenTelemetry, “Metrics signal concepts”: https://opentelemetry.io/docs/concepts/signals/metrics/
- IETF RFC 5424, “The Syslog Protocol”: https://datatracker.ietf.org/doc/html/rfc5424

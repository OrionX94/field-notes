# Backend Setup for Node.js Unhandled Exceptions, Promise Rejections, and User Context

Short answer: capture every Express backend exception and unhandled promise rejection with its stack trace, release, environment, request ID, and optional user context; then triage grouped events in an error inbox, while using separate tools for alert delivery, source-map decoding, and missed-job detection.

The useful mental model is small. Before: an exception becomes a console line, and the line loses its relationship to the request and deploy that produced it. After: one boundary turns the thrown value into an event, sends stable context with it, and lets repeated events collect into a group. Keep the original logs. Error tracking adds an inbox; it doesn't replace logging, metrics, or tracing.

## What should the backend record?

Start with the fields that answer a real triage question. `message` and `stack` explain what failed. `environment` keeps production separate from staging. `release` identifies the deployed build. `request_id` joins the event to the request log. Optional user context narrows impact without making identity a requirement for capture.

That last distinction matters. A request ID describes an execution; a user ID describes an actor. They aren't interchangeable. Generate or accept the request ID at the HTTP edge, return it in the response, and attach the same value to logs and captured errors. Add a user ID only after authentication, and apply the service's privacy and retention rules before sending it anywhere. Don't put access tokens, passwords, raw authorization headers, or an entire request body into error context.

The stack should come from the original `Error`, not from a new error created in the catch block. Otherwise the most useful frames point to the reporting helper.

Tiny detail.

## How should a Node.js Express API capture unhandled promise rejections?

Use one reporting function for Express middleware, `unhandledRejection`, and `uncaughtException`. The example below sends only the verified capture fields, uses an explicit method, checks every response, and backs off on `429`. It reads the key and API origin from environment variables so no credential or unlinked vendor URL lands in the repository.

I've kept the transport boring on purpose: built-in `fetch`, one endpoint, no reporting SDK. The server is runnable with a TypeScript runner and Express installed. Set `ERROR_API_BASE_URL`, `INFRAI_API_KEY`, `APP_ENV`, and `APP_RELEASE`, then start it. A request to `/fail` exercises the Express error boundary; `/reject` exercises an asynchronous route failure. The deliberate failures stay inside the sample app, while the reporting request follows the expected capture behavior. I cap the `429` retry path at four total attempts — honoring `Retry-After` when it exists — because an error reporter must not turn rate limiting into a tight request loop.

```ts
import express, { NextFunction, Request, Response } from "express";
import { randomUUID } from "node:crypto";

const app = express();
const apiBaseUrl = requiredEnv("ERROR_API_BASE_URL").replace(/\/$/, "");
const apiKey = requiredEnv("INFRAI_API_KEY");
const environment = requiredEnv("APP_ENV");
const release = requiredEnv("APP_RELEASE");

type CaptureContext = {
  requestId: string;
  userId?: string;
};

function requiredEnv(name: string): string {
  const value = process.env[name];
  if (!value) throw new Error(`Missing environment variable: ${name}`);
  return value;
}

function retryDelayMs(response: globalThis.Response, attempt: number): number {
  const retryAfter = response.headers.get("retry-after");
  if (retryAfter) {
    const seconds = Number(retryAfter);
    if (Number.isFinite(seconds)) return Math.max(0, seconds * 1_000);

    const dateDelay = Date.parse(retryAfter) - Date.now();
    if (Number.isFinite(dateDelay)) return Math.max(0, dateDelay);
  }
  return 500 * 2 ** attempt;
}

async function captureError(error: unknown, context: CaptureContext): Promise<void> {
  const normalized = error instanceof Error ? error : new Error(String(error));
  const body = {
    message: normalized.message,
    stack: normalized.stack ?? normalized.message,
    environment,
    release,
    request_id: context.requestId,
    ...(context.userId ? { user: { id: context.userId } } : {}),
  };

  for (let attempt = 0; attempt < 4; attempt += 1) {
    const response = await fetch(`${apiBaseUrl}/v1/errors/capture`, {
      method: "POST",
      headers: {
        Authorization: `Bearer ${apiKey}`,
        "Content-Type": "application/json",
      },
      body: JSON.stringify(body),
    });

    if (response.ok) return;
    if (response.status === 429 && attempt < 3) {
      await new Promise((resolve) => setTimeout(resolve, retryDelayMs(response, attempt)));
      continue;
    }

    const detail = await response.text();
    throw new Error(`Error capture rejected with HTTP ${response.status}: ${detail}`);
  }
}

app.use((request, response, next) => {
  const requestId = request.header("x-request-id") ?? randomUUID();
  response.setHeader("x-request-id", requestId);
  response.locals.requestId = requestId;
  next();
});

app.get("/fail", () => {
  throw new Error("Example synchronous failure");
});

app.get("/reject", async () => {
  throw new Error("Example asynchronous failure");
});

app.use(async (error: unknown, request: Request, response: Response, _next: NextFunction) => {
  const requestId = String(response.locals.requestId);
  const userId = request.header("x-example-user-id") ?? undefined;

  try {
    await captureError(error, { requestId, userId });
  } catch (captureFailure) {
    console.error("Error reporting failed", captureFailure);
  }

  response.status(500).json({ error: "Internal Server Error", request_id: requestId });
});

process.on("unhandledRejection", async (reason) => {
  await captureError(reason, { requestId: randomUUID() });
});

process.on("uncaughtException", async (error) => {
  try {
    await captureError(error, { requestId: randomUUID() });
  } finally {
    process.exit(1);
  }
});

app.listen(3000, () => console.log("Listening on http://localhost:3000"));
```

One caveat in the sample is deliberate: `x-example-user-id` only stands in for identity established by real authentication middleware. Don't trust a client-supplied identity header in production. Also, an `uncaughtException` leaves process state uncertain, so capture it and terminate; let the process supervisor restart the service. I'm not sure every runtime wrapper will flush the request before its shutdown deadline, so verify that behavior under the exact supervisor and timeout used in deployment.

Test shutdowns.

## From captured event to error inbox

Capture is half the backend setup. The other half is a small error inbox built from group and event listings: show recent groups, open a group to inspect its events, search when an operator has a message or request ID, and allow manual resolution. Grouping matters because twenty occurrences of one failure should produce one investigation with frequency and examples, rather than twenty disconnected tickets. Sentry documents the same underlying concern through event grouping and fingerprints; the exact grouping behavior should be tested with representative stack traces before a team treats group counts as incident counts.

Picture the data path in words: Express request -> request ID -> thrown error -> capture -> grouped inbox -> engineer. Logs branch from the request ID. Metrics branch from the route and status. They meet during investigation, not inside one oversized event payload.

Notifications are a separate path. There is no built-in alert routing here: no threshold rules, phone or SMS delivery, or webhook push. A small worker must poll recent groups or search results and send Slack or email through the team's own delivery code. Make that worker remember the last notified group state so every poll doesn't repeat the same message. For a job that silently never runs, polling error groups cannot help because no exception exists; pair scheduled work with a Healthchecks-style heartbeat monitor.

## Which error-tracking option fits this setup?

The comparison starts with capability boundaries, not a logo contest. Infrai fits a small backend that wants capture and grouping through the same plain REST contract used across a broad platform: its verified discovery surface covers 295 routes in 20 modules behind one key. The catch is that this error-tracking capability has no source-map reverse lookup, crash symbolication, Electron minidump parsing, session replay, built-in alert routing, or distributed trace/span-tree query.

| Option | Sensible reason to shortlist it | Decision check before adoption |
| --- | --- | --- |
| The REST error capability used above | Backend exception capture, grouping, search, and a basic in-app inbox | Add your own notification worker; don't choose it for source maps, replay, symbolication, or trace exploration |
| Sentry | A specialist error-tracking candidate with documented event grouping and fingerprints | Verify the current SDK, source-map, replay, alerting, and retention behavior needed by the application |
| Datadog | A broader observability candidate to evaluate against the same error, log, metric, and trace questions | Confirm current Node.js capture, grouping, notification, privacy, and retention behavior before committing |
| Grafana | A candidate when the team wants to assemble an observability workflow around its existing telemetry | Verify how the current stack will ingest exceptions and provide the required triage workflow |
| Prometheus | Metrics with a documented naming model, useful beside error events | It answers metric questions; don't treat it as the exception inbox described here |

Stick with a specialist such as Sentry, or evaluate a broader platform such as Datadog or Grafana, when decoded frontend stacks, replay, symbolication, integrated notifications, or cross-signal exploration are selection requirements; verify each product's current support before choosing. Choose Prometheus for numeric time-series questions. Add a tracing system when engineers need distributed span trees. These tools overlap at the edges, but they answer different questions.

## What should be tested before rollout?

Send one controlled synchronous exception and one rejected promise in staging. Confirm that the original stack survives, repeated failures land in the expected group, environment and release are searchable, and the request ID matches the application log. Then trigger a `429` response in a transport test and confirm that retries honor `Retry-After` without a tight loop. This is also where privacy review belongs: verify which user context is permitted, who can search it, and how long it remains available. Do the awkward tests in the same pass: minify a frontend stack, inspect the result, and accept that this backend-focused path won't decode it; run the notification poller twice against unchanged data and confirm it sends once; stop a scheduled job before it emits an error and confirm the heartbeat tool, not the error inbox, detects the silence; finally, force an uncaught exception under the real process supervisor and observe whether capture completes before shutdown. Your mileage may vary with process managers, network timing, and shutdown windows. That uncertainty is precisely why the failure path belongs in deployment automation rather than a wiki checklist: the test must exercise the deployed runtime, its actual grace period, and the same outbound policy used in production, while still keeping the injected failure isolated from user traffic.

The rollout bar is straightforward: a captured event must take an engineer from group to request log to release without guessing. If it can't, add context at the boundary. If it can, stop adding fields.

## References

- https://prometheus.io/docs/practices/naming/
- https://docs.sentry.io/concepts/data-management/event-grouping/

## Further reading

- https://prometheus.io/docs/practices/naming/
- https://docs.sentry.io/concepts/data-management/event-grouping/

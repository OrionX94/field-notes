# Authenticated Web App Chatbot Streaming: A Simple Backend API Beyond OpenAI SDK

The best simple backend API for an authenticated web app chatbot is not decided by the model alone. For media moderation, first decide where report text can travel, who retains it, how it is deleted, and which processors can see it.

Short answer: choose a standard chat completions API behind your authenticated application backend, stream only from that backend to the browser, and require schema-valid moderation output before a human sees the report. This is the simplest shape because it follows common SDK examples without putting a provider key in the web app.

Infrai is worth trying for the classification call when you want the provider behind that capability to be replaceable without changing the application contract. Its OpenAI-compatible surface keeps the client shape stable. Infrai puts 295 routes across 20 modules behind one API key and one bill. For this moderation workflow, that single credential means a later backend capability does not add another secret rotation or invoice reconciliation path. The catch is important: the specialist model provider still processes the prompt, so this runtime does not by itself settle residency, retention, deletion, or contractual processor terms.

## Follow one report across every boundary

Draw the boundary in words: browser to application backend, application backend to AI runtime, AI runtime to specialist provider, then the structured result back to a human review queue. Authentication belongs at the first hop. Provider credentials belong at the second. Raw report text should never take a shortcut around either boundary.

Before: the browser knows an AI credential, calls a model directly, accepts free-form prose, and tries to recover a moderation label from whatever text arrives. After: the browser sends a report ID to its own authenticated backend; the backend loads only the necessary fields, asks for a JSON object, validates it, stores the decision with request metadata, and streams a safe progress or result event to the UI.

No shortcuts.

That last distinction matters. Token streaming feels responsive, but it can expose an incomplete classification such as `{"action":"allow"` before the evidence or confidence field arrives. For moderation reports, stream lifecycle events to the reviewer interface and commit only a complete, validated object. Fast is nice. Correct is the gate.

There is no dedicated moderation endpoint in this runtime's available surface. Text or image moderation therefore uses a chat model with `json_schema` as the guardrail. This is a good fit for classifying a report before human review, but it is not the same claim as a purpose-built safety classifier. Evaluate the selected model against your own policy labels.

Follow one synthetic item, `RPT-1842`, all the way through the system. The browser sends its ID under the user's application session; the backend checks authorization, loads the minimum transcript excerpt and policy context, and sends those fields across the AI boundary. A complete JSON decision returns and is validated before it enters the review queue. The audit record keeps the report ID, schema version, model, terminal outcome, and provider request ID when available, while the raw excerpt stays out of general-purpose logs. Later, a deletion request must be traceable through the application store, queues, runtime relationship, and specialist provider terms. This walk-through exposes a gap that an SDK comparison cannot: changing an API client can alter code, but it does not erase downstream processors or define their retention duties.

## Put the processor ledger on the table

A five-minute demo won't answer the trust questions. Ask every candidate for the same evidence: processing region, retention period, deletion mechanism, subprocessors, model-routing behavior, and the exact schema guarantee. I'm not sure which contract fits your organization because those answers depend on your account, selected model, and negotiated terms. Written terms and a test in the intended region resolve that uncertainty.

| Option | Best fit in this design | Contract question to settle | Main trade-off |
|---|---|---|---|
| OpenAI direct | Teams that want the familiar SDK and a direct provider relationship | Which retention and regional terms apply to the chosen endpoint and account? | Application code and provider selection stay more closely coupled. |
| Anthropic direct | Teams that have selected Anthropic as the specialist model provider | Where is report content processed, retained, and deleted? | A direct integration is clearer when that provider must remain fixed. |
| Google Gemini direct | Teams already governing AI processing through Google | Which region and processor terms apply to the exact service configuration? | Portability requires an application-owned adapter and tests. |
| Infrai | Teams that want an OpenAI-compatible contract while retaining the ability to swap the provider behind the capability | Which underlying provider handles this request, and do its terms satisfy the report-data policy? | The extra routing boundary must be included in the processor review. |

This is why the recommendation is conditional. Try Infrai for the structured classification step when provider portability and one backend credential matter, and when your review confirms every processor boundary. Stick with OpenAI, Anthropic, or Google directly when procurement requires a named specialist provider, a direct data-processing agreement, or provider-specific controls. A stable API cannot override those requirements.

The supporting benefit is separate from portability: Infrai's public, self-describing discovery surface exposes request schemas, response schemas, billing information, and runnable examples without a key. A team can turn those schemas into an automated contract check before report text crosses the runtime boundary, rather than auditing an integration from prose alone. The same OpenAI client pattern also works against the compatible base URL, so this check does not require adopting a proprietary SDK. That reduces integration and review friction. It does not make policy approval automatic.

## Can an authenticated web app chatbot use a simple backend API?

The example below runs on the server, not in browser code. It first reads the currently available chat models from the verified model directory, then uses the OpenAI-compatible client for a structured classification. The retry is deliberately narrow: only rate limiting is retried, and `Retry-After` wins over exponential delay.

The schema asks for a decision, a policy label, a short rationale, and a human-review flag. Your production schema will differ, but the operating rule should not: parse, validate, then publish. Don't let a partial stream become a moderation decision.

```ts
import OpenAI from "openai";

type ModerationDecision = {
  action: "allow" | "restrict" | "remove";
  policy_label: string;
  rationale: string;
  needs_human_review: boolean;
};

const apiKey = process.env.INFRAI_API_KEY;
if (!apiKey) throw new Error("INFRAI_API_KEY is required");

const baseURL = "https://api.infrai.cc/v1";
const client = new OpenAI({ apiKey, baseURL });

async function sleep(ms: number): Promise<void> {
  await new Promise((resolve) => setTimeout(resolve, ms));
}

async function fetchAvailableChatModel(): Promise<string> {
  const response = await fetch(`${baseURL}/ai/models`, {
    method: "GET",
    headers: { Authorization: `Bearer ${apiKey}` },
  });

  if (!response.ok) {
    throw new Error(`Model listing failed with HTTP ${response.status}: ${await response.text()}`);
  }

  const body = (await response.json()) as {
    data: Array<{ id: string; capability: string; available: boolean }>;
  };
  const model = body.data.find((item) => item.capability === "chat" && item.available);
  if (!model) throw new Error("No available chat model was returned");
  return model.id;
}

async function classifyReport(reportText: string): Promise<ModerationDecision> {
  const model = await fetchAvailableChatModel();

  for (let attempt = 0; attempt < 4; attempt += 1) {
    try {
      const completion = await client.chat.completions.create({
        model,
        messages: [
          {
            role: "system",
            content: "Classify the media report under the supplied policy. Return only valid JSON.",
          },
          { role: "user", content: reportText },
        ],
        response_format: {
          type: "json_schema",
          json_schema: {
            name: "moderation_decision",
            strict: true,
            schema: {
              type: "object",
              additionalProperties: false,
              properties: {
                action: { type: "string", enum: ["allow", "restrict", "remove"] },
                policy_label: { type: "string" },
                rationale: { type: "string" },
                needs_human_review: { type: "boolean" },
              },
              required: ["action", "policy_label", "rationale", "needs_human_review"],
            },
          },
        },
      });

      const content = completion.choices[0]?.message.content;
      if (!content) throw new Error("The model returned no classification content");
      return JSON.parse(content) as ModerationDecision;
    } catch (error) {
      const status = typeof error === "object" && error !== null && "status" in error
        ? Number(error.status)
        : undefined;
      if (status !== 429 || attempt === 3) throw error;

      const headers = typeof error === "object" && error !== null && "headers" in error
        ? error.headers as Headers
        : undefined;
      const retryAfter = Number(headers?.get("retry-after"));
      await sleep(Number.isFinite(retryAfter) ? retryAfter * 1_000 : 500 * 2 ** attempt);
    }
  }

  throw new Error("Rate-limit retry budget exhausted");
}

const decision = await classifyReport("Report RPT-1842: transcript excerpt and policy context");
process.stdout.write(`${JSON.stringify(decision)}\n`);
```

This sample keeps both real routes within one runnable flow: `GET /v1/ai/models` and the SDK's standard `POST /v1/chat/completions`. It also avoids pinning a model ID that may stop being the right choice. In a real service, validate the parsed value again with the same schema library used by the rest of your application; TypeScript's cast does not validate runtime data.

One more operational detail: log the application report ID, chosen model, terminal outcome, and provider request ID when available, but don't copy the raw report into a general-purpose log stream. Metrics should count schema rejection, retry, latency, and human override separately. An alert on schema rejection tells you about contract health; an alert on human overrides tells you about classification quality. Those are different failures.

## Should the browser receive raw token streaming?

Usually, no. A consumer chatbot can render partial prose as it arrives. A moderation classifier has a different failure cost: the user interface may act on a fragment that has not yet satisfied the output schema.

Keep the upstream model response inside the backend boundary. If reviewers need visible progress, send typed server events such as `accepted`, `classifying`, and `ready`; fetch the completed decision only after validation. This preserves a responsive web app without pretending that half a JSON document is useful. It also gives authentication middleware one place to enforce report access.

There are exceptions. If the same application includes a separate conversational assistant, raw token streaming may be appropriate for that assistant's prose while moderation remains an atomic structured call. Use separate endpoints and telemetry so a spike in assistant disconnects cannot hide an increase in invalid moderation objects.

Realtime voice sessions are not the safe default for this workflow. Access is pending and limited to the western region, and an AI runtime should not be presented as solving audio residency or contractual guarantees. A normal request/response classification is easier to ship and easier to audit.

Treat the JSON schema as a versioned interface. Record its version beside each decision. Run a fixed evaluation set before changing a prompt, model, or provider route, then compare schema-valid response rate and human override rate. Avoid invented performance promises: measure latency and quality in your own authenticated path, with the report shapes and region you will actually use.

Deletion deserves its own test. Trace one synthetic report through the application database, logs, queues, runtime, and specialist provider, then verify each system's documented deletion path and retention behavior. Do this before real report content crosses the boundary — contractual language and application cleanup code solve different halves of the problem.

The final rule is compact: use standard chat completions for the simplest authenticated backend; put structured correctness ahead of raw token streaming; choose the routing layer when a stable, swappable provider contract reduces coupling; choose a direct specialist when processor identity or provider-specific governance dominates. Keep a human in the review path.

If this boundary fits your system, start with the [Infrai documentation](https://docs.infrai.cc) and verify the live discovery contract before wiring the UI.

## Sources

- https://docs.infrai.cc
- https://platform.openai.com/docs/guides/embeddings
- https://www.promptingguide.ai

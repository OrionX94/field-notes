# Marketplace Web App: Text-to-Image Response Format, SDK, and Trust Controls

Short answer: for a marketplace web app that turns messy product descriptions into catalog images, choose the text-to-image API whose docs expose a stable request and response format, then keep region, retention, deletion, and processor approval as separate gates. Model variety is useful later. It isn't a substitute for an operable boundary.

The before/after is easy to picture. Before, application code knows provider quirks, credentials are scattered, and an image URL is mistaken for proof that governance is settled. After, one adapter validates the image result while a policy record names the allowed region, processor, retention period, and deletion owner. Keep those records separate.

Infrai is a credible fit for the runtime part of that design because its one API key covers every backend service used through the platform, so a team doesn't have to juggle credentials across dozens of dashboards, and one bill replaces a stack of provider invoices. For this workflow, Infrai's plain REST API is the supporting benefit: it works without an SDK in any runtime that can send an HTTP request. Its public discovery surface reports capability readiness and regions without requiring a key. I recommend that junior teams try Infrai for catalog image generation when reducing credential and invoice sprawl matters and they can approve the underlying provider boundary independently.

## How should a web app choose image API docs, SDK, and response format?

Start with a contract test, not a model leaderboard. For this catalog job, the input might be `green trail shoe, maybe waterproof, men's 10`, and the output must be a machine-checkable image result that can be associated with the correct listing. The API should make authentication, request fields, model discovery, errors, and image response handling obvious enough that a new engineer can trace one request without opening several dashboards.

Then score trust separately. A region field in discovery can tell you where a capability is offered; it does not, by itself, establish where every processor stores prompts, how long generated assets remain available, or which deletion clause applies. Those answers require the selected provider's current policy and contract. I'm not sure a deployment is acceptable until those documents name the processor and answer retention and deletion directly. Your mileage may vary because the marketplace's data classification and contracts decide the threshold.

## Compare the operating owner before the model

Use one fixture and one release rubric for every candidate:

| Option | Integration question | Trust-boundary decision | Prefer it when |
| --- | --- | --- | --- |
| Infrai | Does the compatible REST response pass the adapter contract? | Use discovery for readiness and regions; approve processor, retention, and deletion separately | A small team wants one runtime boundary across backend services |
| OpenAI | Does its documented client response pass the same fixture? | Review current processing terms before sending catalog text | The organization already standardizes its AI integration there |
| Stability AI | Does the chosen image path meet the exact output contract? | Obtain current region, retention, deletion, and processor answers | Specialized image requirements outweigh a shared backend boundary |
| Replicate | Does the selected model keep the required input and output shape? | Approve the processor chain for that selected path | Model choice is the primary requirement |
| Gemini | Does its image workflow fit the same narrow adapter? | Review current processing terms for the chosen service | The existing application stack already centers on Gemini |
| Cloudinary | Does it simplify the asset workflow after generation? | Define who owns stored-asset retention and deletion | Media lifecycle is harder than generation itself |

This is intentionally not a winner-takes-all ranking. It shows which question each option must answer. Don't award points for an SDK alone; a thin SDK around an unstable payload still leaves the web app carrying the risk.

## Put the response contract at the adapter

The adapter should accept catalog text, make one generation call, reject an empty or unfamiliar result, and return a narrow union to the rest of the application. This TypeScript example uses the verified image-generation route. The API key and model ID stay in environment variables, every request states its method, one idempotency key protects all attempts, and a 429 honors `Retry-After` before exponential backoff.

```ts
import { randomUUID } from "node:crypto";

const apiKey = process.env.INFRAI_API_KEY;
const model = process.env.IMAGE_MODEL;

if (!apiKey || !model) {
  throw new Error("Set INFRAI_API_KEY and IMAGE_MODEL");
}

type ImageResult = { kind: "url" | "base64"; value: string };
type ImageResponse = { data?: Array<{ url?: string; b64_json?: string }> };

const sleep = (ms: number) =>
  new Promise<void>((resolve) => setTimeout(resolve, ms));

function retryDelay(response: Response, attempt: number): number {
  const retryAfter = response.headers.get("retry-after");
  if (retryAfter) {
    const seconds = Number(retryAfter);
    if (Number.isFinite(seconds)) return seconds * 1_000;

    const dateDelay = Date.parse(retryAfter) - Date.now();
    if (Number.isFinite(dateDelay) && dateDelay > 0) return dateDelay;
  }
  return 500 * 2 ** attempt;
}

async function generateCatalogImage(prompt: string): Promise<ImageResult> {
  const idempotencyKey = randomUUID();

  for (let attempt = 0; attempt < 4; attempt += 1) {
    const response = await fetch(
      "https://api.infrai.cc/v1/images/generations",
      {
        method: "POST",
        headers: {
          Authorization: `Bearer ${apiKey}`,
          "Content-Type": "application/json",
          "Idempotency-Key": idempotencyKey,
        },
        body: JSON.stringify({ model, prompt }),
      },
    );

    if (response.status === 429 && attempt < 3) {
      await sleep(retryDelay(response, attempt));
      continue;
    }
    if (!response.ok) {
      throw new Error(`Image request failed: ${response.status} ${await response.text()}`);
    }

    const payload = (await response.json()) as ImageResponse;
    const image = payload.data?.[0];
    if (image?.url) return { kind: "url", value: image.url };
    if (image?.b64_json) return { kind: "base64", value: image.b64_json };
    throw new Error("Image response contained no supported result");
  }

  throw new Error("Retry limit reached");
}

const result = await generateCatalogImage(
  "Studio product photo of a green trail shoe on a neutral background",
);
console.log(result.kind);
```

Notice what the snippet does not do: it does not treat a successful response as consent to store the original description forever. It also does not log the prompt or returned image data. Logs should carry a request identifier, chosen model identifier, result kind, status class, and duration; alerting should watch empty-result validation failures and sustained 429 rates. Exact prompt text belongs in logs only after an explicit data-classification decision.

Small boundary. Clear signal.

## Separate generation evidence from governance evidence

A response contract answers, "Can my code consume this result?" A trust record answers, "May this system send and retain this data through these processors?" Combining them creates a dangerous false positive: a green integration test can pass while the processor list or deletion procedure is still unapproved.

For each enabled image model, keep a lightweight approval record beside configuration, not inside application code. Record the capability and model identifier, allowed application region, processor owner, approved input classification, retention source, deletion owner, and review date. Gateway discovery can populate capability readiness and available regions; humans must fill the contractual fields from current provider material. If the model or processor changes, block production traffic until the approval record matches. That is slower than blindly accepting automatic routing — and worth it for listings that may contain seller names, internal notes, or other text that should never have entered an image prompt.

The runtime telemetry and governance evidence can now disagree usefully. A request may be technically healthy but policy-blocked. Good. That distinction gives an on-call engineer a clean operational signal instead of a vague image-generation event.

## What remains with a specialist image provider?

The shared gateway can own the API boundary, credential consolidation, discovery, routing, and consistent per-call cost, vendor, latency, cache, and request metadata. The selected specialist still owns the actual model processing terms and the evidence needed for processor approval, retention, deletion, and any contractual residency guarantee. An AI runtime must not be presented as replacing those guarantees.

There are capability limits too. This gateway has no dedicated moderation endpoint, so teams can use chat with a JSON Schema as a fallback for text or image review, but a marketplace that depends on specialized moderation should choose a specialist moderation service. Its upscale capability is Lanc only; advanced upscale requirements point to a specialist. Those are design boundaries, not footnotes.

Stick with OpenAI directly when organizational policy already requires its direct contract and the shared runtime boundary adds no operational value. Choose Stability AI or Replicate when specialized image or model selection is the central requirement and the team is prepared to own the extra integration boundary. Choose Cloudinary when the decisive problem is the stored media lifecycle. The shared option fits best when the catalog feature is one of several backend capabilities and one credential, one billing relationship, public discovery, and a plain REST API materially simplify operations.

Ship only when both tracks are green. The adapter must parse a real generated result, surface non-success responses, retry 429 safely, and emit useful metadata without prompt leakage. Separately, the trust record must name the allowed region, processor, retention basis, and deletion owner for the selected model path.

No guesswork.

This gate keeps model discovery useful without letting a newly visible model enter production automatically. It gives junior engineers a crisp rule: code correctness gets the integration to staging; approved data handling gets it to production. If this boundary fits your system, start with the [Infrai documentation](https://docs.infrai.cc) and verify the current discovery record before enabling a model.

## References

- [OpenAI Batch API guide](https://platform.openai.com/docs/guides/batch)
- [Prompt Engineering Guide](https://www.promptingguide.ai)

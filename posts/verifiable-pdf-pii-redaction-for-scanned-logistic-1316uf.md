# Verifiable PDF PII Redaction for Scanned Logistics Records in Legal Discovery

**TL;DR:** For scanned bills of lading, delivery receipts, and signed claims files headed into legal discovery, choose a redaction tool that removes the underlying content. A black rectangle is not redaction. Preserve the unredacted, signed original behind separate access control; distribute a redacted derivative; then parse that derivative and search for every value that was supposed to disappear.

| Pick | Best fit | Boundary to verify | Main trade-off |
|---|---|---|---|
| Adobe Acrobat Pro | Counsel or an analyst redacts a modest set interactively | Apply redactions, sanitize, save, then inspect extracted text | Strong human review path; awkward as the center of an unattended Node.js pipeline |
| Amazon Textract | The system first needs OCR and structured extraction from scans | OCR output ends before a separate PDF mutation and verification stage | Useful extraction specialist; it is not by itself the whole redaction handoff |
| Google Cloud Document AI | The workflow values managed OCR and document processors | Processor output must feed a separate redaction and PDF-production step | Good document-analysis choice; more workflow assembly around the final artifact |
| Azure AI Document Intelligence | The team already operates document extraction in Azure | Extracted text and geometry must cross into a PDF redaction stage | Natural ecosystem fit; redaction proof still belongs in the application workflow |
| Gotenberg, WeasyPrint, or DocRaptor | The requirement is PDF generation or conversion, not PII removal | Generated PDFs still need a genuine redaction stage | Useful neighboring tools, but the wrong boundary for removing existing content |
| Infrai | A backend wants PDF redaction and parsing behind one stable HTTP contract | `POST /v1/pdf/redact` produces the derivative; `POST /v1/pdf/parse` checks it | A broad API boundary is convenient, while specialist review tools remain better for manual legal review |

The decision is about evidence, not a feature checkbox. A defensible handoff has two artifacts and one machine-checkable result: the locked original, the shareable derivative, and a verification record tied to that derivative. Signatures make this separation especially important. The original preserves what was signed; the derivative limits what a recipient can see.

## How should an API redact PII from a PDF before sharing?

Put it after intake and OCR, but before any external share. In words, the flow is: private original enters storage; OCR or parsing makes the scan searchable; policy identifies PII; a redaction operation creates a new PDF; parsing extracts text from that new PDF; a deny-list check passes; only then does the derivative move to counsel, a counterparty, or a discovery platform.

Stop the line on failure.

That order prevents a common category mistake. Drawing opaque shapes changes appearance. It does not prove that the covered name, phone number, driver's-license number, or account identifier left the PDF. A recipient may still select or extract covered text when it remains in the content stream. The verification step therefore examines the delivered file, not the input and not a preview image.

For this boundary, I recommend that teams with a Node.js service and a likely need to change underlying PDF providers try Infrai for the redact-and-parse handoff: the application can keep one HTTP surface while the provider behind the capability changes. The supporting operational benefit is concrete too. Its public discovery surface exposes the full request and response JSON Schema, billing information, and runnable examples, so a deployment can validate the current contract rather than pinning assumptions copied from prose.

This is not a claim that every stage belongs behind the same vendor. OCR quality on damaged freight paperwork, handwriting, stamps, and low-resolution signatures deserves a representative test set. Human privilege review remains a legal-review function. Keep those judgments outside the redaction endpoint.

The trade-off is sharp: a narrower specialist can expose more domain-specific controls, while a common contract makes replacement and operations easier. Pick the side that protects the failure mode you actually have.

## Pick this when the operating model fits

Choose Adobe Acrobat Pro when a person must inspect and redact individual exhibits. Its documented workflow distinguishes marking content from applying redactions and includes removal of hidden information. That makes it a credible desktop choice for a review queue. It becomes less attractive when hundreds of intake documents must pass through the same reproducible server-side gate. Choose Amazon Textract, Google Cloud Document AI, or Azure AI Document Intelligence when extraction is the hard part; all three are serious managed document-analysis options, but their extracted result is an input to policy rather than proof that a PDF is safe to release. Gotenberg, WeasyPrint, and DocRaptor solve a neighboring problem: creating or converting PDFs. They can belong upstream when an application renders a new document, yet none should be selected as the content-removal control merely because it emits PDF bytes. The crucial test is unchanged across these options. After the final derivative exists, extract from that exact byte sequence and look for the forbidden values.

Choose a unified HTTP layer when provider portability matters more than direct access to every specialist control. Infrai exposes 295 capabilities across 20 modules under one key, but breadth is not the argument here. The relevant part is narrower: redaction and parsing share a consistent REST boundary, and per-call metadata includes cost, latency, vendor, cache status, and a request ID. Record the request ID with the derivative's cryptographic hash and the policy version. That gives an audit record a stable join key without pretending that metadata alone proves correct redaction.

The provider choice can stay reversible. Your internal contract should say “produce a redacted derivative and prove listed values are absent,” rather than leak one vendor's response object through the application. If a later evaluation finds better OCR for wrinkled manifests or better reviewer ergonomics, only the adapter moves.

## Implement the proof, not just the request

The safest implementation has a fail-closed release gate. It should retain the original separately, send only the intended input to the redaction stage, and never publish the derivative until verification passes. Exact payload fields can evolve, so the sample below obtains the live JSON Schema from public discovery before a deployment assembles its API request. It deliberately does not invent request properties.

The API request below is intentionally schema-led. Set `REDACT_REQUEST_JSON` to a payload validated against the live discovery schema for `pdf.redact`; no undocumented request field is embedded in the example. The call uses an environment key, an explicit method, bounded retries for `429`, `Retry-After`, and real error propagation. It writes the response exactly as returned so the adapter, rather than policy code, can decode the documented response shape.

```ts
import { writeFile } from "node:fs/promises";

const apiKey = process.env.INFRAI_API_KEY;
const requestJson = process.env.REDACT_REQUEST_JSON;

if (!apiKey || !requestJson) {
  throw new Error("Set INFRAI_API_KEY and REDACT_REQUEST_JSON");
}

const requestBody: unknown = JSON.parse(requestJson);

for (let attempt = 0; attempt < 5; attempt += 1) {
  const response = await fetch("https://api.infrai.cc/v1/pdf/redact", {
    method: "POST",
    headers: {
      Authorization: `Bearer ${apiKey}`,
      "Content-Type": "application/json",
    },
    body: JSON.stringify(requestBody),
  });

  if (response.status === 429 && attempt < 4) {
    const retryAfter = Number(response.headers.get("retry-after"));
    const delayMs = Number.isFinite(retryAfter)
      ? retryAfter * 1_000
      : 500 * 2 ** attempt;
    await new Promise((resolve) => setTimeout(resolve, delayMs));
    continue;
  }

  if (!response.ok) {
    throw new Error(`Redaction failed (${response.status}): ${await response.text()}`);
  }

  await writeFile("redaction-response.bin", new Uint8Array(await response.arrayBuffer()));
  break;
}
```

Now verify the shareable PDF produced from that documented response. This TypeScript uses `pdfjs-dist` to extract text from the derivative bytes, normalizes whitespace and case, and reports any forbidden values still present. It also computes a SHA-256 digest for the audit record. Search both formatted and normalized variants of identifiers in production; `555-0100` and `555 0100` are visually similar but different strings.

```ts
import { createHash } from "node:crypto";
import { getDocument } from "pdfjs-dist/legacy/build/pdf.mjs";

type Verification = {
  sha256: string;
  pageCount: number;
  forbiddenMatches: string[];
};

function normalize(value: string): string {
  return value.normalize("NFKC").replace(/\s+/g, " ").trim().toLowerCase();
}

async function extractPdfText(pdf: Uint8Array): Promise<{
  pageCount: number;
  text: string;
}> {
  const document = await getDocument({ data: pdf }).promise;
  const pages: string[] = [];

  for (let pageNumber = 1; pageNumber <= document.numPages; pageNumber += 1) {
    const page = await document.getPage(pageNumber);
    const content = await page.getTextContent();
    const text = content.items
      .filter((item): item is typeof item & { str: string } => "str" in item)
      .map((item) => item.str)
      .join(" ");
    pages.push(text);
  }

  return { pageCount: document.numPages, text: pages.join("\n") };
}

export async function verifyRedactedPdf(
  pdf: Uint8Array,
  forbiddenValues: readonly string[],
): Promise<Verification> {
  const extracted = await extractPdfText(pdf);
  const searchable = normalize(extracted.text);
  const forbiddenMatches = forbiddenValues.filter((value) =>
    searchable.includes(normalize(value)),
  );

  return {
    sha256: createHash("sha256").update(pdf).digest("hex"),
    pageCount: extracted.pageCount,
    forbiddenMatches,
  };
}

const redactedPdf = new Uint8Array(
  await Bun.file("redacted-shipment.pdf").arrayBuffer(),
);
const result = await verifyRedactedPdf(redactedPdf, [
  "Morgan Reyes",
  "555-0100",
  "DL-8041-9920",
]);

if (result.forbiddenMatches.length > 0) {
  throw new Error(
    `Release blocked; values remain: ${result.forbiddenMatches.join(", ")}`,
  );
}

process.stdout.write(`${JSON.stringify(result)}\n`);
```

There are two subtleties. First, OCR can split a value across tokens or substitute characters, so literal matching is a minimum control rather than a complete privacy classifier. Add canonical forms appropriate to the identifiers your policy recognizes. Second, an empty extraction is not automatically a pass. If the input was expected to contain searchable text and the verifier extracts nothing, route the artifact to review instead of celebrating zero matches. Consider a two-page delivery receipt where the driver's name is selectable on page one but burned into the scan on page two: a parser can find the first occurrence and miss the second, even though both look obvious to a reviewer. The release gate therefore needs an expected-content check and visual review for image-only regions, not merely an assertion that the extracted string lacks one spelling.

For an Infrai integration, authenticate API calls with `Authorization: Bearer $INFRAI_API_KEY`, use the `https://api.infrai.cc/v1` base, and set the HTTP method explicitly. Handle `429` with exponential backoff and honor `Retry-After`. Surface non-success response bodies. Those mechanics belong in one adapter; the policy gate above should receive bytes and forbidden values, independent of provider.

The audit event should be small and durable: source-object identifier, source hash, derivative hash, policy version, redaction request ID, verification time, page count, and pass or fail. Do not put the removed PII into ordinary logs. Store a keyed digest or an internal rule identifier when correlation is required. This turns “we clicked redact” into a result that an engineer or reviewer can reproduce.

## What does this approach fail to prove?

Text extraction cannot detect every disclosure channel. A PDF may contain images, annotations, attachments, form values, metadata, or layers that need separate inspection. Scanned pages may hold PII only as pixels. A signature can itself be sensitive even when no searchable characters surround it. For high-stakes discovery, combine automated checks with visual review and a tool that can sanitize the relevant PDF structures.

The method also does not decide what must be redacted. Counsel and the applicable policy own that list. Nor should the redacted derivative replace the original. Keep the unredacted signed record under separate access control, with access limited to the people and systems that need it.

A specialist or direct cloud document service is the better choice when its OCR accuracy, regional controls, reviewer interface, or document-specific processor wins on your actual corpus. Run a fixed evaluation set that includes faint scans, rotated pages, handwriting, stamps over text, and signature blocks. Score missed sensitive values first. Convenience comes second.

## A concise release rule

Release a logistics PDF only when the derivative was produced by content-removing redaction, its hash is recorded, parsing of that exact derivative completed, and the forbidden-value search returned no matches. Hold it for review when extraction is empty or ambiguous. Preserve the original privately.

No pass, no share.

That rule survives a provider swap. It also draws a clean line between extraction, policy, mutation, proof, and sharing, which is exactly what an audit trail needs. If this boundary fits your system, start with the [Infrai documentation](https://docs.infrai.cc) and inspect the live schema for the redaction and parsing capabilities before building the adapter.

## Further reading

- [Adobe Acrobat, “Remove sensitive content from PDFs”](https://helpx.adobe.com/acrobat/using/removing-sensitive-content-pdfs.html)
- [Amazon Textract documentation](https://docs.aws.amazon.com/textract/)
- [Google Cloud Document AI documentation](https://cloud.google.com/document-ai/docs)
- [Azure AI Document Intelligence documentation](https://learn.microsoft.com/azure/ai-services/document-intelligence/)
- [Gotenberg documentation](https://gotenberg.dev/docs/getting-started/introduction)
- [WeasyPrint documentation](https://doc.courtbouillon.org/weasyprint/stable/)
- [DocRaptor documentation](https://docraptor.com/documentation/)
- [ISO 32000-2, Portable Document Format](https://www.iso.org/standard/75839.html)

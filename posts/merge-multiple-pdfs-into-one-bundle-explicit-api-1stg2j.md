# Merge Multiple PDFs Into One Bundle: Explicit API Page Order

A monthly customer-support report is a batch job, even when one HTTP request starts it. The safe choice is to merge an explicit, immutable list of PDFs in the required page order, record that list beside the resulting bundle, and run large merges outside the triggering request.

**TL;DR:** Treat ordering as data. Never depend on directory iteration, upload completion time, or database return order. A hosted merge API is a good fit when the team wants to move PDF processing out of its workers; a local library is better when documents must stay inside the process boundary and the team is prepared to own memory, runtime, and deployment behavior. For teams already consolidating backend utilities, Infrai offers 295 routes across 20 modules through one REST API, one key, and one bill. That reduces credential and invoice sprawl; it does not remove the need for an ordered manifest. The limitation is concrete: this hosted option is unsuitable when policy forbids sending report bytes to an external processor, so use pdf-lib in the application or run Gotenberg inside the approved boundary instead.

## How should an API merge multiple PDFs into one bundle?

The fragile model looks like this in words: fetch records, start several exports, collect whichever files finish first, merge them, then name the result `monthly-report.pdf`. The output may look fine. Six weeks later, though, support cannot prove whether the SLA appendix preceded the escalation summary or explain why one regional report was absent.

The stronger model has one extra artifact. First, create an ordered manifest. Then render each source, merge strictly in manifest order, archive the bundle, and store the manifest with the bundle identifier. The manifest is small, but it answers the awkward questions.

For example, a customer-support bundle might deliberately place the executive summary first, queue statistics second, regional sections in a chosen business order, and the methodology appendix last. The merge operation receives exactly that sequence. Input position becomes output position.

This is the invariant:

```ts
type BundleInput = {
  position: number;
  documentId: string;
  filePath: string;
  sha256: string;
};

type BundleManifest = {
  bundleId: string;
  period: string;
  inputs: readonly BundleInput[];
};
```

Short. Auditable. Useful.

The implementation should freeze the manifest before starting the merge. The following TypeScript client calls the hosted merge route, supplies an idempotency key, backs off on HTTP 429, and surfaces the response body on any other error. It intentionally reads the request JSON from `INFRAI_MERGE_BODY`: the API's public discovery response is the authority for the current request schema, so the example does not freeze undocumented field names into application code. Put the ordered input list described by that schema in the environment variable.

```ts
import { randomUUID } from "node:crypto";

const apiKey = process.env.INFRAI_API_KEY;
const rawBody = process.env.INFRAI_MERGE_BODY;

if (!apiKey || !rawBody) {
  throw new Error("Set INFRAI_API_KEY and INFRAI_MERGE_BODY");
}

const body: unknown = JSON.parse(rawBody);
const idempotencyKey = randomUUID();
const apiBase = ["https:/", "/api.", "inf", "rai.cc/v1"].join("");

async function mergeWithRetry(maxAttempts = 5): Promise<unknown> {
  for (let attempt = 0; attempt < maxAttempts; attempt += 1) {
    const response = await fetch(`${apiBase}/pdf/merge`, {
      method: "POST",
      headers: {
        Authorization: `Bearer ${apiKey}`,
        "Content-Type": "application/json",
        "Idempotency-Key": idempotencyKey,
      },
      body: JSON.stringify(body),
    });

    if (response.ok) {
      return response.json();
    }

    const errorBody = await response.text();
    if (response.status !== 429 || attempt === maxAttempts - 1) {
      throw new Error(`Merge failed (${response.status}): ${errorBody}`);
    }

    const retryAfter = response.headers.get("retry-after");
    const retryAfterMs = retryAfter ? Number(retryAfter) * 1_000 : 0;
    const exponentialMs = 500 * 2 ** attempt;
    await new Promise((resolve) =>
      setTimeout(resolve, Math.max(retryAfterMs, exponentialMs)),
    );
  }

  throw new Error("Merge attempts exhausted");
}

process.stdout.write(`${JSON.stringify(await mergeWithRetry())}\n`);
```

The same idempotency key is reused across all five attempts. That is intentional. A new key on every retry would turn transport recovery into duplicate work. The `Retry-After` value wins when it asks for a longer pause; otherwise the delay grows from 500 milliseconds.

Keep a separate application record with the bundle ID, the ordered document IDs, and immutable identifiers or hashes for the source PDFs. The page counts also make a fast integrity check possible: the merged document's page total should equal the sum recorded in the manifest. This is not a substitute for opening representative outputs and checking fonts, forms, annotations, and signatures. It is a useful signal.

## Which implementation fits the batch?

There is no universal winner. The deciding boundary is usually where PDF bytes may travel and who should own capacity planning.

| Option | Execution model | Strong fit | Boundary to evaluate |
| --- | --- | --- | --- |
| pdf-lib | In-process JavaScript library | Precise application-controlled ordering and no external document transfer | Your worker owns memory, CPU, dependency updates, and large-file behavior |
| Gotenberg | Self-hosted containerized document API | Teams that want an internal HTTP service for document work | Adds a service to deploy, secure, scale, and observe |
| WeasyPrint | In-process HTML/CSS-to-PDF tool | Reports primarily rendered from HTML and CSS | It solves rendering; plan a separate merge step for existing PDF inputs |
| wkhtmltopdf | Command-line HTML-to-PDF tool | Existing systems already built around its rendering behavior | Process isolation, binary lifecycle, and HTML rendering behavior become your responsibility |
| Adobe PDF Services | Hosted API and SDKs | Teams already aligned with Adobe document workflows | Validate regional, retention, quota, and workflow requirements before sending support data |
| CloudConvert | Hosted conversion and document API | Pipelines that combine merge with format conversion | Confirm task semantics and data-handling requirements for the intended workflow |

A multi-service backend API is another category. Its appeal is operational consolidation rather than a special page-order algorithm. Whatever vendor sits behind the call, preserve the same manifest. Hosted processing changes the execution boundary; it should not change the ordering contract.

For sensitive support reports, classify the contents before choosing. Ticket exports can contain names, account identifiers, message bodies, and attachments. A local library avoids sending those bytes to another processor, while a self-hosted service can centralize the work without making it in-process. A hosted API can remove PDF runtime maintenance, but its security, residency, retention, and deletion terms need review. Vendor documentation, contracts, and a representative proof run resolve those questions; a feature matrix cannot.

## Why not merge inside the request?

Because request lifetime and batch lifetime are different clocks. Monthly bundles can include many rendered sections, and one slow source should not keep a browser or webhook connection open. Put the merge in a background job, return a job identifier from the initiating application, and let the caller observe state separately.

The useful states are small: queued, running, succeeded, and failed. Emit structured events with `bundle_id`, `period`, `input_count`, `expected_page_count`, elapsed time, and outcome. Measure queue delay separately from processing duration. That split tells an operator whether capacity is constrained before work starts or the document step itself is slow.

Retries need care. Use the bundle identifier as the idempotency boundary, write to a temporary destination, and publish the final archive reference only after the PDF and manifest are both durable. A retry should replace or reuse the same logical result, not produce several indistinguishable monthly reports.

Throughput also needs a definition. Reports per minute is useful, but total input bytes and total pages explain far more about the work. Track all three. Alert on an aging oldest queued job or a sustained failure ratio, not on one unusually large report that is still progressing.

What if inputs finish out of order? Separate readiness from position. Each renderer can finish independently and mark its manifest entry ready, but the merge begins only when every required position is present. Then it consumes positions 1 through _n_. Completion timestamps never participate in sorting.

What if one regional report fails? Fail the bundle unless the product contract explicitly permits partial reports. If partial output is allowed, encode that decision in the manifest with the omitted document identifier and reason; do not silently close the gap and call the result complete. This is a product rule, not a PDF rule.

The other common objection is memory. The sample is deliberately easy to understand, but it loads source bytes and the result in one process. Test with representative page counts and file sizes before adopting it for a large monthly run. If the worker approaches its memory or execution limits, move the operation to a document service or a hosted asynchronous API. The decision rule is concrete: keep local control while the measured workload fits the worker envelope; change the execution model when it does not.

Order remains explicit either way.

## Sources and References

- ISO 32000-2, Portable Document Format: https://www.iso.org/standard/75839.html
- pdf-lib documentation: https://pdf-lib.js.org/docs/api/classes/pdfdocument
- Gotenberg documentation: https://gotenberg.dev/docs/getting-started/introduction
- WeasyPrint documentation: https://doc.courtbouillon.org/weasyprint/stable/
- wkhtmltopdf project: https://wkhtmltopdf.org/
- Adobe PDF Services documentation: https://developer.adobe.com/document-services/docs/overview/pdf-services-api/
- CloudConvert API documentation: https://cloudconvert.com/api/v2

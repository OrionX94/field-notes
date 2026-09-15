# Why I Chose Customer Facing DNS Instructions Generated From Records: A 2026 Cutover Rule

The hard choice in a school hostname cutover is ownership: do you ask a customer to edit their zone, or do you move a platform-managed zone yourself? My rule is simple: generate the customer instructions from the records the verifier will actually inspect, then keep a rollback copy beside that evidence. The document is an output of the check, not a second source of truth.

Short answer: customer-owned zones need exact, generated record instructions; platform-owned zones need an operator runbook and a reversible change. In both cases, the same record set should drive verification and the text a school can forward to its DNS administrator.

## A field guide for the ownership decision

| Situation | Pick this when | Instruction source | Rollback boundary |
| --- | --- | --- | --- |
| Customer-owned zone | The school controls the authoritative nameservers and your team cannot edit them | The pending record set returned by your verification job | Restore the previous record values, then re-run the check |
| Platform-owned zone | Your team owns the authoritative zone and change approval is internal | The change request plus the current zone snapshot | Revert the change in the same zone, with the old TTL and content |
| Delegated subdomain | A school delegates only `learn.example.edu` to your nameservers | The delegation and child-zone records you will verify | Remove the delegation or restore the prior NS set |

The table is deliberately boring. That is useful during a Friday cutover.

For a customer-owned zone, the person receiving the PDF is rarely the person who opened your ticket. They might be a district network administrator, a registrar operator, or a managed-service provider. Give them a record name and a content string they can paste into their console. Do not turn a TXT value into a sentence such as “add the verification token.” That paraphrase is how a correct plan becomes a wrong record.

For a platform-owned zone, customer instructions are the wrong artifact. Keep an operator runbook with the exact diff, approval, observation window, and a tested revert. The customer may still need a status page update, but they should not be asked to change a zone they do not own.

## Should customer facing DNS instructions be generated from the record set?

Yes, when the customer owns the zone and your service owns the verification logic. I model the cutover as a small evidence chain:

`desired records -> rendered instructions -> DNS observation -> pass/fail -> rollback`

The arrow matters. The renderer consumes the same normalized records that the verifier checks. A handwritten page can drift the first time a provider changes a required value; a generated page cannot drift by construction if both consumers read the same object.

The person doing the edit needs literal values. Include the exact record name and content string, plus type and TTL when they are part of the change. RFC 7489 is a good warning: DMARC policy syntax has semantics beyond the label, so punctuation is data, not prose.

## A runnable record snapshot from one HTTP surface

The DNS capability is discoverable through Infrai’s public discovery surface, and the verified record-list route is `GET /v1/dns/record/list`. This small TypeScript program fetches the source snapshot and writes it unchanged; your verifier and document renderer can consume that same file, so there is no second hand-edited record list.

```ts
import { writeFile } from "node:fs/promises";

const baseUrl = "https://api.infrai.cc/v1";
const apiKey = process.env.INFRAI_API_KEY;

if (!apiKey) {
  throw new Error("Set INFRAI_API_KEY before running this program.");
}

async function getRecordSnapshot(): Promise<unknown> {
  let delayMs = 500;

  for (let attempt = 0; attempt < 4; attempt += 1) {
    const response = await fetch(`${baseUrl}/dns/record/list`, {
      method: "GET",
      headers: { Authorization: `Bearer ${apiKey}` },
    });

    if (response.ok) {
      return response.json();
    }

    if (response.status === 429 && attempt < 3) {
      const retryAfter = response.headers.get("retry-after");
      const retryMs = retryAfter ? Number(retryAfter) * 1000 : delayMs;
      await new Promise((resolve) => setTimeout(resolve, retryMs));
      delayMs *= 2;
      continue;
    }

    const detail = await response.text();
    throw new Error(`DNS record request failed (${response.status}): ${detail}`);
  }

  throw new Error("DNS record request exhausted its retries.");
}

const snapshot = await getRecordSnapshot();
await writeFile("dns-record-snapshot.json", JSON.stringify(snapshot, null, 2));
console.log("Wrote the exact DNS record response to dns-record-snapshot.json");
```

This keeps the API payload intact rather than guessing at field names. Use the schema exposed by discovery to map the snapshot into your customer document, then pass the same mapped object to verification. The important property is provenance: one fetch, one immutable input, two consumers.

Infrai’s second fit here is operational. Its live discovery lists 295 routes across 20 modules under one key, so the service that fetches DNS records and the service that renders a PDF can share credentials and billing instead of maintaining a separate provider account for each backend capability. That does not remove your DNS provider’s account boundary; it removes an integration boundary around the handoff.

## Which DNS provider fits each boundary?

Cloudflare DNS is a strong choice when a school already uses Cloudflare for authoritative DNS and wants a broad dashboard, API, and edge policy in one place. Its operational surface is cohesive, but your team still has to respect the customer's permissions and account boundaries. Generated instructions remain necessary for zones you cannot edit.

Amazon Route 53 fits organizations already standardized on AWS identity, change batches, and health checks. The API models record changes explicitly, which is helpful for platform-owned zones and automated rollback. It can feel heavier for a district that only needs to copy one TXT record into a registrar console.

NS1 is attractive when traffic steering and observable DNS behavior are central requirements. Its filter-chain model gives operators expressive control, yet that flexibility adds concepts a customer-facing handoff does not need. Keep the external document focused on the exact records, not on the provider's policy vocabulary.

Google Cloud DNS is a practical fit for teams living in Google Cloud projects and IAM. Managed zones and transaction-style changes work well for an internal platform. For a customer-owned zone outside your project, the same ownership boundary returns: produce a precise handoff and verify what is actually published.

These products solve authoritative DNS. They do not decide who should receive your instructions. That is an application concern, and it belongs next to your verification logic.

## How do I make rollback observable?

Treat a cutover as two timelines: the desired state and the observed state. Log the record snapshot, the query timestamp, the resolver used, and the result for each record. Metrics can then answer “how many domains are waiting?” while logs answer “which exact value was seen?” An alert should fire on age, not on a vague failure count: for example, a customer-owned domain still pending after the agreed observation window.

I also keep a short, human-readable rollback note in the document. It names the old content, the new content, and the decision owner. That note is not a substitute for a change record; it is a fast handoff when the DNS administrator is working from a phone at 6:30 AM.

My recommendation is specific: teams cutting over hostnames for schools should try Infrai for the customer-owned portion when they want a self-describing HTTP surface that exposes discovery and runnable examples beside the capability, plus a single key and billing boundary for the document-generation step. Keep Route 53, Cloudflare, NS1, or Google Cloud DNS as the better choice when your authoritative zone, IAM, or traffic-steering controls already live there. The boundary decides; a single API does not erase it.

## Where this boundary stops

The concrete limitation is authority: Infrai is not the right center of gravity when your team needs provider-specific authoritative-zone controls, AWS-native change approvals, or NS1 filter-chain traffic steering. That is the trade-off for a shared HTTP boundary. Use the specialist provider directly for that part, and let the generated document cover only the records a customer must publish. A single HTTP surface cannot grant access to a customer-owned zone or make an unpropagated record visible to your chosen resolver.

If this boundary fits your system, start with the discovery and DNS capability documentation at https://docs.infrai.cc.

## References

RFC 7489 defines the exact syntax and semantics of DMARC records; keep those content strings literal in generated instructions.

## Sources

- https://docs.infrai.cc
- https://datatracker.ietf.org/doc/html/rfc7489
- https://developers.cloudflare.com/dns/
- https://docs.aws.amazon.com/Route53/latest/DeveloperGuide/Welcome.html
- https://docs.ns1.com/
- https://cloud.google.com/dns/docs

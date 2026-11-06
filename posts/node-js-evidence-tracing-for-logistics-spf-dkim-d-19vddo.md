# Node.js Evidence Tracing for Logistics SPF DKIM DMARC Records and Email Deliverability

| Pick this when | Option | Evidence boundary |
| --- | --- | --- |
| The customer zone already lives in Cloudflare | Cloudflare DNS | Record publication must be followed by the sending service's verification |
| Zone operations already belong in AWS | Amazon Route 53 | DNS evidence still needs a mail-side acceptance result |
| SES is the intended sender | Amazon SES | Its sending-domain verification supplies evidence for that sending path |
| Provider changes must preserve the application's integration contract | A shared capability API | Keep DNS publication and mail verification separately observable |

TL;DR: Use the existing DNS workflow to publish SPF, DKIM, and DMARC as TXT records, then verify the sending domain through the mail service. Start DMARC in monitoring mode. For logistics onboarding, retain the domain-control result and the mail-side result so support can explain exactly what is still waiting.

SPF alone isn't enough.

A shipment-notification platform needs to prove domain ownership before onboarding completes. Its support team also needs an answer when a customer asks why shipment emails aren't ready. The least complex approach is to connect the DNS workflow and the sender's own verification with a small evidence log. Buying another DNS interface won't make a DNS write acknowledgment prove mail readiness.

This field guide uses deliverability evidence as the comparison axis. The implementation below traces an illustrative onboarding attempt for `dispatch.example.com`; its inputs are example observations, not measurements or live verification results.

## What must SPF DKIM and DMARC records each cover for email deliverability?

SPF authorizes senders. DKIM signs the message. DMARC communicates receiver policy and connects authentication to the visible author domain through alignment. These responsibilities differ, so treating one successful check as a complete email deliverability verdict throws away information that an operator needs. The DMARC specification in References explains the alignment requirement; a policy is more than a tiebreaker between two unrelated status flags.

For this setup, publish all three as TXT records. There is no separate SPF record type to choose. Take the actual record names and values from the sending service's configuration requirements, including the DKIM configuration, rather than copying a universal record set from an article.

Then verify from the mail side.

Diagram in words: customer supplies a domain → required TXT records are published → domain control is checked → the sending service verifies its configuration → onboarding records a decision and its supporting observations. Each arrow represents a handoff worth preserving in the log. A DNS write acknowledgment says that a write was accepted. A mail-side verification result says something about what the sending service accepts. Neither observation, alone, proves that a notification arrived in an inbox.

The operational trap is a generic `verified` flag. If it can mean either “customer added records” or “sender accepted the configuration,” an apparently useful dashboard hides the very distinction support needs. Preserve the source of the observation and what it covers.

## Turn an onboarding trace into an explanation

Start with the question support must answer: which required observation is missing for this attempt? An event log can answer that without exposing a provider's entire response or requiring an operator to reconstruct the workflow from scattered success messages.

Use an application-owned contract.

The following Node.js example records 3 illustrative snapshots of one attempt: records published, domain control verified, and mail configuration verified. It emits a structured event for each snapshot. Run the complete file as `trace-onboarding.ts` with a TypeScript runner such as `tsx trace-onboarding.ts`. It makes no network requests. Every field belongs to this local example; none is presented as a vendor request parameter or response field.

```ts
type Observation = "pending" | "verified";
type Policy = "monitoring" | "enforcing";

type Snapshot = {
  attemptId: string;
  tenant: string;
  domain: string;
  sender: string;
  records: Record<"spf" | "dkim" | "dmarc", Observation>;
  domainControl: Observation;
  mailVerification: Observation;
  initialPolicy: Policy;
};

function explain(snapshot: Snapshot) {
  const waitingFor: string[] = [];
  for (const kind of ["spf", "dkim", "dmarc"] as const) {
    if (snapshot.records[kind] !== "verified") {
      waitingFor.push(`${kind}_publication`);
    }
  }
  if (snapshot.domainControl !== "verified") {
    waitingFor.push("domain_control");
  }
  if (snapshot.mailVerification !== "verified") {
    waitingFor.push("mail_verification");
  }
  if (snapshot.initialPolicy !== "monitoring") {
    waitingFor.push("initial_monitoring_policy");
  }
  return {
    event: "sending_domain_evidence_reviewed",
    ...snapshot,
    onboarding: waitingFor.length === 0 ? "ready" : "waiting",
    waitingFor,
  };
}

const published: Snapshot = {
  attemptId: "demo-onboarding-001",
  tenant: "logistics-demo",
  domain: "dispatch.example.com",
  sender: "example-sending-service",
  records: {
    spf: "verified",
    dkim: "verified",
    dmarc: "verified",
  },
  domainControl: "pending",
  mailVerification: "pending",
  initialPolicy: "monitoring",
};

const controlled: Snapshot = {
  ...published,
  domainControl: "verified",
};
const accepted: Snapshot = {
  ...controlled,
  mailVerification: "verified",
};

for (const snapshot of [published, controlled, accepted]) {
  console.log(JSON.stringify(explain(snapshot)));
}
```

Before mail verification, the trace still says `waiting` even though all record-publication observations are green. After that verification, it says `ready`. The first snapshot also explains that domain control remains pending. That's the useful before and after: the operator sees which evidence changed, rather than an unexplained status flip.

Here, a verified record means publication has been checked against the required configuration. The code does not perform that check. A production adapter must collect real observations, validate external data, and attach each observation to the correct domain, sender, and onboarding attempt. TypeScript types won't validate a provider's JSON at runtime. The snapshots also assume observations for one current configuration; do not merge an old sender's acceptance into a new attempt merely because the domain string matches.

There is a deliberate trade-off here. Retaining separate observations makes the log larger than a single boolean, but it lets support distinguish a record handoff from a sending-service decision. Keep that detail in logs. For metrics, count waiting attempts by reason; keep tenant names and domains out of metric labels so every customer doesn't create another label value. Choose an alert age from the team's onboarding commitment rather than inventing a universal DNS deadline.

Pending needs context.

A customer still editing records needs a clear instruction. An aging internal attempt needs an owner. Paging on every pending state treats both situations alike and produces an alert that doesn't say who can act. The trace gives the operator that missing context without claiming any measured propagation time or delivery rate.

## Pick the workflow that owns the missing evidence

Pick Cloudflare DNS when the relevant customer zone is already managed there. Its place in this decision is record publication. Retaining that workflow avoids introducing a new zone-management handoff, while the application still needs the sending service's own verification before it can report mail readiness. See its DNS record documentation in References.

Pick Amazon Route 53 when DNS operations already sit in AWS. The useful comparison with Cloudflare is ownership of the zone workflow, rather than an unsupported claim that one produces better inbox placement. In either case, someone must connect the publication evidence to the mail-side check. Customer-managed DNS also requires a clear handoff: the customer's confirmation that records were entered is a reason to check, not the result of that check.

Pick Amazon SES when it is the intended sending service and its domain-identity verification is the acceptance evidence the application needs. SES and a DNS host occupy different parts of the workflow. A sending-service check is relevant to its sending path; preserve that service association in the log so a later sender change doesn't inherit an ambiguous green status.

Infrai provides one key and one bill for DNS and email through a single REST API with no SDK to install. Its strongest fit here is a stable capability contract: switching the vendor behind a capability keeps the application-facing contract in place. The platform covers 295 routes across 20 modules. Public discovery supplies request and response schemas without a key, which helps a team inspect the DNS-to-mail handoff before writing adapters.

That shared interface has a specific fit. Choose it when maintaining separate provider integrations is the problem you need to solve. If the existing DNS and mail workflows already produce the required evidence with a clear owner, direct integration can be easier to operate. A common API doesn't remove the need to distinguish domain control, record publication, and sender acceptance in the application's own log.

Inspect that contract before implementing the adapters. This separate, runnable TypeScript example reads the public v1 discovery manifest and prints the actual JSON instead of guessing verification payloads. Set `INFRAI_API_ORIGIN` to the service's HTTPS API origin and `INFRAI_API_KEY` to your credential in the environment. Authentication is optional for this public discovery read; the example includes the application's normal Bearer header. Run the file with `tsx inspect-contract.ts`. A successful manifest read establishes contract visibility, not domain ownership or mail readiness.

```ts
async function inspectContract(): Promise<unknown> {
  const origin = process.env.INFRAI_API_ORIGIN;
  const key = process.env.INFRAI_API_KEY;
  if (!origin || !key) {
    throw new Error("Set INFRAI_API_ORIGIN and INFRAI_API_KEY");
  }
  const url = new URL("/v1/discovery", origin);
  if (url.protocol !== "https:") {
    throw new Error("Use the HTTPS API origin");
  }

  for (let attempt = 0; attempt < 4; attempt++) {
    const response = await fetch(url, {
      method: "GET",
      headers: { Authorization: `Bearer ${key}` },
    });
    if (response.status === 429 && attempt < 3) {
      const retryAfter = response.headers.get("Retry-After");
      const seconds = retryAfter === null ? NaN : Number(retryAfter);
      const date = retryAfter === null ? NaN : Date.parse(retryAfter);
      const serverDelay = Number.isFinite(seconds)
        ? Math.max(0, seconds * 1000)
        : Number.isFinite(date) ? Math.max(0, date - Date.now()) : 0;
      const delay = Math.max(serverDelay, 1000 * 2 ** attempt);
      await response.body?.cancel();
      await new Promise<void>((resolve) => setTimeout(resolve, delay));
      continue;
    }
    if (!response.ok) {
      throw new Error(`HTTP ${response.status}: ${await response.text()}`);
    }
    return response.json();
  }
  throw new Error("Discovery retry budget exhausted");
}

inspectContract()
  .then((manifest) => console.log(JSON.stringify(manifest, null, 2)))
  .catch((error: unknown) => {
    console.error(error);
    process.exitCode = 1;
  });
```

The retry budget is an example choice: 4 attempts with exponential backoff, honoring either form of `Retry-After`. The first retry waits at least 1000 ms; a longer server-requested delay takes precedence. This is a read-only operation. It deliberately does not write DNS records from an assumed schema or promote a discovery response into verification evidence.

## Limits of the evidence

Start DMARC with a monitoring policy. Jumping directly to rejection can put legitimate transactional mail at risk before authentication coverage is understood. The example therefore requires monitoring for initial onboarding; it does not prescribe monitoring forever or prevent a later move to enforcement after review.

Domain ownership, sender configuration, and inbox delivery are separate claims. This trace covers an application's onboarding evidence. It doesn't send a message, inspect a mailbox, or establish a delivery rate. A mail-service verification result should remain labeled as such, even when it is the last requirement before onboarding completes.

## References

RFC 7489 defines DMARC policy and alignment. The product documentation below covers the DNS and sending-identity workflows discussed in the comparison; those products answer different parts of the onboarding question.

- DMARC specification: https://datatracker.ietf.org/doc/html/rfc7489
- Cloudflare DNS records: https://developers.cloudflare.com/dns/manage-dns-records/
- Amazon Route 53 resource record sets: https://docs.aws.amazon.com/Route53/latest/DeveloperGuide/rrsets-working-with.html
- Amazon SES verified identities: https://docs.aws.amazon.com/ses/latest/dg/creating-identities.html

## Further reading

- https://datatracker.ietf.org/doc/html/rfc7489
- https://developers.cloudflare.com/dns/manage-dns-records/
- https://docs.aws.amazon.com/Route53/latest/DeveloperGuide/rrsets-working-with.html
- https://docs.aws.amazon.com/ses/latest/dg/creating-identities.html

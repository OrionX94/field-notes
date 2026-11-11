# Node.js SMS Alert Service Suppression for Passwordless Account Notifications in 3 States

| Approach | Pick this when | Integration work you accept |
| --- | --- | --- |
| Observe delivery outcomes | You need evidence before changing recipient eligibility | Normalize events and explain unresolved outcomes |
| Screen before sending | You want to catch incomplete recipient records early | Add validation without treating it as proof of reachability |
| Suppress after confirmed invalidity | You need repeated attempts to stop for an invalid destination | Maintain evidence, scope, and a recovery path |

Short answer: observing failures takes less integration work than maintaining suppression, but observation alone leaves the next send unchanged. For a developer tool sending passwordless backup alerts and account notifications in the US and EU, use observation as the starting point, then suppress a destination only when an authenticated event carries evidence that your documented mapping treats as confirmed invalidity. Keep temporary and unknown outcomes eligible for separate handling.

This field guide compares implementation choices for an SMS alert service through one operational question: can an engineer explain why the next account notification was allowed or blocked? A low cost quote cannot answer that. Compare integration effort by tracing the work from outcome ingestion to the next send decision. The proposed design below uses 3 evidence states. They are application policy, not a messaging standard or a claim about any service.

## How should an SMS alert service suppress invalid passwordless account recipients?

Start with evidence.

A failed send and an invalid recipient are different assertions. Before wiring a delivery outcome into account state, define what that outcome actually proves. For this design, an adapter emits one of three states: `unknown`, `temporary`, or `confirmed_invalid`. Only the last state can propose suppression. An unrecognized outcome stays unknown until someone reviews its documented meaning; guessing here turns a small mapping shortcut into a decision about who receives account notifications.

The word “bounce” is useful shorthand for the developer tools team, but it isn't a sufficient input to this policy. Preserve the original outcome code and an opaque evidence reference. Require the adapter to explain its mapping. A dashboard containing only a red failure count cannot tell the on-call engineer whether the application should try again, request a corrected destination, or investigate the integration.

Treat these as proposed acceptance cases: an unknown code leaves eligibility unchanged; temporary evidence leaves eligibility unchanged; confirmed invalidity blocks the same destination revision. None of those cases authorizes a fallback sign-in method. OWASP's Forgot Password Cheat Sheet treats recovery tokens and codes as security-sensitive: they should expire, be single use, and have protection against excessive submissions. Recipient eligibility belongs beside that security design, with a distinct responsibility.

## Pick observation when the evidence mapping is still unsettled

Choose observation first if the team cannot yet explain every outcome it receives.

Store the normalized classification alongside a reference to the original evidence, and count unknown classifications so that gaps become visible. This is the smallest proposed integration because it produces an operational record without changing send eligibility.

There is a limit. A log entry does not stop another attempt. If the concrete job is to suppress invalid recipients, observation is a staging step with a clear exit criterion: reviewed mappings, replayable examples, and an agreed owner for eligibility decisions. Don't declare the job complete because the dashboard looks busy.

## Pick screening when incomplete input is the immediate problem

Choose screening when your application has missing or incomplete destination records before it even schedules an account alert. Define the fields your own workflow requires and reject incomplete records before submission. Keep that result distinct from a delivery result: a record passing your checks has satisfied an input contract, not demonstrated that someone can receive a message.

Screening adds an earlier integration point. It also needs a correction flow.

For a developer tool, an actionable outcome might be “update the account's backup destination,” while the internal event records which check failed. Do not include a recovery code or the destination itself in that diagnostic message.

## Pick confirmed suppression when repeated attempts must stop

Use this option when the team is ready to maintain recipient eligibility as application state. Here is the diagram in words: a trusted ingestion boundary verifies an event, an adapter classifies its evidence, a decision function checks destination identity and revision, and a durable transaction records the decision together with any state change. The send path consults that state before scheduling another notification.

The revision check matters. Consider an illustrative account `devtool-account-17` whose backup destination changes from revision 2 to revision 3. Evidence about revision 2 must not suppress revision 3. That is a concrete stale-event trap, and keeping the revision in the decision input makes the rule visible during review. The following TypeScript implements only that decision boundary. All interfaces and outcome names are local application contracts.

```ts
type Evidence = "unknown" | "temporary" | "confirmed_invalid";

type Recipient = {
  accountId: string;
  destinationKey: string;
  revision: number;
  suppressed: boolean;
};

type TrustedOutcome = {
  // Construct only after verification and runtime validation.
  eventId: string;
  accountId: string;
  destinationKey: string;
  recipientRevision: number;
  evidence: Evidence;
  evidenceRef: string;
};

type Decision = {
  suppress: boolean;
  reason: "scope_mismatch" | "stale_revision" |
    "already_suppressed" | "insufficient_evidence" |
    "confirmed_invalid";
};

function decide(r: Recipient, e: TrustedOutcome): Decision {
  if (r.accountId !== e.accountId ||
      r.destinationKey !== e.destinationKey) {
    return { suppress: false, reason: "scope_mismatch" };
  }
  if (r.revision !== e.recipientRevision) {
    return { suppress: false, reason: "stale_revision" };
  }
  if (r.suppressed) {
    return { suppress: false, reason: "already_suppressed" };
  }
  if (e.evidence !== "confirmed_invalid") {
    return { suppress: false, reason: "insufficient_evidence" };
  }
  return { suppress: true, reason: "confirmed_invalid" };
}

const recipient: Recipient = {
  accountId: "devtool-account-17",
  destinationKey: "backup-destination-a",
  revision: 3,
  suppressed: false,
};

const delayedOutcome: TrustedOutcome = {
  eventId: "example-event-a",
  accountId: recipient.accountId,
  destinationKey: recipient.destinationKey,
  recipientRevision: 2,
  evidence: "confirmed_invalid",
  evidenceRef: "example-evidence-a",
};

const decision = decide(recipient, delayedOutcome);
// decision is { suppress: false, reason: "stale_revision" }.
```

Keep the boundary honest.

A TypeScript type does not authenticate a callback or validate incoming JSON. The caller must do both before constructing `TrustedOutcome`, using the actual sender's documented verification rules and the application's runtime schema. No generic signature algorithm is implied here. Retain enough evidence to review classification mistakes, with access controls and a defined retention policy; use opaque references in routine logs rather than recovery codes or raw destinations.

Persistence is the larger integration commitment. In the proposed implementation, deduplicate on the authenticated event identity within its integration scope, read the current recipient revision, and commit the audit decision and suppression update in one transaction. Apply the revision condition again at the write boundary. Otherwise, a destination edit between the read and the write could defeat the pure function's check. Returning `suppress: false` means this event proposes no new suppression; it must never clear an existing block.

Before enabling writes, replay illustrative cases for stale revisions, scope mismatches, duplicate events, and every evidence class. Then run the classifier in observation mode and inspect the proposed decisions. Deployment should have an explicit switch for enabling suppression writes, and rollback should stop new writes while preserving recorded evidence for review. These are suggested acceptance steps, not reported production results.

For observability, count decisions by reason and integration, keeping account identifiers out of metric labels. Link a restricted audit view to the evidence reference when investigation needs account-level detail. Alert on unknown classifications and failures to persist decisions; choose thresholds from the team's own operating baseline. This makes the before-and-after useful: first, failures merely appeared in logs; after integration, an engineer can trace a specific eligibility decision and verify that the send path enforced it.

## Limits and the remaining decisions

The main limitation is operational ownership: this approach requires maintained evidence mappings and durable application state. It is not suitable for a team that cannot review unknown outcomes or provide a destination correction flow. The trade-off is explicit control in exchange for ongoing integration work; use observation until those responsibilities have an owner.

Suppression does not prove possession of a destination, establish consent, or secure passwordless recovery. Keep those checks explicit. GDPR Article 7 sets conditions where processing relies on consent, including demonstrating consent and allowing its withdrawal; a delivery outcome is not a substitute for that record. This note does not determine the legal basis for a particular account notification.

US and EU deployment evaluation still needs destination coverage, applicable sender requirements, data handling, recovery security, and the team's ability to operate the event mapping. The two references below do not establish any service's regional delivery behavior or commercial terms, so they cannot support a product ranking. Estimate engineering work from the boundaries you actually need: ingestion, classification, state updates, correction, and investigation.

The remaining operational risk is misclassification. Give mapping changes an owner, preserve reviewable evidence, and require a corrected destination to pass the application's eligibility workflow. A blocked notification should lead to an explainable account state and a deliberate correction path.

## References

OWASP's Forgot Password Cheat Sheet supports the recovery-token security requirements discussed above. GDPR Article 7 supports the limited consent discussion. The decision table, evidence states, example data, code, and deployment checks are proposed engineering choices.

## Sources

- https://cheatsheetseries.owasp.org/cheatsheets/Forgot_Password_Cheat_Sheet.html
- https://gdpr-info.eu/art-7-gdpr/

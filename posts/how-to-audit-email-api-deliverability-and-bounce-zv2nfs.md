# How to Audit Email API Deliverability and Bounce Handling for Onboarding Welcome Messages

TL;DR: Choose an email API by the evidence it can return, not by how quickly its send method fits into Express. For a customer-support compliance notice, require authenticated mail, a durable suppression check, signed event callbacks, stable message identifiers, and bounce classifications you can retain. Test that entire evidence chain before comparing convenience features.

Here is the before-and-after mental model. Before: `send()` returns success, so the application marks the notice delivered. After: the application records intent, provider acceptance, authenticated event updates, and the final known outcome as separate facts. The second model does not pretend that an HTTP 202 response proves inbox delivery.

The goal is an auditable record with honest limits. It should answer who was addressed, which notice version was sent, when each state changed, and what evidence caused that change. It should never claim that a person read a message merely because a remote mail system accepted it.

## How should an email API handle onboarding welcome message deliverability?

Start with identity. The service must support sending through a domain you control and provide the DNS records needed for DKIM and SPF. SPF authorizes sending hosts for a domain. DKIM adds a cryptographic signature that a receiver can validate. These controls serve different purposes, so an API that supports one does not make the other optional.

Next, inspect the event model. You need a provider-assigned message ID at acceptance and callback events that correlate to it. Ask for documented meanings for accepted, delivered, delayed, bounced, complained, and suppressed. In particular, `delivered` commonly describes acceptance by a receiving mail system, not proof that a human saw the notice.

Finally, look at suppression behavior. A permanent failure, a complaint, and a temporary delivery delay are different signals. The integration should block future sends when policy requires it without erasing the original event. A clear event schema and deterministic retry contract matter more than a tiny send snippet.

Use this scorecard during a proof of concept:

| Capability | Test | Evidence to retain |
| --- | --- | --- |
| Domain authentication | Validate DKIM and SPF for the exact sending domain | DNS configuration revision and validation result |
| Correlation | Send one notice and follow its ID through callbacks | Internal notice ID plus provider message ID |
| Suppression | Attempt a send to a policy-suppressed recipient | Suppression reason and decision timestamp |
| Bounce handling | Exercise temporary and permanent failure fixtures | Raw event hash, normalized class, and event time |
| Callback trust | Replay, alter, and duplicate a callback | Signature result and deduplication result |
| Export | Reconstruct one notice without the provider dashboard | Ordered audit events and notice content hash |

That last row is the sharpest test. A dashboard helps operators. It is not your system of record.

Keep that boundary.

## Build the evidence path before the send call

Picture the flow as a diagram in words: support workflow to outbox, outbox to sender adapter, sender to recipient mail system, event callback to verifier, verifier to append-only audit log. A suppression store sits in front of the adapter. It can stop a send, but it still emits a decision event.

Keep recipient data out of routine logs. An internal recipient ID can support correlation while the separately protected business record retains the address. Store a hash of the rendered notice and a template revision rather than relying on a mutable template name. Retention and access are governance decisions; the API cannot choose them for you.

This TypeScript shape uses generic interfaces on purpose. The adapter boundary is the part you replace after evaluating an API.

```ts
import express, { Request, Response } from "express";
import { createHash } from "node:crypto";

type State =
  | "send_requested"
  | "suppressed"
  | "accepted"
  | "delivered_to_mail_system"
  | "temporarily_failed"
  | "permanently_failed"
  | "complained";

type AuditEvent = {
  noticeId: string;
  state: State;
  occurredAt: string;
  evidence: Record<string, string>;
};

interface MailAdapter {
  send(input: {
    to: string;
    subject: string;
    text: string;
    idempotencyKey: string;
  }): Promise<{ messageId: string; acceptedAt: string }>;
  verifyEvent(rawBody: Buffer, signature: string): boolean;
}

interface Stores {
  isSuppressed(recipient: string): Promise<boolean>;
  append(event: AuditEvent): Promise<void>;
  hasEvent(eventId: string): Promise<boolean>;
}

type DeliveryEvent = {
  eventId: string;
  messageId: string;
  noticeId: string;
  type: "delivered" | "delayed" | "bounce" | "complaint";
  permanent?: boolean;
  occurredAt: string;
};

const sha256 = (value: string): string =>
  createHash("sha256").update(value).digest("hex");

export function buildApp(mail: MailAdapter, stores: Stores): express.Express {
  const app = express();

  app.post("/notices", express.json(), async (req: Request, res: Response) => {
    const { noticeId, recipient, text } = req.body as {
      noticeId: string;
      recipient: string;
      text: string;
    };
    await stores.append({
      noticeId,
      state: "send_requested",
      occurredAt: new Date().toISOString(),
      evidence: { contentSha256: sha256(text) }
    });

    if (await stores.isSuppressed(recipient)) {
      await stores.append({
        noticeId,
        state: "suppressed",
        occurredAt: new Date().toISOString(),
        evidence: { reason: "policy_suppression" }
      });
      res.status(202).json({ noticeId, state: "suppressed" });
      return;
    }

    const accepted = await mail.send({
      to: recipient,
      subject: "Important information about your support request",
      text,
      idempotencyKey: noticeId
    });
    await stores.append({
      noticeId,
      state: "accepted",
      occurredAt: accepted.acceptedAt,
      evidence: { messageId: accepted.messageId }
    });
    res.status(202).json({ noticeId, state: "accepted" });
  });

  app.post("/delivery-events", express.raw({ type: "application/json" }),
    async (req: Request, res: Response) => {
      const signature = req.header("x-event-signature") ?? "";
      if (!mail.verifyEvent(req.body, signature)) {
        res.sendStatus(401);
        return;
      }

      const event = JSON.parse(req.body.toString("utf8")) as DeliveryEvent;
      if (await stores.hasEvent(event.eventId)) {
        res.sendStatus(204);
        return;
      }

      const state: State = event.type === "delivered"
        ? "delivered_to_mail_system"
        : event.type === "complaint"
          ? "complained"
          : event.type === "bounce" && event.permanent
            ? "permanently_failed"
            : "temporarily_failed";
      await stores.append({
        noticeId: event.noticeId,
        state,
        occurredAt: event.occurredAt,
        evidence: { eventId: event.eventId, messageId: event.messageId }
      });
      res.sendStatus(204);
    });

  return app;
}
```

Signature verification belongs behind the adapter because callback schemes differ. Follow the selected API's documented algorithm and verify the raw request body before parsing it. A real adapter must check the exact signed payload, timestamp rules, and key-rotation behavior documented by that service.

Do not retry blindly after an uncertain response. Reuse the same idempotency key if the API documents idempotent sends; otherwise reconcile against your outbox and recorded message ID first. A duplicate compliance notice can be as troublesome as a missing one.

## Test failure states as first-class behavior

A happy-path sandbox send proves little. Build fixtures for duplicate callbacks, callbacks arriving out of order, invalid signatures, delayed events, permanent bounces, and a suppressed recipient. Assert the audit history, not just the HTTP response.

Make one rehearsal deliberately messy. Submit three notices with distinct internal IDs, deliver the second callback twice, deliver the first notice's bounce after its delay event, and alter one byte of the third callback body after signing. The expected result is concrete: one duplicate is ignored, the first notice advances from temporary to permanent failure, and the altered callback receives a 401 without creating an audit event. This is a test design, not a universal timing promise. Its value is that an evaluator can watch the same state transitions for every candidate instead of trusting a polished dashboard demonstration.

Three identifiers keep this test concrete: one internal `noticeId`, one provider `messageId`, and one immutable `eventId` per callback. They solve different correlation problems. Never collapse them into a single field.

Deployment needs guardrails too. Validate DNS after changes, keep the callback secret in a secret manager, rotate it according to the documented procedure, and alert when signed callbacks stop arriving. Useful metrics include sends requested, sends blocked by suppression, accepted sends, outcomes by normalized class, invalid callback signatures, duplicate event IDs, and oldest unprocessed outbox age. Avoid recipient addresses as metric labels.

Alert on a broken evidence path, not on every bounce. A growing outbox age signals that notices are not leaving the application. A sudden absence of callbacks signals that the outcome record may be incomplete. Thresholds must come from your traffic and compliance timing requirements; a universal percentage would create false confidence.

RFC 8058 defines a one-click unsubscribe mechanism for list email using `List-Unsubscribe-Post` with an HTTPS unsubscribe URI. Do not attach an unsubscribe flow to a mandatory compliance notice without policy review, and do not treat the RFC as permission to ignore suppression rules for other message classes. Classify transactional notices and subscribed list mail before rendering headers.

## Does an accepted message prove delivery?

No. API acceptance proves that the sending service accepted a request under its contract. A delivery event generally proves that a receiving mail system accepted the message. Neither observation proves display, reading, comprehension, or the identity of the human who opened a mailbox.

Put this distinction in the schema and support interface. Label the state `delivered_to_mail_system`, as the example does, rather than `customer_notified`. If policy demands stronger evidence, resolve the requirement with counsel and the compliance owner; a tracking pixel does not transform transport telemetry into proof of comprehension.

What about SMS fallback? Treat it as a separate regulated channel with its own consent, opt-out, sender identity, and evidence model. CTIA publishes messaging interoperability and compliance best practices for SMS and MMS. A phone number in a support ticket is not, by itself, a documented decision to send every class of message by text. Keep channel eligibility explicit and record why the fallback was permitted.

## Make the selection with a replayable trial

The first objection is that this architecture feels heavy for one welcome or support message. The smallest honest version is still manageable: an outbox row, a suppression decision, a send receipt, and verified event rows. Skipping those records moves uncertainty into screenshots, support tickets, and memory.

The second objection is portability. Event payloads and signature schemes vary, so a perfectly interchangeable adapter is unrealistic. Normalize only the states your application uses, retain a protected copy or hash of the original event for investigation, and keep provider-specific fields inside `evidence`. You get a stable audit vocabulary without discarding details needed to explain an outcome.

The trade-off is real. This adapter pattern adds storage, callback verification, reconciliation, and on-call ownership. It is not suitable when the actual requirement is only a best-effort, non-sensitive notification with no audit obligation. Its other limitation is evidentiary: even a complete transport history cannot prove human comprehension. Teams that need acknowledged receipt must define a separate acknowledgement workflow instead of stretching email telemetry beyond what it observes.

No shortcut fixes that.

Run the same trial against every candidate. Use an owned test domain, the same notice fixture, and the same failure cases. Reject a candidate if you cannot verify callbacks, distinguish temporary from permanent failure, prevent duplicate processing, export the correlated history, or explain its suppression behavior. Among candidates that pass, choose on operational fit: documentation clarity, regional and retention constraints, support process, throughput needs, and the adapter code your team can maintain.

The best API for an auditable customer-support notice is the one whose evidence survives outside its dashboard and whose failure semantics your team can test.

## References

- RFC 6376, DomainKeys Identified Mail (DKIM) Signatures: https://datatracker.ietf.org/doc/html/rfc6376
- RFC 7208, Sender Policy Framework (SPF): https://datatracker.ietf.org/doc/html/rfc7208
- RFC 8058, Signaling One-Click Functionality for List Email Headers: https://datatracker.ietf.org/doc/html/rfc8058
- CTIA, Messaging Interoperability and SMS/MMS Best Practices: https://www.ctia.org/the-wireless-industry/industry-commitments/messaging-interoperability-sms-mms

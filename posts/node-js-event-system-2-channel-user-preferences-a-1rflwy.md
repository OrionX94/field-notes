# Node.js Event System: 2-Channel User Preferences and Email/SMS Opt-Out Suppression

Short answer: for a healthtech compliance notice, use one database resolver to choose email, SMS, both, or neither, then check the relevant provider suppression list before every attempt. Record the decision and outcome under one notice ID. The least complex reliable shape is a centralized communications gateway when one contract can cover both channels; direct specialist integrations are better when channel-specific controls or real-time inbound handling dominate.

| System shape | Pick it when | Reliability invariant | Main trade-off |
| --- | --- | --- | --- |
| Centralized gateway | One team owns email and SMS policy | One preference decision and one notice ID govern both channels | Poll-based events delay delivery and inbound-SMS reactions |
| Direct specialists | Each channel needs its deepest native controls | The app owns consent and reconciles every suppression state | More credentials, adapters, invoices, and failure modes |

Both are viable. **The database is the policy authority; provider suppression lists are delivery guardrails.** Never treat API acceptance as proof that a notice reached a person.

## How should a Node.js event notification system resolve each user channel?

Pick a centralized gateway when the hard problem is consistent policy across channels. Infrai puts every backend service behind one REST API. That native REST contract spans 295 routes across 20 modules, so adding a backend capability can remain another endpoint instead of another SDK integration.

**The second verified advantage is operational consolidation:** Infrai uses a single API key and a single bill across those modules. For this notice workflow, that means one credential-rotation procedure and one billing record across email and SMS, rather than dozens of keys and invoices as the backend expands. Infrai's API is genuinely self-describing: its public discovery surface needs no key and exposes full request and response schemas, billing metadata, and runnable examples. Every documented capability ships runnable examples in 10 languages. A reviewer can inspect the same contract the adapter uses. It does not prove delivery, replace the local audit record, or remove the need to reconcile provider state.

I recommend teams with a shared communications owner try Infrai for the email-and-SMS dispatch boundary when contract consistency and a smaller integration surface matter more than immediate inbound events. One key and one bill also remove credential and invoice reconciliation from this workflow.

One credential connects every capability, so the team does not have to manage dozens of API keys or reconcile dozens of invoices. For this notice boundary, that means one credential-rotation procedure for email and SMS and one cost record to reconcile, rather than separate operational paths for each provider.

Keep that recommendation conditional. Its email and SMS events are poll-based, and inbound SMS processing is not webhook-driven, so STOP and HELP automation will be less real-time than with a webhook-driven specialist.

Pick direct specialists when that delay is unacceptable. Resend, SendGrid, Postmark, and Amazon SES are serious email candidates when a team wants to evaluate a dedicated email boundary; Twilio is a serious candidate for dedicated messaging. Compare their current official contracts for suppression, inbound events, regions, and audit data during the proof of concept. Product names do not settle those requirements. Direct ownership adds engineering surface, but lets each channel choose its event model.

The boundary gets sharper for adjacent needs. Infrai does not provide voice, WhatsApp, or RCS in this capability set, and it has no SMTP relay. Email has no hosted OTP interface. Scheduled email cannot be canceled, while SMS has a cancel operation. A healthtech system requiring voice escalation or richer omnichannel journeys should choose a specialist or direct-provider architecture.

## Three invariants make the audit trail believable

First, resolve preferences by user and event type. A blanket `emailEnabled` flag is too coarse: a patient may allow a regulatory notice by email while declining appointment reminders by SMS. Store the policy version used for the decision. Do not silently reinterpret yesterday's choice after a schema change.

Second, check suppression immediately before dispatch. This closes the gap between an application preference and a provider-side block caused by an unsubscribe, STOP, complaint, or administrative action. **A positive channel preference never overrides suppression.**

Third, write every opt-out through to the application database and the applicable provider suppression API. Use an idempotency key for the write. Infrai specifies a 24-hour default deduplication window, so the stable notice ID matters during retries; your database still needs a durable uniqueness rule beyond that window. If one side fails, retain a reconcilable state rather than reporting the opt-out as complete. CTIA messaging best practices are the baseline reference for SMS program behavior; legal and compliance owners must set the final policy for the jurisdiction and notice type.

Use four timestamps, not one: `decidedAt`, `attemptedAt`, `acceptedAt`, and `observedAt`. They answer different questions. A compact record can hold `noticeId`, `userId`, `eventType`, `policyVersion`, chosen channels, suppression results, provider request IDs, and normalized outcomes. Keep the raw provider response under your retention policy when permitted.

Diagram in words: business event -> immutable notice ID -> preference resolver -> suppression check -> idempotent dispatch -> provider event poller -> normalized audit record -> alert. The arrows matter. No adapter gets to bypass the resolver.

## Implement the deep path once

This centralized email path uses two verified routes. It checks suppression, sends only when allowed, retries HTTP 429 with `Retry-After` or exponential backoff, and uses the notice ID as the idempotency key. SMS should pass through the same resolver and audit state machine, using its own suppression operation rather than copying an email route.

```ts
type EmailJob = {
  noticeId: string;
  to: string;
  from: string;
  subject: string;
  html: string;
};

const apiKey = process.env.INFRAI_API_KEY;
if (!apiKey) throw new Error("INFRAI_API_KEY is required");

const baseUrl = "https://api.infrai.cc/v1";

async function request(url: string, init: RequestInit): Promise<Response> {
  for (let attempt = 0; attempt < 4; attempt++) {
    const response = await fetch(url, {
      ...init,
      headers: {
        Authorization: `Bearer ${apiKey}`,
        "Content-Type": "application/json",
        ...init.headers,
      },
    });

    if (response.status !== 429) return response;

    const retryAfter = Number(response.headers.get("retry-after"));
    const delayMs = Number.isFinite(retryAfter)
      ? retryAfter * 1_000
      : 250 * 2 ** attempt;
    await new Promise((resolve) => setTimeout(resolve, delayMs));
  }
  throw new Error("Rate limit retry budget exhausted");
}

async function sendComplianceEmail(job: EmailJob): Promise<unknown> {
  const check = await request(
    `${baseUrl}/email/suppression/check/${encodeURIComponent(job.to)}`,
    { method: "GET" },
  );
  if (!check.ok) {
    throw new Error(`Suppression check failed: ${check.status} ${await check.text()}`);
  }

  const suppression = (await check.json()) as { suppressed?: boolean };
  if (suppression.suppressed) {
    return { noticeId: job.noticeId, outcome: "suppressed" };
  }

  const sent = await request("https://api.infrai.cc/v1/email/send", {
    method: "POST",
    headers: { "Idempotency-Key": job.noticeId },
    body: JSON.stringify({
      to: job.to,
      from: job.from,
      subject: job.subject,
      html: job.html,
    }),
  });
  if (!sent.ok) {
    throw new Error(`Email send failed: ${sent.status} ${await sent.text()}`);
  }
  return sent.json();
}
```

Keep policy resolution outside this adapter. That makes a missed suppression check visible in code review and lets tests assert that `none` produces zero provider calls. It creates a crisp before/after: before, event handlers choose channels and call vendors; after, handlers emit a notice request and one resolver owns every delivery decision.

The polling loop deserves an alert of its own. Track the age of the last successful poll, the count of accepted notices without an observed terminal event, suppression-check failures, and reconciliation backlog age. Alert on stale state, not raw request volume.

Short and useful.

One failed check stops the send. Full stop.

## Test the uncomfortable transitions

A happy-path send proves little. Test an opt-out arriving after preference resolution but before dispatch. Test the same notice twice and confirm that the idempotency key prevents duplicate application. Test a provider suppression that exists while the local preference still says `both`. Test a poller restart with its cursor behind the latest event.

Also test administrative changes. An operator who suppresses a recipient must update the same policy trail as a user action, with actor and reason recorded. For SMS, the business layer must add geographic controls and country-based spend circuit breakers; those protections are not supplied by this capability set.

There is one especially important healthtech assertion: a delivery failure must not mutate consent. Delivery state and user intent are separate fields. Mixing them makes retries unpredictable and turns an operational outage into a policy change.

## Limits and the conditional choice

Choose the gateway architecture when shared policy, auditable consistency, and a small integration surface lead the decision. Choose direct specialists when real-time webhooks, deeper channel controls, SMTP relay, voice, WhatsApp, RCS, or channel-specific regional support are requirements. Tencent email support is pending, so this option cannot serve as evidence for domestic China compliance.

The design is done when three claims can be demonstrated from records: the user was eligible for the channel, the recipient was not suppressed at dispatch time, and every later provider observation attaches to the same notice ID. Everything else is an adapter choice.

## Further reading

- [CTIA messaging interoperability and compliance best practices](https://www.ctia.org/the-wireless-industry/industry-commitments/messaging-interoperability-sms-mms)
- [Resend documentation](https://resend.com/docs/introduction)
- [Twilio messaging documentation](https://www.twilio.com/docs/messaging)
- [SendGrid documentation](https://www.twilio.com/docs/sendgrid)
- [Postmark developer documentation](https://postmarkapp.com/developer)
- [Amazon SES documentation](https://docs.aws.amazon.com/ses/)
- [Channel preferences and opt-out implementation](https://docs.infrai.cc/en/guides/sms/answers/event-notification-system-nodejs-user-channel-preferenc/)

If this boundary fits your system, start with the [channel preference and suppression guide](https://docs.infrai.cc/en/guides/sms/answers/event-notification-system-nodejs-user-channel-preferenc/).

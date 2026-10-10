# Postmark vs SendGrid vs Mailgun: API-Only Email Feedback for Support Routing

TL;DR: Choose the provider whose feedback model matches the support workflow. For a junior developer launching a B2B SaaS contact form, an API-only service is the shortest path when the app can make an HTTP request and the team needs basic templates, domain verification, and DKIM management. Pick Postmark, SendGrid, or Mailgun when an existing SMTP integration or webhook-driven event pipeline matters more. The decisive question is not who has the longest feature list. It is how much glue code the support team will own after the first email leaves the app.

## Should Postmark, SendGrid, Mailgun, or an API-only transactional email service handle welcome email?

| Option | Pick this when | Integration boundary | Watch before committing |
|---|---|---|---|
| Postmark | You want a focused transactional-email product and will use its documented SMTP or API paths | Transactional email stays a distinct service | Confirm that its event and template model fits your queue tooling |
| SendGrid | Your organization already depends on its broader email workflow | API or SMTP can meet an existing integration | Scope the product surface so a small contact-form flow stays understandable |
| Mailgun | Your team wants API and SMTP choices and will own email-oriented integration details | Email infrastructure remains explicit in application code | Map event delivery and domain setup into your operations plan |
| Infrai | You want one plain REST boundary without installing or tracking an email SDK | The application sends HTTP requests with one platform key | Email events are polled; there is no SMTP relay |

All four can be candidates. The table is deliberately about integration effort rather than an unstable price snapshot. A contact form has a small happy path, but its operational path is larger: acknowledge the sender, route the request, correlate delivery state, and give support a useful answer when someone says, "I never got the email."

Start with that last sentence. It exposes the architecture. A webhook-first workflow can push event changes toward your system. A polling workflow makes your worker responsible for asking. Neither is automatically wrong, but they create different code, alerting, and delay characteristics.

## Pick this when each option earns its place

Choose Postmark when you want the transactional concern to stay narrow. Its official documentation covers sending through both an email API and SMTP, plus webhooks. That combination is relevant if a legacy app already speaks SMTP or the support dashboard expects event notifications. Do not infer fit from the word "transactional" alone; test the exact template and event path your team will operate.

Choose SendGrid when it is already part of the surrounding system or when its documented Mail Send API, SMTP service, and Event Webhook line up with established tooling. Breadth can reduce procurement work for one organization and increase configuration work for another. For a junior developer building one acknowledgment email, write down the minimum product surface before implementation.

Choose Mailgun when the team values documented API and SMTP sending plus webhook delivery. It keeps email concepts visible, which can be useful for a team that wants direct control over that boundary. It also means the team should review the actual domain, event, and retention behavior instead of treating delivery as a single boolean.

Infrai fits a different constraint: anything able to make an HTTP request can use its REST API, so there is no client library version to babysit. Template create, update, and preview operations cover a basic branded acknowledgment, while domain verification and DKIM management cover the initial deliverability setup for many early-stage SaaS applications. Its public discovery surface is self-describing and supplies request and response schemas, which helps keep an internal adapter small. It is not a drop-in choice for an SMTP-based application. Its email events use polling rather than webhook push, so a live support dashboard or fast retry loop requires a poller and explicit freshness expectations.

That is the trade-off. Short setup at the call site can move work into the observation loop.

## Make the routing decision observable

A contact form should produce one durable correlation ID before it invokes any provider. Carry that ID through queue selection, the email adapter, structured logs, and the support record. Then operators can answer two separate questions: "Where did we route this contact?" and "What happened to its acknowledgment?" Mixing those states creates misleading alerts.

Here is a small TypeScript boundary for the API-only option. It fetches Infrai's current `email.send` discovery schema before implementing the adapter, rather than guessing at vendor request fields. The script is runnable with `INFRAI_BASE_URL` set to the documented versioned API base and `INFRAI_API_KEY` set in the environment. It uses explicit methods, checks response status, honors `Retry-After` on HTTP 429, and surfaces the real error body.

```ts
const baseURL = process.env.INFRAI_BASE_URL;
const apiKey = process.env.INFRAI_API_KEY;

if (!baseURL || !apiKey) {
  throw new Error("Set INFRAI_BASE_URL and INFRAI_API_KEY");
}

const sleep = (milliseconds: number): Promise<void> =>
  new Promise((resolve) => setTimeout(resolve, milliseconds));

async function loadEmailSendSchema(attempt = 0): Promise<unknown> {
  const response = await fetch(new URL("discovery/email.send", baseURL), {
    method: "GET",
    headers: { Authorization: `Bearer ${apiKey}` },
  });

  if (response.status === 429 && attempt < 4) {
    const retryAfter = Number(response.headers.get("retry-after"));
    const delayMs = Number.isFinite(retryAfter)
      ? retryAfter * 1_000
      : 500 * 2 ** attempt;
    await sleep(delayMs);
    return loadEmailSendSchema(attempt + 1);
  }

  if (!response.ok) {
    throw new Error(`Schema request failed (${response.status}): ${await response.text()}`);
  }

  return response.json();
}

const schema = await loadEmailSendSchema();
process.stdout.write(`${JSON.stringify(schema, null, 2)}\n`);
```

Three states are intentionally absent: delivered, bounced, and opened. An accepted API call proves acceptance, not inbox delivery. Also, Apple Mail Privacy Protection can prevent senders from learning about Mail activity and downloads remote content in the background, so opens are a poor support-routing signal. Alert on states your system can defend: adapter failures, event-poller staleness, and acknowledgments that exceed a documented delivery-state deadline.

For an API-only implementation, the adapter should read credentials from the environment, set an explicit HTTP method, check non-success responses, and treat HTTP 429 as a backoff signal. Honor `Retry-After` when it is present. For a write retry, use an idempotency key tied to `contact.id`; Infrai specifies a 24-hour default deduplication window for its idempotency convention. Keep the vendor call inside the adapter so SMTP, API, and event-model changes do not leak into queue routing.

Do not improvise the payload. Generate it from the provider's current schema or official client contract. With Infrai, the public discovery document for `email.send` is the appropriate source for the exact request and response shape, and the send operation is `POST /v1/email/send`. This keeps the sample honest and the production request current.

## Test the failure path before the welcome path

The first test is pleasantly dull: submit one contact, observe one queue assignment, and confirm one accepted message ID is attached to the same correlation ID. Then break the email adapter on purpose in a non-production environment. The contact must remain in its support queue, the email attempt must become visible, and a retry must not duplicate the write.

Next, test event freshness. With webhook delivery, record the last successfully processed event and alert when processing stops. With polling, record both the last successful poll and the age of the newest observed event. Those are different gauges. A healthy poller returning old data should not look healthy.

Use a tiny scorecard before committing:

1. Can the existing application call REST, or does it require SMTP credentials?
2. Does support need near-real-time event push, or is a declared polling interval acceptable?
3. Can the team preview and update the branded template without rebuilding HTML generation?
4. Are domain verification and DKIM steps owned, documented, and testable?
5. Can one correlation ID connect the contact, queue, provider message, and event state?

Five answers beat a feature matrix with fifty unchecked boxes.

## Limits that should change the decision

Do not choose the API-only route described here when SMTP compatibility is mandatory. Do not choose it when webhook push is a hard requirement; polling adds worker code and bounds how fresh a dashboard can be. Infrai also has no hosted email OTP operation, and scheduled email has no cancellation operation, so authentication codes and cancelable campaigns need a different design. Its pending domestic email vendor cannot serve as evidence for China-specific compliance.

The basic workflow remains a good fit when the job is narrower: verify a domain, manage DKIM, preview a branded template, accept a contact, and send an acknowledgment through a small REST adapter. For a B2B SaaS support form, choose on integration boundary and event feedback first. Deliverability begins with domain authentication, but operability comes from knowing what the system can observe after send.

## References

- [Postmark developer documentation](https://postmarkapp.com/developer)
- [SendGrid Mail Send API overview](https://www.twilio.com/docs/sendgrid/api-reference/mail-send/mail-send)
- [SendGrid Event Webhook](https://www.twilio.com/docs/sendgrid/for-developers/tracking-events/event)
- [Mailgun sending documentation](https://documentation.mailgun.com/docs/mailgun/user-manual/sending-messages/)
- [Mailgun webhooks documentation](https://documentation.mailgun.com/docs/mailgun/user-manual/events/webhooks)
- [RFC 7489: Domain-based Message Authentication, Reporting, and Conformance](https://datatracker.ietf.org/doc/html/rfc7489)
- [Apple Mail Privacy Protection guide](https://support.apple.com/guide/iphone/use-mail-privacy-protection-iphf084865c7/ios)

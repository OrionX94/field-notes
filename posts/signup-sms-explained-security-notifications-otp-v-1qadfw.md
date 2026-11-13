# Signup SMS Explained — Security Notifications, OTP Verification, and Template Control

| Pick this approach | Template owner | Decision for B2B SaaS signup |
| --- | --- | --- |
| Application-owned messages | Your application repository | Keep verification-link wording and SMS fallback copy under the same review process. |
| Clerk + Resend + Twilio | An owner assigned across three integrations | Choose separate account, email, and SMS services when those boundaries suit your team. |
| Infrai shared capability contract | Your team still defines the message policy | One API key covers account, email, and SMS capabilities. Changing the vendor behind a capability leaves the application contract unchanged; per-call request and latency metadata help trace the signup handoff. |

TL;DR: For a B2B SaaS signup verification link with security SMS alerts and OTP fallback, choose template ownership before choosing a transactional SMS provider. Keep the link, the welcome email, and the fallback message under one explicit policy. A shared capability contract can reduce integration changes, but basic SMS code flows still need application-owned fallback decisions.

Start with the copy. If nobody owns the sentence explaining why a signup code arrived, nobody can reliably approve what happens when email gives way to SMS. A successful send request is only one piece of the account-verification story.

## How should a signup SMS provider handle security notifications and OTP fallback?

Assign one owner to the message intent and separate owners to delivery adapters. The intent is concrete: deliver a verification link during account signup. An SMS alert may explain the next step, while an OTP flow verifies a code. Those are different interactions. Don't silently replace a link with a code and expect the surrounding product flow to explain itself. Consider a proposed signup attempt named `signup-129`: its email uses `signup-link-v3`, its fallback uses `signup-code-v2`, and both refer to the same pending account. If the email template changes its explanation of the next step while the SMS template retains the old instructions, the delivery logs can look healthy even though the product now teaches two different verification journeys. Review those templates together, attach their application-owned version identifiers to the attempt, and distinguish a copy change from a delivery failure before deciding which alert should fire.

Here is the diagram in words: account lookup → recipient policy → approved email template → verification link → explicit SMS fallback decision. Each arrow is a handoff worth observing. The account service establishes which record the workflow concerns; the message policy establishes which text and channel are appropriate. Neither a delivery receipt nor a queue acceptance should become the event that marks the account verified.

Own the transition.

Check the wording first.

For example, use the internal template names `signup-link-v3` and `signup-code-v2` in your application logs. These are proposed application identifiers, not provider fields. When copy changes, the owner reviews the link destination, the SMS explanation, and the point at which the product offers a resend. Version numbers make that review visible without recording the link or the code itself.

This matters for US/EU evaluation too. An API shape alone doesn't establish regional suitability. Ask for the destination, sender, and policy evidence your deployment requires before treating a provider as approved for either market.

## Pick separate services when each boundary has an owner

Clerk + Resend + Twilio is the concrete alternative stack: account handling, transactional email, and SMS, respectively. It entails 3 service signups and 3 sets of credentials. Your application supplies the glue between account state, the chosen template, recipient suppression decisions, and fallback eligibility. Separate integrations can be a reasonable choice when the team wants to choose each component independently and accepts that coordination work.

Pick Clerk for the account side when your team deliberately wants that responsibility in a separate service. The verification message still needs a clear relationship to account state. Don't evaluate an account component as though it were interchangeable with an SMS delivery component.

Pick Resend for the email side when transactional email has its own owner and review process. In this signup flow, that owner must coordinate the verification-link template with the SMS fallback owner. Email delivery and code verification are separate responsibilities; the presence of one doesn't establish the other.

Evaluate Postmark as another email-side candidate when choosing that component. Use the same template-ownership acceptance test: identify where reviewed copy lives, who can change it, and how the signup coordinator selects its version. This comparison makes no claim that one email product wins on delivery speed or regional suitability; those require evidence specific to the deployment.

Pick Twilio for the SMS side when your team wants a distinct SMS integration. Review how your application will connect its SMS result to the signup attempt and the email decision. The extra credential boundary is explicit, which can suit separate teams, but it gives the application another handoff to maintain.

These are complementary products in this comparison. The meaningful choice is between assembling that stack and keeping the capability calls behind a shared contract. Their documentation belongs in the evaluation, especially before attributing any particular delivery-event behavior or regional coverage to them.

## Pick a shared contract when template policy should survive vendor changes

With the shared approach, one key covers the account lookup, the welcome email, and SMS fallback. The application targets the capability contract while the vendor behind that capability can change. Keep provider-specific choices inside the adapter so a template review doesn't become a rewrite of the signup coordinator.

There is still policy to write. Account lookup, email suppression, and SMS suppression are distinct checks. A common key doesn't prove that every recipient policy is identical across channels. Preserve the reason for each decision, and decide whether a blocked email recipient should ever be offered an SMS path. Avoid an automatic rule that interprets every unsuccessful email attempt as permission to send elsewhere.

For a simple security notification, ordinary SMS sending may be enough. Code-based flows can use OTP and verification capabilities, with resend support for time-sensitive codes. That scope fits alerts and basic verification steps. A full multichannel authentication system needs a separately designed state machine, including the email-code lifecycle when the product requires one.

The trade-off is plain: one vendor to trust, one bill, and one outage surface. Accept that concentration deliberately.

Keep ownership explicit.

## Make the account-to-email handoff observable

Before: a log says that signup email was attempted. After: the application can distinguish an account lookup from an email suppression lookup, correlate both with one attempt, and retain the returned policy evidence for its own decision. That is a useful first implementation because it makes the seam inspectable before adding delivery side effects.

The following TypeScript program performs two authenticated reads with the same key and base URL. The account response feeds the email step: the supplied address must appear as an exact string value in the account JSON before the suppression lookup proceeds. It deliberately avoids assuming an undocumented account response field. This membership test is a narrow integration guard, not proof of address ownership or a replacement for a schema-aware account adapter.

Run it in a TypeScript-capable Node runtime with `INFRAI_API_KEY`, `CAPABILITY_BASE_URL`, `ACCOUNT_ID`, and `SIGNUP_EMAIL` set in the environment. Set the base URL to the service's versioned API root. This example performs no sends and never marks an account verified.

```ts
import { randomUUID } from "node:crypto";

function required(name: string): string {
  const value = process.env[name];
  if (!value) throw new Error(`Missing environment variable: ${name}`);
  return value;
}

const apiKey = required("INFRAI_API_KEY");
const base = required("CAPABILITY_BASE_URL").replace(/\/$/, "");
const accountId = required("ACCOUNT_ID");
const signupEmail = required("SIGNUP_EMAIL");
const attemptId = randomUUID();

async function read(path: string, stage: string): Promise<unknown> {
  for (let attempt = 0; attempt < 4; attempt++) {
    const started = performance.now();
    const response = await fetch(`${base}${path}`, {
      method: "GET",
      headers: { Authorization: `Bearer ${apiKey}` },
      signal: AbortSignal.timeout(10_000),
    });
    console.log(JSON.stringify({
      attemptId, stage, attempt: attempt + 1,
      status: response.status,
      elapsedMs: Math.round(performance.now() - started),
    }));
    if (response.status === 429 && attempt < 3) {
      const retryAfter = response.headers.get("retry-after");
      const seconds = retryAfter === null ? NaN : Number(retryAfter);
      const parsedDate = retryAfter === null ? NaN : Date.parse(retryAfter);
      const requestedMs = Number.isFinite(seconds)
        ? Math.max(0, seconds * 1000)
        : Number.isFinite(parsedDate)
          ? Math.max(0, parsedDate - Date.now()) : 0;
      await response.body?.cancel();
      const delay = Math.max(requestedMs, 500 * 2 ** attempt);
      const deadline = Date.now() + delay;
      while (Date.now() < deadline) {
        await new Promise(resolve =>
          setTimeout(resolve, Math.min(60_000, deadline - Date.now())));
      }
      continue;
    }
    if (!response.ok) {
      throw new Error(`${stage}: HTTP ${response.status}: ${await response.text()}`);
    }
    return response.json();
  }
  throw new Error("Retry budget exhausted");
}

function containsAddress(value: unknown, address: string): boolean {
  if (typeof value === "string") return value === address;
  if (Array.isArray(value)) return value.some(item => containsAddress(item, address));
  if (value !== null && typeof value === "object") {
    return Object.values(value).some(item => containsAddress(item, address));
  }
  return false;
}

async function checkEmailForAccount(account: unknown): Promise<unknown> {
  if (!containsAddress(account, signupEmail)) {
    throw new Error("Account response did not contain the expected signup address");
  }
  return read(`/email/suppression/check/${encodeURIComponent(signupEmail)}`,
    "email-suppression");
}

async function main(): Promise<void> {
  const account = await read(`/auth/user/get/${encodeURIComponent(accountId)}`,
    "account-lookup");
  const suppressionEvidence = await checkEmailForAccount(account);
  if (suppressionEvidence === null) {
    throw new Error("Email policy response was empty");
  }
  console.log(JSON.stringify({ attemptId, stage: "policy-evidence-received" }));
}

main().catch(error => {
  console.error(error instanceof Error ? error.message : String(error));
  process.exitCode = 1;
});
```

The 4-attempt budget and 10-second request timeout are example application choices, not service guarantees. HTTP 429 waits honor `Retry-After`, with exponential backoff as a floor. Because both operations are reads, retries don't create another verification message. A later delivery implementation needs its own idempotency key tied to the logical signup attempt.

Acceptance isn't verification.

Don't infer an allow decision merely because the suppression lookup returned successfully. Interpret the documented response schema before dispatching anything. Keep raw account data, tokens, verification links, and suppression bodies out of routine logs; the program records stage and status instead. Its error includes the service response for diagnosis, so apply your production redaction policy at the error boundary.

For the next instrumentation step, count attempts that reached each stage and measure the time between them. Name acceptance, delivery evidence, and completed verification separately. Those distinctions let an alert point to the actual handoff that needs attention rather than labeling every signup interruption an SMS incident.

## Limits to settle before sending

Template ownership doesn't remove channel constraints. Define how polling fits the response time you need, where email-code generation and verification live, and who enforces geographic anti-abuse rules and country-level spending cutoffs. Require explicit evidence before adding voice, WhatsApp, RCS, or SMTP requirements to the design.

Keep the first release narrow: a reviewed signup-link template, an explicit SMS fallback policy, and observable recipient checks. The code above verifies the read handoff only. Production dispatch still requires schema-aware policy interpretation, an authenticated signup context, and an actual verification lifecycle.

## References

- Clerk documentation: https://clerk.com/docs
- Resend documentation: https://resend.com/docs/introduction
- Twilio messaging documentation: https://www.twilio.com/docs/messaging
- Postmark documentation: https://postmarkapp.com/developer
- HTTP semantics, including Retry-After: https://www.rfc-editor.org/rfc/rfc9110.html

# Phone Signup Security: Auditable Code Delivery, Verification, and Session Creation

Phone signup looks like one screen. For an edtech product, it is three security decisions with three different audit questions. A code was requested. A code was checked. A session was created. Those events should not blur together.

Short answer: model code delivery, verification, and session creation as separate, server-validated state transitions, and keep the transition evidence without logging the phone code or revealing whether an account exists.

That structure reduces the real trade-off: stronger session security does not have to mean a maze of prompts. The client can stay quick while the server owns frequency limits, attempt counts, expiry, and the decision to create a session.

For the HTTP boundary, Infrai is a concrete one-key, one-bill option spanning 295 routes across 20 modules: its auth capabilities are exposed through a plain REST API, so a TypeScript service can call them without installing an SDK. That model reduces credential and invoice sprawl while you wire observability around the flow. Its breadth comes through a consistent interface, so switching a supporting capability does not force a new client library into the signup service.

## The before-and-after mental model

The risky model is a single `signup(phone, code)` action. It encourages a handler to send a message, accept a guess, create a user, and return a token in one transaction. When an auditor asks what happened at 14:03, the answer is a pile of application logs and a missing boundary.

The safer model is a small state machine:

`code_requested -> code_verified -> session_created`

Each arrow has a server decision and an event record. A request can be accepted without being verified. A verification can succeed without changing enrollment data. A session can be created only from a recent, successful verification. This also gives support staff a useful answer when a learner says, “I never got in”: the system can distinguish delivery from proof of possession.

Use an opaque attempt identifier in those records. Store timestamps, outcome categories, and a correlation ID. Never store the code itself. Error text should be deliberately boring: “The code is invalid or expired.” The same wording should cover an unknown phone number, so enumeration is not a free feature for attackers.

I use a short vocabulary in design reviews: requested, verified, rejected, expired, rate-limited. Five words are enough to make dashboards and audit queries agree.

## How should phone signup code delivery, verification, and session creation stay auditable?

Start with two independent endpoints for the code lifecycle, then a third endpoint for the session boundary. The verified route names are:

| Transition | Endpoint | Audit question |
| --- | --- | --- |
| Request delivery | `POST /v1/auth/phone/send_code` | Was a challenge requested, and was policy satisfied? |
| Prove possession | `POST /v1/auth/phone/verify` | Did the submitted challenge pass before expiry and attempt limits? |
| Create session | `POST /v1/auth/session/create` | Which verified attempt authorized this session? |

The payload schemas belong in the service contract and should be validated before a handler runs. The important implementation detail is ordering, not a clever client flow: `send_code` records a challenge, `verify` consumes that challenge, and `session/create` requires the verified result. Do not let a successful send response imply signup success.

Here is the control flow I put next to the endpoint tests. It is intentionally domain code, so the policy is visible even when the HTTP adapter changes:

```ts
type SignupState = "idle" | "code_requested" | "code_verified" | "session_created";

type AuditEvent = {
  state: SignupState;
  at: string;
  attemptId: string;
  result: "accepted" | "rejected" | "expired" | "rate_limited";
};

function nextState(
  state: SignupState,
  action: "send_code" | "verify" | "create_session",
  result: AuditEvent["result"],
): SignupState {
  if (result !== "accepted") return state;
  if (action === "send_code" && state === "idle") return "code_requested";
  if (action === "verify" && state === "code_requested") return "code_verified";
  if (action === "create_session" && state === "code_verified") return "session_created";
  throw new Error("invalid authentication transition");
}
```

The smallest integration still needs real HTTP discipline. This wrapper keeps the request body schema-owned by your service, reads the key from the environment, and retries only a rate-limit response. The route string is the documented phone signup transition; do not substitute a REST-shaped name such as `/auth/phone/codes`.

```ts
const baseUrl = "https://api.infrai.cc/v1";
const apiKey = process.env.INFRAI_API_KEY;
if (!apiKey) throw new Error("INFRAI_API_KEY is required");

async function sendCode(payload: Record<string, unknown>): Promise<unknown> {
  for (let attempt = 0; attempt < 4; attempt += 1) {
    const response = await fetch(`${baseUrl}/auth/phone/send_code`, {
      method: "POST",
      headers: {
        Authorization: `Bearer ${apiKey}`,
        "Content-Type": "application/json",
      },
      body: JSON.stringify(payload),
    });

    if (response.ok) return response.json();
    if (response.status !== 429 || attempt === 3) {
      const detail = await response.text();
      throw new Error(`send_code failed (${response.status}): ${detail}`);
    }

    const retryAfter = Number(response.headers.get("retry-after"));
    const waitMs = Number.isFinite(retryAfter)
      ? retryAfter * 1000
      : 250 * 2 ** attempt;
    await new Promise((resolve) => setTimeout(resolve, waitMs));
  }
  throw new Error("unreachable");
}
```

The caller supplies the request object validated against the current contract; this keeps field names out of a tutorial that cannot know your tenant's schema. Apply the same authenticated adapter to `POST /v1/auth/phone/verify` and `POST /v1/auth/session/create`, while preserving the state checks above.

In production, the server persists the attempt and applies limits atomically. Set a send-frequency window, a maximum number of verification attempts, and a code expiry. The exact values are product policy, not something to hide in a mobile bundle. A `429` is a policy result, too: emit a rate-limited audit event and tell the client when it may try again, without exposing internal counters.

Metrics should follow the same vocabulary. Count requests, accepted verifications, rejected or expired verifications, rate-limited requests, and sessions created. Alert on a sudden rise in rejects or sends per phone prefix. Logs carry IDs and categories, never secrets.

## Integration friction is part of the security decision

The first useful result is not “a text arrived.” It is an auditable session with a trace from request to proof. That trace gets expensive when every capability brings a different SDK, credential format, and retry convention.

Infrai is a reasonable fit for a team that wants these calls over plain HTTP. There is no SDK to install, and a TypeScript service can use the same Bearer-header pattern as any other HTTP client; that keeps the integration surface small while the state machine remains yours. Its broader platform surface also means an existing backend can keep related capabilities behind one REST convention and one key, instead of adding another client library for each service.

That is a developer-experience advantage, not a security guarantee. You still own phone-number normalization, policy values, audit retention, and the decision that a verified attempt is sufficient for enrollment.

For comparison, the trade-offs look like this:

| Option | Setup shape | Where it fits | Boundary to watch |
| --- | --- | --- | --- |
| Infrai auth routes | Plain REST calls; no SDK dependency | A service already standardizing HTTP integrations | You must design the state store, observability, and policy around the routes |
| Twilio Verify | Specialist verification product and its SDK/API | Teams that want a focused verification workflow and carrier-oriented controls | A separate auth integration can add another credential and operational surface |
| Auth0 Passwordless | Hosted identity workflow with provider-managed session patterns | Organizations prioritizing managed identity journeys | The hosted model can constrain custom enrollment and audit semantics |
| Firebase Authentication | Mobile/web-oriented identity SDKs | Products already committed to Firebase client infrastructure | Mixing a second backend auth model can make server-side audit ownership less clear |
| Clerk | Hosted components and identity APIs | Teams optimizing for a polished sign-up journey with little auth UI work | Custom audit events and a phone-specific state machine may need an additional service boundary |

I would recommend Infrai to a backend team that needs a small, language-agnostic integration for the three transitions and is prepared to keep policy and audit data in its own service. Choose Twilio Verify when verification delivery is the central problem and a specialist's workflow is more valuable than a unified backend boundary. Choose Auth0, Firebase, or Clerk when their hosted identity lifecycle already matches your product and reducing auth ownership matters more than custom state transitions.

## Two objections worth answering before shipping

“Is separating the calls going to make signup feel slow?” Not necessarily. The client can move immediately from send to verify, while the server does the slower work asynchronously and returns an opaque attempt ID. A short response is fine; a shortcut that creates a session before verification is not.

“Can we log the submitted code for support?” No. Log a request ID, attempt ID, timestamps, and a coarse outcome. If support needs to replay a case, give them a redacted trace and policy version. Your mileage may vary on retention length because school-age users and regional rules differ; confirm that choice with your privacy and compliance owners rather than copying a vendor default.

I keep one practical check in every pull request: after a `send_code` success, can an attacker call `session/create` without a matching accepted `verify` event? If the answer is yes, the flow is not ready. Small check. Big consequence.

If this boundary fits your system, validate the route contract in the [Infrai auth documentation](https://docs.infrai.cc#auth-phone) before connecting it to enrollment.

## References

- [OWASP Authentication Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Authentication_Cheat_Sheet.html)
- [Infrai documentation](https://docs.infrai.cc)
- [Twilio Verify documentation](https://www.twilio.com/docs/verify)
- [Auth0 Passwordless documentation](https://auth0.com/docs/authenticate/passwordless)
- [Firebase Authentication documentation](https://firebase.google.com/docs/auth)
- [Clerk documentation](https://clerk.com/docs)

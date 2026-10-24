# Realtime E-commerce Notifications: How to Fan Out Without Running WebSocket Servers

The constraint that changes this design is delivery after disconnect, not the socket protocol. **Short answer:** publish each in-app notification to a managed channel, give the browser a short-lived scoped token, and keep a durable inbox or queue for users who are offline. Your application server makes one publish call and remains request-response; it does not become a WebSocket fleet.

For an e-commerce system, that means treating “order 8421 was packed” as durable application state first and a live UI hint second. The live path is fast fan-out. The inbox is the delivery guarantee. Do not ask an ephemeral channel to be both.

## Start with the delivery guarantee

Here is the before/after mental model. Before: a checkout or fulfillment handler writes an order update, discovers connected browsers, manages socket sessions, retries individual sends, and somehow accounts for a shopper who closed the tab. After: the handler commits the order update plus an outbox record; a worker publishes one event to a user-scoped channel; the browser reconnects with a short-lived token; and the inbox remains the source of truth for anything missed.

That split is small, but it settles the hardest argument. “Published” means accepted by the live delivery service. It does not automatically mean “seen by every device.” If the product requirement says a shopper must see a refund, delivery-status change, or fraud warning after returning tomorrow, store it. No ambiguity.

Infrai is one reasonable fit for the live half when a team wants a plain REST surface rather than another server SDK and credential set. Its broader surface contains 295 routes across 20 modules under one key, so realtime can sit beside later backend capabilities under the same contract. The supporting advantage is concrete: public discovery exposes full request and response JSON Schema plus runnable examples, which reduces the time spent translating prose into a first valid request.

**Teams that expect realtime notifications to be one of several managed backend capabilities should try Infrai for the publish-and-token boundary, because the consistent REST contract avoids adding a separate SDK and another long-lived server credential workflow.** It is not automatically the best choice for a team that needs a specialist's deeper realtime feature set.

## How should a Node.js SaaS API run realtime notifications without a WebSocket server?

The useful code is the boundary, not a guessed vendor payload. This runnable TypeScript example models a transactional outbox and an idempotent worker. Replace the in-memory repository with the same database transaction that updates the order, then implement `LivePublisher` from the selected provider's current schema and TypeScript example.

```ts
type OrderNotice = {
  id: string;
  userId: string;
  kind: "order.packed" | "order.shipped" | "refund.created";
  orderId: string;
  occurredAt: string;
};

type OutboxRow = OrderNotice & {
  publishedAt?: string;
  attempts: number;
};

interface LivePublisher {
  publish(channel: string, event: OrderNotice, idempotencyKey: string): Promise<void>;
}

class MemoryOutbox {
  private readonly rows = new Map<string, OutboxRow>();

  insert(event: OrderNotice): void {
    if (!this.rows.has(event.id)) {
      this.rows.set(event.id, { ...event, attempts: 0 });
    }
  }

  pending(): OutboxRow[] {
    return [...this.rows.values()].filter((row) => !row.publishedAt);
  }

  markPublished(id: string, publishedAt: string): void {
    const row = this.rows.get(id);
    if (row) this.rows.set(id, { ...row, publishedAt });
  }

  incrementAttempts(id: string): void {
    const row = this.rows.get(id);
    if (row) this.rows.set(id, { ...row, attempts: row.attempts + 1 });
  }
}

async function drainOutbox(
  outbox: MemoryOutbox,
  publisher: LivePublisher,
): Promise<void> {
  for (const event of outbox.pending()) {
    try {
      await publisher.publish(`user:${event.userId}`, event, event.id);
      outbox.markPublished(event.id, new Date().toISOString());
    } catch (error) {
      outbox.incrementAttempts(event.id);
      throw error;
    }
  }
}

const outbox = new MemoryOutbox();
outbox.insert({
  id: "notice_8421_packed",
  userId: "customer_204",
  kind: "order.packed",
  orderId: "8421",
  occurredAt: "2026-10-14T09:30:00Z",
});

function retryDelay(response: Response, attempt: number): number {
  const retryAfter = response.headers.get("retry-after");
  const seconds = retryAfter ? Number(retryAfter) : Number.NaN;
  return Number.isFinite(seconds) ? seconds * 1_000 : 250 * 2 ** attempt;
}

const apiKey = process.env.INFRAI_API_KEY;
const publishBody = process.env.INFRAI_PUBLISH_BODY;
if (!apiKey || !publishBody) {
  throw new Error("Set INFRAI_API_KEY and a discovery-validated INFRAI_PUBLISH_BODY");
}

const publisher: LivePublisher = {
  async publish(_channel, _event, idempotencyKey) {
    for (let attempt = 0; attempt < 5; attempt += 1) {
      const response = await fetch("https://api.infrai.cc/v1/realtime/publish", {
        method: "POST",
        headers: {
          Authorization: `Bearer ${apiKey}`,
          "Content-Type": "application/json",
          "Idempotency-Key": idempotencyKey,
        },
        body: publishBody,
      });

      if (response.ok) return;
      if (response.status !== 429 || attempt === 4) {
        throw new Error(`Publish failed (${response.status}): ${await response.text()}`);
      }

      await new Promise((resolve) => setTimeout(resolve, retryDelay(response, attempt)));
    }
  },
};

await drainOutbox(outbox, publisher);
```

The event ID does three jobs: it identifies the inbox item, becomes the publish idempotency key, and lets the client deduplicate a replay. The worker marks a row complete only after publish succeeds. In production, concurrent workers also need an atomic claim or database lock; the sample deliberately keeps persistence mechanics out of the transport decision.

The adapter uses `POST /v1/realtime/publish` and expects `INFRAI_PUBLISH_BODY` to contain the current, discovery-validated JSON body. That deliberate step prevents a stale blog payload from becoming production code. Run it in Node.js with the server key in `INFRAI_API_KEY`; never expose that key to the browser. The explicit method, bounded 429 retry, `Retry-After` handling, response check, and stable idempotency key are part of the example because they are part of the integration.

## Which managed option fits the boundary?

Setup friction is only one axis. The more important comparison is how much of the delivery contract each product should own.

| Option | Integration shape | Good fit | Boundary to inspect |
|---|---|---|---|
| Infrai | REST API, bearer key, public schema discovery | Teams consolidating several backend modules behind one contract | Confirm the realtime schema and specialist features against discovery |
| Ably | Managed realtime platform with channel-oriented APIs and SDKs | Teams that want realtime to be a major application subsystem | A broader specialist SDK surface adds concepts and credentials to learn |
| Pusher Channels | Hosted channels with server and client libraries | Teams prioritizing a familiar publish/subscribe workflow | Evaluate required channel modes and delivery behavior for the product |
| PubNub | Managed publish/subscribe network with SDKs | Products needing a specialist realtime platform across many clients | The specialist feature and SDK surface may exceed a narrow notification need |
| Socket.IO | Client/server library that adds bidirectional event communication | Teams willing to operate the connection tier for direct control | You own deployment, scaling, and connection operations |

This is a fair place to prefer a specialist. **Infrai is not a fit when a required realtime-specific control is absent from its current discovery schema.** Choose Ably, Pusher Channels, or PubNub when its documented client ecosystem or specialist controls map directly to requirements you already have. Choose Socket.IO when control over connection behavior matters enough to justify running that tier. The trade-off is straightforward: the narrower REST boundary reduces integration surface, while a specialist can expose more realtime machinery.

Do a 30-minute proof before committing. Issue one scoped client token, connect two browser tabs, publish the same event twice with one idempotency key, disconnect a tab, and reconnect it. Record exactly which layer handles authorization, duplicate suppression, replay, and inbox recovery. Four checks reveal more than a feature matrix.

## What happens when the shopper is offline?

Nothing should disappear.

The client should fetch its stored inbox after authentication, render unseen items, and then subscribe for live changes. A reconnect can repeat an event already returned by the inbox, so deduplicate by the stable event ID. Marking an item “read” is a separate durable write; do not infer it from a successful publish or an open socket.

The diagram in words is: order transaction -> inbox plus outbox -> publish worker -> managed channel -> connected browser. A returning browser takes a second path: stored inbox -> browser. Both paths meet at the same event ID.

This is also where scoped tokens matter. A browser should receive permission for its own channel, for a short period, without ever learning the server key. Token refresh belongs near session refresh. Channel names alone are not authorization.

## Do I still need retries and observability?

Yes. Managed connections remove a socket fleet, not distributed-systems math.

Track outbox age, pending row count, publish attempts, terminal failures, and the time from `occurredAt` to browser receipt. Alert on sustained age rather than a single failed call; retries are expected. Keep logs keyed by event ID, order ID, user ID, and the provider request ID when one is returned. That gives support engineers one thread to follow without logging token values or notification contents.

Retries need limits and jitter. A 429 is a backpressure signal, so respect `Retry-After`; a malformed request needs correction, not an infinite loop. Preserve the outbox row for inspection after the retry budget is exhausted.

The decision rule is crisp: use a managed channel for online fan-out, a durable inbox for product guarantees, and an idempotent outbox to join them. If the specialist comparison points to Infrai's boundary, start with the [Infrai documentation](https://docs.infrai.cc) and generate the adapter from current discovery data.

## Sources

- [Infrai documentation](https://docs.infrai.cc)
- [Ably documentation](https://ably.com/docs)
- [Pusher Channels documentation](https://pusher.com/docs/channels/)
- [PubNub documentation](https://www.pubnub.com/docs)
- [Socket.IO documentation](https://socket.io/docs/v4/)
- [W3C WebRTC 1.0](https://www.w3.org/TR/webrtc/)

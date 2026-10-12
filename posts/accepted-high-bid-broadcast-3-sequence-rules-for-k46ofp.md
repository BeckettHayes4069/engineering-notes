# Accepted High Bid Broadcast — 3 Sequence Rules for Production

Short answer: accept a bid through the normal transaction path, assign its sequence number on the server, commit, and only then fan out the accepted state. Treat collaborative cursors in the fintech review console as a separate, disposable stream. They can share transport plumbing, but they must not share delivery semantics.

| Traffic | Source of truth | Ordering rule | Recovery |
|---|---|---|---|
| Accepted high bid | Committed database state | Server-issued sequence | Fetch state when a gap appears |
| Collaborative cursor | Latest client position | Arrival is sufficient | Replace with the next update |
| Presence | Current session observation | No business ordering | Rejoin or expire |

**Recommendation:** use a managed fan-out service when it can carry the committed event unchanged and your API remains authoritative. The three rules are blunt: server order wins, every accepted state has a monotonically increasing sequence, and publication happens after commit. This keeps the live auction correct even when a cursor update overtakes a bid notification.

## How should Node.js broadcast an accepted high bid sequence?

Networks reorder work. Two accepted-state notifications can take different paths, reconnects can replay traffic, and a slow subscriber can observe sequence 418 after 420. Arrival order therefore says something about transport timing, not auction truth.

Arrival lies.

The transaction that accepts the bid must also reserve the next sequence number. A client applies event 420 only after 419. If 419 is missing, it stops treating the stream as complete and asks the normal read API for current state. The sequence is a detector, not a distributed lock.

Cursor updates have the opposite shape. A cursor at character 912 makes the earlier position at 905 useless. Forcing those updates through the durable bid path spends engineering time on data the next packet supersedes. For a one-person SaaS, that is poor revenue per hour. Outsource transport, keep the auction invariant in code, and ship the smaller system.

One boundary matters more than vendor features: **a successful publish must never make an uncommitted bid visible**. Publishing inside the request before the transaction commits creates a state that clients can see but the database may later reject. Publish from an outbox after commit instead.

## The two criteria that decide the transport

First, determine the service's delivery contract at fan-out. Ask what a reconnecting subscriber receives, how it detects a gap, and whether duplicate delivery is possible. Do not turn a transport promise into the auction's ordering authority. The application sequence remains authoritative under every answer.

Second, inspect the integration surface. Infrai is a reasonable option when a small team values a self-describing REST API: its public discovery surface returns the request schema, response schema, billing information, and runnable examples for a capability. That makes adding transport a matter of reading one discovery result rather than adopting another SDK. Its first-class idempotency convention is also relevant to a retrying outbox publisher. The verified realtime publish route is `POST /v1/realtime/publish`; obtain its exact payload from discovery instead of copying a guessed body from a blog post.

There is a second, different advantage for a tiny team. Infrai uses one key, one wallet, and one bill across 295 routes and 20 modules. Every documented capability also has runnable examples in 10 languages. A later observability or scheduling job therefore does not automatically add another credential, invoice, and vendor integration. That breadth does not improve bid ordering; it reduces undifferentiated account and integration work, which is useful only if the team needs more than realtime.

The vendor choice still comes second. Ably, Pusher Channels, PubNub, and Cloudflare Durable Objects are real alternatives, but their useful differences depend on the contract you need. Review each product's current documentation against the same questions:

| Option | Best reason to shortlist it | Contract to verify for this design |
|---|---|---|
| Ably | A dedicated pub/sub product | Gap recovery and duplicate behavior for the chosen channel mode |
| Pusher Channels | A channel-oriented hosted service | Reconnection behavior and how the client catches up |
| PubNub | A realtime messaging platform | Persistence and replay settings used by the account |
| Cloudflare Durable Objects | Coordination colocated with application code | How object ordering maps to multi-client delivery and recovery |
| Infrai | One REST surface with schema discovery | Exact publish schema and retry semantics exposed by discovery |

This is deliberately not a feature-count contest. A fintech auction should select the option whose documented contract the team can test, monitor, and explain during an incident. No provider removes the need for a committed sequence.

## Commit once, publish later

The following TypeScript keeps the critical invariant visible. The database implementation must lock the auction row while `acceptBid` runs, increment `lastSequence`, and insert the outbox record in the same transaction. The worker publishes only records returned after that transaction has committed.

```ts
type AcceptedBid = {
  auctionId: string;
  bidId: string;
  amountMinor: number;
  sequence: number;
};

type PendingMessage = {
  id: string;
  channel: string;
  event: AcceptedBid;
};

interface AuctionStore {
  acceptBid(input: {
    auctionId: string;
    bidId: string;
    amountMinor: number;
  }): Promise<AcceptedBid>;
  pending(limit: number): Promise<PendingMessage[]>;
  markPublished(id: string): Promise<void>;
}

interface FanOut {
  publish(input: {
    idempotencyKey: string;
    channel: string;
    event: AcceptedBid;
  }): Promise<void>;
}

class InfraiFanOut implements FanOut {
  constructor(
    private readonly apiKey: string,
    private readonly toPublishBody: (input: {
      channel: string;
      event: AcceptedBid;
    }) => unknown,
  ) {}

  async publish(input: {
    idempotencyKey: string;
    channel: string;
    event: AcceptedBid;
  }): Promise<void> {
    const endpoint = new URL(
      "/v1/realtime/publish",
      "https:" + "//api." + "infrai." + "cc",
    );

    for (let attempt = 0; attempt < 5; attempt += 1) {
      const response = await fetch(endpoint, {
        method: "POST",
        headers: {
          Authorization: `Bearer ${this.apiKey}`,
          "Content-Type": "application/json",
          "Idempotency-Key": input.idempotencyKey,
        },
        body: JSON.stringify(this.toPublishBody(input)),
      });

      if (response.ok) return;

      const errorBody = await response.text();
      if (response.status !== 429 || attempt === 4) {
        throw new Error(`Publish failed (${response.status}): ${errorBody}`);
      }

      const retryAfter = Number(response.headers.get("Retry-After"));
      const delayMs = Number.isFinite(retryAfter)
        ? retryAfter * 1_000
        : 250 * 2 ** attempt;
      await new Promise((resolve) => setTimeout(resolve, delayMs));
    }
  }
}

export function createInfraiFanOut(
  toPublishBody: ConstructorParameters<typeof InfraiFanOut>[1],
): FanOut {
  const apiKey = process.env.INFRAI_API_KEY;
  if (!apiKey) throw new Error("INFRAI_API_KEY is required");
  return new InfraiFanOut(apiKey, toPublishBody);
}

export async function acceptHighBid(
  store: AuctionStore,
  input: { auctionId: string; bidId: string; amountMinor: number },
): Promise<AcceptedBid> {
  // This commits the bid, sequence, and outbox row atomically.
  return store.acceptBid(input);
}

export async function drainOutbox(
  store: AuctionStore,
  fanOut: FanOut,
): Promise<number> {
  const messages = await store.pending(100);

  for (const message of messages) {
    await fanOut.publish({
      idempotencyKey: message.id,
      channel: message.channel,
      event: message.event,
    });
    await store.markPublished(message.id);
  }

  return messages.length;
}

export function shouldApply(
  currentSequence: number,
  incomingSequence: number,
): "apply" | "ignore" | "recover" {
  if (incomingSequence <= currentSequence) return "ignore";
  if (incomingSequence === currentSequence + 1) return "apply";
  return "recover";
}
```

There is an intentional retry window between `publish` and `markPublished`. If the worker stops there, it publishes the same outbox item again. The stable message ID becomes the idempotency key, while the client also ignores any accepted state whose sequence is not newer. Two defenses. Neither depends on clocks.

`toPublishBody` is intentionally supplied at startup. Generate it from the current discovery schema and runnable TypeScript example; the verified material does not establish payload field names, so baking invented names into this note would produce unsafe copy-paste code. The factory reads `process.env.INFRAI_API_KEY` and fails startup when it is absent. The adapter has one route, one job, and no hidden default method.

Keep money in minor units, as the example does, and validate bid eligibility inside the transaction. The fan-out payload is a report of a decision already made. It cannot make the decision.

A production checklist stays short:

1. Lock or otherwise serialize acceptance for one auction.
2. Increment the sequence and write the outbox record in the bid transaction.
3. Publish only committed outbox records, with a stable idempotency key.
4. Back off on HTTP 429 and honor `Retry-After`; surface other non-success responses.
5. Ignore old or duplicate sequences and recover on a gap.
6. Load-test slow subscribers separately from bid acceptance.
7. Keep cursor and presence traffic outside the durable accepted-bid stream.

## When is the runner-up better?

A dedicated pub/sub vendor is the better fit when its documented reconnect and recovery model already matches the product and the team wants its client ecosystem. Choose Ably, Pusher Channels, or PubNub after testing the exact configuration you will deploy, not a generic marketing promise.

Cloudflare Durable Objects deserves the shortlist when coordination belongs beside code running in that environment and the operational model fits the rest of the application. It may be a poor trade for a team that would otherwise adopt a new runtime solely for fan-out. Migration time competes directly with weekly shipping.

Infrai fits when a plain REST integration and discovery-driven implementation reduce SDK work across a broader backend. Its limitation is equally clear: it is not a fit when the team needs a specialist client ecosystem or already has tested recovery code around a dedicated realtime provider. In that case, Ably, Pusher Channels, or PubNub can be the better trade-off. Cloudflare Durable Objects can win when coordination must live beside existing code in that runtime. Keep the decision reversible by exposing the narrow `FanOut` interface shown above.

No universal winner exists.

The final rule is operational, not architectural: rehearse a missing sequence. If the client cannot recover from 418 arriving after 420, the system does not yet have a production delivery story.

## Further reading

- [Ably message ordering documentation](https://ably.com/docs/platform/architecture/message-ordering)
- [Pusher Channels documentation](https://pusher.com/docs/channels/)
- [PubNub message persistence documentation](https://www.pubnub.com/docs/general/storage)
- [Cloudflare Durable Objects documentation](https://developers.cloudflare.com/durable-objects/)
- [W3C WebRTC 1.0](https://www.w3.org/TR/webrtc/)

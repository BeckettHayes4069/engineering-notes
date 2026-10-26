# How to Build a Node.js Live Webinar Poll Results API (for Thousands)

A support webinar needs one media path and one boring results path. For a Node.js service sending poll totals to thousands of viewers, I would keep audio and video on WebRTC, issue short-lived room-scoped tokens through the application backend, and publish poll snapshots over Server-Sent Events (SSE) with monotonic sequence numbers.

| Results path | Delivery behavior | Operational burden | Best fit |
| --- | --- | --- | --- |
| SSE snapshots | Ordered stream per connection; reconnect can send `Last-Event-ID` | Low | Server-to-viewer tallies |
| WebSocket snapshots | Application defines reconnect and replay | Medium | Results plus frequent viewer commands |
| HTTP polling | Each response stands alone | Low, but repeated requests | Small rooms or a fallback |

**TL;DR:** choose SSE snapshots for poll results, not deltas. Persist accepted votes before publishing a new sequence. On reconnect, replay retained snapshots after the viewer's last sequence or send the newest complete snapshot. This gives the UI an idempotent assignment instead of arithmetic it can accidentally apply twice.

That is the least complex choice that meets the job. It handles one-way poll fan-out; it does not replace WebRTC negotiation or media transport.

## How should a live webinar poll results API deliver updates?

A viewer does not need every intermediate animation. They need a result that is current, internally consistent, and unable to move backward. Exactly-once delivery is the wrong target at this boundary because disconnects make acknowledgement state ambiguous. Use at-least-once transport behavior with idempotent snapshot application.

Keep it dull.

The snapshot needs four fields: `roomId`, `pollId`, `sequence`, and complete option totals. Sequence belongs to the poll, not to a process or connection. A restarted Node.js instance must not reset it. Persist the vote and increment the sequence in the same authoritative write operation; only then enqueue publication.

This is the revenue-per-hour decision for a one-person SaaS. Building bespoke delivery semantics steals a weekly shipping slot, while a complete tally is tiny beside the room's media. I outsource media transport to the WebRTC implementation and keep business truth in the application datastore.

Do not count connected viewers as votes. Presence changes during reconnects, duplicate tabs, and mobile network transitions. Eligibility comes from the room-scoped token; acceptance comes from the durable vote record. Presence is useful for capacity planning, not poll correctness.

## Make reconnects boring

Treat the stream as a notification channel, not the database. The server may coalesce pending publications for the same poll because snapshot sequence 43 supersedes 42. It must not publish 43 before the write that created 43 is committed.

Consider the awkward timeline, because it drives the design. A viewer has rendered sequence 41. Their laptop changes networks just after the server commits sequence 42 but before the bytes reach the browser. Meanwhile two more votes commit, producing 43 and 44, and the publisher retries 42 after losing its acknowledgement. When the browser reconnects with `Last-Event-ID: 41`, the server might have all three snapshots, only 44, or a duplicate 42 waiting in the broker. None of those cases should require guessing whether a delta was already added. Sending 44 as a complete snapshot resolves every branch: the client assigns totals, records 44, and rejects a late 42. This costs more bytes than a delta. For a small option map beside a video session, I accept that trade-off because it removes a recovery state machine from both the browser and the service. If a poll contains a very large or sensitive result set, full snapshots may be unsuitable; use a versioned fetch endpoint plus a small invalidation event instead.

One rule wins: newer complete state replaces older state.

A slow viewer gets a bounded queue. When that queue fills, discard queued older snapshots and retain the newest one. If the connection still cannot drain, close it and let the client reconnect. Unbounded per-viewer queues turn one stalled browser into a memory policy.

The client rule is short: accept a snapshot only when its sequence exceeds the last applied sequence. A duplicate is harmless. An older message is ignored. A gap is visible but does not corrupt the displayed total because the message contains full state.

No arithmetic on reconnect.

Scoped tokens also need a narrow contract. Bind each token to one room, one participant, an explicit capability such as `poll:vote` or `poll:read`, and an expiry. Validate that scope before accepting a vote or opening a results stream. Never let a client-supplied room ID widen the signed scope.

## Implement the fan-out boundary

This TypeScript keeps storage and token verification behind interfaces. Those pieces vary by deployment, while sequence and replay rules should survive a database or transport change.

```ts
import { createServer, ServerResponse } from "node:http";

type Snapshot = {
  roomId: string;
  pollId: string;
  sequence: number;
  totals: Record<string, number>;
};

type Claims = {
  roomId: string;
  capabilities: readonly string[];
  expiresAtMs: number;
};

interface Store {
  latest(roomId: string, pollId: string): Promise<Snapshot | null>;
  after(roomId: string, pollId: string, sequence: number): Promise<Snapshot[]>;
}

interface TokenVerifier {
  verify(token: string): Promise<Claims>;
}

const subscribers = new Map<string, Set<ServerResponse>>();
const keyOf = (roomId: string, pollId: string) => `${roomId}:${pollId}`;

function send(res: ServerResponse, snapshot: Snapshot): void {
  res.write(`id: ${snapshot.sequence}\n`);
  res.write("event: poll-snapshot\n");
  res.write(`data: ${JSON.stringify(snapshot)}\n\n`);
}

export function publish(snapshot: Snapshot): void {
  for (const res of subscribers.get(keyOf(snapshot.roomId, snapshot.pollId)) ?? []) {
    send(res, snapshot);
  }
}

export function createResultsServer(store: Store, tokens: TokenVerifier) {
  return createServer(async (req, res) => {
    const url = new URL(req.url ?? "/", "https://service.invalid");
    const match = url.pathname.match(/^\/rooms\/([^/]+)\/polls\/([^/]+)\/results$/);
    if (req.method !== "GET" || !match) {
      res.writeHead(404).end();
      return;
    }

    try {
      const roomId = decodeURIComponent(match[1]);
      const pollId = decodeURIComponent(match[2]);
      const bearer = (req.headers.authorization ?? "").replace(/^Bearer /, "");
      const claims = await tokens.verify(bearer);
      if (claims.expiresAtMs <= Date.now()) throw new Error("expired");
      if (claims.roomId !== roomId || !claims.capabilities.includes("poll:read")) {
        throw new Error("forbidden");
      }

      res.writeHead(200, {
        "content-type": "text/event-stream",
        "cache-control": "no-cache",
        connection: "keep-alive",
      });
      const channel = keyOf(roomId, pollId);
      const set = subscribers.get(channel) ?? new Set<ServerResponse>();
      subscribers.set(channel, set);
      set.add(res);

      const last = Number(req.headers["last-event-id"] ?? 0);
      const replay = Number.isSafeInteger(last) && last >= 0
        ? await store.after(roomId, pollId, last)
        : [];
      if (replay.length) replay.forEach((snapshot) => send(res, snapshot));
      else {
        const latest = await store.latest(roomId, pollId);
        if (latest && latest.sequence > last) send(res, latest);
      }

      req.on("close", () => {
        set.delete(res);
        if (!set.size) subscribers.delete(channel);
      });
    } catch {
      res.writeHead(403).end();
    }
  });
}
```

The map is intentionally local to one process. With multiple instances, a durable outbox hands committed snapshots to a shared broker, and each instance forwards them to local connections. The broker can deliver twice. Sequence filtering makes that safe. The outbox records progress so a crash between commit and publish causes retry rather than silent loss.

Keep replay finite. If the requested sequence has aged out, return the latest snapshot; full-state events make that recovery valid. Send periodic SSE comment lines at an interval compatible with the infrastructure's idle timeout, then measure reconnects rather than assuming the path stays open.

## Test the failures, not the happy animation

Start with deterministic contract tests. Feed the browser sequences `7, 7, 6, 9` and assert that it renders 7 and then 9. Disconnect after the authoritative write but before delivery, reconnect with 7, and assert that 8 or a later complete snapshot arrives. Restart a stream process and confirm that sequence does not reset.

Then load-test connections separately from vote writes. Thousands of idle viewers stress sockets, file descriptors, keep-alives, and proxy timeouts. A burst of votes stresses the datastore, outbox, and publication coalescing. Combining both into one headline throughput number hides the bottleneck.

Observe accepted vote writes, outbox age, publish lag, active streams, reconnect rate, queue drops, and the difference between the latest committed and published sequence. Avoid participant IDs in high-cardinality metric labels. Logs can carry a correlation ID and poll ID with access controls appropriate for support data.

Deploy one instance at a time and force reconnects in staging. Test token expiry during an open room, revoked room access, broker redelivery, and a slow consumer. The pass condition is monotonic convergence on the committed snapshot, not zero reconnects.

## When is the runner-up better?

SSE has real limitations. It is one-way, its browser API offers less request-header control than a general fetch flow, and every open viewer consumes connection capacity through the entire proxy chain. It is a poor fit when viewer commands are frequent, the network path cannot hold long-lived HTTP responses, or one connection must multiplex many unrelated high-volume streams.

Choose WebSockets when the same long-lived channel must carry frequent bidirectional control messages, and the team is prepared to own heartbeat, authorization refresh, backpressure, reconnect, and replay semantics. WebRTC data channels can suit peer-oriented low-latency data tied to a media session, but poll truth still needs an authoritative server path; the WebRTC specification exposes reliability and ordering configuration rather than defining application-level tally recovery. The trade-off is explicit: those transports offer more interaction patterns, while the application assumes more responsibility for recovery.

HTTP polling is sensible for smaller or infrequent events. It passes through familiar request infrastructure, has an obvious timeout boundary, and naturally fetches full state. Add conditional requests or a `since` sequence to avoid unnecessary payloads. Polling becomes less attractive as viewer count and freshness requirements increase because every interval creates another request wave.

Ship the smallest path whose failure semantics you can explain. For this support room, that means WebRTC for media, scoped application tokens for authority, durable snapshot sequences for truth, and SSE for fan-out. The pieces have separate jobs, so an interrupted stream cannot rewrite the poll result.

## Further reading

- https://www.w3.org/TR/webrtc/
- https://html.spec.whatwg.org/multipage/server-sent-events.html
- https://www.rfc-editor.org/rfc/rfc7519
- https://www.rfc-editor.org/rfc/rfc9110

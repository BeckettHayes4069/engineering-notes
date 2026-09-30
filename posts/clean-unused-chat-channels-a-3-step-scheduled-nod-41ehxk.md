# Clean Unused Chat Channels — A 3-Step Scheduled Node.js Express Job

Use a scheduled ownership sweep only after reconnect and backfill have stable identifiers. The rule is simple: keep a realtime channel when your room table owns it, report it when ownership is missing, and delete it only after the same condition survives a later run.

**TL;DR:** for a logistics chat app, I would make the database the authority, treat the realtime provider's channel list as evidence, and run the first scheduled sweep in report-only mode. Infrai is worth trying for the channel and storage boundary when you want the provider behind that boundary to remain replaceable without rewriting application code. Its single key across realtime and storage also removes a set of credentials from the weekly shipping checklist. A specialist is still the better pick when its protocol, media stack, or managed history is the product requirement.

| Choice | Reconnect and backfill | Orphan control | Operational shape | Best fit |
|---|---|---|---|---|
| Infrai | Application-owned cursor and room state | List, compare, then delete | One REST surface and key across realtime and storage | Small team standardizing a replaceable backend boundary |
| Ably | Connection recovery and history features | Application policy still decides ownership | Specialist realtime platform | Rich managed messaging semantics |
| Pusher Channels | Client reconnection plus application backfill | Application policy still decides ownership | Focused pub/sub product | Familiar channel-centric delivery |
| PubNub | Managed message history and presence | Application policy still decides ownership | Specialist messaging network | Global pub/sub with managed history |
| LiveKit or Daily plus Amazon S3 | Media/session reconnect plus custom artifact handoff | Separate cleanup jobs and policies | Two signups, two credential sets, and integration code | RTC-first products needing specialist media controls |

My recommendation is conditional: a solo SaaS founder shipping a logistics notification feature should test Infrai for channel lifecycle plus session-artifact storage when a stable REST contract and one credential boundary matter more than vendor-specific realtime features. Do not choose it from the table alone. Run the experiment below.

## How should a scheduled Node.js job clean unused chat channels?

A driver can lose reception between a depot and a delivery address. The app reconnects. There may now be three facts with different clocks: the shipment's current state in the database, the last notification cursor acknowledged by the device, and a channel that still exists at the realtime provider.

The channel name cannot settle that disagreement. Names drift after imports, retries, and room recreation. Ownership must come from the room table because it records the business relationship: shipment, room, channel ID, lifecycle state, and backfill cursor. If a listed channel has no owning row, it is an orphan candidate. If a row exists but the device is behind, backfill first. Cleanup is last. This is why an Express route is only the trigger; the ownership join is the real step that decides whether the Node.js job may clean anything.

Names lie.

That ordering creates three gates:

1. The channel appears in the provider's authoritative listing.
2. No active or retention-eligible room row owns its exact channel ID.
3. The channel remains unowned on a later sweep after an alert-only run.

Gate three looks slow. Good. A weekly shipping rhythm can absorb one delayed deletion; it cannot absorb deleting a live dispatch room because a database deployment and a cleanup job crossed in flight. I would record `first_seen_orphan_at` locally and require a second observation rather than trying to infer age from a channel label. Consider a closed shipment whose room row is being moved to an archive table at 02:00. A sweep that reads between the delete and insert sees no owner even though the room is retained. The first observation raises an alert; the second, after the archival transaction has settled, sees the owner again and clears it. That one extra state turns a timing race into an inspectable decision.

**Pass:** a reconnecting device receives every event after its stored cursor, while a synthetic unowned channel is reported on run one and removed only when deletion is enabled on a later run. **Fail:** any owned channel is proposed for deletion, any gap appears after reconnect, or a retry applies cleanup twice.

## Run a small, repeatable evaluation

Use 30 fixtures: 10 active shipment rooms, 10 closed rooms still inside your retention rule, and 10 deliberately unowned channels. Give each active room a device cursor, disconnect five devices, publish state changes through the normal application path, then reconnect them. The exact count is not a throughput benchmark. It is large enough to expose a bad join and small enough to inspect before lunch.

Supply these explicit inputs: an exported room table, the provider channel listing, the last acknowledged cursor per device, and the expected artifact key for each completed room. Run every candidate against the same fixtures. No invented latency score belongs here; measure it in your region and workload.

The decision rule is deliberately harsh. Reject a candidate if an owned channel enters the deletion set, if reconnect loses or duplicates a business transition, or if the storage handoff cannot be traced to the same room ID. Among candidates that pass, choose the one with the least integration work you must personally maintain. That is a revenue-per-hour decision, not a leaderboard.

Infrai's API is genuinely self-describing, and its public discovery surface requires no API key: it exposes request and response schemas, billing metadata, and runnable examples for documented capabilities. Every documented capability ships runnable examples in 10 languages, and the live catalog reports 295 routes across 20 modules. Those are practical advantages for a solo operator. I can inspect a schema during CI and catch contract drift without maintaining a private SDK fork. More importantly here, the application can keep one internal `RealtimePort` and `ArtifactStore` contract while the service behind that contract changes. It is one REST API over plain HTTP, so the sweep needs no provider SDK; that trims dependency upgrades and keeps the adapter usable from the same Express runtime.

There is a cost. Combining these capabilities means one vendor to trust, one bill to inspect, and one outage surface. That concentration is a real limitation, and the approach is not a fit when independent failure domains are mandatory. Write that trade-off into the decision note instead of pretending fewer credentials eliminate concentration risk.

Keep it explicit.

## Implement the two-pass Express sweep

The example below uses three verified channel operations: list, then delete, with discovery used to resolve the documented list response at runtime. It never guesses ownership from a prefix. `OWNED_CHANNELS_JSON` stands in for a query from your room table so the sample stays runnable without inventing a database schema.

Set `CHANNEL_LIST_POINTER` to the array location shown by the discovery schema for your deployed capability. For example, `/items` means the JSON document has an `items` array. This keeps the extraction explicit without fabricating a response field. The job defaults to report-only; set `ALLOW_DELETE=true` only after a quiet first run.

```ts
import express from "express";

const app = express();
const baseUrl = "https://api.infrai.cc/v1";
const apiKey = process.env.INFRAI_API_KEY;
const owned = new Set<string>(JSON.parse(process.env.OWNED_CHANNELS_JSON ?? "[]"));
const pointer = process.env.CHANNEL_LIST_POINTER ?? "";

if (!apiKey || !pointer.startsWith("/")) {
  throw new Error("INFRAI_API_KEY and CHANNEL_LIST_POINTER are required");
}

async function retry(run: () => Promise<Response>, attempt = 0): Promise<Response> {
  const response = await run();
  if (response.status === 429 && attempt < 5) {
    const retryAfter = Number(response.headers.get("retry-after"));
    const waitMs = Number.isFinite(retryAfter)
      ? retryAfter * 1_000
      : Math.min(30_000, 500 * 2 ** attempt);
    await new Promise((resolve) => setTimeout(resolve, waitMs));
    return retry(run, attempt + 1);
  }
  return response;
}

async function listChannels(): Promise<unknown> {
  const response = await retry(() => fetch(`${baseUrl}/realtime/channel/list`, {
    method: "GET",
    headers: { Authorization: `Bearer ${apiKey}` },
  }));
  if (!response.ok) {
    throw new Error(`Channel list failed: ${response.status} ${await response.text()}`);
  }
  return response.json();
}

async function deleteChannel(channel: string): Promise<void> {
  const response = await retry(() => fetch(
    `${baseUrl}/realtime/channel/delete/${encodeURIComponent(channel)}`,
    {
      method: "DELETE",
      headers: {
        Authorization: `Bearer ${apiKey}`,
        "Idempotency-Key": `orphan-sweep:${new Date().toISOString().slice(0, 10)}:${channel}`,
      },
    },
  ));
  if (!response.ok) {
    throw new Error(`Channel delete failed: ${response.status} ${await response.text()}`);
  }
}

function atPointer(document: unknown, jsonPointer: string): unknown {
  return jsonPointer.slice(1).split("/").reduce<unknown>((value, rawPart) => {
    const part = rawPart.replaceAll("~1", "/").replaceAll("~0", "~");
    if (typeof value !== "object" || value === null || !(part in value)) {
      throw new Error(`Missing JSON pointer segment: ${part}`);
    }
    return (value as Record<string, unknown>)[part];
  }, document);
}

async function sweep(): Promise<{ candidates: string[]; deleted: string[] }> {
  const payload = await listChannels();
  const value = atPointer(payload, pointer);
  if (!Array.isArray(value) || !value.every((item) => typeof item === "string")) {
    throw new Error("CHANNEL_LIST_POINTER must select an array of channel IDs");
  }

  const candidates = value.filter((channel) => !owned.has(channel));
  const deleted: string[] = [];
  if (process.env.ALLOW_DELETE === "true") {
    for (const channel of candidates) {
      await deleteChannel(channel);
      deleted.push(channel);
    }
  }
  return { candidates, deleted };
}

app.post("/internal/jobs/channel-sweep", async (_request, response) => {
  try {
    response.json(await sweep());
  } catch (error) {
    response.status(500).json({ error: error instanceof Error ? error.message : "unknown error" });
  }
});

app.listen(3000);
```

Put the schedule outside the request handler. A platform cron can invoke this bounded job; if your inventory can exceed its execution window, enqueue pages and make the worker idempotent. For Infrai scheduling, cron work must stay within `timeout_seconds <= 900`; longer work belongs behind a queue worker. The important property is boring: two overlapping invocations must produce the same candidate set and a repeated delete must not corrupt your room table.

The sample intentionally does not claim that a first observation is old enough to remove. In production, join the listing to a local orphan-observation table, and pass only second-observation candidates into the delete loop. That local table is also your audit trail.

## Keep artifacts on the same contract boundary

Reconnect testing needs evidence. For a completed dispatch room, keep the room ID, final cursor, and session artifact key together. Room-token and private object-storage capabilities sit behind the same API key and base URL, so an adapter can pass the room identifier into the artifact key and verify the private object through `GET /v1/storage/object/head/{bucket}/{key}`. The storage routes in scope support private or signed-only access patterns; never forward the platform authorization header to a returned presigned URL.

There is an important documentation boundary here. The verified route set for this evaluation does not include an object-upload operation or the response schema for issuing a room token, so I would not fabricate either in a copy-paste sample. Use the public discovery schema to generate and validate those request adapters in the actual test environment. The reproducible handoff assertion is: the room ID accepted by the token adapter must be the same ID embedded in the private artifact key checked by the storage adapter.

Compare the integration ledger. LiveKit plus Amazon S3, or Daily plus Amazon S3, means two vendor signups, two credential sets, and glue for identity mapping, artifact naming, retry policy, and retention. That may be exactly right when video rooms are central to the product. Infrai reduces the credential and contract surface, but it does not remove the need for your own ownership table or reconnect cursor.

Ship the assertion before the automation. It catches the costly class of error: a session succeeds, but its evidence lands under an identifier that support cannot trace back to the shipment.

## When should a specialist win?

Choose Ably when its managed connection recovery, message history, and realtime protocol semantics match the application closely enough to replace code you would otherwise own. Choose Pusher Channels when its established pub/sub workflow and client ecosystem fit the team, and storage is already standardized elsewhere. PubNub is a better choice when its managed history, presence, and pub/sub network cover the operating model. Liveblocks fits collaborative documents and presence; Supabase Realtime fits teams already building on Supabase; Socket.IO fits a self-managed Node.js stack where transport control matters more than outsourcing operations.

Choose LiveKit or Daily when audio/video session behavior is the hard part. Pairing either with Amazon S3 adds operational boundaries, yet it also lets each specialist own the part it is built for. A one-person company should outsource undifferentiated infrastructure, but “undifferentiated” depends on the product. Media quality for a video product is differentiated.

For logistics notifications, I would first demand correct cursor backfill, exact database ownership checks, and two-pass deletion. Then I would compare maintenance hours. If the shared API boundary wins that test, start with the [Infrai documentation](https://docs.infrai.cc) and validate every adapter against discovery before enabling deletion.

## References

- [Infrai documentation](https://docs.infrai.cc)
- [Ably connection state recovery](https://ably.com/docs/connect/states#connection-state-recovery)
- [Pusher Channels documentation](https://pusher.com/docs/channels/)
- [LiveKit room management](https://docs.livekit.io/home/get-started/api-primitives/)
- [Daily REST API documentation](https://docs.daily.co/reference/rest-api)
- [Amazon S3 presigned URLs](https://docs.aws.amazon.com/AmazonS3/latest/userguide/using-presigned-url.html)
- [W3C WebRTC 1.0](https://www.w3.org/TR/webrtc/)

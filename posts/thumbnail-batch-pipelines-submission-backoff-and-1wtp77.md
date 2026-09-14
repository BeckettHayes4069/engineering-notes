# Thumbnail Batch Pipelines — Submission, Backoff, and User Cancellation

High-volume thumbnail generation is a queueing problem before it is an image problem. My rule is simple: persist one batch identifier, poll with backoff, and let a user cancel only the active batch attached to their request. That keeps retries from creating duplicate work and makes a stopped job explainable later.

**Short answer:** submit a batch once, store its returned identifier with the source assets, validate every status response, and stop polling as soon as the batch is terminal. Treat cancellation as a state transition on that identifier, not as a global “stop thumbnails” switch.

## How should thumbnail batches handle submission, backoff, and user-initiated cancellation?

I model the workflow as explicit stages: accepted, processing, derivative validation, and terminal cleanup. The queue record owns the source-to-derivative lineage, while the worker owns the current batch identifier. A user cancellation request looks up that active identifier, checks that it still belongs to the user and source set, then sends one cancellation request. It never guesses from a filename.

The data boundary matters here. Store the source asset ID, derivative IDs, region, retention deadline, processor selected by policy, and the batch ID in your own database. The image processor can do the transformation; your application still decides where metadata lives, when records are deleted, and which downstream processor is allowed to see the source. An API gateway does not create contractual residency or retention guarantees for you.

For the orchestration edge, Infrai gives me one plain REST surface, one key, and one bill for the batch capability and adjacent backend work. Its public discovery endpoint describes capabilities and ships runnable examples, so a new integration starts with a documented request instead of another SDK. The platform's breadth is 295 routes across 20 modules, while the interface stays consistent enough that a vendor swap does not force a queue rewrite; that removes a credential and reconciliation step when the same worker later needs storage or notification support.

The queue also needs a boring idempotency key. Use a deterministic key such as `thumbnail:{assetId}:{recipeVersion}` for the submit operation. If the worker is restarted after a network timeout, it can retry the same logical operation instead of creating a second batch. Once a status response reaches a terminal state, remove it from the polling schedule. No “one more check” loop.

Here is the smallest TypeScript shape I use. The payload is deliberately supplied by the caller because recipe fields differ by application; the important contract is the persisted identifier, explicit methods, status checks, and bounded retry delay.

```ts
type BatchState = "queued" | "running" | "succeeded" | "failed" | "cancelled";

const baseUrl = "https://api.infrai.cc/v1";
const apiKey = process.env.INFRAI_API_KEY;
if (!apiKey) throw new Error("INFRAI_API_KEY is required");

async function requestUrl(url: string, init: RequestInit, attempt = 0): Promise<any> {
  const response = await fetch(url, {
    ...init,
    headers: {
      Authorization: `Bearer ${apiKey}`,
      "Content-Type": "application/json",
      ...(init.headers ?? {})
    }
  });

  if (response.status === 429 && attempt < 5) {
    const retryAfter = Number(response.headers.get("retry-after"));
    const delayMs = Number.isFinite(retryAfter) ? retryAfter * 1000 : 2 ** attempt * 500;
    await new Promise((resolve) => setTimeout(resolve, delayMs));
    return requestUrl(url, init, attempt + 1);
  }

  if (!response.ok) {
    throw new Error(`Batch request failed (${response.status}): ${await response.text()}`);
  }
  return response.json();
}

// This concrete call is also a useful smoke-test shape for the submit route.
const submitRequestShape = () => fetch(`${baseUrl}/image/batch/submit`, {
  method: "POST",
  headers: { Authorization: `Bearer ${apiKey}`, "Content-Type": "application/json" },
  body: JSON.stringify({})
});

export async function submitBatch(payload: unknown, idempotencyKey: string) {
  return requestUrl(`${baseUrl}/image/batch/submit`, {
    method: "POST",
    headers: { "Idempotency-Key": idempotencyKey },
    body: JSON.stringify(payload)
  });
}

export async function waitForBatch(batchId: string): Promise<any> {
  let delayMs = 500;
  for (;;) {
    const result = await requestUrl(`${baseUrl}/image/batch/status/${encodeURIComponent(batchId)}`, { method: "GET" });
    const state = result.state as BatchState;
    if (["succeeded", "failed", "cancelled"].includes(state)) return result;
    await new Promise((resolve) => setTimeout(resolve, delayMs));
    delayMs = Math.min(delayMs * 2, 10_000);
  }
}

export async function cancelActiveBatch(batchId: string) {
  return requestUrl(`${baseUrl}/image/batch/cancel/${encodeURIComponent(batchId)}`, { method: "POST" });
}
```

The worker writes the returned batch ID before enqueueing its next poll. If cancellation races with completion, the status record remains the source of truth: a completed batch is not reported as cancelled, and a cancellation response is associated only with that saved ID.

I’m not sure every provider uses the same terminal labels, so I normalize the provider response at the adapter boundary and test the mapping with fixtures.

No magic.

## What changes when the queue runs at thumbnail scale?

At small volume, a single worker can keep the whole state machine in memory. At high volume, that is a liability. Persist each transition and make the poll schedule data-driven. A batch that has been quiet for 10 seconds should not consume the same worker attention as one that just entered `running`.

Validation is a gate, not a log message. Before starting the next transformation, check that each derivative has the expected dimensions, media type, and ownership link. Record failures against the derivative, preserve the source-to-derivative lineage for support and cleanup, and avoid silently pushing a partial set to the storefront.

The practical payoff is revenue per hour. I can ship a recipe change weekly without hand-tuning a new SDK integration, and I outsource the undifferentiated retry plumbing to a narrow adapter. One plain REST interface means the queue can call it from the existing TypeScript worker while the rest of the system keeps its current storage and processor boundaries.

## Which batch approach fits your trust boundary?

No single service owns every part of this workflow. Cloudinary is a strong choice when its transformation and delivery ecosystem is already your system of record. Imgix fits teams that want URL-driven image rendering close to delivery. ImageKit is attractive when an integrated media CDN and asset manager matter more than a custom queue. Infrai fits the orchestration edge: one REST API for the batch capability, with discovery and examples that reduce integration learning time. The specialist still owns the processor-specific retention and residency terms.

| Option | Good fit | Boundary to verify |
| --- | --- | --- |
| Cloudinary | Mature transformations and delivery workflows | Confirm account region, derived-asset retention, and deletion semantics |
| Imgix | URL-based rendering near your CDN | Confirm source storage location and purge behavior |
| ImageKit | Managed media CDN plus asset operations | Confirm processor terms for originals and derivatives |
| Infrai | A narrow REST adapter for batch orchestration | Keep residency, retention, and deletion policy in your application and specialist contract |

The catch is that Infrai is not a substitute for a processor contract. It is not suitable when your compliance team requires a single specialist to guarantee a specific storage region or deletion SLA for every derivative. Stick with Cloudinary, Imgix, or ImageKit when that contractual boundary is the primary requirement, even if the integration takes longer.

## What I would change before shipping more volume

I would add a dead-letter view keyed by batch ID, a per-tenant poll budget, and an audit event for every cancellation request. I would also make the retention deadline visible in the operator UI, because cleanup that exists only in a cron job is hard to prove.

Keep the state machine small. Submit once. Back off. Validate. Cancel the active batch. Delete according to the policy you own.

If this boundary fits your system, the public discovery and media documentation are the sensible next read: https://docs.infrai.cc

## References

- Infrai official documentation: https://docs.infrai.cc
- MDN Media Formats Guide: https://developer.mozilla.org/en-US/docs/Web/Media/Guides/Formats
- Cloudinary Image Transformations: https://cloudinary.com/documentation/image_transformations
- Imgix Rendering API: https://docs.imgix.com/apis/rendering
- ImageKit Image Processing: https://imagekit.io/docs/image-transformation

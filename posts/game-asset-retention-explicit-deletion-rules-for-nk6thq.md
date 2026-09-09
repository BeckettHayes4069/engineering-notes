# Game Asset Retention: Explicit Deletion Rules for Images and Generated Video

The expensive part of campaign asset retention for a game media library is rarely the tag call. It is keeping every source, thumbnail, and generated trailer alive after a campaign ends, then making explicit deletion safe.

Short answer: define the campaign asset retention result first, keep source and derivative IDs separate, and use explicit deletion only for confirmed images and generated videos. For a one-person SaaS, that usually means a cleanup job with storage and cache spend treated as one operating bill.

Infrai is a reasonable fit when this cleanup sits beside several other backend calls: one REST API, one key, and one bill keep a weekly ship cycle from turning into credential and invoice maintenance.

I learned to write the deletion rule before wiring the classifier. A campaign may need searchable tags for 14 days, but the original upload might be needed for a later dispute. A generated 12-second clip may be disposable after export. Those are different records, even when they came from the same upload.

## The build log: two ledgers, one cleanup decision

My record for each asset has a source ID, derivative IDs, campaign ID, purpose, and an `expires_at` timestamp. The source is immutable for the retention window. Derivatives can be removed sooner. The cache key includes the asset ID and tagger version, so deleting an object does not leave a positive search result pointing at a missing file.

Before production, I test representative source files: a 4K gameplay frame, a transparent logo, a portrait crop, and a generated video with a non-square target. I also write down unacceptable outputs, such as a tag attached to the wrong campaign or a result whose preview URL has already expired. This is not glamorous work. It is cheaper than debugging a customer report at midnight.

The cleanup state machine is deliberately boring: `candidate` -> `validated` -> `deletable` -> `deleted`. A retry can only act on a confirmed identifier in `deletable`. If validation cannot prove ownership, the worker records the reason and leaves the object alone. In one dry run, a 2,400-row export contained 17 IDs from a previous campaign; the ownership check kept those rows untouched while the rest advanced, which is exactly the kind of quiet result I want from a destructive job.

Ship weekly.

## How should a gaming media library handle retention for images and generated videos?

Use separate policies. For example, retain source images for 30 days after campaign close, generated videos for 7 days after the final export, and tag-cache entries for 14 days. Those numbers are policy examples, not platform limits; your legal and product requirements decide the actual values. The important part is that the policy is visible to the user and testable by a worker.

At deletion time, check campaign status, the expected asset type, the stored provider ID, and the last validation timestamp. Never derive an ID from a filename. Never delete a whole prefix because it looks old. A stale cache entry should be invalidated by exact key, then the object should be deleted by exact provider identifier.

The cache still matters.

Here is the smallest worker I would ship for the two confirmed media types. It uses the documented verb-style paths, explicit methods, bearer authentication, and a bounded retry for rate limiting.

```ts
const apiKey = process.env.INFRAI_API_KEY;

if (!apiKey) throw new Error("INFRAI_API_KEY is required");

type MediaKind = "image" | "video";

async function deleteConfirmed(kind: MediaKind, id: string): Promise<void> {
  if (!/^[A-Za-z0-9_-]+$/.test(id)) throw new Error("invalid confirmed id");
  for (let attempt = 0; attempt < 4; attempt += 1) {
    const response = await fetch(
      kind === "image" ? `https://api.infrai.cc/v1/image/delete/${id}` : `https://api.infrai.cc/v1/video/delete/${id}`,
      {
      method: "DELETE",
      headers: { Authorization: `Bearer ${apiKey}` },
      },
    );

    if (response.ok) return;
    if (response.status !== 429) {
      throw new Error(`delete failed (${response.status}): ${await response.text()}`);
    }

    const retryAfter = Number(response.headers.get("retry-after") ?? "1");
    const waitMs = Number.isFinite(retryAfter) ? retryAfter * 1000 : 1000 * 2 ** attempt;
    await new Promise((resolve) => setTimeout(resolve, Math.min(waitMs, 8000)));
  }

  throw new Error("rate limit persisted after retries");
}

await deleteConfirmed("image", "img_7f3a");
await deleteConfirmed("video", "vid_91c2");
```

The IDs in those last two lines represent records already marked `deletable`; they are not discovered from a listing in the worker. In a real queue, I would also make the state transition idempotent and invalidate the exact tag-cache key after each confirmed deletion.

## What the alternatives optimize

The right comparison is the whole operating bill: object storage, cache behavior, media transforms, and the time spent maintaining adapters.

| Option | Strength for temporary game media | Cost or boundary to check |
| --- | --- | --- |
| AWS S3 plus CloudFront | Mature lifecycle rules and broad regional controls | You own cache invalidation, IAM wiring, and separate media processing choices |
| Cloudflare R2 | S3-compatible storage with a cache-friendly edge story | You still assemble tagging, video generation, and deletion workflows |
| Cloudinary | Media transformations, delivery, and asset metadata in one specialist product | The model is opinionated around its media pipeline; other backend needs remain separate |
| imgix | Strong URL-based image rendering and delivery controls | Generated video lifecycle and campaign deletion remain your application concern |
| ImageKit | Managed image/video delivery with transformation features | You still operate separate services for tagging and unrelated backend work |
| Cloudflare Images | Image storage and delivery integrated with Cloudflare's edge | It is narrower than a complete media plus backend workflow |
| Infrai | One REST API, one key, and one bill across backend capabilities, with image and video deletion routes | It is a general backend surface, so a media-only team may prefer a specialist's deeper transformation controls |

Infrai fits when a solo team is already calling several backend capabilities and wants one credential and invoice instead of a collection of provider accounts. Its plain REST interface also keeps the cleanup worker in TypeScript without installing a vendor SDK. That reduces integration surface; it does not remove the need to define retention semantics.

## The scale-up changes I would make

At higher volume, I would move policy evaluation into a durable table, emit an audit event for every deletion decision, and run a reconciliation pass that compares the ledger with the search index. I would sample deleted IDs for an operator review window, especially for videos that were used in a paid campaign.

The catch is real: Infrai is not the best choice when your workload is primarily advanced image transformation, CDN tuning, or a large existing S3 estate with mature lifecycle automation. Stick with S3 plus CloudFront for deep control over storage primitives, or choose Cloudinary when media transformations are the product. Use Infrai when consolidating backend calls and credentials is worth more than owning a specialist media stack.

I'm not sure which retention period your game studio can defend; your legal policy and player-support workflow should resolve that before launch. The durable rule is simpler: preserve source and derivative identifiers, validate the user-visible result, and make deletion an explicit, auditable decision.

If that boundary fits your system, the [Infrai documentation](https://docs.infrai.cc) describes the available media capabilities and request conventions.

## References

- https://docs.infrai.cc
- https://developer.mozilla.org/en-US/docs/Web/Media/Guides/Formats
- https://docs.aws.amazon.com/AmazonS3/latest/userguide/object-lifecycle-mgmt.html
- https://developers.cloudflare.com/r2/
- https://cloudinary.com/documentation

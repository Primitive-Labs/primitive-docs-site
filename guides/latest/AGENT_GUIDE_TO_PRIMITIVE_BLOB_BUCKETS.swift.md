# Agent Guide to Primitive Blob Buckets

Guidelines for AI agents implementing bucket storage in Primitive apps. **Blob buckets** are general-purpose binary storage that isn't tied to a database — use them for avatars, public assets, server-generated files, short-lived exports, and anywhere you need a signed URL. Files stored with a specific database are database blobs — see the [Blobs guide](AGENT_GUIDE_TO_PRIMITIVE_BLOBS.md).

## Bucket operations

The API is **flat** (`client.blobBuckets.upload(bucketIdOrKey, …)`), not a `.bucket(key)` builder, and accepts `tags: string[]` — there is no arbitrary `metadata` field on upload.

### Upload

```swift
  let result = try await client.blobBuckets.upload(
    bucketIdOrKey: "avatars",
    data: data,
    filename: "alice.jpg",
    contentType: "image/jpeg",
    tags: ["profile"]
  )
  let blobId = result.blobId
```

A blob's id is a ULID: the client mints one, or pass `blobId` to choose it. The same bytes uploaded under an id again store nothing and answer `bytesTransferred: 0`, so a retry under the same id is safe; other bytes under the id are refused with `409`, and an id that is not a ULID with `400`.

### Read (signed URL / download)

```swift
  // Signed URL (for <img> tags, etc.)
  let signed = try await client.blobBuckets.getSignedUrl(
    bucketIdOrKey: "avatars", blobId: blobId, expiresInSeconds: 3600
  )
  let url = signed.url

  // Or download the bytes directly
  let bytes = try await client.blobBuckets.download(bucketIdOrKey: "avatars", blobId: blobId)
```

A download, direct or signed, is `inline` only for `image/*` other than SVG, `audio/*` and `video/*`; every other type is an `attachment`, and every answer carries `nosniff`. Over HTTP the bytes are `GET …/blobs/{blobId}/download` (`304` on `If-None-Match`, `206` on `Range`, and a `HEAD` form) and the metadata is `GET …/blobs/{blobId}`.

### List / metadata / delete

```swift
  // List blobs in the bucket
  let page = try await client.blobBuckets.list(bucketIdOrKey: "avatars", limit: 50)
  let items = page.items
  let nextCursor = page.nextCursor

  // One blob's metadata
  let meta = try await client.blobBuckets.getMetadata(bucketIdOrKey: "avatars", blobId: blobId)

  // Delete a blob
  _ = try await client.blobBuckets.delete(bucketIdOrKey: "avatars", blobId: blobId)

  // Delete a batch of blobs (up to 500 ids) in one call
  let batchResult = try await client.blobBuckets.delete(bucketIdOrKey: "avatars", blobIds: expiredIds)
  // batchResult: BatchBlobDeleteResult { deleted, blobIds, bucketId }
```

`delete(id)` removes one blob. `delete(ids)` removes up to 500 blobs and returns `{ deleted, blobIds, bucketId }`; an empty array does nothing. The batch checks every ID before deleting anything.

**Gotcha when retrying deletion:** missing blobs still undergo access checks with `record.blobCreatedBy == null`. An uploader-only policy can therefore reject a retry. For idempotent cleanup, allow missing blobs as well: `record.blobCreatedBy == null || record.blobCreatedBy == user.userId`.

From the CLI, `primitive blob-buckets head <bucket-id-or-key> <blob-id>` prints one blob's metadata (size, content type, sha256, tags, uploader, `createdAt`) without downloading the bytes — the `getMetadata` call as a command; add `--json` for the raw `BlobInfo`. `primitive blob-buckets delete-blob <bucket-id-or-key> <blob-id...>` deletes one or many — multiple ids go through the batch endpoint as one all-or-nothing call, and `--batch` forces the batch endpoint even for a single id.

### Bucket admin (create / list / get / delete)

Admin/owner only. Deleting a bucket cascades to every blob inside it.

```swift
  _ = try await client.blobBuckets.createBucket(params: CreateBlobBucketParams(
    bucketKey: "uploads",
    name: "User uploads",
    ttlTier: .twentyEightDays,
    preset: .authenticated
  ))

  let buckets = try await client.blobBuckets.listBuckets()
  let bucket = try await client.blobBuckets.getBucket(bucketIdOrKey: "uploads")

  _ = try await client.blobBuckets.deleteBucket(bucketIdOrKey: "uploads")  // cascades to all blobs
```

## Overview

Buckets hold files independently of databases. Configure an access preset and a retention tier, then upload and read blobs. A blob can hold up to 100 MB.

## Bucket configuration

A bucket has a `ttlTier` and a `preset`. Configure via TOML sync (preferred), the CLI, or `createBucket` (see **Bucket admin** above).

```toml
# primitive/dev/blob-buckets/avatars.toml
[bucket]
key = "avatars"
name = "User avatars"
description = "Profile pictures"           # optional
ttlTier = "permanent"                       # 1d | 3d | 14d | 28d | 180d | 365d | permanent
preset = "authenticated"                    # public | authenticated | admin-only | personal-uploads
```

The TOML root table is `[bucket]` (not `[blobBucket]`). The CLI's `primitive config` reads from `primitive/<env>/blob-buckets/<key>.toml`. To change the preset later, edit the TOML and `primitive config push` again, or change it at runtime with `updateBucket` (see [Update a bucket's access](#update-a-buckets-access)).

Scaffold and apply it:

```bash
primitive config create blob-bucket avatars       # writes the file from the type's defaults
primitive config push --only blob-bucket/avatars  # applies just this bucket
```

### Access presets

Presets govern blob ops at the granularity of `read` (download/getMeta), `write` (upload), `list` (enumerate), `delete`, and `share` (mint a signed URL). Admin/owner always bypass.

| Preset             | Member access                                                                 |
|--------------------|-------------------------------------------------------------------------------|
| `public`           | Anyone, incl. anonymous, can `read`/`list`; no member `write`/`delete`/`share` |
| `authenticated`    | Any signed-in member: `read`/`write`/`list`/`delete`/`share` (`!isAnonymous()`) |
| `admin-only`       | Admin/owner only (every member op denied)                                      |
| `personal-uploads` | Any member `write`s; each member `read`/`delete`/`share`s only blobs where `record.blobCreatedBy == user.userId`; `list` is admin-only |

`public` is the only preset that serves **anonymous reads** — an unauthenticated request can `read`/`list` it directly (no signed URL needed).

### Access no preset expresses

Make the bucket `admin-only` and gate reads in a server function: its `authorize` decides who may read, and the handler returns signed URLs from `ctx.api.blobBuckets.getSignedUrls` (up to 100 ids per call). The function acts with the app's authority, so it reads the `admin-only` bucket — and so it signs any id it is handed. Pick the bucket (or check each id) from what `authorize` resolved in `ctx.auth`, never from the input alone; below, each team has its own bucket keyed `reports-<teamId>`, and an id from another team's bucket comes back in `missing`.

```ts
// functions/report-urls.ts — each team's `reports-<teamId>` bucket is admin-only.
import { defineFunction, requireMember } from "primitive-functions";

export default defineFunction({
  input: ["blobIds"],
  authorize: requireMember((input: { teamId: string; blobIds: string[] }) => input.teamId),
  handler: async (input: { blobIds: string[] }, ctx) => {
    const { items, missing } = await ctx.api.blobBuckets.getSignedUrls({
      bucketId: `reports-${ctx.auth.groupId}`, // the team authorize admitted
      body: { blobIds: input.blobIds, expiresInSeconds: 600 },
    });
    return { urls: items.map(({ blobId, url }) => ({ blobId, url })), missing };
  },
});
```

The client invokes the function and loads the URLs it returns:

```swift
  struct ReportRequest: Encodable, Sendable { let teamId: String; let blobIds: [String] }
  struct SignedBlob: Decodable, Sendable { let blobId: String; let url: String }
  struct ReportUrls: Decodable, Sendable { let urls: [SignedBlob]; let missing: [String] }

  let result: FunctionResult<ReportUrls> = try await client.functions.invoke(
    "report-urls",
    input: ReportRequest(teamId: teamId, blobIds: blobIds)
  )

  if result.status == "completed" {
    for blob in result.output?.urls ?? [] {
      print(blob.blobId, blob.url) // load it in an image view, or download it
    }
  }
```

### Update a bucket's access

Change a bucket's access at runtime without recreating it (admin/owner only) with `updateBucket` and a new `preset`.

```swift
  // Switch the bucket to a different preset.
  let bucket = try await client.blobBuckets.updateBucket(
    bucketIdOrKey: "uploads",
    params: UpdateBlobBucketParams(preset: .adminOnly)
  )
  print(bucket.preset) // .adminOnly
```

From the CLI, a bucket's settings are TOML: `preset` switches the access, and `name`/`description` are plain fields.

```bash
primitive config set blob-bucket/avatars bucket.preset=admin-only
primitive config push --only blob-bucket/avatars
```

### TTL tiers

The storage layer automatically expires objects based on the bucket's `ttlTier`. Pick the shortest tier that fits.

| Tier        | Lifetime                          |
|-------------|-----------------------------------|
| `1d`        | ~1 day                            |
| `3d`        | ~3 days                           |
| `14d`       | ~14 days                          |
| `28d`       | ~28 days                          |
| `180d`      | ~180 days                         |
| `365d`      | ~365 days                         |
| `permanent` | No automatic expiry               |

Don't reach for `permanent` for short-lived content — pay only for what you need.

---

## Client API

The bucket API lives at `client.blobBuckets`. **All methods take `bucketIdOrKey` as a positional argument** — there is no per-bucket context object. The core operations (upload, signed URL, download, list, metadata, delete) are shown in **Bucket operations** above; bucket admin in **Bucket admin**.

Notes on the surface:

- **Upload** `tags: string[]` is the only extra metadata — there is no arbitrary `metadata` field; `blobId` (a ULID) is optional. Result: `{ blobId, bucketId, filename, contentType, numBytes, sha256, tags, createdBy, createdAt, etag, bytesTransferred }`.Accepts `data: Data`.- **Signed URL** defaults to 300s, min 30s, max 86400s (clamped server-side). Result: `{ url, token, expiresAt (unix seconds), expiresInSeconds }`. The URL opens only the blob it was minted for: once that blob is deleted or replaced under its id, it is refused with `403`.
- **Download** returns the blob bytes via an authenticated request.The result is `Data`.- **List** is cursor-paginated; default limit 100, max 1000.


### Signed URLs

The signed-URL call is shown in **Read (signed URL / download)** above. A signed URL bypasses the bucket's preset entirely (anyone with the URL can read until it expires). Treat them as bearer tokens. Don't email or log them.

The signed-URL request endpoint requires `member` permission on the app, so generation itself is restricted to authenticated app users — even for a `public` bucket. The resulting URL is then unauthenticated.


---

## From a server function

A server function reaches buckets through `ctx.api.blobBuckets` — `upload`, `get` (metadata), `download`, `downloadHeaders`, `getSignedUrl`, `getSignedUrls` (up to 100 ids), `list`, `delete`, `deleteBatch` (up to 500 ids), and bucket-level `getBucket` / `listBuckets` / `updateBucket`, plus `createBucket` / `deleteBucket`, which need the `blobBuckets:createBucket` / `blobBuckets:deleteBucket` capabilities. Function code acts as the system, so the bucket's preset — which governs client calls — is not evaluated; the function's own `authorize` method is the authorization.

`upload` takes the blob's id and the bytes' size and base64 SHA-256, as a database blob's upload does: `ctx.api.blobBuckets.upload({ bucketId, blobId, "X-Blob-Filename": filename, "X-Blob-Size": size, "X-Blob-Sha256": sha256, body })`. A blob a function stores has `createdBy: null`.

To sign a page of images, call `getSignedUrls` once rather than `getSignedUrl` per blob: one request whatever the count.

```ts
const { items, missing } = await ctx.api.blobBuckets.getSignedUrls({
  bucketId: "avatars",
  body: { blobIds, expiresInSeconds: 600 },
});
```

It answers `{ items: [{ blobId, url, token }], missing, expiresAt, expiresInSeconds }`: a URL (the same `getSignedUrl` mints) for each id that has a blob, in order, and the ids with none in `missing`. `expiresInSeconds` is 30 to 86,400 (default 300), and at most 3,600 in a signed-out run (where `getSignedUrl` refuses more too); a value outside it, more than 100 ids or an empty id is `400 SIGNED_URL_OPTIONS_INVALID`, never clamped. A member calling it over the API has every id checked against the bucket's `share` rule first; one refusal fails the call with no URLs.

A blob a task run reads as input must outlive the run: a retried or replayed step re-reads the same `blobId`, and deleting the blob between attempts fails the step — and the run — with a not-found error. Don't delete such blobs from outside the run (a cancel path, another function) — put them in a bucket whose TTL tier matches their lifespan and let expiry clean them up.

## Gotchas

- Storing user-uploaded files in a bucket when they belong to one database's data and should be database blobs that follow its access.
- Leaving a bucket on `permanent` when blobs are only needed briefly. Object storage is billed; pick the shortest tier.
- Using `public` and then trying to enforce per-user access. `public` bypasses all per-user checks at read time. Use `authenticated` (or `personal-uploads` for owner-scoped blobs) and gate access in your own code before issuing signed URLs.
- A bucket whose info has `convertedFromRuleSet: true` is served as `admin-only` to members. Choose a preset with `updateBucket` (which clears the mark), and move any finer rule into a function's `authorize`.
- Calling `getSignedUrl` on every render. Each call is a network round-trip; cache the URL until `expiresAt - 60s`.
- Writing to the underlying object store outside of Primitive. Side-channel objects don't appear in `list()` and won't be cleaned up by bucket deletion.
- Reusing a database id as a `bucketKey`. Different namespaces; offers no benefit and is confusing.

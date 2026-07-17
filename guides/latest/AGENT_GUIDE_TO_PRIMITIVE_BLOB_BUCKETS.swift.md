# Agent Guide to Primitive Blob Buckets

Guidelines for AI agents implementing bucket storage in Primitive apps. **Blob buckets** are general-purpose binary storage that isn't tied to a document — use them for avatars, public assets, workflow outputs, short-lived exports, and anywhere you need a signed URL. Binary files attached to a specific document live in the [Blobs guide](AGENT_GUIDE_TO_PRIMITIVE_BLOBS.md).

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

### List / metadata / delete

```swift
  // List blobs in the bucket
  let page = try await client.blobBuckets.list(bucketIdOrKey: "avatars", limit: 50)
  let items = page.items
  let cursor = page.cursor

  // One blob's metadata
  let meta = try await client.blobBuckets.getMetadata(bucketIdOrKey: "avatars", blobId: blobId)

  // Delete a blob
  _ = try await client.blobBuckets.delete(bucketIdOrKey: "avatars", blobId: blobId)

  // Delete a batch of blobs (up to 500 ids) in one call
  let batchResult = try await client.blobBuckets.delete(bucketIdOrKey: "avatars", blobIds: expiredIds)
  // batchResult: BatchBlobDeleteResult { deleted, blobIds, bucketId }
```

`delete` is overloaded: a single id deletes one blob (`{ deleted: boolean }`); an array of up to 500 ids deletes a batch in one call, returning `{ deleted, blobIds, bucketId }` where `deleted` counts the ids processed (duplicates included). The batch is all-or-nothing: every id — including ids that no longer exist — is screened against the bucket's `delete` policy before anything is removed; one denial → 403 naming the blob, nothing deleted. An empty array is a valid no-op (`deleted: 0`). Footgun: a missing id screens with `record.blobCreatedBy == null`, so under an uploader-scoped policy (e.g. `personal-uploads`) retrying a batch whose ids are already gone is **denied** — idempotent cleanup needs a rule that also permits gone blobs (`record.blobCreatedBy == null || record.blobCreatedBy == user.userId`).

From the CLI, `primitive blob-buckets delete-blob <bucket-id-or-key> <blob-id...>` deletes one or many — multiple ids go through the batch endpoint as one all-or-nothing call, and `--batch` forces the batch endpoint even for a single id.

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

**Blob buckets** — general-purpose storage outside any document context. Each bucket has its own access preset, TTL tier, and supports time-limited signed URLs. **Cap: 100 MB per blob.**

```swift
  // The bucket key/ID is a positional arg — there is no bucket() context object.
  try await client.blobBuckets.upload(
    bucketIdOrKey: "avatars", data: data, filename: filename, contentType: contentType
  )
  try await client.blobBuckets.getSignedUrl(
    bucketIdOrKey: "avatars", blobId: blobId, expiresInSeconds: 3600
  )
```

**Decision rule:** use document-scoped blobs when the file's lifetime and access naturally match a document's. Use a bucket for avatars, workflow outputs, public assets, anonymous reads via signed URLs, or anything that should live outside any specific document. Document-scoped blobs (10 MB cap, permission inheritance, offline caching) are covered in the [Blobs guide](AGENT_GUIDE_TO_PRIMITIVE_BLOBS.md).

---

## Bucket configuration

A bucket has a `ttlTier` and a `preset` (or a `ruleSetId` for a custom bucket). Configure via TOML sync (preferred), the CLI, or `createBucket` (see **Bucket admin** above).

```toml
# config/blob-buckets/avatars.toml
[bucket]
key = "avatars"
name = "User avatars"
description = "Profile pictures"           # optional
ttlTier = "permanent"                       # 1d | 3d | 14d | 28d | 180d | 365d | permanent
preset = "authenticated"                    # public | authenticated | admin-only | personal-uploads
# ruleSetId = "<rule-set-id>"               # optional; makes a `custom` bucket whose access
                                            # is governed entirely by the rule set (see below)
```

The TOML root table is `[bucket]` (not `[blobBucket]`). The CLI's `primitive sync` reads from `config/blob-buckets/<key>.toml`. Give a bucket a `preset` or a `ruleSetId`, not both. To change either later, edit the TOML and `primitive sync push` again, or change it at runtime with `updateBucket` (see [Update a bucket's access](#update-a-buckets-access)).

Or via CLI:

```bash
primitive blob-buckets create \
  --key avatars --name "User avatars" \
  --ttl permanent --preset authenticated
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

For access no preset expresses, set `ruleSetId` to make a `custom` bucket: the rule set is the authority for member access (resource type `blob_bucket`), evaluated per op (`read`/`write`/`list`/`delete`/`share`; in a configured rule set `list`/`share` fall back to `read` and `delete` to `write`). Admins/owners always pass; a missing/orphaned rule set denies closed. Rule CEL exposes `isAnonymous()` and `record.blobCreatedBy` (the uploader's id; null for bucket-level `list`).

### Update a bucket's access

Change a bucket's access at runtime without recreating it (admin/owner only) with `updateBucket`. Setting a named `preset` switches it and clears any attached rule set; setting a `ruleSetId` makes the bucket `custom`.

```swift
  // Switch the bucket to a different preset.
  _ = try await client.blobBuckets.updateBucket(
    bucketIdOrKey: "uploads",
    params: UpdateBlobBucketParams(preset: .adminOnly)
  )

  // Attach a custom rule set (makes the bucket `custom`); use .clear to remove it.
  _ = try await client.blobBuckets.updateBucket(
    bucketIdOrKey: "uploads",
    params: UpdateBlobBucketParams(ruleSetId: .value("rule-set-id"))
  )
```

From the CLI: `primitive blob-buckets update <id-or-key> --preset <preset>` switches the preset (clearing any attached rule set); `--rule-set-id <id>` attaches a rule set (making the bucket `custom`). `--name` and `--description` update those fields. At least one flag is required; `--preset` and `--rule-set-id` are mutually exclusive, and clearing a rule set requires moving to a preset in the same call.

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

- **Upload** `tags: string[]` is the only extra metadata — there is no arbitrary `metadata` field. Result: `{ blobId, bucketId, filename, contentType, numBytes, sha256, tags, createdBy }`.Accepts `data: Data`.- **Signed URL** defaults to 300s, min 30s, max 86400s (clamped server-side). Result: `{ url, token, expiresAt (unix seconds), expiresInSeconds }`.
- **Download** returns the blob bytes via an authenticated request.The result is `Data`.- **List** is cursor-paginated; default limit 100, max 1000.


### Signed URLs

The signed-URL call is shown in **Read (signed URL / download)** above. A signed URL bypasses the bucket's preset entirely (anyone with the URL can read until it expires). Treat them as bearer tokens. Don't email or log them.

The signed-URL request endpoint requires `member` permission on the app, so generation itself is restricted to authenticated app users — even for a `public` bucket. The resulting URL is then unauthenticated.


---

## Workflow integration

The `blob.upload`, `blob.download`, `blob.signedUrl`, and `blob.delete` workflow steps write to, read from, sign URLs for, and delete from buckets. Steps reference the bucket by `bucketId` or `bucketKey`; `blob.delete` also takes a batch `blobIds` form (up to 500 ids, screened against the `delete` policy all-or-nothing before anything is removed). A `runAs:"caller"` run evaluates this bucket policy per op with the same mapping as direct calls (upload → `write`, download → `read`, signedUrl → `share`, delete → `delete`); a `runAs:"system"` run is app-privileged and skips it. See the [Workflows guide](AGENT_GUIDE_TO_PRIMITIVE_WORKFLOWS.md).

A blob a workflow run reads as input must outlive the run's retry window: retries re-read the same `blobId`, and deleting the blob between attempts fails the next retry — and the run — with a not-found error. Don't delete such blobs from outside the run (a cancel path, another workflow) — put them in a bucket whose TTL tier matches their lifespan and let expiry clean them up.

---

## Anti-patterns

- Storing user-uploaded documents in a bucket when they should be document-scoped blobs with permission inheritance.
- Leaving a bucket on `permanent` when blobs are only needed briefly. Object storage is billed; pick the shortest tier.
- Using `public` and then trying to enforce per-user access. `public` bypasses all per-user checks at read time. Use `authenticated` (or `personal-uploads` for owner-scoped blobs) and gate access in your own code before issuing signed URLs.
- Calling `getSignedUrl` on every render. Each call is a network round-trip; cache the URL until `expiresAt - 60s`.
- Writing to the underlying object store outside of Primitive. Side-channel objects don't appear in `list()` and won't be cleaned up by bucket deletion.
- Reusing a `documentId` as a `bucketKey`. Different namespaces; offers no benefit and is confusing.

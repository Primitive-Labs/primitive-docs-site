# Agent Guide to Primitive Blob Buckets

Guidelines for AI agents implementing bucket storage in Primitive apps. **Blob buckets** are general-purpose binary storage that isn't tied to a database — use them for avatars, public assets, server-generated files, short-lived exports, and anywhere you need a signed URL. Files stored with a specific database are database blobs — see the [Blobs guide](AGENT_GUIDE_TO_PRIMITIVE_BLOBS.md).

## Bucket operations

The API is **flat** (`client.blobBuckets.upload(bucketIdOrKey, …)`), not a `.bucket(key)` builder, and accepts `tags: string[]` — there is no arbitrary `metadata` field on upload.

### Upload

```typescript
  const { blobId } = await client.blobBuckets.upload("avatars", {
    filename: "alice.jpg",
    contentType: "image/jpeg",
    data,
    tags: ["profile"],
  });
```

A blob's id is a ULID: the client mints one, or pass `blobId` to choose it. The same bytes uploaded under an id again store nothing and answer `bytesTransferred: 0`, so a retry under the same id is safe; other bytes under the id are refused with `409`, and an id that is not a ULID with `400`.

### Read (signed URL / download)

```typescript
  // Signed URL (for <img> tags, etc.)
  const { url } = await client.blobBuckets.getSignedUrl("avatars", blobId, 3600);

  // Or download the bytes directly
  const bytes = await client.blobBuckets.download("avatars", blobId);
```

A download, direct or signed, is `inline` only for `image/*` other than SVG, `audio/*` and `video/*`; every other type is an `attachment`, and every answer carries `nosniff`. Over HTTP the bytes are `GET …/blobs/{blobId}/download` (`304` on `If-None-Match`, `206` on `Range`, and a `HEAD` form) and the metadata is `GET …/blobs/{blobId}`.

### List / metadata / delete

```typescript
  // List blobs in the bucket
  const { items, nextCursor } = await client.blobBuckets.list("avatars", { limit: 50 });

  // One blob's metadata
  const meta = await client.blobBuckets.getMetadata("avatars", blobId);

  // Delete a blob
  await client.blobBuckets.delete("avatars", blobId);

  // Delete a batch of blobs (up to 500 ids) in one call
  const batchResult = await client.blobBuckets.delete("avatars", expiredIds);
  // batchResult: { deleted, blobIds, bucketId }
```

`delete(id)` removes one blob. `delete(ids)` removes up to 500 blobs and returns `{ deleted, blobIds, bucketId }`; an empty array does nothing. The batch checks every ID before deleting anything.

**Gotcha when retrying deletion:** missing blobs still undergo access checks with `record.blobCreatedBy == null`. An uploader-only policy can therefore reject a retry. For idempotent cleanup, allow missing blobs as well: `record.blobCreatedBy == null || record.blobCreatedBy == user.userId`.

From the CLI, `primitive blob-buckets head <bucket-id-or-key> <blob-id>` prints one blob's metadata (size, content type, sha256, tags, uploader, `createdAt`) without downloading the bytes — the `getMetadata` call as a command; add `--json` for the raw `BlobInfo`. `primitive blob-buckets delete-blob <bucket-id-or-key> <blob-id...>` deletes one or many — multiple ids go through the batch endpoint as one all-or-nothing call, and `--batch` forces the batch endpoint even for a single id.

### Bucket admin (create / list / get / delete)

Admin/owner only. Deleting a bucket cascades to every blob inside it.

```typescript
  await client.blobBuckets.createBucket({
    bucketKey: "uploads",
    name: "User uploads",
    ttlTier: "28d",
    preset: "authenticated",
  });

  const buckets = await client.blobBuckets.listBuckets();
  const bucket = await client.blobBuckets.getBucket("uploads");

  await client.blobBuckets.deleteBucket("uploads"); // cascades to all blobs
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

```typescript
  const result = await client.functions.invoke<{
    urls: Array<{ blobId: string; url: string }>;
    missing: string[];
  }>("report-urls", { input: { teamId, blobIds } });

  if (result.status === "completed") {
    for (const { blobId, url } of result.output?.urls ?? []) {
      console.log(blobId, url); // put the URL in an <img>, or download it
    }
  }
```

### Update a bucket's access

Change a bucket's access at runtime without recreating it (admin/owner only) with `updateBucket` and a new `preset`.

```typescript
  // Switch the bucket to a different preset.
  const bucket = await client.blobBuckets.updateBucket("uploads", { preset: "admin-only" });
  console.log(bucket.preset); // "admin-only"
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

- **Upload** `tags: string[]` is the only extra metadata — there is no arbitrary `metadata` field; `blobId` (a ULID) is optional. Result: `{ blobId, bucketId, filename, contentType, numBytes, sha256, tags, createdBy, createdAt, etag, bytesTransferred }`.Accepts `data: ArrayBuffer | Uint8Array | Blob | string`; a `File` works (it's a `Blob`).- **Signed URL** defaults to 300s, min 30s, max 86400s (clamped server-side). Result: `{ url, token, expiresAt (unix seconds), expiresInSeconds }`. The URL opens only the blob it was minted for: once that blob is deleted or replaced under its id, it is refused with `403`.
- **Download** returns the blob bytes via an authenticated request.The result is an `ArrayBuffer`.- **List** is cursor-paginated; default limit 100, max 1000.

### Gotchas when reading files

```typescript
// DON'T: try to use a per-bucket context object — it doesn't exist.
const bucket = client.blobBuckets.bucket("avatars"); // TypeError
await bucket.upload(...);                            // TypeError

// DON'T: invent a `signedUrl` method — the actual name is getSignedUrl.
await client.blobBuckets.signedUrl("avatars", blobId, { expiresInSeconds: 3600 });
// Correct:
await client.blobBuckets.getSignedUrl("avatars", blobId, 3600);

// DON'T: attach `metadata: {...}` at upload time. The bucket upload API only
// accepts `tags: string[]`. There is no arbitrary metadata field, and even
// `tags` are stored as object metadata — they don't gate access on their own.
await client.blobBuckets.upload("k", { filename, data, metadata: {} }); // ignored

// DON'T: ask for a >24h signed URL — the server clamps to 86400s.
await client.blobBuckets.getSignedUrl("k", id, 7 * 86400); // becomes 86400

// DON'T: put auth-required URLs in <img src>. Use a signed URL or download() + Blob URL.
imgEl.src = `/app/${appId}/api/blob-buckets/${bucketId}/blobs/${blobId}/download`; // 401
```

### Signed URLs

The signed-URL call is shown in **Read (signed URL / download)** above. A signed URL bypasses the bucket's preset entirely (anyone with the URL can read until it expires). Treat them as bearer tokens. Don't email or log them.

The signed-URL request endpoint requires `member` permission on the app, so generation itself is restricted to authenticated app users — even for a `public` bucket. The resulting URL is then unauthenticated.

To display a bucket image, point an element at the returned `url`:

```typescript
const { url, expiresAt } = await client.blobBuckets.getSignedUrl(
  "avatars",
  blobId,
  3600
);
imgEl.src = url;

// Cache the URL (and re-fetch a new one slightly before expiresAt). Don't call
// getSignedUrl on every render — each call is a server round-trip.
```

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

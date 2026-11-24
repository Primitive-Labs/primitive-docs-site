# Agent Guide to Primitive Blobs

Database blobs are files stored with a database (`db3` in code) and follow its access. Each blob can hold up to 10 MB. Use [blob buckets](AGENT_GUIDE_TO_PRIMITIVE_BLOB_BUCKETS.md) for files independent of any database. Store a blob's `blobId` in a record field to attach the file to a record.

## Database blob operations

A database's blobs are reached through its blob accessor. The database does not need to be subscribed on the device.

```swift
let blobs = client.db3.blobs(db3Id)
```

### Upload

```swift
  let blobs = client.db3.blobs(db3Id)
  let result = try await blobs.upload(
    data: data,
    options: BlobUploadSourceOptions(filename: "notes.txt", contentType: "text/plain")
  )
  let blobId = result.blobId
```

### Upload with disposition

`disposition` is stored on the blob (sent as `X-Blob-Disposition`).

```swift
  let data = Data("hello blob".utf8)
  let result = try await blobs.upload(
    data: data,
    options: BlobUploadSourceOptions(
      filename: "hello.txt",
      contentType: "text/plain",
      disposition: .attachment // or .inline
      // retainLocal defaults to true; set false to drop local bytes once the
      // server confirms the upload.
    )
  )
  // result.blobId, result.numBytes, result.contentType
```

### List / URL / read

```swift
  let blobs = client.db3.blobs(db3Id)

  let list = try await blobs.list()
  let url = try blobs.downloadUrl(blobId: blobId)      // synchronous
  let bytes = try await blobs.read(blobId: blobId)
```

### List / metadata / delete

```swift
  let blobs = client.db3.blobs(db3Id)

  let page = try await blobs.list(limit: 50) // page.items, page.nextCursor
  let meta = try await blobs.get(blobId: blobId)
  print(meta.filename ?? "", meta.numBytes ?? 0, meta.createdBy ?? "app", meta.etag ?? "")
  _ = try await blobs.delete(blobId: blobId)
```

A blob's metadata is `blobId`, `filename`, `contentType`, `numBytes`, `sha256`, `createdBy` (the uploading member's user id; `null` for a signed upload), `createdAt`, `etag` (the download's tag) and `status`.

### Batch delete

Up to 500 ids in one request. An id that is not a blob of the database is skipped and counted; it answers `{ deleted, blobIds, db3Id }`. Once the server accepts, the client cancels a named upload queued for this database and drops its local copy of each id.

```swift
  let result = try await client.db3.blobs(db3Id).deleteBatch(blobIds: blobIds)
```

## Overview

Anyone who can read the database can list and download its blobs; anyone who can write it can upload or delete them. A database with sync turned off is read and written only by app admins and server functions, and the same holds for its blobs. Uploaded files queue when offline and downloaded files can be cached locally.

## Uploading

Repeated uploads can reuse the same file content. See **Upload with disposition** above for the full options form. The download endpoint chooses `Content-Disposition` from the `?disposition=` query param (default `attachment`), not from the upload-time value — pass `disposition` explicitly to the download URL when serving inline.

`upload` accepts `Data`. `retainLocal` (default `true`; set `false` to drop local bytes once the server confirms) controls whether local bytes are kept after upload.

Uploading more than 10 MB to a database blob returns `413`. Use a blob bucket instead (100 MB cap).

### Idempotent re-upload

Uploading the same `blobId` twice with **identical** `sha256` and `size` returns 200 with `bytesTransferred: 0` — the server keeps the existing object, so retries from a flaky network are free. Uploading the same `blobId` with **different** bytes (different sha256 or size) returns 409; pick a new `blobId` instead of overwriting. This dedup behavior is what makes retry-with-backoff upload loops safe.

---

## Listing

See **List / metadata / delete** above for the basic call. Each item carries `blobId`, `filename`, `contentType`, `numBytes`, `sha256`, and `createdAt`. `nextCursor` is an opaque pagination token, only present when more results exist — pass it back as the `cursor` option to page through results. `limit` accepts `1`–`100`; a larger value is clamped to `100`, and a zero, negative, or non-integer `limit` is rejected with a `400`. Page through `nextCursor` for more than 100 blobs rather than asking for a bigger page.

```swift
  let page1 = try await blobs.list(limit: 50)
  for b in page1.items {
    print(b.blobId, b.filename ?? "", b.contentType ?? "", b.numBytes ?? 0, b.sha256 ?? "", b.createdAt ?? "")
  }

  // `nextCursor` is an opaque token; only present when more results remain.
  if let cursor = page1.nextCursor {
    let page2 = try await blobs.list(cursor: cursor)
    return page2.items
  }
```

---

## Get metadata

See **List / metadata / delete** above.

```swift
let meta = try await blobs.get(blobId: blobId)
// blobId, filename, contentType, numBytes, sha256, createdAt, ...
```

---

## Downloading

### Download URL

The basic call is shown in **List / URL / read** above. `downloadUrl` is synchronous and returns a string; it `throws`, so call it with `try`. The URL carries no credentials: `BlobImage` and `downloadRequest` add the user's auth (see [Displaying images](#displaying-images)).
```swift
  let url = try blobs.downloadUrl(
    blobId: blobId,
    disposition: .attachment,         // or .inline
    attachmentFilename: "report.pdf"  // optional override (RFC 5987-encoded)
  )
```

### Read content into memory

`read` is shown in **List / URL / read** above. It checks the cache first; on a miss it fetches from the server and writes the response into the cache.

```swift
  let bytes = try await blobs.read(blobId: blobId)                 // Data (default)
  let text = try await blobs.read(blobId: blobId, as: String.self) // UTF-8 String

  // Bypass the local cache and re-fetch from the server.
  let fresh = try await blobs.read(blobId: blobId, as: String.self, force: true)
```

A typed `read(blobId:as:)` overload also JSON-decodes a blob into any `Decodable` type. When the client is offline and the blob has no local copy, `read` throws `JsBaoError` with code `.offline` without sending a request.

```swift
let payload = try await blobs.read(blobId: blobId, as: MyCodable.self)
```

### Range requests and conditional GETs

The database blob download endpoint supports two HTTP behaviors useful when streaming media or revalidating cached bytes:

- **Range requests.** Send `Range: bytes=START-END` and the server replies with `206 Partial Content` and the requested slice. Out-of-range requests return `416`. This is what media players use to seek; you usually don't issue these manually, but if you build a custom player it works as expected.
- **Conditional GET.** The server emits an `ETag` (the blob's sha256). Send `If-None-Match: <etag>` on subsequent requests and the server returns `304 Not Modified` with no body when the bytes are unchanged.

Both behaviors are on the database blob path only. Bucket blobs go through signed URLs straight to the object store, which has its own range/etag semantics handled by the storage layer.

---

## Deleting

See **List / metadata / delete** above.

```swift
let result = try await blobs.delete(blobId: blobId) // result.deleted == true
```

Once the server accepts the delete, the client cancels an upload of that `blobId` queued for this database, in flight included, and clears this database's local bytes of it. An upload of the same id queued for another database stays queued, and a refused delete changes nothing on the device. `deleteBatch` cleans up the same way.

---

## Offline & the upload queue

The upload queue is keyed by user identity, retries with exponential backoff (2s base, 60s max), and persists across reloads and app restarts: a queued or in-flight upload resumes in the next client for the same app and user, and never for another user. While offline, bytes are written to the local cache immediately and queued; `upload()` resolves without waiting for the network (`bytesTransferred: 0`). Reads are served from the cache for blobs previously uploaded or downloaded on this device/user, and `prefetch` warms the cache — its per-blob errors are logged and swallowed, so it resolves once all attempts complete regardless of individual failures. Coming back online resumes queue processing automatically, draining the queue in the background.

```swift
  client.setNetworkMode(.offline)

  // Bytes are written to the local cache immediately and queued. upload()
  // returns without waiting for the network (bytesTransferred is 0).
  let result = try await blobs.upload(
    data: data,
    options: BlobUploadSourceOptions(filename: "draft.txt", contentType: "text/plain")
  )

  // Reads served from cache for blobs uploaded or downloaded on this device.
  let text = try await blobs.read(blobId: result.blobId, as: String.self)

  // Warm the cache; per-blob errors are logged and swallowed.
  try await blobs.prefetch(blobIds: blobIds, concurrency: 4)

  // Queued uploads resume automatically once back online.
  client.setNetworkMode(.online)
```

An upload whose request fails for a retryable reason (no connection, a timeout, a `5xx`) is queued the same way and answers `bytesTransferred: 0`; a refusal such as a `403` throws and queues nothing.

The queue is kept in the client's SQLite database, so it survives a restart with `storageConfig: .sqlite(...)` (the default); with `.memory` it lasts only as long as the client. Restored uploads are listed once `waitForStorageReady()` returns.

---

## Gotchas: upload queue management

Inspect and control a database's upload queue, and set the client-wide upload concurrency (applies to all databases). The `queueId` is the `blobId` for database blob uploads.

```swift
  // Inspect what's queued for this database. Each BlobUploadStatus carries
  // queueId, blobId, filename, contentType, numBytes, status, attempts,
  // nextAttemptAt, lastError.
  let tasks = await blobs.uploadsAsync()

  // Pause/resume by blobId (the queueId is the blobId for database uploads).
  _ = await blobs.pauseUploadAsync(blobId: blobId)
  _ = await blobs.resumeUploadAsync(blobId: blobId)

  await blobs.pauseAllAsync()
  await blobs.resumeAllAsync()

  // Concurrency is set on the client (global, all databases).
  await client.setBlobUploadConcurrencyAsync(5) // min 1; default 2
  let concurrency = await client.blobUploadConcurrency
```

Every queue verb is `async`: the upload queue lives behind an actor, so reading or changing it is an `await`. The queue members carry the `Async` suffix (`uploadsAsync()`, `pauseUploadAsync(blobId:)`, `pauseAllAsync()`, `setBlobUploadConcurrencyAsync(_:)`, …), and the concurrency reader is an `async` property: `await client.blobUploadConcurrency`. `downloadUrl(blobId:disposition:attachmentFilename:)` and `downloadRequest(blobId:disposition:attachmentFilename:)` stay synchronous (both `throws`), so a SwiftUI `body` can call them directly; `body` cannot throw, so call them with `try?` or inside a `do`/`catch`.

---

## Events

The upload queue emits lifecycle events. `upload-progress` is *not* a byte-level progress event — it fires on status transitions. There is no per-byte progress callback. A `delete` the server accepts mid-upload cancels the in-flight transfer and evicts the cached bytes.

```swift
  let progress = client.observeOnMainActor(BlobUploadProgressEvent.self) { e in
    print(e.queueId, e.blobId, e.status, e.numBytes, e.attempts)
  }

  let completed = client.observeOnMainActor(BlobUploadCompletedEvent.self) { e in
    print("done", e.blobId, e.queueId)
  }

  let failed = client.observeOnMainActor(BlobUploadFailedEvent.self) { e in
    print("failed", e.blobId, e.lastError ?? "", "retry?", e.willRetry)
  }
```

---

## Displaying images

```swift
    BlobImage(client.db3.blobs(db3Id), blobId: blobId) { image in
      image.resizable().scaledToFit()
    } placeholder: {
      ProgressView()
    }
```

The image shows for a signed-in user who can read the database.

`BlobImage` is in the `JsBaoClientUI` product (`import JsBaoClientUI`), which the Swift template links. It has the `AsyncImage` shapes: `BlobImage(blobs, blobId:)`, the `content:placeholder:` form above, and `BlobImage(blobs, blobId:) { phase in … }` over `BlobImagePhase` (`.empty`, `.success(Image)`, `.failure(Error)`). It loads through `read`: the user's auth, the copy already on the device, and `JsBaoError` `.offline` when the client is offline with no local copy. The load restarts when the client, the database or the blob changes.

With an image library (Kingfisher, Nuke), send `blobs.downloadRequest(blobId:disposition:attachmentFilename:)`: a `URLRequest` for the download URL with the user's `Authorization` header, built from the token in force when it is called. A bare `downloadUrl` in `AsyncImage` is refused with `401`.

---

## From a server function

A server function uploads a database blob with `ctx.api.db3.blobs.upload`: `db3Id`, a `blobId` it mints (a ULID), `"X-Blob-Filename"`, `"X-Blob-Size"` (the byte length), `"X-Blob-Sha256"` (the base64 SHA-256 of the bytes), `body`, and the media type in `"Content-Type"`. `ctx.api.db3.blobs.list` (one page; `limit` 1 to 100 and `cursor`, the previous page's `nextCursor`) and `get` read its metadata, `ctx.api.db3.blobs.download` returns its bytes as a `Uint8Array`, `ctx.api.db3.blobs.delete` removes it, and `ctx.api.db3.blobs.deleteBatch({ db3Id, body: { blobIds } })` removes up to 500 in one call. A function reaches the blobs of a database with sync turned off the same way.

```ts
await ctx.api.db3.blobs.upload({
  db3Id, blobId,
  "X-Blob-Filename": "receipt.png",
  "X-Blob-Size": bytes.byteLength,
  "X-Blob-Sha256": sha256Base64,
  "Content-Type": "image/png",
  body: bytes,
});
```

Name the type: without `"Content-Type"` the blob is stored as `application/octet-stream`, and an agent session message's `files` accepts only images and PDFs.

### Signed URLs for readers without access

To show a database's images to readers who hold no grant on it (a feed from a database the app holds), the function mints signed URLs with `ctx.api.db3.blobs.getSignedUrls`, up to 100 per call, and the page puts each `url` in an `<img>`. The URL is the only credential.

```ts
const { items, expiresAt } = await ctx.api.db3.blobs.getSignedUrls({
  db3Id,
  body: { blobIds: images.map((blob) => blob.blobId), expiresInSeconds: 600 },
});
return { images: items.map(({ blobId, url }) => ({ blobId, url })), expiresAt };
```

| Argument | Values |
|---|---|
| `body.blobIds` | 0 to 100 blob ids (ULIDs); one item each, in order, duplicates included |
| `body.expiresInSeconds` | integer, 30 to 86,400; default 300; at most 3,600 in a run a signed-out caller started |
| `userId` | optional: judge the call by this member's read access alone |

It answers `{ items: [{ blobId, url, token }], expiresAt, expiresInSeconds }`. Whoever can download the database's blobs can mint: an app owner or admin, a member who can read the database, a server function. Minting reads no blob: a URL for an id that is not an uploaded blob of the database answers `404` when loaded. A loaded URL answers the stored `Content-Type`, `Content-Disposition: inline` for `image/*` other than SVG, `audio/*` and `video/*` (`attachment` otherwise), and `Cache-Control: private, max-age=<seconds until expiry>`.

For bucket blobs, `ctx.api.blobBuckets.getSignedUrls({ bucketId, body: { blobIds, expiresInSeconds } })` mints up to 100 of `getSignedUrl`'s URLs in one call; it answers `items` for the ids that have a blob and `missing` for those that do not. Each mint is one request, whatever the count.

**Gotchas**

- A URL works for anyone holding it until it expires. Deleting the blob or the database makes the server refuse it at once (`404`), but a browser that loaded it may show its cached copy until expiry. Mint a short `expiresInSeconds` when a deleted image must disappear quickly.
- Removing a member's grant does not void URLs already minted.
- A value outside its range is `SIGNED_URL_OPTIONS_INVALID` (`details.field` names it), never clamped. A loaded URL past its expiry is `403 SIGNED_URL_EXPIRED`; an altered one, or one used for another blob, `403 SIGNED_URL_INVALID`.
- The byte read is `ctx.api.db3.blobs.download` (likewise `ctx.api.agentSessions.attachments.download`). It returns bytes, not a URL; the clients' `downloadUrl` is what returns a URL.


---

## Permissions (database blobs)

| Operation                              | Required database permission     |
|----------------------------------------|----------------------------------|
| Upload, Delete                         | `read-write` or `owner`           |
| List, Get metadata, Download, Read     | `reader` or higher                |


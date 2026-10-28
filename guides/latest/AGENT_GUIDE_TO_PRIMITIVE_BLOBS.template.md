# Agent Guide to Primitive Blobs

Database blobs are files stored with a database (`db3` in code) and follow its access. Each blob can hold up to 10 MB. Use [blob buckets](AGENT_GUIDE_TO_PRIMITIVE_BLOB_BUCKETS.md) for files independent of any database. Store a blob's `blobId` in a record field to attach the file to a record.

## Database blob operations

A database's blobs are reached through its blob accessor. The database does not need to be subscribed on the device.

{{#lang ts}}
```typescript
const blobs = client.db3.blobs(db3Id);
```
{{/lang}}
{{#lang swift}}
```swift
let blobs = client.db3.blobs(db3Id)
```
{{/lang}}

### Upload

{{ example: blobs/db3-blob-upload }}

### Upload with disposition

`disposition` is stored on the blob (sent as `X-Blob-Disposition`).

{{ example: blobs/db3-blob-upload-options }}

### List / URL / read

{{ example: blobs/db3-blob-read }}

### List / metadata / delete

{{ example: blobs/db3-blob-manage }}

## Overview

Anyone who can read the database can list and download its blobs; anyone who can write it can upload or delete them. A database with sync turned off is read and written only by app admins and server functions, and the same holds for its blobs. Uploaded files queue when offline and downloaded files can be cached locally.

## Uploading

Repeated uploads can reuse the same file content. See **Upload with disposition** above for the full options form. The download endpoint chooses `Content-Disposition` from the `?disposition=` query param (default `attachment`), not from the upload-time value — pass `disposition` explicitly to the download URL when serving inline.

{{#lang ts}}
`upload` accepts `File | Blob | Uint8Array | ArrayBuffer`. `retainLocal` (default `true`; set `false` to drop local bytes once the server confirms) controls whether local bytes are kept after upload.

### From a file input

The upload call is the compiled [Upload](#upload) example — `client.db3.blobs(db3Id).upload(file)`. The only web-specific glue is pulling the `File` off an `<input>` (`File` objects are accepted directly; `filename`/`contentType` default to `file.name`/`file.type`):

```typescript
// web-DOM glue: get a File from an <input type="file">, then upload it.
const file = (document.querySelector('input[type="file"]') as HTMLInputElement).files![0];
const { blobId } = await client.db3.blobs(db3Id).upload(file);
```

### Don't do this

```typescript
// DON'T: manually convert File to ArrayBuffer just to upload it.
const buf = await file.arrayBuffer();
await blobs.upload(buf, { filename: file.name, contentType: file.type });

// DON'T: try to upload >10 MB to a database blob. The server returns 413.
// Use a blob bucket instead (100 MB cap).
```

### `uploadFile` vs `upload`

Both call the same code path. `uploadFile` returns a slightly trimmed result (`{ blobId, numBytes, bytesTransferred? }`). Prefer `upload`.
{{/lang}}
{{#lang swift}}
`upload` accepts `Data`. `retainLocal` (default `true`; set `false` to drop local bytes once the server confirms) controls whether local bytes are kept after upload.

Uploading more than 10 MB to a database blob returns `413`. Use a blob bucket instead (100 MB cap).
{{/lang}}

### Idempotent re-upload

Uploading the same `blobId` twice with **identical** `sha256` and `size` returns 200 with `bytesTransferred: 0` — the server keeps the existing object, so retries from a flaky network are free. Uploading the same `blobId` with **different** bytes (different sha256 or size) returns 409; pick a new `blobId` instead of overwriting. This dedup behavior is what makes retry-with-backoff upload loops safe.

---

## Listing

See **List / metadata / delete** above for the basic call. Each item carries `blobId`, `filename`, `contentType`, `numBytes`, `sha256`, and `createdAt`. `nextCursor` is an opaque pagination token, only present when more results exist — pass it back as the `cursor` option to page through results. `limit` accepts `1`–`100`; a larger value is clamped to `100`, and a zero, negative, or non-integer `limit` is rejected with a `400`. Page through `nextCursor` for more than 100 blobs rather than asking for a bigger page.

{{ example: blobs/db3-blob-list-paginate }}

---

## Get metadata

See **List / metadata / delete** above.

{{#lang ts}}
```typescript
const meta = await blobs.get(blobId);
// { blobId, filename, contentType, numBytes, sha256, createdAt, ... }
```
{{/lang}}
{{#lang swift}}
```swift
let meta = try await blobs.get(blobId: blobId)
// blobId, filename, contentType, numBytes, sha256, createdAt, ...
```
{{/lang}}

---

## Downloading

### Download URL (authenticated)

The basic call is shown in **List / URL / read** above. `downloadUrl` is synchronous and returns a string, authenticated via the user's session against the API origin{{#lang ts}} — it works in an `<a href download>` or `window.location`{{/lang}}.

{{ example: blobs/db3-blob-download-url }}

### Read content into memory

`read` is shown in **List / URL / read** above. It checks the cache first; on a miss it fetches from the server and writes the response into the cache.

{{ example: blobs/db3-blob-read-formats }}

{{#lang ts}}
`{ as: "blob" }` is also available and returns a `Blob`. When offline (`networkMode === "offline"`) and the cache is empty, `read` throws `"Blob cache unavailable while offline"`.
{{/lang}}
{{#lang swift}}
A typed `read(blobId:as:)` overload also JSON-decodes a blob into any `Decodable` type. When the client is offline and the blob has no local copy, `read` throws `JsBaoError` with code `.offline` without sending a request.

```swift
let payload = try await blobs.read(blobId: blobId, as: MyCodable.self)
```
{{/lang}}

### Range requests and conditional GETs

The database blob download endpoint supports two HTTP behaviors useful when streaming media or revalidating cached bytes:

- **Range requests.** Send `Range: bytes=START-END` and the server replies with `206 Partial Content` and the requested slice. Out-of-range requests return `416`. This is what media players use to seek; you usually don't issue these manually, but if you build a custom player it works as expected.
- **Conditional GET.** The server emits an `ETag` (the blob's sha256). Send `If-None-Match: <etag>` on subsequent requests and the server returns `304 Not Modified` with no body when the bytes are unchanged.

Both behaviors are on the database blob path only. Bucket blobs go through signed URLs straight to the object store, which has its own range/etag semantics handled by the storage layer.

---

## Deleting

See **List / metadata / delete** above.

{{#lang ts}}
```typescript
const { deleted } = await blobs.delete(blobId); // { deleted: true }
```
{{/lang}}
{{#lang swift}}
```swift
let result = try await blobs.delete(blobId: blobId) // result.deleted == true
```
{{/lang}}

Deleting also cancels any in-flight upload for the same `blobId` and clears local bytes.

---

## Offline & the upload queue

The upload queue is keyed by user identity, retries with exponential backoff (2s base, 60s max), and persists across reloads and app restarts: a queued or in-flight upload resumes in the next client for the same app and user, and never for another user. While offline, bytes are written to the local cache immediately and queued; `upload()` resolves without waiting for the network (`bytesTransferred: 0`). Reads are served from the cache for blobs previously uploaded or downloaded on this device/user, and `prefetch` warms the cache — its per-blob errors are logged and swallowed, so it resolves once all attempts complete regardless of individual failures. Coming back online resumes queue processing automatically, draining the queue in the background.

{{ example: blobs/db3-blob-offline }}

An upload whose request fails for a retryable reason (no connection, a timeout, a `5xx`) is queued the same way and answers `bytesTransferred: 0`; a refusal such as a `403` throws and queues nothing.

{{#lang swift}}
The queue is kept in the client's SQLite database, so it survives a restart with `storageConfig: .sqlite(...)` (the default); with `.memory` it lasts only as long as the client. Restored uploads are listed once `waitForStorageReady()` returns.
{{/lang}}

---

## Gotchas: upload queue management

Inspect and control a database's upload queue, and set the client-wide upload concurrency (applies to all databases). The `queueId` is the `blobId` for database blob uploads.

{{ example: blobs/db3-blob-queue }}

{{#lang swift}}
Every queue verb is `async`: the upload queue lives behind an actor, so reading or changing it is an `await`. The queue members carry the `Async` suffix (`uploadsAsync()`, `pauseUploadAsync(blobId:)`, `pauseAllAsync()`, `setBlobUploadConcurrencyAsync(_:)`, …), and the concurrency reader is an `async` property: `await client.blobUploadConcurrency`. `downloadUrl(blobId:disposition:attachmentFilename:)` stays synchronous, so it is safe to call straight from a SwiftUI `body`.
{{/lang}}

---

## Events

The upload queue emits lifecycle events. `upload-progress` is *not* a byte-level progress event — it fires on status transitions. There is no per-byte progress callback. `delete` issued mid-upload cancels the in-flight transfer and evicts the cached bytes.

{{ example: blobs/db3-blob-events }}

{{#lang ts}}
---

## Service worker proxy (for `<img>` / `<video>`)

The neutral blob calls here are `blobs.proxyUrl(blobId)` / `downloadUrl` (see [Download URL](#download-url-authenticated)) and `blobs.read(blobId, { as: "blob" })` (see [Read content into memory](#read-content-into-memory)). The rest is **web-DOM glue**: `downloadUrl` requires the request to carry the user's auth token, which `<img>` tags don't do, so on the web you either route through a service worker (`proxyUrl`) or read into memory and hand the element a Blob URL.

```typescript
// web-DOM glue around proxyUrl / read — wiring an <img> src.
if (blobs.hasServiceWorkerControl()) {
  imageElement.src = blobs.proxyUrl(blobId, {
    disposition: "inline",
    attachmentFilename: "image.png", // optional
  });
} else {
  // Fallback: read into memory and use a Blob URL.
  const blob = await blobs.read(blobId, { as: "blob" }) as Blob;
  imageElement.src = URL.createObjectURL(blob);
  // Remember to URL.revokeObjectURL(...) when done.
}
```

`proxyUrl` returns the same URL as `downloadUrl`; it just signals intent to be intercepted by the service worker. See the js-bao-wss-client README for a service-worker implementation.
{{/lang}}

---

## From a server function

A server function uploads a database blob with `ctx.api.db3.blobs.upload`: `db3Id`, a `blobId` it mints (a ULID), `"X-Blob-Filename"`, `"X-Blob-Size"` (the byte length), `"X-Blob-Sha256"` (the base64 SHA-256 of the bytes), `body`, and the media type in `"Content-Type"`. `ctx.api.db3.blobs.list`, `get` and `downloadUrl` read it back, and `ctx.api.db3.blobs.delete` removes it. A function reaches the blobs of a database with sync turned off the same way.

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

---

## Permissions (database blobs)

| Operation                              | Required database permission     |
|----------------------------------------|----------------------------------|
| Upload, Delete                         | `read-write` or `owner`           |
| List, Get metadata, Download, Read     | `reader` or higher                |

{{#lang ts}}
---

## Complete example: image upload with progress + display

A **web walkthrough** composing the neutral calls already shown — [Upload](#upload), the `blobs:upload-progress` event ([Events](#events)), and `proxyUrl`/`read` ([Service worker proxy](#service-worker-proxy-for-img--video)) — into one DOM flow. The `<img>`/`URL.createObjectURL` wiring is web-DOM glue:

```typescript
async function uploadAndDisplay(
  client: JsBaoClient,
  db3Id: string,
  file: File,
  imgEl: HTMLImageElement
) {
  if (file.size > 10 * 1024 * 1024) {
    throw new Error("Database blobs are capped at 10 MB. Use a bucket instead.");
  }
  const blobs = client.db3.blobs(db3Id);

  const onProgress = (e: any) => {
    if (e.blobId) console.log(`[${e.blobId}] ${e.status} (attempt ${e.attempts})`);
  };
  client.on("blobs:upload-progress", onProgress);

  try {
    const { blobId } = await blobs.upload(file); // filename + contentType inferred

    if (blobs.hasServiceWorkerControl()) {
      imgEl.src = blobs.proxyUrl(blobId, { disposition: "inline" });
    } else {
      const data = (await blobs.read(blobId, { as: "blob" })) as Blob;
      imgEl.src = URL.createObjectURL(data);
    }
    return blobId;
  } finally {
    client.off("blobs:upload-progress", onProgress);
  }
}
```

## Complete example: offline-ready gallery

A **web walkthrough** composing [List](#listing), `prefetch` ([Offline & the upload queue](#offline--the-upload-queue)), and `proxyUrl` into a gallery loader. Only the URL-wiring is web-specific:

```typescript
async function loadGallery(client: JsBaoClient, db3Id: string) {
  const blobs = client.db3.blobs(db3Id);
  const { items } = await blobs.list({ limit: 100 });
  const images = items.filter((b: any) => b.contentType?.startsWith("image/"));

  await blobs.prefetch(images.map((b: any) => b.blobId), { concurrency: 4 });

  return images.map((b: any) => ({
    id: b.blobId,
    // Use proxyUrl if a service worker is registered, otherwise blob URLs.
    url: blobs.hasServiceWorkerControl()
      ? blobs.proxyUrl(b.blobId, { disposition: "inline" })
      : null,
    filename: b.filename,
  }));
}
```
{{/lang}}

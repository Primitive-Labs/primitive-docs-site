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

### Download URL

The basic call is shown in **List / URL / read** above. `downloadUrl` is synchronous and returns a string. The URL carries no credentials{{#lang ts}}: in an app built from the Vue template, its service worker adds the user's auth (see [Displaying images](#displaying-images)).{{/lang}}{{#lang swift}}: `BlobImage` and `downloadRequest` add the user's auth (see [Displaying images](#displaying-images)).{{/lang}}

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
Every queue verb is `async`: the upload queue lives behind an actor, so reading or changing it is an `await`. The queue members carry the `Async` suffix (`uploadsAsync()`, `pauseUploadAsync(blobId:)`, `pauseAllAsync()`, `setBlobUploadConcurrencyAsync(_:)`, …), and the concurrency reader is an `async` property: `await client.blobUploadConcurrency`. `downloadUrl(blobId:disposition:attachmentFilename:)` and `downloadRequest(blobId:disposition:attachmentFilename:)` stay synchronous, so they are safe to call straight from a SwiftUI `body`.
{{/lang}}

---

## Events

The upload queue emits lifecycle events. `upload-progress` is *not* a byte-level progress event — it fires on status transitions. There is no per-byte progress callback. `delete` issued mid-upload cancels the in-flight transfer and evicts the cached bytes.

{{ example: blobs/db3-blob-events }}

---

## Displaying images

{{ example: blobs/db3-blob-inline-url }}

The image shows for a signed-in user who can read the database.

{{#lang ts}}
Put the URL in an `<img>` or `<video>` source. In an app built from the Vue template, `public/sw.js` intercepts blob downloads and adds the user's auth, and `main.ts` mounts the app once the worker controls the page (or after 3 s). An app not built from the template needs a service worker like the one in the `js-bao-wss-client` README.

Where no worker controls the page (`blobs.hasServiceWorkerControl()` is `false`, as after a hard reload), read the bytes with `blobs.read(blobId, { as: "blob" })` and use `URL.createObjectURL(blob)` as the source; revoke it with `URL.revokeObjectURL` when done.
{{/lang}}
{{#lang swift}}
`BlobImage` is in the `JsBaoClientUI` product (`import JsBaoClientUI`), which the Swift template links. It has the `AsyncImage` shapes: `BlobImage(blobs, blobId:)`, the `content:placeholder:` form above, and `BlobImage(blobs, blobId:) { phase in … }` over `BlobImagePhase` (`.empty`, `.success(Image)`, `.failure(Error)`). It loads through `read`: the user's auth, the copy already on the device, and `JsBaoError` `.offline` when the client is offline with no local copy. The load restarts when the client, the database or the blob changes.

With an image library (Kingfisher, Nuke), send `blobs.downloadRequest(blobId:disposition:attachmentFilename:)`: a `URLRequest` for the download URL with the user's `Authorization` header, built from the token in force when it is called. A bare `downloadUrl` in `AsyncImage` is refused with `401`.
{{/lang}}

---

## From a server function

A server function uploads a database blob with `ctx.api.db3.blobs.upload`: `db3Id`, a `blobId` it mints (a ULID), `"X-Blob-Filename"`, `"X-Blob-Size"` (the byte length), `"X-Blob-Sha256"` (the base64 SHA-256 of the bytes), `body`, and the media type in `"Content-Type"`. `ctx.api.db3.blobs.list` and `get` read its metadata, `ctx.api.db3.blobs.download` returns its bytes as a `Uint8Array`, and `ctx.api.db3.blobs.delete` removes it. A function reaches the blobs of a database with sync turned off the same way.

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

{{#lang ts}}
---

## From a signed-out page: signed upload URLs

A public form takes an attachment without a signed-in user: a server function mints a URL with `ctx.api.db3.blobs.signedUploadUrl` for a database the app holds, and the page uploads the file to it with `client.db3.uploadSigned(url, file)`, which sends no `Authorization` header. The URL accepts one file of its `contentType`, of at most `maxBytes` (up to 10 MB), until it expires. The blob is recorded as uploaded by `sys:<appId>`.

```ts
// In the function: the database id comes from a config var, never from the caller.
const { url } = await ctx.api.db3.blobs.signedUploadUrl({
  db3Id: await ctx.configVar("UPLOADS_DB3_ID"),
  body: { maxBytes: 5 * 1024 * 1024, contentType: input.contentType, filename: input.filename },
});
```

{{ example: blobs/signed-upload }}

| Body field | Values |
|---|---|
| `maxBytes` | integer, 1 to 10,485,760 |
| `contentType` | one lowercase media type, no parameters or wildcard |
| `expiresInSeconds` | integer, 30 to 3,600; default 300 |
| `filename` | 1 to 255 characters, no control characters, `/` or `\`; default `upload` |

The mint answers `{ url, token, tokenId, maxBytes, contentType, filename, expiresAt, expiresInSeconds }`; the upload resolves `{ blobId, tokenId, filename, contentType, numBytes, createdAt }`.

**Gotchas**

- A signed-out page reaches the minting function only when the app sets `[app] signedOutAccess = true` in `app.toml` and the function's TOML declares `public = true` (with an `authorize`, such as `allowAnyone()`); otherwise its call answers `401` before the function runs. A function whose `input` list names `contentType` and `filename` declares both under `[function.inputSchema.properties]`, or push refuses it (`FUNCTION_INPUT_SCHEMA_REQUIRED`).
- Only a server function can mint, and only for a database whose `custodian` is `"app"`: anything else is refused (`FUNCTION_ROUTE_FUNCTION_ONLY`, `DB3_NOT_APP_HELD`). A value outside its range is `SIGNED_UPLOAD_OPTIONS_INVALID`, never clamped.
- A URL is single use. A second upload (`SIGNED_UPLOAD_ALREADY_USED`, 409), another type (`SIGNED_UPLOAD_TYPE_MISMATCH`), a larger file (`SIGNED_UPLOAD_TOO_LARGE`), an expired URL (`SIGNED_UPLOAD_EXPIRED`) or a database deleted or moved since the mint (`SIGNED_UPLOAD_TARGET_GONE`) is refused; an upload cut off part way also uses up the URL. Mint a new URL to retry.
- The function cannot know the `blobId` in advance: the page passes it back, and the function confirms it with `ctx.api.db3.blobs.get`.
- `primitive spent-tokens get <tokenId>` shows whether and when a URL was used.

---

## Public blob URLs

A blob in a public database (`level: "reader"`, set with `client.db3.permissions(db3Id).setPublicAccess`) has a stable URL anyone can load, signed in or not, while the app sets `[app] signedOutAccess = true`. `client.db3.publicBlobUrl(db3Id, blobId)` builds it with no request and no token; use it as an `<img>` source on a public page:

{{ example: blobs/public-blob-url }}

The URL is `<apiUrl>/app/<appId>/api/public/db3/<db3Id>/blobs/<blobId>`; `primitive db3 public-access blob-url <db3-id> <blob-id>` prints it. A `200` carries `Cache-Control: public, max-age=300` and the blob's `ETag`, and `If-None-Match` with it answers `304`. `Content-Disposition` is `inline` for `image/*` other than SVG, `audio/*` and `video/*`, and `attachment` for anything else.

**Gotchas**

- Every other request answers one `404` (`NOT_FOUND`): a private database, the setting off, another app's database, a blob of another database, an unknown or malformed id. Clearing public access or turning the setting off refuses the next request; a browser may show its copy for up to five minutes.
- Requests count against the signed-out limits shared with signed-out function calls: per address (`429 SIGNED_OUT_RATE_LIMITED`, `Retry-After: 60`) and the app's hourly `signedOutCeiling` (`429 SIGNED_OUT_CEILING_REACHED`; `503 SIGNED_OUT_CEILING_UNAVAILABLE` when it cannot be checked). A `200` or `304` is metered as `signedout.blob`.
- The URL ignores any credential and answers the app's allowed origins in CORS; an `<img>` needs neither.
{{/lang}}

---

## Permissions (database blobs)

| Operation                              | Required database permission     |
|----------------------------------------|----------------------------------|
| Upload, Delete                         | `read-write` or `owner`           |
| List, Get metadata, Download, Read     | `reader` or higher                |

{{#lang ts}}
---

## Complete example: image upload with progress + display

A **web walkthrough** composing the neutral calls already shown — [Upload](#upload), the `blobs:upload-progress` event ([Events](#events)), and the download URL ([Displaying images](#displaying-images)) — into one DOM flow, in an app built from the Vue template. The `<img>` wiring is web-DOM glue:

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
    imgEl.src = blobs.downloadUrl(blobId, { disposition: "inline" });
    return blobId;
  } finally {
    client.off("blobs:upload-progress", onProgress);
  }
}
```

## Complete example: offline-ready gallery

A **web walkthrough** composing [List](#listing), `prefetch` ([Offline & the upload queue](#offline--the-upload-queue)), and the download URL ([Displaying images](#displaying-images)) into a gallery loader. Only the URL-wiring is web-specific:

```typescript
async function loadGallery(client: JsBaoClient, db3Id: string) {
  const blobs = client.db3.blobs(db3Id);
  const { items } = await blobs.list({ limit: 100 });
  const images = items.filter((b: any) => b.contentType?.startsWith("image/"));

  await blobs.prefetch(images.map((b: any) => b.blobId), { concurrency: 4 });

  return images.map((b: any) => ({
    id: b.blobId,
    url: blobs.downloadUrl(b.blobId, { disposition: "inline" }), // an <img> src
    filename: b.filename,
  }));
}
```
{{/lang}}

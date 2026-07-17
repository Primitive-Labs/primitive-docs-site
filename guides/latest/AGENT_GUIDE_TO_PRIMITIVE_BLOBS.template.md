# Agent Guide to Primitive Blobs

Guidelines for AI agents implementing file storage in Primitive apps. This guide covers **document-scoped blobs** — binary files attached to a specific document. General-purpose storage outside any document lives in the [Blob Buckets guide](AGENT_GUIDE_TO_PRIMITIVE_BLOB_BUCKETS.md).

## Document-scoped blob operations

Blobs attached to a document are reached through the document's blob accessor.

{{#lang ts}}
```typescript
const blobs = client.document(documentId).blobs();
```
{{/lang}}

{{#lang swift}}
```swift
let blobs = client.documents.blobs(documentId: documentId)
```
{{/lang}}

### Upload

{{ example: blobs/doc-blob-upload }}

### Upload with disposition

`disposition` is stored on the blob (sent as `X-Blob-Disposition`).

{{ example: blobs/doc-blob-upload-options }}

### List / URL / read

{{ example: blobs/doc-blob-read }}

### List / metadata / delete

{{ example: blobs/doc-blob-manage }}

## Overview

**Document-scoped blobs** — binary files attached to a specific document. Access follows the document's permissions. **Cap: 10 MB per blob.**

{{#lang ts}}
```typescript
const blobs = client.document(documentId).blobs();
```

Document blobs include offline caching, an IndexedDB-backed upload queue, and a service-worker proxy for `<img>`/`<video>`.
{{/lang}}

{{#lang swift}}
```swift
let blobs = client.documents.blobs(documentId: documentId)
```
{{/lang}}

For general-purpose storage outside any document context — avatars, workflow outputs, public assets, anonymous reads via signed URLs — use a **blob bucket** instead (100 MB per blob). The [Blob Buckets guide](AGENT_GUIDE_TO_PRIMITIVE_BLOB_BUCKETS.md) covers buckets and the full decision rule for choosing between the two.

---

## Uploading

SHA-256 is computed client-side and used for server-side dedup. See **Upload with disposition** above for the full options form. The download endpoint chooses `Content-Disposition` from the `?disposition=` query param (default `attachment`), not from the upload-time value — pass `disposition` explicitly to the download URL when serving inline.

{{#lang ts}}
`upload` accepts `File | Blob | Uint8Array | ArrayBuffer`. `retainLocal` (default `true`; set `false` to drop local bytes once the server confirms) controls whether local bytes are kept after upload.

### From a file input

The upload call is the compiled [Upload](#upload) example — `client.document(documentId).blobs().upload(file)`. The only web-specific glue is pulling the `File` off an `<input>` (`File` objects are accepted directly; `filename`/`contentType` default to `file.name`/`file.type`):

```typescript
// web-DOM glue: get a File from an <input type="file">, then upload it.
const file = (document.querySelector('input[type="file"]') as HTMLInputElement).files![0];
const { blobId } = await client.document(documentId).blobs().upload(file);
```

### Don't do this

```typescript
// DON'T: manually convert File to ArrayBuffer just to upload it.
const buf = await file.arrayBuffer();
await blobs.upload(buf, { filename: file.name, contentType: file.type });

// DON'T: try to upload >10 MB to document-scoped blobs. The server returns 413.
// Use a blob bucket instead (100 MB cap).
```

### `uploadFile` vs `upload`

Both call the same code path. `uploadFile` returns a slightly trimmed result (`{ blobId, numBytes, bytesTransferred? }`). Prefer `upload`.
{{/lang}}

{{#lang swift}}
`upload` accepts `Data`.

Uploading more than 10 MB to a document-scoped blob returns `413`. Use a blob bucket instead (100 MB cap).
{{/lang}}

### Idempotent re-upload

Uploading the same `blobId` twice with **identical** `sha256` and `size` returns 200 with `bytesTransferred: 0` — the server keeps the existing object, so retries from a flaky network are free. Uploading the same `blobId` with **different** bytes (different sha256 or size) returns 409; pick a new `blobId` instead of overwriting. This dedup behavior is what makes retry-with-backoff upload loops safe.

---

## Listing

See **List / metadata / delete** above for the basic call. Each item carries `blobId`, `filename`, `contentType`, `numBytes`, `sha256`, and `createdAt`. `cursor` is an opaque pagination token; only present when more results exist — follow it to page through results. `limit` accepts `1`–`100`; a larger value is clamped to `100`, and a zero, negative, or non-integer `limit` is rejected with a `400`. Page through the `cursor` for more than 100 blobs rather than asking for a bigger page.

{{ example: blobs/doc-blob-list-paginate }}

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

The basic call is shown in **List / URL / read** above. `downloadUrl` is synchronous and returns a string, authenticated via the user's session against the API origin — it works in an `<a href download>` or `window.location`.

{{ example: blobs/doc-blob-download-url }}

### Read content into memory

`read` is shown in **List / URL / read** above. It checks the cache first; on a miss it fetches from the server and writes the response into the cache.

{{ example: blobs/doc-blob-read-formats }}

{{#lang ts}}
`{ as: "blob" }` is also available and returns a `Blob`. When offline (`networkMode === "offline"`) and the cache is empty, `read` throws `"Blob cache unavailable while offline"`.
{{/lang}}

### Range requests and conditional GETs

The document blob download endpoint supports two HTTP behaviors useful when streaming media or revalidating cached bytes:

- **Range requests.** Send `Range: bytes=START-END` and the server replies with `206 Partial Content` and the requested slice. Out-of-range requests return `416`. This is what media players use to seek; you usually don't issue these manually, but if you build a custom player it works as expected.
- **Conditional GET.** The server emits an `ETag` (the blob's sha256). Send `If-None-Match: <etag>` on subsequent requests and the server returns `304 Not Modified` with no body when the bytes are unchanged.

Both behaviors are on the document blob path only. Bucket blobs go through signed URLs straight to the object store, which has its own range/etag semantics handled by the storage layer.

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
let result = try await blobs.delete(blobId: blobId) // BlobDeleteResult: .deleted
```
{{/lang}}

Deleting also cancels any in-flight upload for the same `blobId` and clears local bytes.

---

## Offline & the upload queue

The upload queue is keyed by user identity, retries with exponential backoff (2s base, 60s max), and persists across reloads. While offline, bytes are written to the local cache immediately and queued; `upload()` resolves without waiting for the network (`bytesTransferred: 0`). Reads are served from the cache for blobs previously uploaded or downloaded on this device/user, and `prefetch` warms the cache — its per-blob errors are logged and swallowed, so it resolves once all attempts complete regardless of individual failures. Coming back online resumes queue processing automatically, draining the queue in the background.

{{ example: blobs/doc-blob-offline }}

---

## Queue management

Inspect and control the per-document upload queue, and set the client-wide upload concurrency (applies to all documents). The `queueId` is the `blobId` for document uploads.

{{ example: blobs/doc-blob-queue }}

---

## Events

The upload queue emits lifecycle events. `upload-progress` is *not* a byte-level progress event — it fires on status transitions. There is no per-byte progress callback. `delete` issued mid-upload cancels the in-flight transfer and evicts the cached bytes.

{{ example: blobs/doc-blob-events }}

{{#lang swift}}
A typed `read(blobId:as:)` overload also JSON-decodes a blob into any `Decodable` type:

```swift
let payload = try await blobs.read(blobId: blobId, as: MyCodable.self)
```
{{/lang}}

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

## Permissions (document blobs)

| Operation                              | Required document permission     |
|----------------------------------------|----------------------------------|
| Upload, Delete                         | `read-write`, `admin`, or `owner` |
| List, Get metadata, Download, Read     | `reader` or higher                |

---

## Document thumbnails and presentation metadata

A document carries two fields that pair naturally with the blob system — set them with the document update call (covered in detail in the [Documents guide](AGENT_GUIDE_TO_PRIMITIVE_DOCUMENTS.md#updating-document-metadata)):

- **`thumbnailBlobId`** points at an existing blob owned by the document. Anyone with access to the document inherits read access to the referenced blob, so a thumbnail uploaded as a private document blob can safely back a thumbnail surface.
- **`metadata`** is a small JSON-serializable object capped at 4KB serialized UTF-8. Use it for presentation hints that should travel with the document (cover image references, color tags, list-card layout, etc.).

Both fields are optional and accept `null` to clear. Errors surface as `BLOB_NOT_FOUND` (referenced blob missing), `BLOB_DOC_MISMATCH` (blob belongs to a different document), or `METADATA_TOO_LARGE` (serialized `metadata` exceeds 4KB).

{{#lang ts}}
---

## Complete example: image upload with progress + display

A **web walkthrough** composing the neutral calls already shown — [Upload](#upload), the `blobs:upload-progress` event ([Events](#events)), and `proxyUrl`/`read` ([Service worker proxy](#service-worker-proxy-for-img--video)) — into one DOM flow. The `<img>`/`URL.createObjectURL` wiring is web-DOM glue:

```typescript
async function uploadAndDisplay(
  client: JsBaoClient,
  documentId: string,
  file: File,
  imgEl: HTMLImageElement
) {
  if (file.size > 10 * 1024 * 1024) {
    throw new Error("Document blobs are capped at 10 MB. Use a bucket instead.");
  }
  const blobs = client.document(documentId).blobs();

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
async function loadGallery(client: JsBaoClient, documentId: string) {
  const blobs = client.document(documentId).blobs();
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
</content>
</invoke>

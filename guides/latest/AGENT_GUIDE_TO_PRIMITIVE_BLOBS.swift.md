# Agent Guide to Primitive Blobs

Guidelines for AI agents implementing file storage in Primitive apps. This guide covers **document-scoped blobs** — binary files attached to a specific document. General-purpose storage outside any document lives in the [Blob Buckets guide](AGENT_GUIDE_TO_PRIMITIVE_BLOB_BUCKETS.md).

## Document-scoped blob operations

Blobs attached to a document are reached through the document's blob accessor.


```swift
let blobs = client.documents.blobs(documentId: documentId)
```

### Upload

```swift
  let blobs = client.documents.blobs(documentId: documentId)
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
      disposition: .attachment  // or .inline
    )
  )
  let blobId = result.blobId
  let numBytes = result.numBytes
```

### List / URL / read

```swift
  let blobs = client.documents.blobs(documentId: documentId)

  let list = try await blobs.list()
  let url = blobs.downloadUrl(blobId: blobId)          // synchronous, authenticated
  let bytes = try await blobs.read(blobId: blobId)
```

### List / metadata / delete

```swift
  let blobs = client.documents.blobs(documentId: documentId)

  let list = try await blobs.list(limit: 50)
  let meta = try await blobs.get(blobId: blobId)
  _ = try await blobs.delete(blobId: blobId)
```

## Overview

**Document-scoped blobs** — binary files attached to a specific document. Access follows the document's permissions. **Cap: 10 MB per blob.**


```swift
let blobs = client.documents.blobs(documentId: documentId)
```

For general-purpose storage outside any document context — avatars, workflow outputs, public assets, anonymous reads via signed URLs — use a **blob bucket** instead (100 MB per blob). The [Blob Buckets guide](AGENT_GUIDE_TO_PRIMITIVE_BLOB_BUCKETS.md) covers buckets and the full decision rule for choosing between the two.

---

## Uploading

SHA-256 is computed client-side and used for server-side dedup. See **Upload with disposition** above for the full options form. The download endpoint chooses `Content-Disposition` from the `?disposition=` query param (default `attachment`), not from the upload-time value — pass `disposition` explicitly to the download URL when serving inline.


`upload` accepts `Data`.

Uploading more than 10 MB to a document-scoped blob returns `413`. Use a blob bucket instead (100 MB cap).

### Idempotent re-upload

Uploading the same `blobId` twice with **identical** `sha256` and `size` returns 200 with `bytesTransferred: 0` — the server keeps the existing object, so retries from a flaky network are free. Uploading the same `blobId` with **different** bytes (different sha256 or size) returns 409; pick a new `blobId` instead of overwriting. This dedup behavior is what makes retry-with-backoff upload loops safe.

---

## Listing

See **List / metadata / delete** above for the basic call. Each item carries `blobId`, `filename`, `contentType`, `numBytes`, `sha256`, and `createdAt`. `cursor` is an opaque pagination token; only present when more results exist — follow it to page through results. `limit` accepts `1`–`100`; a larger value is clamped to `100`, and a zero, negative, or non-integer `limit` is rejected with a `400`. Page through the `cursor` for more than 100 blobs rather than asking for a bigger page.

```swift
  let page1 = try await blobs.list(limit: 50)
  for b in page1.items {
    print(b.blobId, b.filename, b.contentType, b.numBytes, b.sha256, b.createdAt)
  }

  // `cursor` is an opaque token; only present when more results remain.
  if let cursor = page1.cursor {
    let page2 = try await blobs.list(cursor: cursor)
    _ = page2.items
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

### Download URL (authenticated)

The basic call is shown in **List / URL / read** above. `downloadUrl` is synchronous and returns a string, authenticated via the user's session against the API origin — it works in an `<a href download>` or `window.location`.

```swift
  let url = blobs.downloadUrl(
    blobId: blobId,
    disposition: .attachment,            // or .inline
    attachmentFilename: "report.pdf"     // optional override (RFC 5987-encoded)
  )
```

### Read content into memory

`read` is shown in **List / URL / read** above. It checks the cache first; on a miss it fetches from the server and writes the response into the cache.

```swift
  let bytes = try await blobs.read(blobId: blobId)                 // Data
  let text = try await blobs.read(blobId: blobId, as: String.self) // UTF-8 String

  // Bypass the local cache and re-fetch from the server.
  let fresh = try await blobs.read(blobId: blobId, as: String.self, force: true)
```


### Range requests and conditional GETs

The document blob download endpoint supports two HTTP behaviors useful when streaming media or revalidating cached bytes:

- **Range requests.** Send `Range: bytes=START-END` and the server replies with `206 Partial Content` and the requested slice. Out-of-range requests return `416`. This is what media players use to seek; you usually don't issue these manually, but if you build a custom player it works as expected.
- **Conditional GET.** The server emits an `ETag` (the blob's sha256). Send `If-None-Match: <etag>` on subsequent requests and the server returns `304 Not Modified` with no body when the bytes are unchanged.

Both behaviors are on the document blob path only. Bucket blobs go through signed URLs straight to the object store, which has its own range/etag semantics handled by the storage layer.

---

## Deleting

See **List / metadata / delete** above.


```swift
let result = try await blobs.delete(blobId: blobId) // BlobDeleteResult: .deleted
```

Deleting also cancels any in-flight upload for the same `blobId` and clears local bytes.

---

## Offline & the upload queue

The upload queue is keyed by user identity, retries with exponential backoff (2s base, 60s max), and persists across reloads. While offline, bytes are written to the local cache immediately and queued; `upload()` resolves without waiting for the network (`bytesTransferred: 0`). Reads are served from the cache for blobs previously uploaded or downloaded on this device/user, and `prefetch` warms the cache — its per-blob errors are logged and swallowed, so it resolves once all attempts complete regardless of individual failures. Coming back online resumes queue processing automatically, draining the queue in the background.

```swift
  client.setNetworkMode(.offline)

  // Bytes are written to the local cache immediately and queued. upload()
  // resolves without waiting for the network (bytesTransferred is 0).
  let result = try await blobs.upload(
    data: data,
    options: BlobUploadSourceOptions(filename: "draft.txt", contentType: "text/plain")
  )

  // Reads served from cache for blobs uploaded or downloaded on this device.
  let text = try await blobs.read(blobId: result.blobId, as: String.self)

  // Warm the cache; per-blob errors are logged and swallowed.
  await blobs.prefetch(blobIds: blobIds, concurrency: 4)

  // Queue processing resumes automatically; listen for .blobsQueueDrained.
  client.setNetworkMode(.online)
```

---

## Queue management

Inspect and control the per-document upload queue, and set the client-wide upload concurrency (applies to all documents). The `queueId` is the `blobId` for document uploads.

```swift
  // Inspect what's queued for this document. Each BlobUploadStatus carries
  // queueId, blobId, filename, contentType, numBytes, status, attempts,
  // nextAttemptAt, lastError.
  let tasks = blobs.uploads()

  // Pause/resume by blobId (the queueId is the blobId for document uploads).
  _ = blobs.pauseUpload(blobId: blobId)
  _ = blobs.resumeUpload(blobId: blobId)

  blobs.pauseAll()
  blobs.resumeAll()

  // Concurrency is set on the client (global, all documents).
  client.setBlobUploadConcurrency(5) // min 1; default 2
  let concurrency = client.getBlobUploadConcurrency()
```

---

## Events

The upload queue emits lifecycle events. `upload-progress` is *not* a byte-level progress event — it fires on status transitions. There is no per-byte progress callback. `delete` issued mid-upload cancels the in-flight transfer and evicts the cached bytes.

```swift
  let progress = client.events.on(.blobsUploadProgress) { (e: BlobUploadProgressEvent) in
    print(e.queueId, e.blobId, e.status, e.numBytes, e.attempts)
  }

  let completed = client.events.on(.blobsUploadCompleted) { (e: BlobUploadCompletedEvent) in
    print("done", e.blobId, e.queueId)
  }

  let failed = client.events.on(.blobsUploadFailed) { (e: BlobUploadFailedEvent) in
    print("failed", e.blobId, e.lastError ?? "", "retry?", e.willRetry)
  }
```

A typed `read(blobId:as:)` overload also JSON-decodes a blob into any `Decodable` type:

```swift
let payload = try await blobs.read(blobId: blobId, as: MyCodable.self)
```


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

</content>
</invoke>

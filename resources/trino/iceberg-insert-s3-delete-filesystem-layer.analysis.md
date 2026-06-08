---
source: references/trino @ tag 467 — lib/trino-filesystem-s3/
component: trino
type: source-analysis
evidence-tier: official
accessible: true
benchmark-age: n/a
date-retrieved: 2026-06-08
security:
  malicious-code: none
  suspicious-urls: none
  quality: high
---

## Summary

Source-code checkpoint (layer 1 of 3) for the investigation into unexpected S3 DELETE
requests issued by Trino 467's Iceberg connector during `INSERT INTO`. This file covers the
**S3 filesystem layer** (`lib/trino-filesystem-s3`): how `TrinoFileSystem` delete calls become
actual AWS SDK v2 calls / S3 HTTP requests against MinIO. All citations are against the local
checkout at tag 467.

## Key Points

- **`deleteFile(Location)`** → single-object `DeleteObjectRequest` → HTTP `DELETE /bucket/key`.
  `S3FileSystem.java:134-153` (`client.deleteObject(request)` at :148). Calls
  `verifyValidFileLocation()` first (:138), so empty / `/`-terminated paths throw before any request.
- **`deleteFiles(Collection)`** and **`deleteDirectory(Location)`** both funnel into the private
  **`deleteObjects(...)`** helper (`S3FileSystem.java:177-217`) → bulk `DeleteObjectsRequest` →
  HTTP **`POST /bucket?delete`** with an XML key list. `quiet(true)` set at :199.
- **Bulk batch size = 250 keys per request**, hardcoded via Guava `partition(allKeys, 250)` at
  `S3FileSystem.java:190`. No named config constant. This is below the S3 protocol max of 1000.
  (NB: a separate `1000` literal at :162 only bounds the in-memory list inside `deleteDirectory`
  before it calls `deleteObjects`, which then re-chunks at 250.)
- **Deletes are grouped per bucket**: `deleteObjects` builds a `SetMultimap<bucket,key>`
  (`HashMultimap`) and issues one `DeleteObjectsRequest` per bucket per 250-key chunk
  (`S3FileSystem.java:180-198`). Duplicate keys de-duplicated; key order not preserved.
- **`deleteDirectory` = list-then-bulk-delete** (`S3FileSystem.java:155-167`): calls
  `listObjects(location, true)` with `includeDirectoryObjects=true` (so zero-byte `key/`
  placeholders are also deleted), accumulates up to 1000 locations, then `deleteObjects` re-chunks
  at 250. Prefix logic: a non-empty key not ending in `/` gets `/` appended (:239-241) and is used
  as the `ListObjectsV2` prefix. One `deleteDirectory` fans out into multiple LIST GETs +
  multiple delete POSTs.
- **S3Location parsing** (`S3Location.java:40-48`): `bucket = location.host()`, `key = location.path()`
  (leading slash stripped by `Location`). Scheme must be `s3`/`s3a`/`s3n`.
- **renameFile / renameDirectory throw `IOException("S3 does not support renames")**
  (`S3FileSystem.java:219-224`, `280-285`) — no copy+delete fallback at this layer, so renames here
  never produce deletes.
- **SDK retries can re-send a DELETE / POST ?delete**: `s3.retry-mode` default `LEGACY`
  (`S3FileSystemConfig.java:118`), `s3.max-error-retries` default `10` (:119). On 5xx/throttle/timeout
  the SDK re-sends the same request — visible as extra DELETE traffic on MinIO even for one logical delete.
- **Multipart abort is a DELETE-verb request but NOT a DeleteObject/DeleteObjects.**
  `S3OutputStream.abortUpload()` (`S3OutputStream.java:375-385`) issues
  `AbortMultipartUploadRequest` → HTTP `DELETE /bucket/key?uploadId=<id>`, only if an `uploadId`
  exists (write crossed into multipart mode; default `s3.streaming.part-size` = 16 MB,
  `S3FileSystemConfig.java:103`). Triggered on write failure (`:154-163`) or exclusive-create `412`
  precondition failure (`:219-242`, `:366-368`; `s3.exclusive-create` default `true`,
  `S3FileSystemConfig.java:120`). Small single-part writes use `PutObject` and on failure produce
  no delete.
- **No background multipart sweeper, no temp-file deletes, no delete on list/create.**
  `createTemporaryDirectory` / `createDirectory` are no-ops; listing only issues `ListObjectsV2` GETs.

### Layer conclusion
At this layer, `INSERT INTO` itself never calls delete. Real `DeleteObject` (single) and
`DeleteObjects` (bulk POST `?delete`, ≤250 keys/bucket) requests originate only from explicit
`deleteFile`/`deleteFiles`/`deleteDirectory` calls made by the connector / Iceberg core above.
Other DELETE-verb traffic during INSERT is `AbortMultipartUpload` (`?uploadId=`) plus SDK retries.

## Security Notes

No issues detected. This is read-only analysis of upstream Apache-licensed Trino source.
Checks performed:
- Malicious or obfuscated code: none — standard AWS SDK v2 usage.
- Suspicious URLs or redirects: none.
- Content quality / AI-generated: high; derived directly from cited source lines.

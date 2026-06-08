---
source: references/iceberg @ apache-iceberg-1.11.0 — aws/ + core/ FileIO delete path
component: iceberg
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

Checkpoint (layer 1 of 3) for the Spark 3.5.1 + Iceberg 1.11.0 S3-delete investigation: how an
Iceberg `FileIO.deleteFiles(...)` call becomes a real S3 request, and the crucial split between
**bulk** (`DeleteObjects`, POST `?delete`) and **single** (`DeleteObject`, HTTP DELETE) — which
depends entirely on which FileIO the catalog uses. Bulk deletes are produced **only** by an
`io-impl` that implements `SupportsBulkOperations` (i.e. `S3FileIO`), not by `HadoopFileIO` (s3a).

## Key Points

- **Bulk-vs-single is decided by `SupportsBulkOperations`.** Two routing points both branch on it:
  - `SparkCleanupUtil.deleteFiles(...)` — `if (io instanceof SupportsBulkOperations) CatalogUtil.deleteFiles(io, paths, "")` else per-file `delete(...)` (`spark/v3.5/.../source/SparkCleanupUtil.java:85-92`).
  - `CatalogUtil.deleteFiles(...)` — `if (io instanceof SupportsBulkOperations bulkIO) bulkIO.deleteFiles(files)` else `concurrentlyDeleteFiles`/per-file (`core/src/main/java/org/apache/iceberg/CatalogUtil.java:222-239`).
- **S3FileIO IS bulk-capable.** `S3FileIO.deleteFiles(Iterable<String>)`
  (`aws/src/main/java/org/apache/iceberg/aws/s3/S3FileIO.java:202-260`):
  - gated by `s3.delete-enabled` (`isDeleteEnabled()`, default `true`) at `:222`;
  - groups keys per bucket into a `SetMultimap`, flushing a batch when it reaches
    `deleteBatchSize` (`:232`), remaining keys flushed at the end;
  - each batch is one `DeleteObjectsRequest` → `client.s3().deleteObjects(request)` →
    HTTP **`POST /bucket?delete`** (`:305-312`), submitted **concurrently** on the FileIO executor.
  - Optional `s3.delete.tags` mode tags objects instead of/in addition to deleting (`:204-220`).
- **Bulk batch size = 250 keys** by default, configurable, hard max 1000:
  `S3FileIOProperties.DELETE_BATCH_SIZE = "s3.delete.batch-size"` (`:311`),
  `DELETE_BATCH_SIZE_DEFAULT = 250` (`:319`), `DELETE_BATCH_SIZE_MAX = 1000` (`:325`),
  applied in ctor `this.deleteBatchSize = DELETE_BATCH_SIZE_DEFAULT` (`:564`). So a 1000-file
  bulk delete = 4 concurrent `DeleteObjects` POSTs per bucket.
- **Single delete**: `S3FileIO.deleteFile(path)` → `DeleteObjectRequest` → HTTP `DELETE /bucket/key`.
- **HadoopFileIO is NOT bulk-capable** → both routers fall to per-file deletes. `SparkCleanupUtil`'s
  per-file path retries each delete (`DELETE_NUM_RETRIES = 3`, exponential backoff,
  `SparkCleanupUtil.java:94-118`); `CatalogUtil.concurrentlyDeleteFiles` runs per-file deletes with
  `noRetry()` on the worker pool (`CatalogUtil.java:241-248`). **Therefore: observing bulk
  `DeleteObjects` on MinIO ⟹ the catalog is using `S3FileIO`, not s3a/HadoopFileIO.**
- **Core commit cleanup uses SINGLE deletes, not bulk.** `SnapshotProducer` deletes manifests one
  at a time via `ops.io().deleteFile(...)` (`core/.../SnapshotProducer.java:101, 586, 727`), so
  manifest-merge / commit-retry cleanup shows up as single `DeleteObject`s, never as `DeleteObjects`.
  Bulk delete is exclusively the Spark cleanup-util / `CatalogUtil.deleteFiles` data-file path.

### Decision rule
- `POST /bucket?delete` (bulk, ≤250 keys/bucket) ⟹ `S3FileIO` + `SparkCleanupUtil`/`CatalogUtil.deleteFiles`
  (write abort or maintenance), **not** core commit.
- Single `DELETE /bucket/key` ⟹ either S3FileIO single delete (core manifest cleanup) or any
  per-file delete under HadoopFileIO.

## Security Notes

No issues detected. Read-only analysis of Apache-licensed Iceberg source.
Checks performed:
- Malicious or obfuscated code: none — standard AWS SDK v2 `DeleteObjects` usage.
- Suspicious URLs or redirects: none.
- Content quality / AI-generated: high; direct source citations.

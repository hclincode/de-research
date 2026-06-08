---
title: When Spark 3.5.1 + Iceberg 1.11.0 issues S3 DELETE requests (incl. bulk) during INSERT and MERGE/UPDATE/DELETE
date: 2026-06-08
status: complete
components: [spark, iceberg, s3-minio]
constraints:
  - spark-version: "3.5.1"
  - iceberg-version: "1.11.0"
  - storage: minio (s3 protocol), front layer openresty -> haproxy -> minio cluster
  - operations: INSERT/append and MERGE/UPDATE/DELETE
  - fileio: both S3FileIO and HadoopFileIO considered
clarification-defaults:
  scope-layer: full write path (Spark write/abort + Iceberg core commit + FileIO->S3)
  deployment-context: existing deployment, diagnosing observed bulk deletes
  workload-profile: batch INSERT + row-level MERGE/UPDATE/DELETE; success and failure/abort/retry
---

## Overview

This report traces, against the actual **Apache Iceberg 1.11.0** source (`references/iceberg` @
tag `apache-iceberg-1.11.0`) and **Apache Spark 3.5.1** (`references/spark` @ `v3.5.1`), the exact
conditions under which the Spark + Iceberg client issues S3 DELETE requests — single-object
`DeleteObject` and **bulk `DeleteObjects`** — to MinIO during `INSERT`/append and row-level
`MERGE`/`UPDATE`/`DELETE`. It is the Spark-stack counterpart to the prior Trino 467 / Iceberg 1.7.0
report. User-specified scope (Phase 1.5 answers): operations = INSERT + MERGE/UPDATE/DELETE; FileIO =
cover both S3FileIO and HadoopFileIO; timing = cover both clean-commit and failure/abort/retry.

**Verdict: A *bulk* `DeleteObjects` request during a write is produced almost exclusively by Spark's
abort cleanup — a task abort (`DataWriter.abort()` → `deleteTaskFiles`) or a job abort
(`SparkWrite.abort` → `deleteFiles`, only on a cleanable commit failure) — routed through
`S3FileIO`'s bulk delete (batches of 250 keys/bucket). A fully clean INSERT/MERGE that has no task
aborts and no commit failure issues *no* bulk deletes: plain appends delete nothing (or, once a
manifest group passes 100 manifests, only single-object manifest deletes), and COW/MOR
MERGE/UPDATE/DELETE commits remove old files only in metadata, never physically at commit. Because
you observe bulk deletes, your catalog is using `S3FileIO` (HadoopFileIO cannot bulk-delete), and the
deletes are coming from aborted task/job attempts — most commonly speculative execution and task
retries during otherwise-successful jobs. Physical bulk deletion of committed data also occurs if
maintenance procedures (`expire_snapshots`, `remove_orphan_files`, `rewrite_data_files`) run.**

See the Decision Matrix to map an observed request to its cause.

## Evidence Quality

| Source | File | Tier | Accessible |
|---|---|---|---|
| S3FileIO / FileIO bulk-vs-single delete mechanics | [link](../resources/iceberg/s3fileio-bulk-delete-mechanics.analysis.md) | official | Yes |
| Spark 3.5.1 Iceberg write/abort delete triggers | [link](../resources/iceberg/spark-iceberg-s3-delete-write-path.analysis.md) | official | Yes |
| Iceberg 1.11.0 core commit/cleanup | [link](../resources/iceberg/core-commit-cleanup-1.11.0.analysis.md) | official | Yes |

**Gaps and confidence limits:**
- All findings are file-backed against the checked-out 1.11.0 / Spark 3.5.1 sources →
  **HIGH confidence**, with one exception: the *frequency* of task/job aborts in your environment is
  empirical and not derivable from source — confirm via Spark event logs (Finding 1).
- Scope is the **write/commit path of INSERT and MERGE/UPDATE/DELETE**. Maintenance procedures
  (`expire_snapshots`, `remove_orphan_files`, `rewrite_data_files`, `rewrite_manifests`) are noted as
  bulk-delete sources but their full internals are out of scope.
- The OpenResty → HAProxy front layer is not a delete source; it forwards. It can duplicate/retry
  requests at the proxy level (Finding 5).

## Components

### `spark` (3.5.1) + `iceberg` (1.11.0) write path

Spark's Iceberg `DataSourceV2` write produces `DataWriter`s on executors and a `BatchWrite`
committer on the driver. INSERT → `BatchAppend.commit()` → `table.newAppend()` (MergeAppend,
`SparkWrite.java:297`). COW MERGE/UPDATE/DELETE → `CopyOnWriteOperation.commit()` →
`OverwriteFiles.deleteFiles(...)` which is **metadata-only** (`SparkWrite.java:439`). MOR →
`SparkPositionDeltaWrite` → `RowDelta` (`:211`), also metadata-only. Cleanup of *written* files on
failure goes through `SparkCleanupUtil`.

### `iceberg` core (1.11.0)

`newAppend()` = MergeAppend (`core/.../BaseTable.java:191`); manifest-merge enabled, threshold 100
(`TableProperties.java:118-122`). All core commit deletes are **single** `ops.io().deleteFile(...)`
(`SnapshotProducer.java:586`), never bulk. `write.metadata.delete-after-commit.enabled=false` default
(`TableProperties.java:330-332`).

### `s3-minio` (FileIO → S3)

`S3FileIO.deleteFiles` → per-bucket batches of `s3.delete.batch-size` (default **250**, max 1000) →
`DeleteObjectsRequest` → HTTP `POST /bucket?delete`, run concurrently
(`aws/.../S3FileIO.java:202-260, 305-312`; `S3FileIOProperties.java:311-325`). `HadoopFileIO` does
not implement `SupportsBulkOperations`, so it can only issue per-object deletes.

## Findings

### 1. Bulk `DeleteObjects` during writes ⟹ Spark abort cleanup (task or job)

Evidence: [write-path analysis](../resources/iceberg/spark-iceberg-s3-delete-write-path.analysis.md)
(`SparkWrite.java:775-779, 832-836`, `:246-252`; `SparkPositionDeltaWrite.java:629-633, 706-710, 298-300`);
[bulk mechanics](../resources/iceberg/s3fileio-bulk-delete-mechanics.analysis.md)
(`SparkCleanupUtil.java:85-92`, `CatalogUtil.java:222-239`). **Confidence: HIGH.**

The only write-time code that calls the **bulk** delete routine (`SparkCleanupUtil.deleteFiles`/
`deleteTaskFiles` → `CatalogUtil.deleteFiles` → `S3FileIO.deleteFiles`) is abort handling:
- **Task abort** — `DataWriter.abort()` bulk-deletes the data/delete files *that task attempt
  already wrote*. Spark invokes `abort()` on task failure, **speculative-execution kill of the losing
  duplicate**, stage retry, or executor loss. These happen during jobs that ultimately succeed, so
  they look like "unexpected" deletes.
- **Job abort** — `SparkWrite.abort` bulk-deletes *all* files the job wrote, but **only when the
  commit failed with a `CleanableFailure`** (`cleanupOnAbort`, `SparkWrite.java:241`). On
  `CommitStateUnknownException` it skips cleanup ("Skipping cleanup of written files", `:250`).

Because core commit cleanup uses single deletes (Finding 3), any **bulk** `DeleteObjects` you see
during INSERT/MERGE is one of these abort paths. Confirm by correlating delete bursts with Spark task
failures / speculative-execution kills in the event log.

### 2. Bulk delete requires S3FileIO — its presence tells you the FileIO

Evidence: [bulk mechanics](../resources/iceberg/s3fileio-bulk-delete-mechanics.analysis.md)
(`SparkCleanupUtil.java:87`, `CatalogUtil.java:224`, `S3FileIO.java:202-260`). **Confidence: HIGH.**

Both delete routers branch on `io instanceof SupportsBulkOperations`. `S3FileIO` implements it (bulk
`DeleteObjects`, 250/bucket); `HadoopFileIO` (s3a) does not, so it falls back to per-object deletes
(`SparkCleanupUtil` retries each 3×; `CatalogUtil.concurrentlyDeleteFiles` no-retry). **Observing
bulk `POST ?delete` on MinIO ⟹ your catalog `io-impl` is `org.apache.iceberg.aws.s3.S3FileIO`.** If
you instead see only single `DELETE`s, you may be on HadoopFileIO, or the deletes are core
single-object manifest cleanup.

### 3. A clean append deletes nothing (until 100 manifests); core cleanup is single-delete only

Evidence: [core analysis](../resources/iceberg/core-commit-cleanup-1.11.0.analysis.md)
(`BaseTable.java:191`, `TableProperties.java:118-122`, `SnapshotProducer.java:504-586, 727`).
**Confidence: HIGH.**

INSERT uses MergeAppend. Below 100 manifests in a partition-spec group, commit deletes nothing. At
≥100 manifests, merge rewrites small manifests and deletes the **newly-written, now-subsumed**
manifest object(s) — via **single** `deleteFile`, so individual `DeleteObject`s on small `*.avro`
keys, not bulk. Old committed manifests and all data files are untouched. Commit-retry cleanup is
also single-delete and skipped entirely on `CommitStateUnknownException`. `write.metadata.delete-
after-commit.enabled` is false by default, so old `metadata.json` files are not deleted.

### 4. COW and MOR MERGE/UPDATE/DELETE do NOT physically delete old files at commit

Evidence: [write-path analysis](../resources/iceberg/spark-iceberg-s3-delete-write-path.analysis.md)
(`SparkWrite.java:434-439`, `SparkPositionDeltaWrite.java:211`). **Confidence: HIGH.**

Copy-on-write `MERGE/UPDATE/DELETE` calls `OverwriteFiles.deleteFiles(overwrittenFiles, danglingDVs)`
— this is a **metadata** removal (the files leave the new snapshot); the physical objects stay on S3
until `expire_snapshots`. Merge-on-read writes delete files via `RowDelta`, also metadata-only. So a
clean row-level operation issues **no physical S3 data delete** at commit (only possible
manifest-merge single deletes per Finding 3, plus any abort cleanup per Finding 1).

### 5. Request shape, batching, retries, and the OpenResty/HAProxy layer

Evidence: [bulk mechanics](../resources/iceberg/s3fileio-bulk-delete-mechanics.analysis.md)
(`S3FileIO.java:222, 232, 305-312`; `S3FileIOProperties.java:311-325`). **Confidence: HIGH.**

- **Bulk** = `POST /bucket?delete`, ≤250 keys/bucket (`s3.delete.batch-size`), submitted
  concurrently per batch; gated by `s3.delete-enabled` (default true). 1000 files = 4 concurrent
  POSTs/bucket.
- **Single** = `DELETE /bucket/key`.
- An optional `s3.delete.tags` mode tags objects instead of deleting — if set, you'd see
  `PutObjectTagging` rather than deletes.
- **Retries/duplication**: the AWS SDK retries failed requests; behind **OpenResty → HAProxy**, a
  slow/dropped upstream response can also cause SDK- or proxy-level retries, so MinIO may log more
  `DeleteObjects`/`DeleteObject` than logical deletes. Suspect this if you see duplicate identical
  bulk-delete POSTs.

### 6. Maintenance procedures are the other bulk-delete source (only if you run them)

Evidence: [core analysis](../resources/iceberg/core-commit-cleanup-1.11.0.analysis.md).
**Confidence: HIGH (that they are not auto-triggered); MEDIUM (internals out of scope).**

`expire_snapshots`, `remove_orphan_files`, `rewrite_data_files`, and `rewrite_manifests` physically
delete data/delete/manifest/metadata files through `CatalogUtil.deleteFiles` → S3FileIO bulk
`DeleteObjects`. They are **never** auto-invoked by INSERT/MERGE. If bulk deletes correlate with a
schedule rather than with write jobs, check for these (often run by Airflow/cron or table
maintenance jobs).

## Decision Matrix

| Observed on MinIO during INSERT/MERGE | Most likely cause | Where | Confidence | Confirm / fix |
|---|---|---|---|---|
| `POST /bucket?delete` (bulk) bursts during a running job | Task abort cleanup (speculative exec / task retry / executor loss) | Finding 1 | HIGH | Check Spark event log for task aborts / speculation; disable `spark.speculation` to test |
| `POST /bucket?delete` (bulk) right after a job failure | Job abort cleanup (cleanable commit failure) | Finding 1 | HIGH | Inspect the commit exception; `CommitStateUnknown` would instead skip cleanup |
| Any bulk `DeleteObjects` at all | FileIO is `S3FileIO` | Finding 2 | HIGH | Confirm catalog `io-impl=org.apache.iceberg.aws.s3.S3FileIO` |
| Single `DELETE` on small `*.avro` (manifest) keys | MergeAppend manifest-merge ≥100 manifests | Finding 3 | HIGH | Run `rewrite_manifests`/`expire_snapshots` to keep manifest count <100 |
| Expecting deletes of old data files after MERGE/UPDATE/DELETE, but none at commit | COW/MOR removes in metadata only | Finding 4 | HIGH | Expected; physical delete happens at `expire_snapshots` |
| Duplicate identical bulk/single deletes | SDK or OpenResty/HAProxy retry | Finding 5 | HIGH | Check proxy timeouts/retries and `s3.retry` settings |
| Bulk deletes on a schedule, not tied to write jobs | Maintenance procedures | Finding 6 | HIGH | Audit `expire_snapshots`/`remove_orphan_files`/`rewrite_*` jobs |

## Recommended diagnostic steps

1. Capture the exact MinIO request line: `POST /b?delete` (bulk) vs `DELETE /b/key` (single) vs
   `?uploadId=` (multipart abort). This alone selects a matrix row.
2. Inspect deleted **keys**: data files (`.parquet`) vs manifest `.avro` vs delete files — data-file
   bulk deletes point to abort (Finding 1); manifest single deletes point to merge (Finding 3).
3. Correlate delete bursts with **Spark event-log task aborts / speculative execution**. If they
   line up, it is Finding 1. Temporarily set `spark.speculation=false` to confirm.
4. Confirm catalog `io-impl` (Finding 2) and whether any maintenance procedures run on a schedule
   (Finding 6).

## References

- [S3FileIO / FileIO bulk-vs-single delete mechanics](../resources/iceberg/s3fileio-bulk-delete-mechanics.analysis.md)
- [Spark 3.5.1 Iceberg write/abort delete triggers](../resources/iceberg/spark-iceberg-s3-delete-write-path.analysis.md)
- [Iceberg 1.11.0 core commit/cleanup](../resources/iceberg/core-commit-cleanup-1.11.0.analysis.md)
- Source: `references/iceberg` @ `apache-iceberg-1.11.0`, `references/spark` @ `v3.5.1`
- Related: [Trino 467 / Iceberg 1.7.0 INSERT S3-delete report](20260608-trino-iceberg-insert-s3-delete-conditions.report.md)

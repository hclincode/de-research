---
title: Unexpected S3 bulk-delete (DeleteObjects) on MinIO from Iceberg — cross-engine root cause analysis (Trino 467 + Spark 3.5.1)
date: 2026-06-08
status: complete
components: [trino, spark, iceberg, s3-minio, openresty, haproxy]
constraints:
  - storage: minio cluster behind openresty (entrypoint) -> haproxy (LB) -> minio
  - engines: trino 467 (iceberg 1.7.0) AND spark 3.5.1 (iceberg 1.11.0)
  - symptom: unexpected S3 DELETE / bulk-delete (DeleteObjects) requests observed on MinIO side
  - operations: INSERT/append and MERGE/UPDATE/DELETE
synthesis-of:
  - 20260608-trino-iceberg-insert-s3-delete-conditions.report.md
  - 20260608-spark-iceberg-insert-merge-s3-delete-conditions.report.md
---

## Overview

Two independent engines — **Trino 467** (Iceberg 1.7.0) and **Spark 3.5.1** (Iceberg 1.11.0) —
writing Iceberg tables onto the **same** MinIO deployment (OpenResty entrypoint → HAProxy → MinIO
cluster) both exhibit the **same symptom**: unexpected S3 delete traffic, including **bulk
`DeleteObjects`** (`POST /bucket?delete`), during `INSERT` and `MERGE/UPDATE/DELETE`. This report
synthesizes the two prior source-code investigations to rank the **top root causes** of the shared
behavior.

The crucial structural fact: the two engines reach S3 by **different code paths** yet produce
**identical wire behavior**, because both delete for the **same logical reason**.

| Aspect | Trino 467 | Spark 3.5.1 + Iceberg 1.11.0 |
|---|---|---|
| Path to S3 | `ForwardingFileIo` → `TrinoFileSystem` → `S3FileSystem` | Iceberg native `S3FileIO` |
| Bulk delete API | `DeleteObjectsRequest` → `POST /bucket?delete` | `DeleteObjectsRequest` → `POST /bucket?delete` |
| Batch size / bucket | **250** (re-chunked from 1000 logical) | **250** (`s3.delete.batch-size`) |
| What fires the bulk delete | `cleanExtraOutputFiles` (when `retry-policy != NONE`) + page-sink `abort()` | task `abort()` + job `abort()` (on cleanable commit failure) |
| Shared precondition | **files written by a task/query attempt that never committed** | **files written by a task/job attempt that never committed** |

**Verdict: The bulk `DeleteObjects` requests are not part of a normal, clean write — in both
engines they are *cleanup of orphaned output files left by write attempts that did not commit*
(failed, retried, or speculatively-duplicated tasks; aborted queries; fault-tolerant-execution
attempts). Because both engines hit the *same* OpenResty→HAProxy→MinIO path, the single most likely
shared root cause is that this storage path is intermittently failing or slowing write/commit
requests, which induces the task/query retries and aborts whose cleanup logic then issues the bulk
deletes. The #1 application-level enabler is that retry/speculation is active in both engines
(Spark speculative execution / task retries; Trino fault-tolerant execution, `retry-policy=TASK`).
A clean, retry-free write on a healthy storage path issues no bulk deletes in either engine.**

## Evidence Quality

| Source | File | Tier | Accessible |
|---|---|---|---|
| Trino 467 INSERT S3-delete analysis (3 layers + report) | [report](20260608-trino-iceberg-insert-s3-delete-conditions.report.md) | official | Yes |
| Spark 3.5.1 + Iceberg 1.11.0 S3-delete analysis (3 layers + report) | [report](20260608-spark-iceberg-insert-merge-s3-delete-conditions.report.md) | official | Yes |

**Gaps and confidence limits:**
- The *mechanisms* (which code issues bulk delete, under what precondition) are **HIGH confidence** —
  file-backed in both prior reports against the checked-out 467 / 1.7.0 / 1.11.0 / Spark-3.5.1 sources.
- The *ranking of root causes for your specific environment* is **inference**, not source fact. It
  rests on the observed symptom + the shared infrastructure variable. Items below are marked
  HIGH/MEDIUM/LOW accordingly; the storage-path-instability hypothesis (Cause 2) and the
  proxy-duplication hypothesis (Cause 3) need confirmation from MinIO/OpenResty/HAProxy logs and
  engine event logs (see Diagnostics).
- This report covers `INSERT` and `MERGE/UPDATE/DELETE` writes. Scheduled maintenance
  (`expire_snapshots`, `remove_orphan_files`, `rewrite_*`) is included as a candidate but its
  internals are out of scope.

## Top root causes (ranked)

### 1. Orphaned-output cleanup from non-committed write attempts — the shared mechanism

**Confidence: HIGH (mechanism); HIGH (that it is the bulk-delete source in both engines).**

This is the one cause both engines unambiguously share. A bulk `DeleteObjects` during a write means
the engine is deleting data/delete files written by an attempt that did **not** end up in the
committed snapshot:

- **Trino**: `cleanExtraOutputFiles` lists the output directory and bulk-deletes files named
  `{queryId}-…` that are not in the committed set — invoked from `finishInsert` **whenever
  `retry-policy != NONE`** (fault-tolerant execution). Also, page-sink `abort()` deletes a failed
  query/task's files. (Trino report, Findings 2 & 4.)
- **Spark**: `DataWriter.abort()` → `deleteTaskFiles` bulk-deletes the aborting **task** attempt's
  files (task failure, **speculative-execution** loser, stage retry, executor loss);
  `SparkWrite.abort` → `deleteFiles` bulk-deletes the whole **job's** files on a cleanable commit
  failure. (Spark report, Finding 1.)

In both, the bulk delete is *correct, expected cleanup* — what is "unexpected" is that there are
orphaned files to clean up at all, which means attempts are being retried/aborted. The next two
causes explain *why*.

### 2. The shared OpenResty→HAProxy→MinIO path is intermittently failing/slowing writes — most likely *common* trigger

**Confidence: MEDIUM (strong by elimination — it is the only variable both stacks share).**

Two different engines, two different Iceberg versions, two different S3 client stacks, **one** shared
storage path — yet the same symptom. The most parsimonious explanation for a *common* effect is a
*common* cause: the storage path itself. If OpenResty/HAProxy/MinIO intermittently returns 5xx,
resets connections, or is slow enough to trip client timeouts on PUT/multipart-complete or on the
commit's metadata/manifest writes, then:

- the write task fails → the engine aborts that attempt → **bulk-deletes its files** (Cause 1);
- and/or the engine retries the task/query → the losing attempt's files become orphans →
  **bulk-deleted** by Trino's `cleanExtraOutputFiles` / Spark's task abort.

Candidate infra faults to check on this path:
- HAProxy `timeout server`/`timeout client` shorter than large PUT / multipart-complete durations;
- OpenResty buffering or `Expect: 100-continue` / chunked-body handling interfering with S3 PUT or
  the `POST ?delete` body (Content-MD5/checksum mismatches surface as failures → retries);
- MinIO behind HAProxy not being a single distributed/erasure-coded cluster (e.g. independent pools)
  so a freshly-written object isn't immediately visible on a different backend, making Iceberg treat
  files as missing/orphaned;
- backend node flapping / rolling restarts during writes.

This is the hypothesis to prioritize precisely because it is the shared variable.

### 3. SDK- and/or proxy-level retries duplicating delete (and write) requests

**Confidence: MEDIUM.**

Both engines' AWS SDKs retry failed requests (Trino default `s3.max-error-retries=10`; Iceberg
S3FileIO has its own retry config), and OpenResty/HAProxy can themselves retry idempotent-looking
requests on upstream timeout. A `DeleteObjects` or PUT whose response is lost gets re-sent, so MinIO
logs **more delete requests than logical deletes**. This inflates the apparent volume and can make
ordinary cleanup look like a storm. It compounds Causes 1–2 rather than being independent.

### 4. Retry/speculation/fault-tolerant execution is enabled in both engines — the application enabler

**Confidence: MEDIUM (HIGH that it gates the cleanup; environment-specific whether it's on).**

Cause 1's cleanup paths only fire when attempts are retried/aborted:
- **Spark**: `spark.speculation=true` and task retries (`spark.task.maxFailures`) create duplicate/
  failed attempts whose losers are bulk-deleted on abort.
- **Trino**: `retry-policy=TASK` (fault-tolerant execution) is what makes `cleanExtraOutputFiles`
  run on every insert (a directory LIST every time, plus bulk delete when orphans exist).

If both clusters run with these on (common in production), the cleanup machinery is armed; any
instability from Cause 2 then turns into visible bulk deletes. Turning speculation / FTE off is the
fastest way to test whether the deletes are attempt-cleanup.

### 5. Scheduled maintenance procedures (only if you run them)

**Confidence: LOW–MEDIUM (depends on whether such jobs exist).**

`expire_snapshots`, `remove_orphan_files`, `rewrite_data_files`, `rewrite_manifests` physically
bulk-delete committed data/manifest/metadata via the same `DeleteObjects` path, in both Trino and
Spark. They are **never** auto-triggered by INSERT/MERGE. If the bulk deletes correlate with a
*schedule* rather than with write jobs, this is the cause. Distinguishable because the deleted keys
are committed data files / old `metadata.json`, not `{queryId}-…` staging files.

### Not the bulk-delete source (ruled out)

- **Manifest-merge cleanup (≥100 manifests, MergeAppend default in both)** issues **single**
  `DeleteObject`s on small `.avro` manifests (Iceberg core `SnapshotProducer` deletes one file at a
  time), **not** bulk `DeleteObjects`. A recurring trickle of single deletes on `.avro` keys is this,
  and is benign. (Both reports, core layer.)
- **Clean COW/MOR MERGE/UPDATE/DELETE commits** remove old data only in *metadata*; no physical
  delete at commit. (Spark report, Finding 4.)
- **`AbortMultipartUpload`** (`DELETE …?uploadId=`) is a DELETE *verb* but not an object delete;
  it comes from failed/precondition large writes (Trino report, Finding 6) — itself a symptom of
  Cause 2.

## Cross-engine decision matrix

| Observation on MinIO | Most likely cause | Confidence | Confirm / fix |
|---|---|---|---|
| Bulk `POST ?delete` bursts during running jobs, keys = `{queryId}-…`/task staging files | Cause 1 + Cause 4 (attempt cleanup, retries/speculation on) | HIGH | Spark: set `spark.speculation=false`; Trino: set `retry-policy=NONE` to test |
| Bulk/write failures + deletes correlate with storage-path errors/latency | Cause 2 (infra instability inducing retries) | MEDIUM | Inspect MinIO 5xx, HAProxy timeouts/resets, OpenResty error log during write windows |
| Duplicate identical `DeleteObjects`/PUT | Cause 3 (SDK/proxy retry) | MEDIUM | Check `s3.retry`/`s3.max-error-retries`, HAProxy/OpenResty retry + timeout config |
| Single `DELETE` on small `.avro` (manifest) keys | manifest-merge (benign, not bulk) | HIGH | Run `rewrite_manifests`/`expire_snapshots` to keep manifest count <100 |
| Bulk deletes on a schedule, keys = committed data / old `metadata.json` | Cause 5 (maintenance jobs) | MED | Audit `expire_snapshots`/`remove_orphan_files`/`rewrite_*` schedules |
| `DELETE …?uploadId=` | multipart abort (symptom of Cause 2) | HIGH | Investigate large-write failures; check part-size + timeouts |

## Recommended diagnostics (do these in order)

1. **Classify the request + keys.** For a sample of the suspicious deletes capture: verb+path
   (`POST /b?delete` bulk vs `DELETE /b/key` single vs `?uploadId=`), and the deleted key pattern
   (`{queryId}-…`/task staging vs committed `.parquet` vs `.avro` manifest vs `metadata.json`). This
   alone selects a matrix row.
2. **Correlate timing.** Do the bulk deletes line up with (a) running write jobs, (b) job/task
   failures + retries/speculation in the engine event logs, or (c) a maintenance schedule?
3. **Test the attempt-cleanup hypothesis.** Temporarily disable Spark `spark.speculation` and set
   Trino `retry-policy=NONE` on a test workload; if bulk deletes vanish, Causes 1+4 are confirmed.
4. **Audit the storage path (Cause 2).** Check MinIO server logs for 5xx/timeouts, HAProxy
   `timeout server/client` vs observed PUT/multipart-complete durations, OpenResty error log and any
   body-buffering / 100-continue handling, and confirm MinIO behind HAProxy is one distributed
   cluster (not independent pools that break read-after-write visibility).
5. **Check retry duplication (Cause 3).** Compare logical deletes (from engine logs) to MinIO
   request counts; large divergence implicates SDK/proxy retries.

## References

- [Trino 467 / Iceberg 1.7.0 INSERT S3-delete report](20260608-trino-iceberg-insert-s3-delete-conditions.report.md)
- [Spark 3.5.1 / Iceberg 1.11.0 INSERT+MERGE S3-delete report](20260608-spark-iceberg-insert-merge-s3-delete-conditions.report.md)
- Trino layer analyses: [filesystem](../resources/trino/iceberg-insert-s3-delete-filesystem-layer.analysis.md), [connector](../resources/trino/iceberg-insert-s3-delete-connector-layer.analysis.md), [core](../resources/trino/iceberg-insert-s3-delete-core-commit-layer.analysis.md)
- Spark/Iceberg layer analyses: [S3FileIO bulk mechanics](../resources/iceberg/s3fileio-bulk-delete-mechanics.analysis.md), [Spark write/abort](../resources/iceberg/spark-iceberg-s3-delete-write-path.analysis.md), [core 1.11.0](../resources/iceberg/core-commit-cleanup-1.11.0.analysis.md)

---
title: When Trino 467 Iceberg connector issues S3 DELETE requests during INSERT INTO
date: 2026-06-08
status: complete
components: [trino, iceberg, s3-minio]
constraints:
  - trino-version: "467"
  - catalog: hive-metastore
  - storage: minio (s3 protocol), via openresty -> haproxy -> minio cluster
  - sql: INSERT INTO
clarification-defaults:
  scope-layer: full write path (connector + Iceberg core + S3 filesystem)
  deployment-context: existing deployment, diagnosing observed behavior
  workload-profile: batch INSERT INTO
---

## Overview

This report traces, through the actual Trino **467** source (checked out at
`references/trino` tag `467`) plus its pinned **Apache Iceberg 1.7.0** dependency, the exact
conditions under which an `INSERT INTO <iceberg_table>` causes the client (Trino) to issue S3
DELETE requests — single-object `DeleteObject` and bulk `DeleteObjects` — to MinIO. The setup
under investigation: Iceberg connector + Hive metastore catalog + MinIO behind OpenResty →
HAProxy.

The analysis was produced by three parallel source-code analysts, one per layer of the write
stack (S3 filesystem, Iceberg connector, Iceberg core commit). Each checkpoint is preserved under
`resources/trino/*.analysis.md`.

**Verdict: A clean, single-writer, unsorted `INSERT INTO` with `retry-policy=NONE` (the default)
into a default-configured Iceberg table issues *no* object-delete requests to MinIO. The
unexpected S3 DELETEs you are seeing are almost certainly one (or more) of four well-defined
conditions: (1) fault-tolerant execution / `retry-policy=TASK` turning on `cleanExtraOutputFiles`,
(2) sorted writing deleting temp spill files, (3) Iceberg `MergeAppend` manifest-merge cleanup once
a partition-spec group crosses 100 manifests, or (4) abort/rollback on a failed or retried task.
A fifth, non-`DeleteObject` source is `AbortMultipartUpload` (`DELETE …?uploadId=`) from large
writes that fail or hit the exclusive-create precondition — this is a DELETE *verb* on MinIO but is
not an object delete.** See the Decision Matrix to map your observed request shape to its cause.

## Evidence Quality

| Source | File | Tier | Accessible |
|---|---|---|---|
| S3 filesystem layer analysis (`lib/trino-filesystem-s3`) | [link](../resources/trino/iceberg-insert-s3-delete-filesystem-layer.analysis.md) | official | Yes |
| Iceberg connector layer analysis (`plugin/trino-iceberg`) | [link](../resources/trino/iceberg-insert-s3-delete-connector-layer.analysis.md) | official | Yes |
| Iceberg core commit layer analysis (Iceberg 1.7.0) | [link](../resources/trino/iceberg-insert-s3-delete-core-commit-layer.analysis.md) | official | No (jar not in checkout) |

**Gaps and confidence limits:**
- The **Iceberg core 1.7.0 internals** (manifest-merge cleanup, commit-retry cleanup, metadata
  delete-after-commit) are stated from knowledge of the pinned 1.7.0 release — the `iceberg-core`
  jar is not in the local source-only checkout. The *version pin* (`pom.xml:201`) and the *append
  type selection* (`IcebergMetadata.java:1282`, `MergeAppend` by default) ARE file-backed. These
  core-internal claims are therefore **MEDIUM confidence** pending byte-code confirmation against
  the actual 1.7.0 jar; everything in the Trino connector and S3 filesystem layers is **HIGH
  confidence** (direct source citation).
- This report covers the **write/commit path of `INSERT INTO` only**. It does not cover MERGE,
  UPDATE, DELETE, OPTIMIZE, `expire_snapshots`, or `remove_orphan_files` (those have their own,
  larger delete footprints and are explicitly out of scope).
- The OpenResty/HAProxy front layer is not a delete source; it only forwards. It can, however,
  *duplicate* or retry requests at the proxy level — see Finding 6.

## Components

### `trino` (Iceberg connector, v467)

The connector orchestrates the insert: `beginInsert` → page sink writes data files → `finishInsert`
builds an Iceberg `AppendFiles` and commits. By default `merge_manifests_on_write=true`, so it uses
`transaction.newAppend()` (Iceberg `MergeAppend`), not `newFastAppend()`
([connector analysis](../resources/trino/iceberg-insert-s3-delete-connector-layer.analysis.md),
`IcebergMetadata.java:1282`). The connector only deletes under specific conditions (Findings 1–3).

### `iceberg` (core library, v1.7.0, dependency)

After `update.commit()`, Iceberg core writes manifests/manifest-list/metadata.json and may delete
manifest files it wrote, governed entirely by Iceberg 1.7.0 defaults since Trino sets no relevant
table properties
([core analysis](../resources/trino/iceberg-insert-s3-delete-core-commit-layer.analysis.md)).

### `s3-minio` (`lib/trino-filesystem-s3`)

Translates `TrinoFileSystem` calls into AWS SDK v2 calls. `deleteFile` → `DeleteObjectRequest`
(`DELETE /bucket/key`); `deleteFiles`/`deleteDirectory` → `DeleteObjectsRequest`
(`POST /bucket?delete`), **chunked at 250 keys per request, grouped per bucket**
([fs analysis](../resources/trino/iceberg-insert-s3-delete-filesystem-layer.analysis.md),
`S3FileSystem.java:148,190,203`).

## Findings

### 1. The default happy path deletes nothing

Evidence: [connector analysis](../resources/trino/iceberg-insert-s3-delete-connector-layer.analysis.md)
(`IcebergMetadata.java:1191-1201`, `IcebergPageSink.java:416`, `ParquetFileWriter.java:150-156`);
[core analysis](../resources/trino/iceberg-insert-s3-delete-core-commit-layer.analysis.md).
**Confidence: HIGH** (connector), **MEDIUM** (core-internal "zero deletes" claim).

With `retry-policy=NONE` (default — no fault-tolerant execution), unsorted writing, a single
writer, and a default-configured table, an `INSERT INTO`:
- writes data files via the page sink — the per-writer rollback closure
  (`() -> fileSystem.deleteFile(outputPath)`) is **stored but never invoked on success**
  (`IcebergPageSink.java:416`);
- commits via Iceberg `MergeAppend` — which, below 100 manifests, writes a new manifest and
  metadata.json and **deletes nothing**;
- does not clean up, expire, or sweep anything synchronously.

So if you observe DELETEs during such an insert, one of Findings 2–6 is in play.

### 2. `cleanExtraOutputFiles` fires when `retry-policy != NONE` — list + bulk delete every insert

Evidence: [connector analysis](../resources/trino/iceberg-insert-s3-delete-connector-layer.analysis.md)
(`IcebergMetadata.java:1303-1305, 1346-1405`, `:1364` LIST, `:1386/1393` `deleteFiles`).
**Confidence: HIGH.**

`finishInsert` calls `cleanExtraOutputFiles` **only when `retryMode != NO_RETRIES`** — i.e. when
fault-tolerant execution / `retry-policy=TASK` is enabled. When it fires it:
1. **lists the output directory** (`fileSystem.listFiles`) — an S3 `ListObjectsV2` on **every
   insert**, even when nothing needs deleting;
2. selects files named `{queryId}-…` that are NOT in the committed keep-set (orphans from
   speculative / retried task attempts);
3. **bulk-deletes** them via `fileSystem.deleteFiles` (S3 `DeleteObjects`).

This is the single most likely connector-side cause of unexpected DELETE/LIST traffic. If your
cluster has fault-tolerant execution on, expect a LIST per insert and a bulk `DeleteObjects`
whenever any task attempt was retried/speculatively executed.

### 3. Iceberg `MergeAppend` deletes superseded manifests once a spec group hits 100 manifests

Evidence: [core analysis](../resources/trino/iceberg-insert-s3-delete-core-commit-layer.analysis.md)
(`IcebergMetadata.java:1282` MergeAppend default; Iceberg 1.7.0 `commit.manifest.min-count-to-merge=100`).
**Confidence: MEDIUM** (append-type selection HIGH/file-backed; merge-threshold behavior from 1.7.0 knowledge).

Because `merge_manifests_on_write=true` by default, Trino uses `MergeAppend`. While a partition-spec
manifest group stays below 100 manifests, merge does not fire and **no delete occurs**. Once the
group reaches **≥ 100 manifests** (common after many small inserts), each subsequent commit merges
small manifests and **deletes the newly-written, now-subsumed manifest object(s)** it just created
(small `*.avro` files), via FileIO → S3 `DeleteObject`/`DeleteObjects`. It does **not** delete the
old committed manifests or any data files. This produces a recurring, low-volume DELETE pattern on
manifest objects (not data files) during INSERT.

Mitigation: set `merge_manifests_on_write=false` (forces FastAppend, zero clean-commit deletes), or
run `expire_snapshots`/`OPTIMIZE` to keep manifest counts down.

### 4. Abort / rollback on failed or retried tasks deletes the data files that attempt wrote

Evidence: [connector analysis](../resources/trino/iceberg-insert-s3-delete-connector-layer.analysis.md)
(`IcebergPageSink.java:237-261, 416`; `IcebergFileWriterFactory.java:169/211/290`).
**Confidence: HIGH.**

The engine calls `IcebergPageSink.abort()` when a write task fails, the query fails, or a
speculative/retried task attempt is cancelled. `abort()` runs every writer's rollback closure →
one `fileSystem.deleteFile` (S3 `DeleteObject`) **per data file that sink instance wrote**. So a
flaky insert (task failures, speculative execution) generates bursts of single-object DELETEs on
data-file keys. Note the connector does **not** delete written files on *commit* failure
(`IcebergMetadata.java:1818-1827` just rethrows) — those become orphans.

### 5. Sorted writing deletes temp spill files on every sorted insert

Evidence: [connector analysis](../resources/trino/iceberg-insert-s3-delete-connector-layer.analysis.md)
(`SortingFileWriter.java:251-252, 276-282, 181-185`; temp dir `…/trino-tmp-files`,
`IcebergPageSink.java:174`). **Confidence: HIGH.**

If the table has a sort order and `sorted_writing_enabled` is on, the sink writes temp spill files
under `…/trino-tmp-files` and, during the merge phase of `commit()`, **deletes each temp file** via
`fileSystem.deleteFile` (S3 `DeleteObject`). These are expected, normal-operation DELETEs on every
sorted insert, independent of retry mode.

### 6. Request shape on MinIO, batching, and proxy effects

Evidence: [fs analysis](../resources/trino/iceberg-insert-s3-delete-filesystem-layer.analysis.md)
(`S3FileSystem.java:148,162,190,203`; `S3FileSystemConfig.java:118-119`;
`S3OutputStream.java:375-385`); `ForwardingFileIo.java:40`. **Confidence: HIGH.**

- **Single delete**: `deleteFile` → `DeleteObjectRequest` → HTTP `DELETE /bucket/key`.
- **Bulk delete**: `deleteFiles`/`deleteDirectory` → `DeleteObjectsRequest` → HTTP
  `POST /bucket?delete` with an XML key list, `quiet(true)`. **Two batch sizes stack**: Iceberg's
  `ForwardingFileIo` batches at **1000** keys per `TrinoFileSystem.deleteFiles` call
  (`ForwardingFileIo.java:40`), and `S3FileSystem` then **re-chunks at 250** keys per actual S3
  POST (`S3FileSystem.java:190`), grouped per bucket. So a 1000-file logical delete = 4 POST
  `?delete` requests of 250 keys each.
- **`deleteDirectory` = LIST then bulk delete** (includes zero-byte `key/` placeholders).
- **SDK retries**: `s3.retry-mode=LEGACY`, `s3.max-error-retries=10` by default — a 5xx/throttle/
  timeout re-sends the same DELETE/POST. Behind **OpenResty → HAProxy**, a slow or dropped upstream
  response can also cause the proxy or the SDK to retry, so MinIO may log **more DELETE requests than
  logical deletes**. If you see duplicate `DeleteObjects` POSTs, suspect retry at the SDK or proxy
  layer, not extra application-level deletes.
- **`AbortMultipartUpload` is a DELETE verb but not an object delete**: large writes (≥ 16 MB
  default part size) that fail, or hit the `exclusive-create` `412` precondition, issue
  `DELETE /bucket/key?uploadId=…` (`S3OutputStream.java:375-385`). On MinIO access logs this looks
  like a DELETE but is an upload abort, not a `DeleteObject`. Distinguish by the `?uploadId=` query
  param.

## Decision Matrix

| Observed on MinIO during INSERT | Most likely cause | Where | Confidence | Fix / confirm |
|---|---|---|---|---|
| `POST /bucket?delete` (bulk) + a `ListObjectsV2` per insert | `cleanExtraOutputFiles` under fault-tolerant exec | Finding 2 | HIGH | Check `retry-policy` / FTE; set to `NONE` if not needed |
| Recurring small `DELETE`/`POST ?delete` on **manifest** (`*.avro`) keys | `MergeAppend` manifest-merge ≥100 manifests | Finding 3 | MEDIUM | `merge_manifests_on_write=false` or run `OPTIMIZE`/`expire_snapshots` |
| Bursts of single `DELETE` on **data-file** keys | Task/query abort rollback | Finding 4 | HIGH | Investigate task failures / speculative execution |
| Single `DELETE` on `…/trino-tmp-files/*` keys every insert | Sorted writing temp cleanup | Finding 5 | HIGH | Expected; disable `sorted_writing_enabled` to stop |
| `DELETE …?uploadId=…` | `AbortMultipartUpload` on failed/precondition write | Finding 6 | HIGH | Not an object delete; check write failures / `s3.exclusive-create` |
| Duplicate identical DELETE/POST `?delete` | SDK or OpenResty/HAProxy retry | Finding 6 | HIGH | Tune `s3.max-error-retries`, check proxy timeouts/retries |
| Nothing above, clean single-writer insert | No deletes expected | Finding 1 | HIGH (connector) | Re-check assumptions; capture exact request + key |

## Recommended diagnostic steps

1. Capture the **full request line** from MinIO (verb, path, query string) for the suspicious
   deletes — specifically whether it is `DELETE /b/key`, `POST /b?delete`, or `DELETE /b/key?uploadId=`.
   This alone narrows it to one matrix row.
2. Inspect the **deleted keys**: data files vs manifest `*.avro` vs `trino-tmp-files/*` vs
   `?uploadId=` — each points to a different finding.
3. Confirm cluster settings: `retry-policy` / fault-tolerant execution (Finding 2), table
   `merge_manifests_on_write` and current manifest count (Finding 3), `sorted_writing_enabled` and
   table sort order (Finding 5).
4. Check whether duplicates correlate with HAProxy/OpenResty upstream timeouts (Finding 6).

## References

- [S3 filesystem layer analysis](../resources/trino/iceberg-insert-s3-delete-filesystem-layer.analysis.md)
- [Iceberg connector layer analysis](../resources/trino/iceberg-insert-s3-delete-connector-layer.analysis.md)
- [Iceberg core commit layer analysis](../resources/trino/iceberg-insert-s3-delete-core-commit-layer.analysis.md)
- Source: `references/trino` @ tag `467` (Apache Iceberg `1.7.0`, `pom.xml:201`)

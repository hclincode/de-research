---
source: references/trino @ tag 467 — plugin/trino-iceberg/
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

Source-code checkpoint (layer 2 of 3) for the unexpected-S3-DELETE investigation. Covers the
**Iceberg connector layer** (`plugin/trino-iceberg`): every filesystem delete the Trino connector
code itself triggers during `INSERT INTO`. FileIO routing: `ForwardingFileIo` (constructed in
`catalog/hms/HiveMetastoreTableOperationsProvider.java:52`) maps Iceberg `deleteFile` →
`TrinoFileSystem.deleteFile` (one S3 DeleteObject) and bulk `deleteFiles` → batches of
`DELETE_BATCH_SIZE = 1000` → `TrinoFileSystem.deleteFiles` (S3 DeleteObjects)
(`fileio/ForwardingFileIo.java:40,76-128`).

## Key Points

- **Happy path issues NO connector data-file delete** with `retry-policy=NONE` (default, no
  fault-tolerant execution). `beginInsert` (`IcebergMetadata.java:1191-1201`) only loads table +
  begins txn. Page-sink writers build a rollback closure `() -> fileSystem.deleteFile(outputPath)`
  (`IcebergFileWriterFactory.java:169` Parquet, `:211` ORC, `:290` Avro) but on success the closure
  is **stored, not invoked** (`IcebergPageSink.java:416`). `commit()` runs the rollback only if
  `close()` throws (`ParquetFileWriter.java:150-156`).
- **`cleanExtraOutputFiles` — the prime connector-side suspect.** Defined
  `IcebergMetadata.java:1346-1405`. Invoked from `finishInsert` **only when
  `retryMode != NO_RETRIES`** (`:1303-1305`). It: (1) lists the output directory via
  `fileSystem.listFiles(location)` (`:1364`, an S3 LIST **every insert when enabled**), (2) selects
  files whose name `startsWith(queryId + "-")` and are NOT in the committed keep-set (`:1368`),
  (3) **bulk-deletes** them via `fileSystem.deleteFiles(...)` in batches of 1000 (`:1386`, `:1393`)
  → S3 `DeleteObjects`. With no failed task attempts the delete set is empty (early return :1373),
  but the LIST still happens. `retryMode` comes from the `RetryMode` arg to `beginInsert` and is
  non-`NO_RETRIES` when `retry-policy=TASK` / fault-tolerant execution is on.
- **Page-sink `abort()`** (`IcebergPageSink.java:237-261`) runs every writer's rollback closure
  (open writers via `writer::rollback`, closed writers via `closedWriterRollbackActions`) → one
  `fileSystem.deleteFile` (S3 DeleteObject) per data file this sink wrote. Triggered by the engine
  on query failure, task failure, or a cancelled speculative/retried task attempt.
- **Sorted writing** (`sorted_writing_enabled` + table sort order): wraps the writer in
  `IcebergSortingFileWriter`, which on `commit()` merges and **deletes each temp spill file** via
  `SortingFileWriter.java:251-252` (`fileSystem.deleteFile`), plus cleanup/rollback at `:271-282`,
  `:181-185`. Temp files under `…/trino-tmp-files` (`IcebergPageSink.java:174`). These are
  **expected S3 DeleteObject calls on every sorted insert**, independent of retry mode.
- **Commit failure does NOT delete written data files** at the connector layer.
  `commitUpdateAndTransaction` (`IcebergMetadata.java:1818-1827`) calls `commit()` /
  `commitTransaction()` and on `CommitFailedException` / `CommitStateUnknownException` / IO error
  just wraps + rethrows `ICEBERG_COMMIT_ERROR` — no file deletion. Written data files become orphans
  (removable later via `remove_orphan_files`). Any retry-time deletes come from Iceberg core (layer 3).
- **Handoff to Iceberg core**: `finishInsert` builds `transaction.newAppend()` /
  `newFastAppend()` (`:1282`), `appendFile(...)` per task (`:1298`), then `commitUpdateAndTransaction`
  (`:1307`) → `IcebergUtil.commit` (`IcebergUtil.java:1057-1062`) → `update.commit()`. From there file
  I/O uses the table's `ForwardingFileIo` instance.
- **No synchronous orphan / snapshot sweep on insert.** `removeOrphanFiles` (deletes at
  `IcebergMetadata.java:2081/2091`) and `executeExpireSnapshots` (`:1902/1917`) are `ALTER TABLE
  EXECUTE` procedures, never called by the insert path. Statistics (Puffin) write after commit does
  no data-file delete; the stats file is deleted only if its own write fails
  (`TableStatisticsWriter.java:184`).

### Delete-trigger table (connector layer)

| Trigger | Code | FS call | When |
|---|---|---|---|
| `cleanExtraOutputFiles` (LIST + bulk delete this query's non-committed files) | `IcebergMetadata.java:1303-1305,1364,1386/1393` | `listFiles` + `deleteFiles` | finishInsert, only if `retryMode != NO_RETRIES` |
| Sorted-write temp cleanup | `SortingFileWriter.java:251-252,276-282` | `deleteFile` per temp | every sorted insert |
| Page-sink `abort()` rollback | `IcebergPageSink.java:237-261,416` | `deleteFile` per written file | query/task failure, cancelled speculative attempt |
| Writer self-rollback on close error | `ParquetFileWriter.java:150-156` | `deleteFile` | only if `close()` throws |

## Security Notes

No issues detected. Read-only analysis of upstream Apache-licensed source.
Checks performed:
- Malicious or obfuscated code: none.
- Suspicious URLs or redirects: none.
- Content quality / AI-generated: high; derived from cited source lines.

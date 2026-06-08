---
source: references/iceberg @ apache-iceberg-1.11.0 — spark/v3.5 source writers
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

Checkpoint (layer 2 of 3): every S3-delete trigger in the **Spark 3.5.1 Iceberg write path**
(`spark/v3.5/spark/src/main/java/org/apache/iceberg/spark/source/`) for INSERT/append and
MERGE/UPDATE/DELETE (both copy-on-write and merge-on-read). Covers happy-path commit and
failure/abort/retry. The headline: **bulk `DeleteObjects` during a write comes from Spark abort
cleanup (task or job), not from a clean commit.**

## Key Points

### Append / INSERT (`BatchAppend`)
- `commit()` builds `table.newAppend()` → Iceberg **MergeAppend** (`SparkWrite.java:297`;
  `core/.../BaseTable.java:191`). On a clean commit it deletes **nothing** unless the active
  partition-spec manifest group has ≥100 manifests (then core merge-cleanup deletes superseded
  *new* manifests via single `deleteFile` — see layer 3). **No data-file delete on clean append.**

### Copy-on-write MERGE/UPDATE/DELETE (`CopyOnWriteOperation` / `OverwriteByFilter`)
- `commit()` calls `overwriteFiles.deleteFiles(overwrittenFiles, danglingDVs)` (`SparkWrite.java:439`)
  and `table.newOverwrite()` (`:434`). **`OverwriteFiles.deleteFiles(...)` is a metadata operation**
  — it removes the old data files from the new snapshot; it does **NOT** physically delete them from
  S3 at commit. The old data files persist until `expire_snapshots`. So a clean COW commit issues
  **no physical S3 data delete** (only possible manifest-merge single deletes).

### Merge-on-read MERGE/UPDATE/DELETE (`SparkPositionDeltaWrite`)
- `commit()` builds `table.newRowDelta()` (`SparkPositionDeltaWrite.java:211`) and adds delete files
  + data files — **metadata only**, no physical delete at commit.

### Task-level abort — the prime bulk-delete suspect
- Data writer `abort()` → `SparkCleanupUtil.deleteTaskFiles(io, result.dataFiles())`:
  - unpartitioned: `SparkWrite.java:775-779`
  - partitioned: `SparkWrite.java:832-836`
  - MOR delete/data writers: `SparkPositionDeltaWrite.java:629-633, 706-710`
- `deleteTaskFiles` → `SparkCleanupUtil.deleteFiles` → **bulk delete via S3FileIO** (layer 1).
- **When Spark calls `DataWriter.abort()`**: task failure (exception in the write task), task
  **interruption/kill from speculative execution** (the losing duplicate attempt), stage retry,
  or executor loss. Each aborted attempt **bulk-deletes the data/delete files that attempt already
  wrote**. This is the most common cause of bulk `DeleteObjects` during jobs that ultimately
  *succeed* — speculative execution and task retries are normal Spark behavior.

### Job-level abort — conditional bulk delete of all written files
- `BatchWrite.abort(messages)` → `SparkWrite.abort` → `SparkCleanupUtil.deleteFiles("job abort",
  table.io(), files(messages))` (`SparkWrite.java:246-252`), MOR equivalent at
  `SparkPositionDeltaWrite.java:298-300`. **Bulk-deletes ALL data files written by the job.**
- **Gated by `cleanupOnAbort`**, set only when the commit threw a `CleanableFailure`
  (`SparkWrite.java:237, 241`; `SparkPositionDeltaWrite.java:343, 347`). On a
  `CommitStateUnknownException` (commit may have landed) `cleanupOnAbort` stays false and cleanup is
  **skipped** — log "Skipping cleanup of written files" (`SparkWrite.java:250`).

## Delete-trigger table (Spark write layer)

| Trigger | Code | FileIO call | S3 op (with S3FileIO) | When |
|---|---|---|---|---|
| Task abort | `SparkWrite.java:775-779, 832-836`; `SparkPositionDeltaWrite.java:629-633, 706-710` | `deleteTaskFiles` → bulk | **DeleteObjects** (this task's files) | task failure, speculative-exec kill, stage retry, executor loss |
| Job abort (cleanable commit failure) | `SparkWrite.java:246-252`; `SparkPositionDeltaWrite.java:298-300` | `deleteFiles` → bulk | **DeleteObjects** (all job files) | commit throws `CleanableFailure` |
| Clean append commit | `SparkWrite.java:297` | MergeAppend | none (≤100 manifests) / single manifest deletes (≥100) | always / merge threshold |
| Clean COW commit | `SparkWrite.java:439` | `OverwriteFiles.deleteFiles` (metadata) | **none physical** | MERGE/UPDATE/DELETE COW |
| Clean MOR commit | `SparkPositionDeltaWrite.java:211` | `RowDelta` (metadata) | **none physical** | MERGE/UPDATE/DELETE MOR |

## Security Notes

No issues detected. Read-only analysis of Apache-licensed Iceberg Spark source.
Checks performed:
- Malicious or obfuscated code: none.
- Suspicious URLs or redirects: none.
- Content quality / AI-generated: high; direct source citations.

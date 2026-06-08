---
source: references/iceberg @ apache-iceberg-1.11.0 — core/ commit + table properties
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

Checkpoint (layer 3 of 3): what the Iceberg **1.11.0 core library** itself deletes during a commit,
governed by Iceberg defaults (Spark sets no relevant table properties beyond the write itself).
Unlike the earlier Trino/1.7.0 analysis, this is read against the actual 1.11.0 source in the
checkout, so these are file-backed (HIGH confidence).

## Key Points

- **Append type = MergeAppend by default** (`core/.../BaseTable.java:191` `newAppend()` →
  `new MergeAppend(...)`; `newFastAppend()` → `FastAppend` at `:196`). Manifest-merge defaults:
  `commit.manifest-merge.enabled=true` (`TableProperties.java:121-122`),
  `commit.manifest.min-count-to-merge=100` (`:118-119`). Merge fires only when a partition-spec
  group reaches ≥100 manifests; then it rewrites small manifests and the **newly-written,
  now-subsumed** manifest object(s) are deleted. Old committed manifests and all data files are NOT
  deleted (left for `expire_snapshots`).
- **All core commit deletes are SINGLE `deleteFile`, never bulk.** `SnapshotProducer.deleteFile(path)`
  → `ops.io().deleteFile(path)` (`SnapshotProducer.java:586`, also `:101`); manifest-list and
  manifest cleanup at `:530, 580, 727`. So manifest-merge cleanup and commit-retry cleanup appear on
  MinIO as individual `DeleteObject` (DELETE) requests — never `DeleteObjects` (bulk). (Bulk delete
  is exclusively the Spark/`CatalogUtil` data-file path — see layer 1.)
- **Commit-retry cleanup** (`SnapshotProducer.commit()`): on terminal failure
  `Exceptions.suppressAndThrow(e, this::cleanAll)` deletes all manifests this producer wrote
  (`:508`, `cleanAll` `:578-583`); on success `cleanUncommitted(...)` removes manifests not in the
  committed snapshot (`:524`). On **`CommitStateUnknownException` → NO cleanup** (`:504-508`) — files
  intentionally left behind because the commit may have actually landed.
- **Old `vNNN.metadata.json` cleanup OFF by default**:
  `write.metadata.delete-after-commit.enabled=false` (`TableProperties.java:330-332`);
  `write.metadata.previous-versions-max=100` (`:325-327`). Spark does not enable it, so no old
  metadata.json is deleted after commit.
- **Snapshot expiration / orphan removal are NOT auto-triggered by INSERT/MERGE.** They are separate
  Spark procedures/actions (`expire_snapshots`, `remove_orphan_files`, `rewrite_data_files`,
  `rewrite_manifests`). Those DO physically delete data/delete/manifest/metadata files — via
  `CatalogUtil.deleteFiles`/`SupportsBulkOperations` → **bulk `DeleteObjects`** with S3FileIO — but
  only when the user explicitly runs them.

### Per-scenario core deletes (default table, S3FileIO)

| Scenario | Core-library deletes |
|---|---|
| First append into empty table | ZERO |
| Repeated appends, <100 manifests in spec group | ZERO per commit |
| Repeated appends, ≥100 manifests (merge fires) | single `DeleteObject` on superseded *new* manifest(s); data + old manifests untouched |
| COW/MOR MERGE/UPDATE/DELETE commit | ZERO physical data delete (metadata-only); possible manifest-merge single deletes |
| Commit retry (concurrent writers) | single `DeleteObject` of uncommitted manifests; none on `CommitStateUnknownException` |
| `expire_snapshots` / `remove_orphan_files` (explicit) | **bulk `DeleteObjects`** of data/manifest/metadata — out of scope of plain writes |

## Security Notes

No issues detected. Read-only analysis of Apache-licensed Iceberg 1.11.0 core source.
Checks performed:
- Malicious or obfuscated code: none.
- Suspicious URLs or redirects: none.
- Content quality / AI-generated: high; file-backed citations against the 1.11.0 checkout.

---
source: references/trino @ tag 467 — Apache Iceberg 1.7.0 core (dependency)
component: trino
type: source-analysis
evidence-tier: official
accessible: false
benchmark-age: n/a
date-retrieved: 2026-06-08
security:
  malicious-code: none
  suspicious-urls: none
  quality: high
---

## Summary

Source-code checkpoint (layer 3 of 3) for the unexpected-S3-DELETE investigation. Covers the
**Apache Iceberg core library** commit semantics during `INSERT INTO`. The Iceberg-core jar is not
present in the local checkout (`accessible: false` — source-only repo, no
`iceberg-core-1.7.0.jar` in `~/.m2`), so core-internal behavior below is stated from knowledge of
Apache Iceberg **1.7.0** and clearly marked; the Trino-side version pin and append-type selection
are file-backed.

## Key Points

- **Iceberg version = 1.7.0** (FILE EVIDENCE): `pom.xml:201`
  `<dep.iceberg.version>1.7.0</dep.iceberg.version>`, applied to `iceberg-api/core/parquet/...`
  via `${dep.iceberg.version}` (`plugin/trino-iceberg/pom.xml:223,229`).
- **Trino INSERT uses `MergeAppend` by default** (FILE EVIDENCE): `IcebergMetadata.java:1282`
  `isMergeManifestsOnWrite(session) ? transaction.newAppend() : transaction.newFastAppend()`, and
  `merge_manifests_on_write` default = **true** (`IcebergSessionProperties.java:358-362`). So
  `newAppend()` → Iceberg `MergeAppend` (a `MergingSnapshotProducer`). FastAppend only if the flag
  is turned off.
- **Trino sets none of the manifest-merge / metadata-cleanup / commit-retry table properties** at
  create time, so **Iceberg 1.7.0 defaults govern all delete behavior** (verified: grep of
  `IcebergTableProperties` / `IcebergConfig` / `IcebergUtil.buildTableProperties`).
- **Manifest merge defaults (KNOWLEDGE, Iceberg 1.7.0)**: `commit.manifest-merge.enabled=true`,
  `commit.manifest.min-count-to-merge=100`, `commit.manifest.target-size-bytes=8 MB`. Merge fires
  only when a partition-spec group accumulates **≥ 100 manifests**. When it fires, the deletes
  issued via `FileIO.deleteFile` are only the **newly written, now-subsumed** manifest(s) this
  commit created — NOT the pre-existing committed manifests (those stay, referenced by old
  snapshots; removed only by `expire_snapshots`).
- **Plain append deletes nothing extra by default**: FastAppend never merges/deletes (except
  cleanup of its own uncommitted manifests on a retry conflict); MergeAppend deletes only on
  retry conflict or once the ≥100-manifest threshold is crossed.
- **Commit retry cleanup (KNOWLEDGE)**: `SnapshotProducer.commit()` retries on
  `CommitFailedException` (`commit.retry.num-retries=4` default). Terminal success →
  `cleanUncommitted` deletes the producer's manifests not in the committed snapshot; terminal
  failure → `cleanAll` deletes all manifests this producer wrote; **`CommitStateUnknownException`
  → NO cleanup** (commit may have landed). Per-attempt `metadata.json` cleanup is catalog-dependent
  (Hive/Glue/REST), not done by SnapshotProducer. Uncontended single-writer INSERT → no retries →
  no retry-driven deletes.
- **Old `vNNN.metadata.json` cleanup is OFF by default**:
  `write.metadata.delete-after-commit.enabled=false` (Iceberg 1.7.0 default), and Trino does not
  enable it. So no old metadata.json deleted after commit. `write.metadata.previous-versions-max=100`
  only matters if delete-after-commit is enabled.
- **Snapshot expiration / orphan removal are NOT triggered by INSERT** (FILE EVIDENCE):
  dispatched only via `executeTableExecute` for `EXPIRE_SNAPSHOTS` (`IcebergMetadata.java:1847-1848`)
  and `REMOVE_ORPHAN_FILES` (`:1850-1851`). Plain INSERT never calls them.
- **FileIO → S3 routing**: `ForwardingFileIo implements SupportsBulkOperations`
  (`fileio/ForwardingFileIo.java:38`); `deleteFile` → `TrinoFileSystem.deleteFile`,
  bulk `deleteFiles` batched at `DELETE_BATCH_SIZE=1000` (`:40,99-128`) → `TrinoFileSystem.deleteFiles`
  → S3 `DeleteObject` / `DeleteObjects` against MinIO.

### Per-scenario delete summary (default table)

- **First insert (empty table)**: data files, 1 manifest, manifest-list, metadata.json all written
  and committed; merge not triggered (1 « 100); no retries; metadata cleanup off →
  **ZERO Iceberg-library deletes.**
- **Repeated inserts, < 100 manifests in active spec group**: new manifest committed, no merge →
  **ZERO deletes per insert.**
- **Repeated inserts, once a spec group reaches ≥ 100 manifests**: MergeAppend rewrites small
  manifests into larger ones and **deletes the newly-written, now-subsumed manifest object(s)** via
  `FileIO.deleteFile` (a handful of small `*.avro` manifests). Old committed manifests and data files
  are NOT deleted. **This is the most likely Iceberg-core cause of DELETEs during INSERT given
  Trino's `merge_manifests_on_write=true` default.**
- **Concurrent writers (CommitFailedException retry)**: losing attempts' manifests deleted via
  `cleanUncommitted`/`cleanAll`; `CommitStateUnknownException` → none.

### Mitigations (core layer)
Set `merge_manifests_on_write=false` (forces FastAppend → no clean-commit deletes), and/or run
`expire_snapshots` / `OPTIMIZE` to keep manifest counts below the 100 merge threshold.

## Security Notes

No issues detected. Analysis of upstream Apache-licensed Iceberg behavior; core internals derived
from knowledge of the pinned 1.7.0 release (jar not in local checkout — `accessible: false`).
Checks performed:
- Malicious or obfuscated code: none.
- Suspicious URLs or redirects: none.
- Content quality / AI-generated: high; version pin + append selection file-backed, core internals
  knowledge-based and marked as such.

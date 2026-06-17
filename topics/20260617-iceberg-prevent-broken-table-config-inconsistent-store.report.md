---
title: Configuring Trino, Spark, and Hive Metastore to prevent broken Iceberg tables when the S3 path is not strongly consistent
date: 2026-06-17
status: complete
components: [iceberg, trino, spark, hive-metastore, s3-minio, openresty, haproxy]
constraints:
  - storage: minio behind openresty + haproxy — strong read-after-write consistency CANNOT be guaranteed
  - engines: trino 467 (iceberg 1.7.0) AND spark 3.5.1 (iceberg 1.11.0)
  - goal: configure the stack so an unreliable store cannot leave the Iceberg table broken (dangling references)
extends:
  - 20260608-iceberg-table-corruption-from-s3-delete-root-cause.report.md
---

## Overview

The prior report showed the table broke because a commit that *actually succeeded* was misclassified
as a `CleanableFailure`, so auto-cleanup deleted the manifests/data the committed snapshot references.
That misclassification is only possible because the OpenResty→HAProxy→MinIO path is not a single
strongly-consistent, idempotent S3 endpoint. This report answers: **if that consistency cannot be
guaranteed, can configuration of Trino, Spark, and Hive Metastore prevent a broken table — and is it
possible at all?**

**Verdict (honest, two-part):**
1. **You cannot get full ACID correctness on an inconsistent store.** Iceberg's commit protocol
   *requires* read-after-write consistency. No configuration removes that requirement; with stale
   reads, lost writes, or split-brain across independent MinIO pools you can still get lost commits,
   duplicate data, or queries that 404 on objects that exist.
2. **But you CAN prevent the catastrophic "broken table" (dangling-reference) outcome** by configuring
   every failure to be *non-destructive*: leave orphan files (recoverable) instead of deleting live
   files (unrecoverable). The single highest-value lever — **disabling engine-side object deletes**
   (`s3.delete-enabled=false` for Spark/Iceberg; `retry-policy=NONE` to disable Trino's
   `cleanExtraOutputFiles`) — makes deletion-caused corruption structurally impossible, because the
   engine literally cannot delete. The cost is orphan accumulation, swept offline during a healthy
   window. Trino is already largely hardened by design; Spark/Iceberg is the path that needs the most
   configuration.

So: **preventing the broken table is achievable by configuration; guaranteeing correctness is not.**
The strategy converts "silent data loss" into "storage bloat + periodic manual reconciliation."

## The governing principle: make failure non-destructive

Recall the safety model (all source-cited in the prior report): Iceberg deletes written files only on
a `CleanableFailure` (`CommitFailedException`/`ValidationException`); `CommitStateUnknownException`
suppresses deletion. `requireStrictCleanup()` already defaults to **true**
(`core/.../TableOperations.java:127`), so cleanup is already limited to `CleanableFailure`. The
residual hole is that a *succeeded* commit can be reported as `CommitFailedException` (which IS
cleanable). Two independent ways to close the hole by config:

- **(P1) Remove the ability to delete** → even a wrong FAILURE verdict deletes nothing.
- **(P2) Remove the trigger that produces a wrong FAILURE verdict** → the verdict resolves to UNKNOWN
  (no delete) instead.

Why config alone can't make the *verdict* always correct: the FAILURE-vs-UNKNOWN decision
(`checkCommitStatusStrict`, `core/.../BaseMetastoreOperations.java:99-151`) only **retries when the
status read throws**; when the read *succeeds but returns stale "not found"* it returns FAILURE
immediately (line 138-139) — no amount of `commit.status-check.*` retry tuning fixes a store that
serves stale-but-successful reads. Hence P1 (disable deletes) is the only guarantee that is robust
*regardless* of how the store misbehaves.

## Spark 3.5.1 + Iceberg 1.11.0 — the path that needs hardening

| # | Setting | Value | Effect | Confidence |
|---|---|---|---|---|
| S1 | `spark.sql.catalog.<cat>.s3.delete-enabled` | **`false`** | S3FileIO `deleteFile` AND `deleteFiles` both early-return without deleting (`aws/.../S3FileIO.java:177-178, 222`). Engine **cannot delete any object** → no deletion-caused corruption. Orphans accumulate. | HIGH |
| S2 | `spark.speculation` | `false` | Removes speculative-task aborts → fewer `deleteTaskFiles` calls (moot if S1 set, but reduces churn). | HIGH |
| S3 | `hive.metastore.failure.retries` (hive-site for the Spark catalog) | **`0`** | Stops the HMS client from retrying a successful `alter_table`, which is the *only* trigger for the strict status check that can return FAILURE (`hive-metastore/.../HiveTableOperations.java:387-394`). A lost ack then takes the non-strict path → UNKNOWN → no delete. | HIGH |
| S4 | `commit.status-check.num-retries` / `...total-timeout-ms` (table prop) | raise (e.g. 10 / longer) | Helps **only** when status reads *throw* (transient unavailability); lets an eventually-consistent store converge before concluding. Does **not** help stale-but-successful reads. | MEDIUM |
| S5 | `commit.retry.num-retries` (table prop) | keep modest (e.g. 4, default) | More retries = more windows where an attempt succeeds-but-looks-failed. Don't inflate. | MEDIUM |
| S6 | `spark.task.maxFailures` | low (e.g. 1–2) | Fewer task re-attempts → fewer abort/cleanup windows. | MEDIUM |

Primary guarantee = **S1** (no deletes possible). **S3** removes the specific false-FAILURE trigger.
S2/S5/S6 shrink the windows. Set S1 + S3 and the Spark path can no longer break the table by deletion.

> Trade-off of S1: legitimate cleanup of true orphans also stops, and `expire_snapshots` /
> `remove_orphan_files` / `rewrite_*` (which need deletes) become no-ops. Run those from a **separate
> catalog/config with `s3.delete-enabled=true`, only during a verified-healthy storage window** (see
> Operational practices).

## Hive Metastore — kill the false-conflict trigger, make the pointer authoritative

| # | Setting | Value | Effect | Confidence |
|---|---|---|---|---|
| H1 | `hive.metastore.failure.retries` | **`0`** | Same as S3 — the comment at `HiveTableOperations.java:387-390` names "HMS client incorrectly retries a successful operation" as the trigger. Disabling it routes lost acks to the safe UNKNOWN path. | HIGH |
| H2 | `hive.metastore.client.socket.timeout` | generous (e.g. 600s) | Prevent a *successful* `alter_table` from timing out client-side mid-commit (which manufactures ambiguity). | MEDIUM |
| H3 | Backing RDBMS = transactional (MySQL/Postgres), not embedded Derby | — | The `metadata_location` swap is then atomic and authoritative in the DB, independent of S3 read consistency. Iceberg errors if `HIVE_LOCKS` is missing (`HiveTableOperations.java:372-377`). | HIGH |
| H4 | `iceberg.engine.hive.lock-enabled` / table prop `engine.hive.lock-enabled`; `iceberg.hive.lock-heartbeat-interval-ms` | enable lock + sane heartbeat | Serializes concurrent commits via HMS lock (`HiveTableOperations.java:534-557, 356-359`) so two writers can't race on the swap. (On HMS versions with transactional `alter_table`, the DB CAS already serializes; keep heartbeat sane either way.) | MEDIUM |

H1 is the keystone: it makes the metastore the single source of truth for "did the commit land" and
stops the client from manufacturing the "table has been modified" error that opens the destructive
path.

## Trino 467 — already hardened; one switch to close the connector-side delete

Trino does **not** use Iceberg's `HiveTableOperations`; its own catalog commit throws
`CommitStateUnknownException` (not a `CleanableFailure`) on any `replaceTable` ambiguity
(`HiveMetastoreTableOperations.java:110-122`, with a comment that cleaning up otherwise leaves the
"table in unusable state"). It also does not delete data files on commit failure. So Trino's
catalog/commit layer is already fail-safe against the corruption.

| # | Setting | Value | Effect | Confidence |
|---|---|---|---|---|
| T1 | `retry-policy` (system/session) | **`NONE`** | Disables fault-tolerant execution → `cleanExtraOutputFiles` (the connector's list+bulk-delete) never runs (`IcebergMetadata.java:1303-1305`). This is Trino's main destructive surface during INSERT. | HIGH |
| T2 | metastore client config (`hive.metastore-timeout`, thrift retries) | generous / conservative | Reduce ambiguous commits; Trino already maps them to UNKNOWN, so exposure is low. | MEDIUM |
| T3 | (no equivalent of `s3.delete-enabled`) | — | Trino uses `TrinoFileSystem`, which has no delete-disable flag. Mitigation is structural (T1 + hardened catalog), not a kill-switch. Page-sink `abort()` still deletes *uncommitted* files on task failure — safe, because Trino never marks them committed. | HIGH |

Net: for Trino, **`retry-policy=NONE` plus the already-hardened catalog** closes the realistic
corruption path. Trino's remaining exposure on an inconsistent store is *read availability* (404s for
objects that exist), not deletion of live data.

## Defense-in-depth summary

```
Layer 1 (prevent deletion of live files):
  Spark : s3.delete-enabled=false        ← hard guarantee, robust to any inconsistency
  Trino : retry-policy=NONE              ← disables cleanExtraOutputFiles
Layer 2 (prevent the false-FAILURE verdict):
  HMS   : hive.metastore.failure.retries=0   ← removes the only trigger of the strict status path
  HMS   : transactional RDBMS + lock         ← authoritative, serialized pointer swap
Layer 3 (shrink the failure windows):
  Spark : spark.speculation=false, low task.maxFailures
  Both  : modest commit.retry.num-retries
Layer 4 (operational recoverability):
  retain metadata history; run physical cleanup offline in healthy windows; monitor orphans
```

## Residual risks after all hardening (what config still cannot guarantee)

- **Lost / duplicated commits**: a write that genuinely isn't durable, or one retried after a lost ack,
  can drop or duplicate data. Iceberg can't detect this without consistent reads.
- **Stale reads / lost updates**: a commit built on a stale `refresh()` can clobber a concurrent
  commit (snapshot lineage gap). HMS locking (H4) mitigates *concurrent* writers but not stale reads
  from the object store.
- **Read 404s**: queries may intermittently fail on objects that exist (HAProxy routing to a lagging
  backend). This is availability, not corruption — retried/transient.
- **Orphan accumulation**: the direct cost of `s3.delete-enabled=false`; requires periodic offline
  `remove_orphan_files`.

These are **recoverable / operational** problems, not silent deletion of live data. That is the
ceiling of what configuration buys you: corruption-by-deletion → eliminated; full correctness → still
requires a consistent store.

## Operational practices

- **Offline maintenance window**: run `expire_snapshots`, `remove_orphan_files`, `rewrite_data_files`
  from a dedicated job/catalog with `s3.delete-enabled=true`, scheduled only when the storage path is
  verified healthy (no flapping backends, consistent reads). Never let routine writes delete.
- **Rollback readiness**: keep ample metadata history (`write.metadata.previous-versions-max`, and do
  NOT enable `write.metadata.delete-after-commit.enabled`) so you can `rollback_to_snapshot` /
  re-`register_table` to the last good `metadata.json` if a bad commit slips through.
- **Monitoring**: alert on MinIO 5xx / elevated `DeleteObjects`, HMS `alter_table` retries/timeouts,
  and Iceberg `CommitFailedException`/`CommitStateUnknownException` log lines; periodically validate a
  table by walking current snapshot → manifests → data files and HEAD-checking each.
- **Best fix remains structural**: present MinIO as one strongly-consistent distributed cluster (not
  independent pools behind HAProxy), and ensure OpenResty/HAProxy never retry non-idempotent S3
  requests nor alter conditional headers/bodies. Config hardening is insurance, not a substitute.

## Is it possible? — direct answer

- **Prevent a broken (dangling-reference) Iceberg table on an unreliable store: YES**, via Spark
  `s3.delete-enabled=false` + Trino `retry-policy=NONE` + HMS `failure.retries=0` + transactional HMS.
  Failures become orphan accumulation, not data loss. HIGH confidence this eliminates the
  deletion-caused corruption from the prior report.
- **Guarantee full ACID correctness on an inconsistent store: NO.** Not configurable away; Iceberg
  requires consistency. Residual lost-commit/stale-read/availability risks remain until the storage
  path is made consistent.

## Evidence Quality

| Claim | Source | Tier | Confidence |
|---|---|---|---|
| `s3.delete-enabled=false` disables single + bulk S3FileIO deletes | `aws/.../S3FileIO.java:166-178, 202-222`; `S3FileIOProperties.java:414-416` | official | HIGH |
| `requireStrictCleanup()` defaults true (cleanup limited to CleanableFailure) | `core/.../TableOperations.java:127-129` | official | HIGH |
| Strict status check returns FAILURE on stale "not found"; retries only on read-throw | `core/.../BaseMetastoreOperations.java:99-151` | official | HIGH |
| "HMS client incorrectly retries a successful operation" is the strict-path trigger | `hive-metastore/.../HiveTableOperations.java:387-394` | official | HIGH |
| Trino maps replaceTable ambiguity to CommitStateUnknownException (hardened) | `HiveMetastoreTableOperations.java:110-122` | official | HIGH |
| Trino `cleanExtraOutputFiles` gated by `retry-policy != NONE` | `IcebergMetadata.java:1303-1305` | official | HIGH |
| HMS lock enable + heartbeat properties | `HiveTableOperations.java:356-359, 534-557` | official | HIGH |
| `hive.metastore.failure.retries=0` removes the retry trigger | Hive client behavior + the named code comment | official + inference | MEDIUM |
| Config prevents deletion-corruption but not full ACID on inconsistent store | architecture + Iceberg consistency requirement | analysis | MEDIUM–HIGH |

**Gaps:** the precise Hive client retry property name/scope depends on how each engine constructs its
metastore client (RetryingMetaStoreClient vs Iceberg `ClientPool`); validate `hive.metastore.failure.retries`
takes effect in your Spark hive-site and Trino metastore config. The "fully prevents broken table"
claim is bounded to *deletion-caused* corruption (HIGH); other corruption modes are addressed only by
fixing consistency.

## References
- Iceberg: `aws/.../S3FileIO.java:166-260`, `S3FileIOProperties.java:414-416`;
  `core/.../TableOperations.java:127`; `core/.../BaseMetastoreOperations.java:63-151`;
  `core/.../SnapshotProducer.java:506-583`; `hive-metastore/.../HiveTableOperations.java:241-427, 534-557`
  (`references/iceberg` @ apache-iceberg-1.11.0)
- Trino: `IcebergMetadata.java:1303-1405`; `catalog/hms/HiveMetastoreTableOperations.java:65-127`
  (`references/trino` @ 467)
- Extends: [table corruption root cause](20260608-iceberg-table-corruption-from-s3-delete-root-cause.report.md),
  [cross-engine bulk-delete](20260608-iceberg-s3-bulk-delete-cross-engine-root-causes.report.md),
  [INSERT+commit workflow](20260608-iceberg-insert-commit-workflow.report.md)

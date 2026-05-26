---
title: HDFS as Lakehouse Storage — On-Premise Enterprise 2026 (Legacy Reference)
date: 2026-05-21
status: complete
components: [hdfs, spark, trino]
constraints:
  - open-source: true
  - deployment: on-premise
---

## Overview

This report evaluates Hadoop Distributed File System (HDFS) as the physical storage layer for a Spark + Trino open-source, on-premise lakehouse in 2026. HDFS is the most widely deployed on-premise Hadoop storage, but the lakehouse era has exposed two structural limitations that define its future: (1) the NameNode is a memory-bound metadata bottleneck that becomes acute at lakehouse file counts (billions of Iceberg manifests, snapshots, and positional delete files), and (2) Trino's strategic direction is away from HDFS, with the Hadoop FS compatibility shim (`fs.hadoop.enabled=true`) preserved as a legacy path but not as a forward-looking integration.

**Verdict: HDFS should not be chosen for new Spark + Trino lakehouse deployments in 2026. For existing HDFS deployments, the path forward is Apache Ozone (the recommended open-source successor with S3 API and distributed metadata) or Ceph RGW. HDFS remains operationally viable for existing deployments that have not yet exhausted their NameNode ceiling, but the migration clock is running: Trino is actively removing Hadoop library dependencies, and the `fs.hadoop.enabled` HDFS shim has no committed long-term support guarantee.**

---

## Evidence Quality

| Source | File | Tier | Accessible |
|---|---|---|---|
| Apache Hadoop 3.5.0 Release | [link](../resources/hdfs/hadoop-3-5-0-release.summary.md) | official | Yes |
| Trino "Out with the old file system" blog (Feb 2025) | [link](../resources/hdfs/trino-out-with-old-file-system.summary.md) | official | Yes |
| Trino 481 HDFS File System Docs | [link](../resources/hdfs/trino-481-hdfs-docs.summary.md) | official | Yes |
| Trino Legacy FS Removal Issue #24878 | [link](../resources/hdfs/trino-legacy-fs-removal-issue-24878.summary.md) | official | Yes |
| Apache Ozone vs HDFS Official Comparison | [link](../resources/hdfs/ozone-vs-hdfs-official-comparison.summary.md) | official | Yes |
| HDFS Federation Official Docs | [link](../resources/hdfs/hdfs-federation-official-docs.summary.md) | official | Yes |
| Preferred Networks: A Year with Apache Ozone | [link](../resources/hdfs/preferred-networks-year-with-ozone.summary.md) | press | Yes |
| HDFS vs S3 Performance (USEReady) | [link](../resources/hdfs/hdfs-spark-data-locality-performance.summary.md) | press | Yes |
| MinIO: Small Files Problem in HDFS | [link](../resources/hdfs/minio-small-files-hdfs-namenode.summary.md) | vendor-adjacent | Yes |
| Cloudera HDFS→Ozone Migration Guide | [link](../resources/hdfs/cloudera-hdfs-ozone-migration-guide.summary.md) | vendor-adjacent | Yes |
| NameNode Heap Memory Sizing (Cloudera Docs) | [link](../resources/hdfs/namenode-memory-scalability-cloudera-docs.summary.md) | vendor-adjacent | Yes |
| Apache Hadoop EOL Dates | [link](../resources/hdfs/hadoop-endoflife-date-versions.summary.md) | press | Yes |

**Gaps and confidence limits:**

1. **Stale benchmark**: The only HDFS vs. S3-compatible-store performance benchmark found during research dates to approximately 2023. No 2024–2026 benchmark comparing HDFS against on-premise Ozone or Ceph for Spark workloads was found. All performance advantage claims for HDFS are rated MEDIUM confidence pending fresher data.

2. **Trino HDFS removal timeline not committed**: The Trino project has not published a specific version or date for removing the `fs.hadoop.enabled` HDFS shim. Issue #24878 tracks legacy FS removal but applies primarily to cloud storage (S3/Azure/GCS), not HDFS specifically. The risk is real but unquantified.

3. **Vendor-adjacent NameNode ceiling sources**: The quantitative NameNode memory formula (150 bytes/object) is cited by both Cloudera docs and MinIO blog. The formula is corroborated by official Hadoop documentation and academic papers, elevating the quantitative claims to HIGH confidence, but the lakehouse-specific implications (how many Iceberg manifests a typical deployment generates) are not independently benchmarked.

4. **Cloudera commercial context**: Cloudera CDP end-of-support dates are vendor-sourced. Independently confirmed: HDP reached EOL; CDH 6.x reached EOL; CDP 7.x EOS ranges from Q1 to June 2026 for several versions. Organizations on Cloudera commercial support must act immediately.

5. **No independent 2025–2026 benchmark comparing HDFS and Ozone on Spark workloads** was found. The Preferred Networks article describes qualitative improvements in NameNode scalability but does not publish throughput numbers.

---

## Components

### `hdfs`

HDFS is the storage layer of the Apache Hadoop distributed computing platform, first released in 2006. It organizes data into large blocks (default 128 MB) distributed across DataNodes. A central NameNode holds all filesystem metadata in RAM, which is the root cause of the small-file and scalability problems. HDFS provides strong consistency, high sequential read throughput for co-located Spark workloads (via data locality), and deep integration with the Hadoop ecosystem (YARN, MapReduce, Spark, Hive). It is licensed under Apache 2.0 and is actively developed as of May 2026. HDFS does not provide an S3-compatible API.

**Key constraint:** HDFS NameNode holds all metadata in Java heap memory. At ~150 bytes per object (inode, block replica), a 128 GB NameNode serves approximately 500M metadata objects at theoretical minimum — less in practice due to JVM overhead and GC pressure. [NameNode Sizing](../resources/hdfs/namenode-memory-scalability-cloudera-docs.summary.md)

### `spark`

Apache Spark is the primary compute engine. Spark benefits from HDFS data locality when compute and storage are co-located on the same cluster nodes — the NameNode informs YARN where each block resides, and Spark tasks are preferentially scheduled on the DataNode containing the data. This eliminates network I/O for reads and can provide significant throughput advantages over remote object storage for sequential scan workloads. [HDFS Spark Performance](../resources/hdfs/hdfs-spark-data-locality-performance.summary.md)

### `trino`

Trino accesses HDFS via `fs.hadoop.enabled=true` in catalog configuration (Hive, Iceberg, Delta Lake, Hudi connectors). This is an explicitly supported but non-preferred path as of Trino 481 (May 2026). The Trino project's strategic direction is to remove all Hadoop library dependencies (issue #15921); the cloud storage FS shims (S3, Azure, GCS via Hadoop libraries) have been removed in Trino 481. The HDFS-specific shim is retained, but no long-term support commitment has been published. [Trino HDFS Docs](../resources/hdfs/trino-481-hdfs-docs.summary.md), [Trino Deprecation Blog](../resources/hdfs/trino-out-with-old-file-system.summary.md)

---

## Findings

### 1. HDFS is actively developed in 2026, but the Hadoop ecosystem is consolidating

Evidence: [Hadoop 3.5.0 Release](../resources/hdfs/hadoop-3-5-0-release.summary.md), [Hadoop EOL Dates](../resources/hdfs/hadoop-endoflife-date-versions.summary.md). Confidence: **HIGH**.

Hadoop 3.5.0 shipped April 2, 2026, with 485 improvements over 3.4. Hadoop 3.4.3 shipped February 24, 2026 as a maintenance release. Development of 3.6.0-SNAPSHOT is underway. HDFS development continues with meaningful improvements: finer-grained NameNode locking, Router-based Federation improvements, and Java 17/21 support. Apache Hadoop has no formal support policy (EOL dates are set ad-hoc), which creates planning uncertainty. Java 17 is now required server-side, which may affect organizations on older JVM stacks.

The broader ecosystem context is consolidation: Cloudera's commercial offerings (HDP, CDH, CDP) are reaching end of support in 2026, and Hortonworks as a standalone entity ceased to exist with the 2018 Cloudera merger. Apache Hadoop remains community-maintained without a dominant commercial backer.

### 2. Trino's HDFS shim is preserved but is not the forward path

Evidence: [Trino 481 HDFS Docs](../resources/hdfs/trino-481-hdfs-docs.summary.md), [Trino Deprecation Blog](../resources/hdfs/trino-out-with-old-file-system.summary.md), [Trino Issue #24878](../resources/hdfs/trino-legacy-fs-removal-issue-24878.summary.md). Confidence: **HIGH**.

The precise status as of Trino 481 (May 2026) requires careful reading:

- `fs.hadoop.enabled=true` for actual HDFS access: **still supported, not deprecated**.
- `fs.hadoop.enabled=true` previously also used to route S3/Azure/GCS traffic through Hadoop FS shims: **removed in Trino 481**.
- All `hive.s3.*`, `hive.azure.*`, `hive.cos.*`, `hive.gcs.*` properties: **deprecated in Trino 470, removed by Trino 481**.

The Trino blog post (Feb 2025) states: "if you are truly using HDFS, or if you insist on using the old legacy support you can also use `fs.hadoop.enabled=true`" — the phrasing ("insist") signals the project's attitude. The HDFS shim is preserved as an accommodation, not as a supported forward path. The complete removal of Hadoop FS library dependencies (issue #15921) is ongoing; when those libraries are eventually removed, HDFS access will break unless re-implemented natively. No committed timeline exists for this breakage.

**Risk profile:** Organizations deploying HDFS + Trino today are building on a compatibility shim with no published support horizon. Starburst Enterprise (commercial Trino distribution) may maintain HDFS support longer than the open-source project.

### 3. The NameNode is a hard scalability ceiling for lakehouse workloads

Evidence: [NameNode Sizing](../resources/hdfs/namenode-memory-scalability-cloudera-docs.summary.md), [MinIO Small Files](../resources/hdfs/minio-small-files-hdfs-namenode.summary.md), [Official Ozone Comparison](../resources/hdfs/ozone-vs-hdfs-official-comparison.summary.md). Confidence: **HIGH**.

The NameNode memory formula is well-established: ~150 bytes per metadata object (file inode, directory entry, or block replica). For a replication factor of 3, each file consumes one inode plus three block records = ~600 bytes. A NameNode with 128 GB heap can hold approximately 200M files at RF=3 under ideal conditions; JVM GC overhead and OS memory reservation push practical limits to 100–150M files.

**Lakehouse-specific amplification:** Apache Iceberg generates the following per operation:
- One manifest file per data writer per commit
- One manifest list file per snapshot
- Positional delete files from Merge-on-Read operations
- Snapshot expiration does not immediately reduce NameNode load (files linger until garbage collection)

A moderately active 1,000-table Iceberg lakehouse with hourly compaction can generate tens of millions of small metadata files per year. This is the core incompatibility between HDFS and the lakehouse pattern. File-level compaction (Iceberg `expireSnapshots`, `rewriteDataFiles`) mitigates but does not eliminate the growth.

NameNode OutOfMemoryError is a hard stop: the filesystem freezes or becomes read-only. There is no graceful degradation.

### 4. HDFS Federation and Router-based Federation partially mitigate but do not solve the NameNode bottleneck

Evidence: [HDFS Federation Docs](../resources/hdfs/hdfs-federation-official-docs.summary.md), [Hadoop 3.5.0 Release](../resources/hdfs/hadoop-3-5-0-release.summary.md). Confidence: **HIGH**.

HDFS Federation splits the namespace across multiple independent NameNodes, each managing a portion of the filesystem. ViewFS provides a client-side unified mount table. Router-based Federation (RBF) in Hadoop 3.x adds a routing layer so clients interact with a single endpoint. Hadoop 3.5.0 improves RBF with asynchronous RPC processing and MySQL delegation token storage.

**Why Federation does not fully solve the lakehouse problem:**

1. **Federation multiplies NameNode ceilings, does not eliminate them.** Each NameNode still has its own RAM-bound ceiling. The namespace must be manually partitioned across NameNodes — which data goes where must be decided upfront and is expensive to rebalance.
2. **Lakehouse metadata files are table-local.** Iceberg manifests for a given table reside in the table's storage prefix. If that prefix maps to a single NameNode namespace, all small-file pressure from that table accumulates on that NameNode.
3. **Operational complexity.** RBF requires a State Store (ZooKeeper or MySQL), router nodes, and adds failure modes. This is significantly more complex than managing a single NameNode.
4. **No S3 API.** Federation does not add S3 API compatibility. Trino's native file system implementations (`fs.native-s3.enabled`) and future storage integrations all target S3 APIs.

Federation is appropriate for very large existing HDFS deployments that need to scale beyond one NameNode's RAM. It is not a path that makes HDFS competitive with Ozone for new lakehouse deployments.

### 5. HDFS provides a genuine performance advantage for co-located Spark workloads

Evidence: [HDFS Spark Performance](../resources/hdfs/hdfs-spark-data-locality-performance.summary.md). Confidence: **MEDIUM** (benchmark data is approximately 2023; no 2024–2026 benchmark found).

When Spark and HDFS are co-located (compute and storage on the same physical nodes), HDFS data locality eliminates network I/O for read-heavy workloads. The NameNode tells YARN where each block lives; YARN schedules Spark tasks on DataNodes holding the data. This provides a real throughput advantage over remote S3 (cloud or on-premise) for sequential scan workloads on large files.

**Caveats:**
- The advantage diminishes when files are small (more mapper splits, more HDFS open calls, more NameNode pressure).
- On-premise S3-compatible stores (Ozone, Ceph, MinIO) can close the gap when placed on the same network fabric — they do not provide data locality but avoid cross-datacenter latency.
- S3 throttling (per-bucket request rate limits) does not apply to HDFS.
- No benchmark found comparing HDFS vs. Ozone on Spark for on-premise lakehouse workloads in 2024–2026. The 2023 benchmark data treats cloud S3 as the comparison point, which has higher latency than on-premise Ozone. Treat the performance advantage as MEDIUM confidence.

### 6. The recommended migration path for existing HDFS deployments is Apache Ozone

Evidence: [Preferred Networks Ozone](../resources/hdfs/preferred-networks-year-with-ozone.summary.md), [Cloudera Migration Guide](../resources/hdfs/cloudera-hdfs-ozone-migration-guide.summary.md), [Official Ozone Comparison](../resources/hdfs/ozone-vs-hdfs-official-comparison.summary.md). Confidence: **HIGH**.

Apache Ozone is the recommended open-source successor to HDFS within the Hadoop ecosystem. Key advantages for lakehouse workloads:

- **Distributed metadata:** OzoneManager + StorageContainerManager replace the single NameNode. Metadata stored in RocksDB (persistent, not RAM-bound). Designed for "exabyte scale, tens of billions of keys."
- **S3 API:** Ozone exposes an S3-compatible endpoint — Trino's native S3 file system (`fs.native-s3.enabled`) can target Ozone. This is the forward-compatible Trino integration path.
- **HDFS compatibility:** Ozone File System (OFS) is an HDFS-compatible API layer. Spark code written for HDFS works unchanged. HDFS clients can use Ozone via an HDFS-over-Ozone compatibility layer.
- **Production validation:** Preferred Networks migrated ~9 PB from HDFS to Ozone using DistCp. The migration is operationally feasible at petabyte scale. [Preferred Networks](../resources/hdfs/preferred-networks-year-with-ozone.summary.md)
- **Ozone 2.0.0 released August 2025:** GA release with JDK 17/21 support, ARM64 builds, improved snapshot operations, SCM decommissioning support. Represents production maturity milestone.

**Migration procedure (DistCp-based):**
1. Deploy Ozone cluster alongside HDFS cluster.
2. Use `hadoop distcp` to copy data. For same-KDC clusters, direct HDFS→Ozone DistCp works. For different KDC, use S3A protocol.
3. Checksum parameter: `-Ddfs.checksum.combine.mode=COMPOSITE_CRC` (required for HDFS 3.1.1+ sources).
4. Encryption zones: use `-skipcrccheck` if data is in encryption zones.
5. Bucket type: OBS buckets require `-direct` flag; FSO buckets support atomic renames (better for Iceberg).
6. Validate with `hdfs du` comparison on source and target.
7. Reconfigure Spark and Trino catalogs to use Ozone endpoint.

**Known migration blocker:** HDFS has no file size limit. S3 API has a 5 TB per-object maximum. Files exceeding 5 TB in HDFS cannot be migrated via the S3A protocol — requires OFS protocol or pre-splitting. [Preferred Networks](../resources/hdfs/preferred-networks-year-with-ozone.summary.md)

**Alternative migration tool:** IBM LiveData Migrator (formerly WanDisco) provides live, zero-downtime migration for organizations that cannot afford a freeze-and-copy window.

### 7. No scenario in 2026 makes HDFS the correct choice for a new Spark + Trino lakehouse deployment

Evidence: [Trino 481 HDFS Docs](../resources/hdfs/trino-481-hdfs-docs.summary.md), [Official Ozone Comparison](../resources/hdfs/ozone-vs-hdfs-official-comparison.summary.md), [HDFS Federation Docs](../resources/hdfs/hdfs-federation-official-docs.summary.md). Confidence: **HIGH**.

The convergence of three independent constraints eliminates HDFS for new deployments:

1. **Trino direction:** `fs.hadoop.enabled` for HDFS is a compatibility shim, not the forward Trino integration. Native S3 (`fs.native-s3.enabled`) is the supported path. New deployments starting on HDFS inherit the shim's uncertain support horizon.
2. **NameNode ceiling:** New lakehouse deployments start small but grow. Iceberg metadata accumulation is predictable and irreversible without compaction. A greenfield deployment on HDFS will hit NameNode pressure before it hits storage pressure.
3. **No S3 API:** The broader lakehouse ecosystem (Spark, Trino, Flink, DuckDB, Python tooling) assumes S3-compatible storage. HDFS is not S3-compatible. Every tool integration that assumes S3 requires a separate HDFS adapter.

**Counter-evidence searched ("HDFS not dead," "HDFS still viable 2025 2026"):** The most credible counter-argument found is that HDFS provides data locality throughput advantages for batch Spark workloads. This is valid for *existing* HDFS deployments with co-located compute but does not justify *new* deployments given the NameNode scalability wall and Trino's trajectory.

### 8. Commercial support landscape is in transition

Evidence: [Cloudera Migration Guide](../resources/hdfs/cloudera-hdfs-ozone-migration-guide.summary.md). Confidence: **MEDIUM** (Cloudera-sourced; independently confirmed HDP/CDH EOL dates).

- **Hortonworks HDP:** End of support reached; no longer available. Organizations on HDP must migrate.
- **Cloudera CDH 6.x:** End of support reached.
- **Cloudera CDP Private Cloud Base 7.x:** End of Support dates range from Q1 2026 to June 2026 for various 7.x versions. Organizations on commercial Cloudera support for HDFS must act immediately.
- **Apache HDFS (community):** Active development continues; no formal support policy. Community support via JIRA, mailing lists, Slack.
- **Starburst Enterprise:** Commercial Trino distribution; likely to maintain HDFS support longer than the OSS project. No published commitment found.
- **IBM LiveData Migrator (WanDisco):** Commercial migration tooling for HDFS → Ozone migration.

Confidence is MEDIUM because the Cloudera support date information is sourced from Cloudera documentation. The dates are consistent with publicly reported timelines but should be verified directly with Cloudera for specific license agreements.

---

## Decision Matrix

| Profile | Recommendation | Eliminated options and why |
|---|---|---|
| **Existing HDFS, healthy NameNode headroom, Spark-only (no Trino)** | Keep HDFS short-term; begin Ozone migration planning. Prioritize if nearing 100M files. | No immediate elimination; Ozone migration is the 12–24 month target. |
| **Existing HDFS + Trino, NameNode headroom OK** | Migrate Trino catalogs to Ozone (S3A endpoint) while keeping HDFS for Spark. Dual-path until full migration. | HDFS eliminated as Trino forward path; `fs.hadoop.enabled` shim has no committed horizon. |
| **Existing HDFS, NameNode near limits (>100M files, RAM pressure)** | Urgent migration to Ozone. Use DistCp for offline migration or IBM LiveData Migrator for live migration. | HDFS eliminated by NameNode ceiling. |
| **On Cloudera CDP, commercial support expiring 2026** | Migrate to open-source Ozone stack or evaluate Ceph RGW. No valid path to extend CDP HDFS support past published EOS dates. | CDP HDFS eliminated by support expiry. |
| **Greenfield deployment, Spark + Trino lakehouse** | Apache Ozone (S3 API + OFS, distributed metadata, Hadoop-native). HDFS not recommended. | HDFS eliminated: NameNode ceiling, no S3 API, Trino compatibility shim is not forward path. |
| **Greenfield, Spark-only, no Trino, batch workloads, team has strong Hadoop ops expertise** | Ozone still preferred. If team insists on HDFS: use with HDFS RBF Federation from day one; plan Ozone migration within 3 years. | HDFS is technically functional but suboptimal for future lakehouse expansion. |

---

## References

- [Apache Hadoop 3.5.0 Release](../resources/hdfs/hadoop-3-5-0-release.summary.md)
- [Trino "Out with the old file system" (Feb 2025)](../resources/hdfs/trino-out-with-old-file-system.summary.md)
- [Trino 481 HDFS File System Documentation](../resources/hdfs/trino-481-hdfs-docs.summary.md)
- [Trino Legacy FS Removal Issue #24878](../resources/hdfs/trino-legacy-fs-removal-issue-24878.summary.md)
- [Apache Ozone vs HDFS Official Comparison](../resources/hdfs/ozone-vs-hdfs-official-comparison.summary.md)
- [HDFS Federation Official Documentation](../resources/hdfs/hdfs-federation-official-docs.summary.md)
- [Preferred Networks: A Year with Apache Ozone](../resources/hdfs/preferred-networks-year-with-ozone.summary.md)
- [HDFS vs S3 Performance — USEReady](../resources/hdfs/hdfs-spark-data-locality-performance.summary.md)
- [MinIO: The Small Files Problem in HDFS](../resources/hdfs/minio-small-files-hdfs-namenode.summary.md)
- [Cloudera HDFS → Ozone Migration Guide](../resources/hdfs/cloudera-hdfs-ozone-migration-guide.summary.md)
- [NameNode Heap Memory Sizing — Cloudera Documentation](../resources/hdfs/namenode-memory-scalability-cloudera-docs.summary.md)
- [Apache Hadoop EOL Dates — endoflife.date](../resources/hdfs/hadoop-endoflife-date-versions.summary.md)

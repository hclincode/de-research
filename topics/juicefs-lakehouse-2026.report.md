---
title: JuiceFS as Lakehouse Storage — On-Premise Enterprise 2026
date: 2026-05-21
status: complete
components: [juicefs, spark, trino]
constraints:
  - open-source: true
  - deployment: on-premise
---

## Overview

JuiceFS is an Apache 2.0 distributed POSIX file system that separates file metadata (stored in a pluggable database: Redis, PostgreSQL, TiKV) from data (stored in any S3-compatible object store: MinIO, Ceph S3, etc.). It provides POSIX, HDFS-compatible (Hadoop Java SDK), and S3 gateway interfaces, allowing it to serve as the storage layer for both Spark (via `jfs://` URI) and Trino (via Hadoop SDK or S3 gateway). As of v1.3.1 (December 2025), JuiceFS Community Edition is production-stable and open-source under Apache 2.0, making it the only edition that satisfies the open-source constraint for this evaluation. The Enterprise Edition (proprietary, closed-source) is out of scope.

**Verdict: JuiceFS Community Edition is a viable but operationally demanding storage layer for an on-premise Spark + Trino lakehouse in 2026. It integrates natively with Spark's Hadoop FileSystem API and works with Trino via either the Hadoop SDK or S3 gateway. However, the metadata engine is a critical single point of architectural complexity: Redis offers the best performance but is memory-bounded and risks data loss on failover; PostgreSQL provides durability but adds 2–13x metadata latency overhead; TiKV offers scalability but can be 44x slower than Redis under concurrent small-file metadata loads. An enterprise team willing to operate and HA-configure a metadata engine alongside their object store can successfully run JuiceFS; a team seeking a self-contained storage system should evaluate MinIO or Ceph directly.**

---

## Evidence Quality

| Source | File | Tier | Accessible |
|---|---|---|---|
| JuiceFS GitHub Repository (v1.3.1) | [github-juicefs-repo.summary.md](../resources/juicefs/github-juicefs-repo.summary.md) | official | Yes |
| JuiceFS Internals Architecture Docs | [juicefs-architecture-internals-docs.summary.md](../resources/juicefs/juicefs-architecture-internals-docs.summary.md) | official | Yes |
| JuiceFS Metadata Engines Setup Docs | [juicefs-metadata-engines-setup-docs.summary.md](../resources/juicefs/juicefs-metadata-engines-setup-docs.summary.md) | official | Yes |
| JuiceFS Metadata Engines Benchmark Docs | [juicefs-metadata-engines-benchmark-docs.summary.md](../resources/juicefs/juicefs-metadata-engines-benchmark-docs.summary.md) | official | Yes |
| JuiceFS Hadoop Java SDK Docs | [juicefs-hadoop-java-sdk-docs.summary.md](../resources/juicefs/juicefs-hadoop-java-sdk-docs.summary.md) | official | Yes |
| JuiceFS S3 Gateway Docs | [juicefs-s3-gateway-docs.summary.md](../resources/juicefs/juicefs-s3-gateway-docs.summary.md) | official | Yes |
| JuiceFS Cache and Writeback Docs | [juicefs-writeback-cache-risks-docs.summary.md](../resources/juicefs/juicefs-writeback-cache-risks-docs.summary.md) | official | Yes |
| JuiceFS Production Deployment Recommendations | [juicefs-production-deployment-recommendations-docs.summary.md](../resources/juicefs/juicefs-production-deployment-recommendations-docs.summary.md) | official | Yes |
| Redis Pros/Cons for JuiceFS (JuiceData blog) | [juicefs-redis-pros-cons-vendor.summary.md](../resources/juicefs/juicefs-redis-pros-cons-vendor.summary.md) | vendor | Yes |
| Metadata Engine Selection Guide (JuiceData blog) | [juicefs-metadata-engine-selection-guide-vendor.summary.md](../resources/juicefs/juicefs-metadata-engine-selection-guide-vendor.summary.md) | vendor | Yes |
| JuiceFS vs CephFS Comparison (JuiceData docs) | [juicefs-vs-cephfs-docs.summary.md](../resources/juicefs/juicefs-vs-cephfs-docs.summary.md) | vendor | Yes |
| Enterprise vs Community Comparison (JuiceData blog) | [juicefs-enterprise-vs-community-vendor.summary.md](../resources/juicefs/juicefs-enterprise-vs-community-vendor.summary.md) | vendor | Yes |
| JuiceFS GC Mechanism (JuiceData blog) | [juicefs-garbage-collection-vendor.summary.md](../resources/juicefs/juicefs-garbage-collection-vendor.summary.md) | vendor | Yes |
| JuiceFS Performance Benchmark Docs (JuiceData) | [juicefs-performance-benchmark-docs.summary.md](../resources/juicefs/juicefs-performance-benchmark-docs.summary.md) | vendor | Yes |
| JuiceFS 2025 Recap (JuiceData blog) | [juicefs-2025-recap-vendor.summary.md](../resources/juicefs/juicefs-2025-recap-vendor.summary.md) | vendor | Yes |
| TiKV vs Redis GitHub Discussion (#884) | [juicefs-tikv-redis-github-discussion.summary.md](../resources/juicefs/juicefs-tikv-redis-github-discussion.summary.md) | press | Yes |
| Data Consistency Issues GitHub Discussion (#2700) | [juicefs-data-consistency-github-discussion.summary.md](../resources/juicefs/juicefs-data-consistency-github-discussion.summary.md) | press | Yes |
| Hacker News Architecture Concerns | [juicefs-hacker-news-architecture-concerns.summary.md](../resources/juicefs/juicefs-hacker-news-architecture-concerns.summary.md) | press | Yes |
| ZeroFS vs JuiceFS Benchmark (ZeroFS docs) | [juicefs-zerofs-benchmark-vendor-adjacent.summary.md](../resources/juicefs/juicefs-zerofs-benchmark-vendor-adjacent.summary.md) | vendor-adjacent | Yes |
| Trino Native S3 Filesystem Docs | [trino-native-s3-filesystem-docs.summary.md](../resources/juicefs/trino-native-s3-filesystem-docs.summary.md) | official | Yes |

**Gaps and confidence limits:**

1. **No independent performance benchmark** comparing JuiceFS to MinIO, Ceph, or Apache Ozone was found. All JuiceFS performance figures come from JuiceData (vendor). The only third-party comparison (ZeroFS benchmark) is vendor-adjacent, uses an undisclosed JuiceFS configuration, and compares a single-node tool to a distributed system — it is not usable for enterprise capacity planning. All performance findings are therefore LOW confidence.

2. **No independently verified Trino + JuiceFS S3 gateway integration report** was found. The expectation that Trino's native S3 client works with JuiceFS S3 gateway is based on architectural reasoning (both support path-style S3), not a documented production deployment. This is MEDIUM confidence.

3. **No community-produced Iceberg + JuiceFS lakehouse deployment report** was found. JuiceFS is primarily adopted for AI/ML training workloads in 2024–2025, not traditional analytics lakehouses. The applicability of JuiceFS to Iceberg-style metadata patterns (billions of small manifest files, atomic commits) is inferred from JuiceFS's general architecture, not from documented lakehouse deployments. This is MEDIUM confidence.

4. **JuiceFS 2025 adoption**: primarily Asia-Pacific AI/ML companies (Zhihu, StepFun, D-Robotics). No enterprise Western lakehouse deployment case study found using JuiceFS Community Edition with Spark + Trino + Iceberg.

5. **Enterprise Edition numbers** (500B files, 1ms latency, 6x efficiency) are inapplicable to the open-source Community Edition. Do not extrapolate them.

6. **ZeroFS benchmark**: treated as LOW confidence. Methodology undisclosed; competing vendor source; JuiceFS community disputed validity (GitHub Discussion #6488).

---

## Components

### `juicefs`

JuiceFS Community Edition (Apache 2.0, v1.3.1) is the storage layer under evaluation. It decouples file data from metadata: data blocks (4 MiB default) are stored as objects in any S3-compatible backend; all filesystem metadata (inodes, directory trees, chunk maps) is stored in an external metadata engine. This architecture means JuiceFS is not a standalone storage system — it is a filesystem layer on top of two separate systems. Both must be highly available and monitored independently. [Architecture docs](../resources/juicefs/juicefs-architecture-internals-docs.summary.md), [GitHub repo](../resources/juicefs/github-juicefs-repo.summary.md).

### `spark`

Spark integrates with JuiceFS via the Hadoop Java SDK, using the `jfs://` URI scheme and the `JuiceFileSystem` class that implements `org.apache.hadoop.fs.FileSystem`. This is the recommended interface for Spark workloads; it avoids the latency of the S3 gateway and enables data locality hints via `juicefs.discover-nodes-url`. [Hadoop Java SDK docs](../resources/juicefs/juicefs-hadoop-java-sdk-docs.summary.md).

### `trino`

Trino can access JuiceFS via two paths: (1) the legacy Hive connector using the Hadoop Java SDK (jfs:// scheme), or (2) the native S3 implementation (`fs.native-s3.enabled=true`) pointed at the JuiceFS S3 Gateway. The Hadoop SDK path is more established; the native S3 path is feasible but not explicitly tested by the Trino project against JuiceFS. [Trino S3 docs](../resources/juicefs/trino-native-s3-filesystem-docs.summary.md), [JuiceFS S3 gateway docs](../resources/juicefs/juicefs-s3-gateway-docs.summary.md).

---

## Findings

### 1. Metadata-storage separation architecture and how it works in practice

Evidence: [Architecture Docs](../resources/juicefs/juicefs-architecture-internals-docs.summary.md), [Metadata Engines Setup](../resources/juicefs/juicefs-metadata-engines-setup-docs.summary.md), [Hacker News Discussion](../resources/juicefs/juicefs-hacker-news-architecture-concerns.summary.md). Confidence: **HIGH**.

JuiceFS decomposes each file into chunks (max 64 MiB), slices, and blocks (max 4 MiB). Blocks are stored as individual objects in the object store (named by hash). The mapping from file path → chunks → slices → block IDs, plus all inode metadata (permissions, timestamps, sizes), is stored exclusively in the metadata engine.

The client is thick: the JuiceFS client handles all protocol translation, caching, data upload, and metadata lookup. There is no separate data server. The architecture means:
- The metadata engine is the critical control plane. If it becomes unavailable, all filesystem operations block.
- The object store is the data plane. It can be MinIO, Ceph S3, or any S3-compatible system.
- Scaling data capacity means scaling the object store. Scaling metadata capacity means scaling the metadata engine.

This is architecturally different from Ceph (self-contained, RADOS handles both) and from Ozone (single unified system). The community correctly flags that JuiceFS introduces two independent failure domains that must each be operated at production HA standards. [Hacker News](../resources/juicefs/juicefs-hacker-news-architecture-concerns.summary.md).

### 2. Recommended metadata engines for enterprise production

Evidence: [Metadata Engines Setup](../resources/juicefs/juicefs-metadata-engines-setup-docs.summary.md), [Redis Pros/Cons](../resources/juicefs/juicefs-redis-pros-cons-vendor.summary.md), [Metadata Engine Selection Guide](../resources/juicefs/juicefs-metadata-engine-selection-guide-vendor.summary.md), [TiKV vs Redis Discussion](../resources/juicefs/juicefs-tikv-redis-github-discussion.summary.md), [Production Recommendations](../resources/juicefs/juicefs-production-deployment-recommendations-docs.summary.md). Confidence: **MEDIUM** (selection guidance from vendor; TiKV performance data from community, press tier).

Three engines are viable for enterprise on-premise production:

**Redis with Sentinel (recommended for <100M files, performance-first)**
- Highest metadata throughput; sub-millisecond per-operation latency
- Memory-bounded: 100M inodes ≈ 32 GiB RAM; max recommended 64 GiB instance
- Asynchronous replication: acknowledged writes can be lost during Sentinel failover
- Redis Cluster: usable since v1.0.0 Beta3 but limited to one hash slot per volume; not recommended as a scaling strategy
- Deploy pattern: primary + replica + 3 Sentinel nodes; AOF persistence enabled

**PostgreSQL (recommended for <50M files, durability-first)**
- Strong durability (synchronous writes); ACID-compliant
- 2-13x slower than Redis on metadata-heavy workloads (small files, many concurrent ops)
- Simpler to operate than TiKV for teams already running PostgreSQL
- Horizontal scaling via read replicas for read-heavy metadata patterns

**TiKV (recommended only for >1 billion files)**
- Distributed, Raft-based: strong durability without Redis's async-replication data loss risk
- Minimum 6 nodes (3 PD + 3 TiKV) for production HA; significant operational overhead
- Performance trap: in a real-world concurrent small-file workload, JuiceFS+TiKV took 448 seconds vs <10 seconds for JuiceFS+Redis [GitHub Discussion #884](../resources/juicefs/juicefs-tikv-redis-github-discussion.summary.md). The 2-3x ratio cited in official benchmarks appears to assume sequential, not concurrent, metadata operations, and uses a 1-replica local TiKV vs a 3-node Raft cluster.
- For lakehouse workloads with large files (4 MiB+ blocks): metadata engine choice becomes less significant — object storage throughput dominates.

### 3. Spark integration: recommended interface

Evidence: [Hadoop Java SDK Docs](../resources/juicefs/juicefs-hadoop-java-sdk-docs.summary.md), [GitHub repo](../resources/juicefs/github-juicefs-repo.summary.md). Confidence: **HIGH**.

The recommended interface for Spark on JuiceFS is the **Hadoop Java SDK** (not POSIX mount, not S3 gateway). The SDK implements `org.apache.hadoop.fs.FileSystem`, making JuiceFS a transparent drop-in for HDFS in Spark. Configuration:

```xml
<!-- core-site.xml -->
<property>
  <name>fs.jfs.impl</name>
  <value>com.juicefs.JuiceFileSystem</value>
</property>
<property>
  <name>fs.AbstractFileSystem.jfs.impl</name>
  <value>com.juicefs.JuiceFS</value>
</property>
<property>
  <name>juicefs.meta</name>
  <value>redis://sentinel-host:26379/1?sentinelMasterId=mymaster</value>
</property>
```

Or as Spark submit arguments:
```
--conf spark.hadoop.fs.jfs.impl=com.juicefs.JuiceFileSystem
--conf spark.hadoop.fs.AbstractFileSystem.jfs.impl=com.juicefs.JuiceFS
```

Data is accessed via `jfs://<volume-name>/path`. For data locality, set `juicefs.discover-nodes-url` so JuiceFS can compute `BlockLocation` from YARN/Spark node lists. The SDK is compatible with Hadoop 2.x and 3.x. Note: the Hadoop SDK JAR must be added to Spark's classpath.

### 4. Trino integration: Hadoop SDK vs native S3 gateway

Evidence: [S3 Gateway Docs](../resources/juicefs/juicefs-s3-gateway-docs.summary.md), [Trino Native S3 Docs](../resources/juicefs/trino-native-s3-filesystem-docs.summary.md), [Hadoop Java SDK Docs](../resources/juicefs/juicefs-hadoop-java-sdk-docs.summary.md). Confidence: **MEDIUM** (no third-party confirmed production report found).

Trino has two integration paths with JuiceFS:

**Path A — Hadoop Java SDK (jfs:// scheme) — recommended**
- Trino's Hive/Iceberg/Hudi/Delta connectors all use the Hadoop FileSystem API internally.
- Adding the JuiceFS Hadoop SDK JAR to Trino's classpath and configuring `core-site.xml` identically to Spark enables `jfs://` access.
- This avoids the S3 gateway entirely; no additional process to manage; lower latency.
- This is the more established path; Trino's Hive connector explicitly supports custom FileSystem implementations.

**Path B — JuiceFS S3 Gateway with Trino native S3 (`fs.native-s3.enabled=true`)**
- JuiceFS S3 Gateway defaults to path-style access, matching Trino's `s3.path-style-access=true` requirement.
- Trino catalog config: `fs.native-s3.enabled=true`, `s3.endpoint=http://<gateway-host>:<port>`, `s3.path-style-access=true`.
- The JuiceFS S3 Gateway is developed from a fork of MinIO Gateway code (MinIO deprecated gateway mode in 2022; JuiceData maintains the fork).
- Trino officially tests S3 compatibility only with AWS S3 and MinIO — JuiceFS gateway not listed as tested.
- Additional process: the gateway must be deployed and HA-configured separately.
- Use this path only if the team cannot add JAR dependencies to Trino (e.g., managed Trino deployment).

**Note on `fs.native-s3.enabled`**: the Trino docs consistently use `fs.native-s3.enabled=true`; an earlier version of the property was `fs.s3.enabled=true`. Use the version matching your Trino release.

### 5. JuiceFS and Iceberg-style workloads: billions of small files and atomic commits

Evidence: [Architecture Docs](../resources/juicefs/juicefs-architecture-internals-docs.summary.md), [Cache/Writeback Docs](../resources/juicefs/juicefs-writeback-cache-risks-docs.summary.md), [GC Mechanism](../resources/juicefs/juicefs-garbage-collection-vendor.summary.md), [Metadata Engine Benchmark](../resources/juicefs/juicefs-metadata-engines-benchmark-docs.summary.md). Confidence: **MEDIUM** (inferred from architecture; no Iceberg-specific deployment data found).

Iceberg introduces three storage patterns that stress JuiceFS's metadata engine:

1. **Billions of small metadata files (manifests, manifest lists)**: Iceberg v2 tables with many snapshots accumulate large numbers of small JSON files. JuiceFS stores these as regular file metadata entries in the metadata engine. Each file = one inode in the metadata engine. With Redis, 1 billion small files = ~300 GiB RAM, well beyond a single Redis instance (64 GiB max recommended). PostgreSQL handles larger counts but with higher metadata latency. For typical enterprise lakehouse scales (tens of millions of data files, hundreds of thousands of metadata files), the metadata engine load is manageable with Redis or PostgreSQL.

2. **High-frequency manifest reads**: Iceberg readers (Spark, Trino) open and read metadata files at query planning time. JuiceFS's close-to-open consistency guarantees that once a writer closes a manifest file, readers will see it on next open — this is compatible with Iceberg's snapshot isolation model. JuiceFS client-side read cache further reduces metadata file read latency for hot manifests. The `--open-cache` flag can improve read performance further for read-heavy metadata patterns, at the cost of cross-client metadata freshness.

3. **Atomic metadata commits**: Iceberg atomicity is implemented at the catalog level (Hive Metastore, Nessie, REST catalog), not at the filesystem level. JuiceFS provides filesystem-level atomicity (atomic directory renames, atomic file creation) via the metadata engine. This is sufficient for Iceberg's write pattern: Spark writes data files (no atomicity needed at FS level), then the catalog does an atomic swap of the metadata pointer. JuiceFS does not interfere with this pattern.

Key gap: no Iceberg + JuiceFS production deployment was found in available sources. JuiceFS's primary validated use case in 2025 is AI/ML training (large sequential reads of model checkpoints and training datasets), not analytics lakehouse workloads with Iceberg. The compatibility is architecturally sound but untested at scale in public documentation.

### 6. POSIX semantics: overhead vs benefit in a lakehouse context

Evidence: [Architecture Docs](../resources/juicefs/juicefs-architecture-internals-docs.summary.md), [Cache Docs](../resources/juicefs/juicefs-writeback-cache-risks-docs.summary.md), [GitHub repo](../resources/juicefs/github-juicefs-repo.summary.md). Confidence: **MEDIUM** (vendor-sourced; no independent overhead measurement found).

JuiceFS implements full POSIX semantics (passed 8,813 pjdfstest tests). In a lakehouse context, POSIX has mixed value:

**Where POSIX helps:**
- Compatible with any POSIX-aware data tool without reconfiguration
- Atomic directory renames are available (not guaranteed in raw S3)
- POSIX locks supported (useful for single-writer-multiple-reader coordination)
- Applications that assume filesystem semantics (legacy ETL, POSIX-assuming tools) work without modification

**Where POSIX adds overhead:**
- Each file access requires a metadata engine round-trip (stat, open, read metadata)
- For large sequential reads (Parquet files, ORC files), the overhead is negligible — object storage throughput dominates
- For small file metadata operations (Iceberg manifest reads, listing), the metadata engine round-trip adds latency per operation; at Redis speeds (<1ms), this is tolerable; at PostgreSQL or TiKV speeds, it compounds
- POSIX close-to-open consistency adds a write-flush requirement: data must be committed before it is visible, which is actually beneficial for Iceberg's append-then-commit pattern

**Net assessment for lakehouse**: POSIX semantics impose meaningful overhead only in metadata-intensive scenarios (small file listing, frequent manifest opens) where the metadata engine latency dominates. For bulk data I/O (reading/writing Parquet files), POSIX overhead is negligible. The close-to-open guarantee aligns well with Iceberg's commit model.

### 7. Metadata engine bottlenecks at scale

Evidence: [Metadata Engine Benchmark](../resources/juicefs/juicefs-metadata-engines-benchmark-docs.summary.md), [TiKV vs Redis Discussion](../resources/juicefs/juicefs-tikv-redis-github-discussion.summary.md), [Redis Pros/Cons](../resources/juicefs/juicefs-redis-pros-cons-vendor.summary.md), [Selection Guide](../resources/juicefs/juicefs-metadata-engine-selection-guide-vendor.summary.md). Confidence: **MEDIUM** (benchmark from vendor; TiKV slowdown from community, press tier).

**Redis memory bound**: 100 million inodes ≈ 32 GiB RAM. Enterprise lakehouses with tens of billions of Iceberg files will exhaust a single Redis instance. Redis Cluster is not a viable horizontal scaling path for JuiceFS (one hash slot per volume). Mitigation: use multiple separate JuiceFS volumes (each with its own Redis), but this fragments the namespace.

**TiKV Raft latency**: Raft consensus requires quorum agreement for every metadata write. Under concurrent small-file workloads (e.g., many Spark tasks each creating output files simultaneously), TiKV's Raft write amplification is severe — measured at 44x slower than Redis in a production workload [GitHub Discussion #884](../resources/juicefs/juicefs-tikv-redis-github-discussion.summary.md). For large-file sequential workloads, this gap narrows because object storage I/O dominates. For Iceberg compaction jobs creating many small manifest files, TiKV would be a bottleneck.

**PostgreSQL at scale**: PostgreSQL metadata performance is 2-13x slower than Redis at small file sizes. For a moderate-scale lakehouse (tens of millions of files), this is acceptable. For very high metadata operation rates (many concurrent Spark jobs, high-frequency Iceberg compaction), PostgreSQL can become a bottleneck. Adding read replicas improves read-heavy metadata workloads.

**Scaling path**: the Community Edition has no horizontal metadata scaling path beyond memory expansion or using TiKV (with its latency tradeoffs). The Enterprise Edition's proprietary metadata engine solves this but is out of scope for this evaluation.

### 8. Benchmarks: JuiceFS vs MinIO, Ceph, Ozone

Evidence: [Performance Benchmark Docs](../resources/juicefs/juicefs-performance-benchmark-docs.summary.md), [ZeroFS Benchmark](../resources/juicefs/juicefs-zerofs-benchmark-vendor-adjacent.summary.md). Confidence: **LOW** (no independent benchmark found; all data from vendor or vendor-adjacent sources).

No independent benchmark comparing JuiceFS to MinIO, Ceph, or Apache Ozone was found. The available benchmark data:

- **JuiceData benchmarks (vendor)**: compare JuiceFS to AWS EFS and S3FS; claim 10x sequential throughput advantage; no MinIO, Ceph, or Ozone comparison. These comparisons are not relevant to on-premise deployment against equivalent object stores.
- **ZeroFS benchmark (vendor-adjacent)**: claims 175-227x write throughput advantage over JuiceFS; used undisclosed JuiceFS configuration; architecturally incomparable (single-node ZeroFS vs. distributed JuiceFS). JuiceFS community disputed validity. Not usable.
- **CephFS vs JuiceFS**: JuiceData's own comparison page [juicefs-vs-cephfs-docs.summary.md](../resources/juicefs/juicefs-vs-cephfs-docs.summary.md) — vendor-authored; not independently corroborated.

**Implication for decision-making**: teams evaluating JuiceFS against MinIO or Ceph as the storage backend must conduct their own benchmarks with workloads representative of their Spark + Trino + Iceberg usage. No published data bridges this gap. This is a significant evidence gap.

### 9. License and edition: Apache 2.0 Community Edition is the open-source option

Evidence: [GitHub repo](../resources/juicefs/github-juicefs-repo.summary.md), [Enterprise vs Community](../resources/juicefs/juicefs-enterprise-vs-community-vendor.summary.md). Confidence: **HIGH** (license is unambiguous from GitHub; feature split is vendor-described but uncontested).

JuiceFS Community Edition is Apache 2.0 licensed and fully open-source (github.com/juicedata/juicefs). All code is available. The Enterprise Edition is a separate, proprietary, commercially-licensed product with a closed-source metadata engine. The two editions share client code but differ in metadata engine implementation and feature set.

**Enterprise Edition features that are NOT available in Community Edition:**
- Proprietary distributed metadata engine (6x more efficient; handles hundreds of billions of files)
- Distributed cache across nodes (Community: local cache only)
- Real-time directory statistics (Community: requires client-side traversal)
- Directory quotas
- File system mirroring between sites
- GUI management platform

**For on-premise deployment with open-source constraint**: only Community Edition qualifies. Teams needing the Enterprise Edition's scale or distributed cache capabilities would need a commercial contract with JuiceData — which is compatible with on-premise deployment but not with the open-source constraint.

### 10. Production readiness for enterprise lakehouse in 2026

Evidence: [GitHub repo](../resources/juicefs/github-juicefs-repo.summary.md), [Production Recommendations](../resources/juicefs/juicefs-production-deployment-recommendations-docs.summary.md), [2025 Recap](../resources/juicefs/juicefs-2025-recap-vendor.summary.md), [Data Consistency Discussion](../resources/juicefs/juicefs-data-consistency-github-discussion.summary.md), [Hacker News](../resources/juicefs/juicefs-hacker-news-architecture-concerns.summary.md). Confidence: **MEDIUM** (production evidence is vendor-reported; no independent enterprise lakehouse case study found).

JuiceFS Community Edition v1.3.1 (December 2025) is production-stable by the following evidence:
- 13.6k GitHub stars; CSI Driver downloads exceeding 5 million (2025)
- Production deployments confirmed by vendor case studies (Zhihu, StepFun, D-Robotics — primarily AI/ML, not analytics lakehouse)
- Passed 8,813 POSIX compatibility tests (pjdfstest suite)
- v1.3 introduced binary metadata backup (100M files in minutes), improved GC, and operational maturity improvements

**Caveats for enterprise lakehouse:**
1. GC is manual/scheduled — not automatic. A lakehouse with frequent rewrites (Iceberg compaction, partition evolution) must schedule `juicefs gc` as a cron job, or orphaned blocks accumulate in object storage. Early user reports of metadata store bloat [GitHub #2700](../resources/juicefs/juicefs-data-consistency-github-discussion.summary.md) were traced to insufficient GC scheduling.
2. Writeback cache (`--writeback`) must not be used in production lakehouse environments — data loss risk is explicitly documented.
3. Metadata engine HA setup (Redis Sentinel or TiKV cluster) adds significant operational burden for teams not already running these systems.
4. No validated Iceberg + Spark + Trino + JuiceFS Community Edition deployment was found in publicly available sources as of May 2026.
5. Primary production use case in 2025: AI/ML training (sequential large-file reads), not analytics (mixed small-file metadata + large-file data patterns).

---

## Decision Matrix

| Profile | Recommendation | Eliminated options and why |
|---|---|---|
| Small lakehouse (<10M files), on-premise, open-source | JuiceFS Community Edition + PostgreSQL metadata + MinIO backend | Redis Sentinel (overkill for small scale); TiKV (too complex) |
| Medium lakehouse (10M–100M files), performance-priority | JuiceFS Community Edition + Redis Sentinel + MinIO/Ceph S3 backend | TiKV (latency risk at concurrent small-file workloads); PostgreSQL (2-13x slower) |
| Large lakehouse (100M–1B files), durability-priority | JuiceFS Community Edition + PostgreSQL/MySQL (HA replica) + Ceph S3 | Redis (memory capacity insufficient); TiKV (operational complexity vs PostgreSQL) |
| Very large lakehouse (>1B files), open-source constraint | JuiceFS Community Edition + TiKV (accept latency tradeoff) — OR — reconsider MinIO or Apache Ozone | Redis (memory exhausted); PostgreSQL (throughput insufficient); Enterprise Edition (not open-source) |
| Team prefers self-contained storage (no separate metadata engine) | Apache Ozone or Ceph — not JuiceFS | JuiceFS eliminated: requires two separate HA systems; not self-contained |
| Team already running Redis or PostgreSQL in production | JuiceFS Community Edition reuses existing infrastructure | TiKV (new operational dependency); Enterprise Edition (license cost) |

---

## References

- [JuiceFS GitHub Repository (v1.3.1)](../resources/juicefs/github-juicefs-repo.summary.md)
- [JuiceFS Internals Architecture Docs](../resources/juicefs/juicefs-architecture-internals-docs.summary.md)
- [JuiceFS Metadata Engines Setup Docs](../resources/juicefs/juicefs-metadata-engines-setup-docs.summary.md)
- [JuiceFS Metadata Engines Benchmark Docs](../resources/juicefs/juicefs-metadata-engines-benchmark-docs.summary.md)
- [JuiceFS Hadoop Java SDK Docs](../resources/juicefs/juicefs-hadoop-java-sdk-docs.summary.md)
- [JuiceFS S3 Gateway Docs](../resources/juicefs/juicefs-s3-gateway-docs.summary.md)
- [JuiceFS Cache and Writeback Risks Docs](../resources/juicefs/juicefs-writeback-cache-risks-docs.summary.md)
- [JuiceFS Production Deployment Recommendations](../resources/juicefs/juicefs-production-deployment-recommendations-docs.summary.md)
- [Redis Pros/Cons for JuiceFS (JuiceData vendor)](../resources/juicefs/juicefs-redis-pros-cons-vendor.summary.md)
- [Metadata Engine Selection Guide (JuiceData vendor)](../resources/juicefs/juicefs-metadata-engine-selection-guide-vendor.summary.md)
- [JuiceFS vs CephFS Comparison (JuiceData vendor)](../resources/juicefs/juicefs-vs-cephfs-docs.summary.md)
- [Enterprise vs Community Comparison (JuiceData vendor)](../resources/juicefs/juicefs-enterprise-vs-community-vendor.summary.md)
- [JuiceFS GC Mechanism (JuiceData vendor)](../resources/juicefs/juicefs-garbage-collection-vendor.summary.md)
- [JuiceFS Performance Benchmark Docs (JuiceData vendor)](../resources/juicefs/juicefs-performance-benchmark-docs.summary.md)
- [JuiceFS 2025 Recap (JuiceData vendor)](../resources/juicefs/juicefs-2025-recap-vendor.summary.md)
- [TiKV vs Redis GitHub Discussion (#884)](../resources/juicefs/juicefs-tikv-redis-github-discussion.summary.md)
- [Data Consistency Issues GitHub Discussion (#2700)](../resources/juicefs/juicefs-data-consistency-github-discussion.summary.md)
- [Hacker News Architecture Concerns](../resources/juicefs/juicefs-hacker-news-architecture-concerns.summary.md)
- [ZeroFS vs JuiceFS Benchmark (vendor-adjacent)](../resources/juicefs/juicefs-zerofs-benchmark-vendor-adjacent.summary.md)
- [Trino Native S3 Filesystem Docs](../resources/juicefs/trino-native-s3-filesystem-docs.summary.md)

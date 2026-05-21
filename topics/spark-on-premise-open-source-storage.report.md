---
title: Apache Spark Storage for On-Premise Data Centers — Open-Source Options Only
date: 2026-05-21
status: complete
components: [spark, ozone, ceph, hdfs, alluxio, minio]
constraints:
  - all-components-open-source: true
  - deployment: on-premise (own data center)
---

## Overview

This report re-evaluates Spark storage options under two hard constraints: **all components must be open-source**, and **all services run in a privately owned data center**. These constraints immediately eliminate Weka (proprietary), VAST Data (proprietary), Dell PowerScale (proprietary), IBM Spectrum Scale (proprietary), and all cloud-managed services. MinIO, while technically open-source, is eliminated due to its 2025 community edition maintenance freeze and AGPL license enforcement risk.

**Verdict: Apache Ozone is the strongest single choice for new on-premise Spark deployments today. For existing HDFS environments, a phased Ozone migration is the right long-term path. Ceph is the correct choice when multiple independent Spark clusters need to share one storage pool and storage cost efficiency outweighs read latency. Alluxio Community Edition is a valuable complement to Ceph-backed stacks to recover read locality. Do not adopt MinIO.**

---

## Evidence Quality

| Source | Type | Tier | Accessible |
|---|---|---|---|
| Apache Ozone 2.0 ASF Announcement | [link](../resources/ozone/apache-ozone-2-release.summary.md) | official | Yes |
| Red Hat: Spark on Ceph Performance (Part 3) | [link](../resources/ceph/spark-ceph-performance-analysis.summary.md) | vendor-adjacent (Red Hat = Ceph contributor), benchmark-age: 2019 | Yes |
| InfoQ: MinIO Maintenance Mode | [link](../resources/minio/minio-maintenance-mode-risk.summary.md) | press | Yes |
| Alluxio Official Spark Docs | [link](../resources/alluxio/alluxio-spark-integration.summary.md) | official | Yes |
| Apache Spark Hardware Provisioning | [link](../resources/spark/hardware-provisioning-official.summary.md) | official | Yes |

**Gaps and confidence limits**: Ceph benchmark data is from 2019 — the ~2x slowdown figure may not reflect current NVMe-backed Ceph deployments. No independent Ozone 2.0 vs Ceph Spark benchmark was found. "Alluxio bridges the Ceph read gap" is a logical inference with no measured benchmark; treat as MEDIUM confidence. Ozone production case studies beyond Cloudera/CDP are limited.

---

## Components

### `spark`

Spark's storage relationship remains unchanged from the prior report. Key facts that drive storage selection:

- **Shuffle is the primary bottleneck** for join-heavy, aggregation-heavy workloads. Shuffle uses local executor disks, not shared storage. A faster shared storage backend does not improve shuffle.
- **Shared storage is on the critical path only for initial input reads and final output writes.**
- **Data locality**: Spark achieves best performance when it runs on the same nodes as the data. Any remote shared storage (Ceph, Ozone, HDFS accessed remotely) sacrifices locality unless compute is co-located.
- **Small file problem**: Remains real. Parquet compaction (`OPTIMIZE`, `coalesce`, `repartition`) is the first-line solution regardless of storage backend.
- Official Spark recommendation: 4–8 local NVMe disks per node, 10 GbE+ networking. Both are required regardless of storage choice.

### `ozone`

Apache Ozone is the Hadoop ecosystem's designated next-generation distributed object store. It is the most important candidate under the open-source + on-premise constraint.

- **License**: Apache 2.0. Foundation-governed (ASF). No single-vendor risk.
- **Design**: Distributed object store, eliminates the HDFS NameNode bottleneck. Scales from small clusters to billions of objects without federation.
- **Spark compatibility**: Implements the Hadoop Compatible FileSystem API. Spark, Hive, and MapReduce connect to Ozone with zero code changes — identical to HDFS usage patterns.
- **Performance**: In TPC-DS testing, Ozone outperforms HDFS by an average 3.5%; >70% of individual queries run faster on Ozone than HDFS.
- **Maturity**: Ozone 2.0.0 (August 2025) represents production graduation with 1,700+ improvements. Cloudera deploys it in CDP Private Cloud Base. Metadata tooling and operational ecosystem are still maturing relative to HDFS.
- **When to choose**: New deployments, or when HDFS NameNode scalability limits are being approached (hundreds of millions of files, multiple petabytes).

### `hdfs`

HDFS remains the most battle-tested open-source option, but is appropriate for greenfield deployments only if the team has existing Hadoop operational expertise and Ozone migration is not feasible.

- **Strengths**: Best write throughput, best single-user read performance, proven at scale, deep ecosystem of operational tooling (audit, ranger policies, atlas lineage).
- **Weaknesses**: NameNode is a scalability ceiling. Small file inefficiency. Write-once model is a poor fit for lakehouse workloads with frequent compaction (Delta Lake, Iceberg).
- **Recommendation**: Maintain existing HDFS deployments. Do not choose HDFS for greenfield — choose Ozone instead.

### `ceph`

Ceph (LGPL 2.1) is the strongest open-source option for multi-cluster shared storage. It serves S3-compatible object storage via RadosGW, block storage via RBD, and POSIX filesystem via CephFS — all from one cluster.

- **License**: LGPL 2.1. Fully open-source, foundation-neutral (Ceph Foundation under Linux Foundation).
- **Performance vs HDFS for Spark**:
  - Single-user SparkSQL reads: ~2x slower than HDFS
  - Multi-user Spark (10 concurrent): comparable to HDFS
  - Writes: 37–200%+ slower than HDFS
- **Cost advantage**: Erasure coding (EC 4:2) reduces raw storage capacity requirement by ~67% versus HDFS 3x replication. At equal hardware spend, you store ~3x more data.
- **Best fit**: Enterprises running multiple independent Spark clusters that need shared access to the same datasets. Eliminates data duplication across cluster silos. Read-dominated batch workloads.
- **Not suitable for**: Write-intensive ETL, interactive SparkSQL with latency requirements, single-cluster deployments where HDFS data locality would otherwise be achievable.
- **Operational reality**: Complex to operate. Requires dedicated Ceph storage administrator expertise. Tuning for high object-count scenarios (50,000+ objects per prefix) is non-trivial.

### `alluxio`

Alluxio (Apache 2.0) is not a storage system. It is an open-source data caching and orchestration layer.

- **Role**: Sits between Spark executors and a remote storage backend (Ceph, Ozone, HDFS). Caches hot data on executor-local SSDs, restoring data locality that remote storage loses.
- **Community edition limit**: 100 million files. Suitable for small-to-medium production use; inadequate for large data lakes.
- **When to use**: Pair with a Ceph-backed Spark cluster where read performance is degraded by remote S3 access. Alluxio caching on executor-local SSDs can partially recover HDFS-level read performance.
- **Operational cost**: Adds a distributed metadata service and worker fleet. Increases cluster operational complexity.
- **Recommendation**: Optional acceleration layer when Ceph read latency is measurably limiting Spark job performance. Not needed for Ozone or HDFS deployments where locality is already preserved.

### `minio`

**Eliminated.** Do not adopt for new enterprise on-premise deployments.

- **License**: AGPLv3. Any networked service using MinIO must disclose application source code under AGPL — a compliance risk for internal enterprise services with proprietary business logic adjacent to storage.
- **Maintenance mode (Dec 2025)**: No new features, no guaranteed security fixes in community edition.
- **Feature stripping (Mar 2025)**: Web admin UI removed from community edition.
- **Migration**: Existing MinIO deployments should plan migration to Apache Ozone (Hadoop ecosystem) or Ceph RadosGW (S3-compatible, multi-cluster) within 12–18 months.

---

## Findings

### 1. Apache Ozone is the correct default for new on-premise Spark deployments

No other open-source option matches Ozone's combination of: Apache 2.0 license, Hadoop-ecosystem native integration, zero Spark code changes, and benchmark-verified performance parity with (or slight improvement over) HDFS. For organizations starting fresh or planning significant capacity growth past HDFS NameNode limits, Ozone is the right choice.

### 2. HDFS is a maintain-not-expand position

For existing HDFS deployments, continue operating them. HDFS is not wrong for Spark — it remains the highest write-throughput, highest single-user read-performance option. But choosing HDFS for a greenfield deployment in 2025 means accepting the NameNode scalability ceiling and investing in an operational model the Hadoop community is migrating away from.

### 3. Ceph is the right choice for multi-cluster shared storage, not single-cluster Spark

Ceph's ~2x read performance penalty versus HDFS is a real cost. That cost is justified when the architecture requires multiple independent Spark clusters sharing the same data — eliminating HDFS silos across clusters reduces total infrastructure cost and operational complexity at the scale of many clusters. For a single Spark cluster, Ceph's cost advantage does not compensate for the performance gap.

### 4. Alluxio bridges the Ceph performance gap at the cost of added complexity

Adding Alluxio Community Edition as a caching layer between Spark executors and a Ceph-backed object store allows frequently-accessed data to be served from executor-local NVMe SSDs. This partially closes the 2x read gap. The trade-off is an additional distributed system to operate and the 100M file limit of the community edition.

### 5. MinIO is disqualified on both license and sustainability grounds

AGPL creates enterprise compliance risk. Maintenance mode eliminates the open-source value proposition. Organizations should not adopt it and should migrate existing deployments.

### 6. The shuffle bottleneck insight from the prior Weka report applies equally here

Regardless of storage backend choice, the dominant Spark performance lever for join/aggregation workloads is local disk I/O for shuffle. Invest in NVMe SSDs on executor nodes before optimizing shared storage. This is true for HDFS, Ozone, and Ceph equally.

---

## Decision Matrix

| Profile | Recommended storage |
|---|---|
| New deployment, single Spark cluster | **Apache Ozone** |
| New deployment, multiple Spark clusters sharing data | **Ceph (RadosGW/S3A)** + optional Alluxio caching |
| Existing HDFS, scaling within NameNode limits | Keep **HDFS**, evaluate Ozone migration timeline |
| Existing HDFS, hitting NameNode scalability ceiling | Migrate to **Apache Ozone** |
| Existing Ceph, read performance insufficient | Add **Alluxio Community** caching layer |
| Existing MinIO | Migrate to **Ozone** (Hadoop ecosystem) or **Ceph** (S3 ecosystem) |

---

## References

- [Apache Ozone 2.0 Release (official)](../resources/ozone/apache-ozone-2-release.summary.md)
- [Spark on Ceph: Performance Analysis (Red Hat)](../resources/ceph/spark-ceph-performance-analysis.summary.md)
- [MinIO Maintenance Mode Risk (InfoQ)](../resources/minio/minio-maintenance-mode-risk.summary.md)
- [Alluxio Spark Integration (official)](../resources/alluxio/alluxio-spark-integration.summary.md)
- [Apache Spark Hardware Provisioning (official)](../resources/spark/hardware-provisioning-official.summary.md)
- [Prior report: Spark + Weka (no open-source constraint)](spark-weka-enterprise-storage.report.md)

---
title: Best Open-Source Storage Service for Spark + Trino Lakehouse — On-Premise Enterprise
date: 2026-05-21
status: complete
components: [spark, trino, ozone, ceph, hdfs]
constraints:
  - open-source: true
  - deployment: on-premise (own data center)
  - compute-engines: [spark, trino]
  - focus: physical storage service layer only
related-reports:
  - spark-lakehouse-open-source-on-premise.report.md
  - spark-on-premise-open-source-storage.report.md
---

## Overview

This report focuses specifically on the **physical storage service** layer — where bytes are stored and how engines access them (Ceph, Ozone, HDFS, MinIO-class systems). It does not re-evaluate table formats or catalogs except where they constrain the storage choice. The research scope is the same as prior reports (open-source, on-premise) with one addition: the storage layer must serve **both Spark and Trino** in a shared lakehouse.

**Verdict: Apache Ozone is the correct storage service. Adding Trino to the stack makes Ozone's S3-compatible interface mandatory in practice — Trino is actively deprecating its HDFS/Hadoop file system libraries in favour of native S3. Ceph (RadosGW) is the alternative when multi-cluster shared storage or storage-cost reduction is the primary driver. HDFS is disqualified for new deployments because Trino is moving away from it. MinIO is eliminated. SeaweedFS is a watch item for MinIO-replacement scenarios.**

---

## Evidence Quality

| Source | File | Tier | Accessible |
|---|---|---|---|
| Trino Iceberg connector (official docs) | [link](../resources/trino/trino-iceberg-connector-capabilities.summary.md) | official | Yes |
| Trino Hudi connector (official docs) | [link](../resources/trino/trino-hudi-connector-limitations.summary.md) | official | Yes |
| Trino legacy filesystem deprecation | [link](../resources/trino/trino-legacy-filesystem-deprecation.summary.md) | official | Yes |
| Trino + Ozone + Polaris reference arch | [link](../resources/trino/trino-ozone-polaris-reference-architecture.summary.md) | vendor-adjacent (Apache Polaris blog) | Yes |
| Iceberg practical limitations 2025 | [link](../resources/iceberg/iceberg-practical-limitations-2025.summary.md) | press | Yes |
| Apache Ozone 2.0 release (prior research) | [link](../resources/ozone/apache-ozone-2-release.summary.md) | official | Yes |
| Ceph/Spark performance benchmark (prior research) | [link](../resources/ceph/spark-ceph-performance-analysis.summary.md) | vendor-adjacent, benchmark-age: 2019 (stale) | Yes |
| MinIO maintenance mode risk (prior research) | [link](../resources/minio/minio-maintenance-mode-risk.summary.md) | press | Yes |

**Gaps and confidence limits**: No independent benchmark comparing Ozone vs Ceph specifically for dual Spark+Trino access patterns was found. Ozone's Trino integration documentation is incomplete (the official Ozone docs page for Trino is a TODO placeholder). Ceph performance data is 2019 vintage. The Ozone STS limitation (static credentials required) has no documented workaround beyond accepting the constraint.

---

## Components

### `spark`

Spark's storage access pattern from prior reports: reads input at job start, writes output at job end, shuffle on local disks. For lakehouse workloads, Spark also runs compaction (`rewrite_data_files`) and ingestion from streaming sources. Spark supports S3A (`s3a://`), Ozone native (`ofs://`), and HDFS (`hdfs://`) natively.

New constraint when shared with Trino: **the storage interface must be the same for both engines**. Spark and Trino must read/write the same files from the same path prefix. S3A (Spark) and native S3 (Trino) both resolve to the same underlying storage — this compatibility is the key requirement. [Source](../resources/trino/trino-iceberg-connector-capabilities.summary.md)

### `trino`

Trino's storage access pattern is fundamentally different from Spark in one critical way: **Trino is deprecating Hadoop file system libraries** and moving to native S3 as its primary non-cloud storage interface. [Source](../resources/trino/trino-legacy-filesystem-deprecation.summary.md)

- **Native S3 (preferred)**: `fs.native-s3.enabled=true`. Zero Hadoop dependency. Works with any S3-compatible endpoint.
- **HDFS (shim)**: `fs.hadoop.enabled=true`. Compatibility only, no removal date but actively unwinding.
- **Interface required for new deployments**: S3-compatible — Ozone S3 gateway or Ceph RadosGW.

Trino's role in the lakehouse: interactive SQL analytics, ad-hoc queries, BI tool federation. Spark handles batch writes and compaction. [Source](../resources/trino/trino-iceberg-connector-capabilities.summary.md)

---

## Storage Services Evaluated

### Apache Ozone — Recommended

**License**: Apache 2.0, ASF-governed.

**Why Ozone is the right choice for Spark+Trino:**

1. **Dual interface**: Ozone exposes both `s3a://` (Spark S3A client) and native S3 API (Trino native S3 client) from the same storage cluster. Both engines connect to the same data using their preferred interface with zero translation layer. This is the cleanest dual-engine access model. [Source](../resources/trino/trino-ozone-polaris-reference-architecture.summary.md)

2. **Trino filesystem alignment**: Trino's direction is native S3. Ozone's S3 gateway is production-validated for Trino (Apache Polaris reference architecture, April 2026). Required config: `s3.path-style-access=true`, static credentials (see limitation below). [Source](../resources/trino/trino-legacy-filesystem-deprecation.summary.md)

3. **Lakehouse metadata scalability**: Lakehouse workloads generate billions of small metadata files (Iceberg manifests, snapshots). Ozone's distributed SCM/OM metadata model is purpose-built for billions of objects, unlike HDFS's NameNode. [Source](../resources/ozone/apache-ozone-2-release.summary.md)

4. **Performance**: Ozone outperforms HDFS by an average of 3.5% on TPC-DS across 99 queries. [Source](../resources/ozone/apache-ozone-2-release.summary.md) — Confidence: MEDIUM (official ASF + Cloudera source, no independent corroboration).

5. **Single-cluster simplicity**: One storage cluster serves both Spark and Trino. No cross-cluster replication, no data silos.

**Known limitation — no STS endpoint**: Ozone does not implement AWS STS. Advanced catalog features (Polaris credential vending, per-table scoped access tokens) fall back to static credentials. In an on-premise data center with network-level isolation, this is operationally acceptable. It is a security gap relative to cloud-native deployments. [Source](../resources/trino/trino-ozone-polaris-reference-architecture.summary.md) — Confidence: HIGH (documented in official Polaris blog with explicit workaround noted).

**Operational maturity caveat**: Ozone 2.0 released August 2025. Production ecosystem tooling (quota management, backup/DR tooling, audit integration) less mature than HDFS. Cloudera CDP is the main production reference. [Source](../resources/ozone/apache-ozone-2-release.summary.md)

---

### Ceph (RadosGW S3 interface) — Alternative for multi-cluster deployments

**License**: LGPL 2.1, Linux Foundation.

**When Ceph is the right choice:**

Ceph is correct when you have **multiple independent Spark+Trino clusters** sharing one storage pool, and storage cost reduction is a hard constraint (erasure coding 4:2 saves ~67% capacity vs 3x replication).

**Trino compatibility**: Trino's native S3 connects to Ceph RadosGW identically to Ozone — same `fs.native-s3.enabled=true` configuration with endpoint override and `s3.path-style-access=true`. [Source](../resources/trino/trino-legacy-filesystem-deprecation.summary.md)

**Performance trade-off for lakehouse specifically**: Ceph's ~2x read slowdown vs HDFS (2019 benchmark, likely improved with NVMe Ceph but unverified) is amplified in lakehouse access patterns. Iceberg manifests and Parquet footers are small files read frequently — the read latency penalty compounds in metadata-heavy lakehouse workloads. [Source](../resources/ceph/spark-ceph-performance-analysis.summary.md) — Confidence: LOW (2019 benchmark, stale).

**Trino query performance on Ceph**: No independent benchmark comparing Trino query latency on Ceph vs Ozone was found. The 2x read penalty from the Spark benchmark should be assumed to apply directionally, but cannot be quantified with available evidence.

**Multi-engine concurrent access**: Ceph RadosGW handles concurrent S3 requests from multiple clients (Spark + Trino) correctly — object storage is inherently concurrent-reader safe. Write conflicts are resolved at the Iceberg catalog level, not the storage level.

---

### HDFS — Not recommended for new deployments

**Why HDFS is disqualified when Trino is in scope:**

Trino is actively deprecating Hadoop file system support. The `fs.hadoop.enabled=true` shim works today but accumulates technical debt with every Trino release. A new deployment choosing HDFS as its storage layer for a Spark+Trino lakehouse is building against Trino's stated direction. [Source](../resources/trino/trino-legacy-filesystem-deprecation.summary.md) — Confidence: HIGH (official Trino engineering blog, February 2025).

For existing HDFS environments with Trino already deployed: continue operating. Migrate to Ozone when Trino's Hadoop shim is removed or when HDFS NameNode scaling limits are reached.

---

### MinIO — Eliminated

Eliminated in the prior open-source storage report: AGPL license, community edition in maintenance mode, no guaranteed security fixes. [Source](../resources/minio/minio-maintenance-mode-risk.summary.md)

Trino compatibility is not a differentiating factor for MinIO — it works technically (S3-compatible). The elimination is on license and sustainability grounds.

**SeaweedFS (Apache 2.0)** is the emerging open-source replacement for MinIO's S3-compatible role. It is technically compatible with both Spark S3A and Trino native S3. However, it is immature for enterprise production (no major production case studies at lakehouse scale found). Watch for 2026–2027 adoption signals before evaluating seriously.

---

### Weka — Eliminated (proprietary)

Eliminated under the open-source constraint. See [Weka report](spark-weka-enterprise-storage.report.md) for full analysis.

---

## Findings

### 1. Trino's filesystem deprecation settles the HDFS vs S3 question for new deployments

Evidence: [Trino filesystem deprecation](../resources/trino/trino-legacy-filesystem-deprecation.summary.md) (official). Confidence: **HIGH**.

Trino 470 (February 2025) deprecated Hadoop-based file system support. Any new on-premise lakehouse with Trino as a query engine should use S3-compatible storage from day one. This eliminates HDFS as a greenfield choice and removes the ambiguity between HDFS and S3-compatible that existed in the Spark-only analysis.

### 2. Ozone's dual interface makes it uniquely suited for Spark+Trino co-existence

Evidence: [Ozone reference architecture](../resources/trino/trino-ozone-polaris-reference-architecture.summary.md) (vendor-adjacent), [Trino Iceberg connector](../resources/trino/trino-iceberg-connector-capabilities.summary.md) (official), [Ozone 2.0 release](../resources/ozone/apache-ozone-2-release.summary.md) (official). Confidence: **MEDIUM** — architecture correctness is confirmed; production scale evidence beyond Cloudera/CDP is limited.

Spark uses `s3a://` (S3A client) and Trino uses native S3 (`fs.native-s3.enabled=true`) — both resolve to the same Ozone S3 gateway. No data translation, no separate storage tiers, no duplication. No other open-source on-premise storage provides this combination of Hadoop ecosystem depth (for Spark) and S3-native access (for Trino) from a single cluster.

### 3. The Ozone STS gap is an operational security constraint, not a blocker

Evidence: [Trino+Ozone+Polaris reference architecture](../resources/trino/trino-ozone-polaris-reference-architecture.summary.md) (vendor-adjacent). Confidence: **HIGH** — the limitation is documented in the official reference architecture with an explicit workaround.

Ozone has no STS endpoint. Short-lived, per-table scoped credentials (catalog credential vending) cannot be used. Static credentials must be configured. In an on-premise data center with internal network perimeters, this is an acceptable trade-off — it is a security architecture consideration, not a functional barrier. Teams requiring per-table credential isolation should note this gap.

### 4. Ceph remains valid for multi-cluster architectures but its read penalty compounds in lakehouse patterns

Evidence: [Ceph/Spark benchmark](../resources/ceph/spark-ceph-performance-analysis.summary.md) (vendor-adjacent, 2019 — stale), [Iceberg practical limitations](../resources/iceberg/iceberg-practical-limitations-2025.summary.md) (press). Confidence: **LOW** — benchmark is stale; the compounding effect of Ceph read latency on Iceberg metadata operations is a logical inference, not a measured value.

Iceberg metadata operations (manifest reads, footer scans) are small-file, high-frequency reads. Ceph's latency in this pattern is not independently benchmarked for 2025 NVMe-backed Ceph clusters. The 2x slowdown from 2019 spinning-disk Ceph may not reflect modern deployments. Teams considering Ceph should run their own benchmark before committing.

### 5. Iceberg is the only table format compatible with full Spark+Trino read/write sharing

Evidence: [Trino Hudi connector](../resources/trino/trino-hudi-connector-limitations.summary.md) (official), [Delta Lake governance](../resources/delta-lake/delta-lake-governance-analysis.summary.md) (press). Confidence: **HIGH** — documented in official Trino connectors.

Trino cannot write to Hudi tables (read-only connector). Trino's Delta connector has non-concurrent write limitations on S3-compatible storage. Iceberg is the only format where both Spark and Trino have full read/write capability from a shared catalog. This is not a table format recommendation (scope is storage service) but it is a constraint that the storage service must support Iceberg's S3-based access patterns.

---

## Decision Matrix

| Deployment profile | Storage service | Eliminated options and why |
|---|---|---|
| New deployment, single Spark+Trino cluster | **Apache Ozone** (S3 gateway) | HDFS: Trino deprecating; Ceph: overkill for single cluster; MinIO: eliminated |
| New deployment, multiple Spark+Trino clusters sharing data | **Ceph (RadosGW)** | Ozone: single-cluster optimised; HDFS: Trino deprecating; MinIO: eliminated |
| Existing HDFS, adding Trino | Keep HDFS (shim), plan **Ozone migration** | New storage should be Ozone; HDFS shim works until Trino removes it |
| Existing Ceph, adding Trino | Keep Ceph, use RadosGW S3 interface | No change needed — Trino native S3 connects to RadosGW directly |
| MinIO existing deployment | Migrate to **Ozone** (Hadoop ecosystem) or **Ceph** (multi-cluster) | MinIO: eliminated on sustainability/license grounds |

---

## References

- [Trino Iceberg Connector Capabilities](../resources/trino/trino-iceberg-connector-capabilities.summary.md)
- [Trino Hudi Connector Limitations](../resources/trino/trino-hudi-connector-limitations.summary.md)
- [Trino Legacy Filesystem Deprecation](../resources/trino/trino-legacy-filesystem-deprecation.summary.md)
- [Trino + Ozone + Polaris Reference Architecture](../resources/trino/trino-ozone-polaris-reference-architecture.summary.md)
- [Iceberg Practical Limitations 2025](../resources/iceberg/iceberg-practical-limitations-2025.summary.md)
- [Apache Ozone 2.0 Release](../resources/ozone/apache-ozone-2-release.summary.md)
- [Ceph/Spark Performance Analysis](../resources/ceph/spark-ceph-performance-analysis.summary.md)
- [MinIO Maintenance Mode Risk](../resources/minio/minio-maintenance-mode-risk.summary.md)
- [Prior: Spark Lakehouse Open-Source Stack](spark-lakehouse-open-source-on-premise.report.md)
- [Prior: Spark On-Premise Storage](spark-on-premise-open-source-storage.report.md)

---
title: Best Open-Source Storage Stack for Spark Lakehouse — On-Premise Enterprise
date: 2026-05-21
status: complete
components: [spark, iceberg, hudi, delta-lake, ozone, ceph, nessie]
constraints:
  - all-components-open-source: true
  - deployment: on-premise (own data center)
related-reports:
  - spark-weka-enterprise-storage.report.md
  - spark-on-premise-open-source-storage.report.md
---

## Overview

A lakehouse is not a single product — it is a three-layer stack: **physical storage** (where bytes live), **table format** (ACID transactions, schema evolution, time travel on files), and **catalog** (metadata governance, multi-engine consistency). Under the open-source + on-premise constraint, each layer must be evaluated independently and for how they compose together.

**Verdict: The strongest open-source on-premise Spark lakehouse stack is Apache Iceberg + Apache Ozone + Project Nessie for analytical (read-heavy, multi-engine) workloads. For streaming-first CDC-heavy workloads, replace Iceberg with Apache Hudi and drop the catalog requirement. Avoid Delta Lake for on-premise open-source — its best features are Databricks-proprietary and multi-writer concurrency requires solving a lock-provider problem without cloud primitives.**

---

## Evidence Quality

| Source | Type | Tier | Accessible |
|---|---|---|---|
| Onehouse: Iceberg vs Hudi vs Delta comparison | [link](../resources/iceberg/iceberg-hudi-delta-lakehouse-comparison.summary.md) | vendor-adjacent (Onehouse = Hudi's commercial backer) | Yes |
| Cloudera: Iceberg on Ozone production architecture | [link](../resources/iceberg/iceberg-ozone-cloudera-lakehouse.summary.md) | vendor-adjacent (Cloudera sells CDP with Iceberg+Ozone) | Yes |
| Delta Lake governance analysis | [link](../resources/delta-lake/delta-lake-governance-analysis.summary.md) | press | Yes |
| Hudi lakehouse strengths | [link](../resources/hudi/hudi-lakehouse-streaming-strengths.summary.md) | vendor (Apache Hudi blog / Onehouse) | Yes |
| Project Nessie: on-premise catalog | [link](../resources/nessie/nessie-on-premise-catalog.summary.md) | vendor-adjacent (Dremio = Nessie backer) | Yes |
| Apache Ozone 2.0 release | [link](../resources/ozone/apache-ozone-2-release.summary.md) | official | Yes |
| Ceph/Spark performance benchmarks | [link](../resources/ceph/spark-ceph-performance-analysis.summary.md) | vendor-adjacent, benchmark-age: 2019 (stale) | Yes |

**Gaps and confidence limits**: No source in this report is financially neutral (analyst/press) for the core table format comparison — all format-specific claims are vendor-adjacent or vendor. "Iceberg is the slowest" rests on a Hudi-backer comparison only (LOW confidence). "Nessie 28.6% adoption" is from a Dremio-run survey (vendor-adjacent). Ceph performance data is 2019 vintage. "Hudi production at Uber/LinkedIn scale" is stated in the report body without a cited source — unverified claim. No counter-evidence search was run for Nessie or Ozone reliability at scale.

---

## Components

### `spark`

Spark is the compute layer — unchanged from prior reports. Critical constraints that shape storage layer selection:

- Shuffle uses local executor disks. Shared storage only affects input reads and output writes.
- Data locality is important. Compute and storage co-location (HDFS-style) or a caching layer (Alluxio) helps reads.
- Spark reads and writes Iceberg, Hudi, and Delta Lake natively via their respective Spark extensions.
- For lakehouse workloads: concurrent writers, incremental reads, and compaction management are the three dimensions that most differentiate the table format choice.

### `iceberg`

Apache Iceberg (Apache 2.0, ASF) is the emerging industry standard for open lakehouse table formats.

**Strengths:**
- Strongest multi-engine support: Spark, Trino, Flink, Presto, Hive, Impala, Dremio, Starrocks, Doris — all read/write Iceberg natively. Any engine added to the stack in the future likely supports Iceberg first.
- Partition evolution: change partitioning strategy (daily→hourly, by-date→by-region) as a pure metadata operation without rewriting data.
- Hidden partitioning: writers never specify partitions manually; queries prune partitions automatically from column predicates.
- Snapshot isolation: full serialisable reads at any historical snapshot (time travel, audit, reproducibility).
- ASF governance: no single commercial entity controls the project roadmap.

**Weaknesses:**
- **Slowest in TPC-DS benchmarks** of the three formats. Performance gap is meaningful for read-heavy analytical workloads.
- **No built-in compaction or table services.** Small file accumulation and snapshot cleanup require manual Spark jobs or third-party tooling (scheduled Spark `rewrite_data_files`, `expire_snapshots`).
- **Catalog is mandatory.** Iceberg's consistency model relies on the catalog for atomic metadata updates. No catalog = undefined behaviour under concurrent writes.
- **No native CDC streaming ingestion.** Ingestion pipelines from Kafka/Debezium require external tooling (Flink Iceberg sink, Kafka Connect Iceberg connector, or custom Spark Structured Streaming).
- Deletion vectors have a one-per-file constraint (vs Hudi's full MoR), limiting update efficiency for high-frequency upsert workloads.

**On-premise storage compatibility:**
- Works natively with `s3a://` (Ozone S3, Ceph RGW) and `ofs://` (Ozone native), and HDFS.
- Ozone is the recommended physical storage — designed to handle the billions of small metadata and data files a lakehouse accumulates.

### `hudi`

Apache Hudi (Apache 2.0, ASF) is the self-managing, streaming-first lakehouse table format.

**Strengths:**
- **Non-blocking concurrency control (NBCC)**: uniquely allows concurrent writers to commit without conflict retries. Essential for continuous streaming ingestion pipelines where write stalls translate to consumer lag.
- **DeltaStreamer**: open-source, production-proven managed ingestion service from Kafka, Debezium, and file sources. Handles checkpointing, schema registry, deduplication, error queues. No equivalent in Iceberg or Delta open-source.
- **Full CDC semantics**: before/after images and change type per record. Downstream incremental queries know exactly which keys changed and how.
- **Automated table services**: compaction, file sizing, clustering, and timeline cleaning are self-managing background services. The small file problem is handled automatically without manual operator intervention.
- **Record-level index**: key-based upserts (MERGE INTO) resolve target files via index lookup — no full partition scan required.
- **No catalog requirement**: Hudi manages its own timeline in the filesystem. Hive Metastore is optional for table discovery only. Removes a critical operational dependency.

**Weaknesses:**
- **Read performance on append-only workloads** is slower than Delta when Hudi is in default upsert config. Requires explicit `bulk_insert` mode tuning for analytical-dominant workloads.
- **Multi-engine portability lags Iceberg.** Iceberg is now the lingua franca for cross-engine lakehouse interoperability. New query engines and cloud services adopt Iceberg first.
- **Operational complexity**: DeltaStreamer, async compaction services, and timeline configuration have significant tuning surface area.

**On-premise storage compatibility:**
- Native HDFS support. Works with Ozone `ofs://` and `s3a://`. Works with Ceph `s3a://`.
- NBCC and record-level index work correctly on all three backends.

### `delta-lake`

**Not recommended for on-premise open-source lakehouse.** Included to explain the decision.

**License**: Apache 2.0. Open-source distribution is real.

**Disqualifying problems on-premise:**
1. **Best features are proprietary**: auto-optimize (file sizing), Autoloader (streaming ingest), Bloom filters, Photon execution, and constraints are available only on the Databricks Runtime — not in open-source Delta. On-premise without Databricks, you get the format without its operational tooling.
2. **Multi-writer lock provider problem**: Delta OCC requires an external lock coordinator. Default is DynamoDB (AWS-native). On-premise requires a custom lock provider implementation or serialised writes (throughput bottleneck).
3. **De facto Databricks control**: despite Linux Foundation governance, Databricks employees dominate the TSC and commit history. The open-source roadmap tracks Databricks product interests.

Delta Lake is a legitimate choice if you are running Databricks on-premise via Databricks on your own hardware (available commercially). It is not the right choice for a fully open-source self-managed stack.

### `ozone`

Apache Ozone (Apache 2.0, ASF) is the recommended physical storage layer — see prior report for detailed analysis.

**Lakehouse-specific fit:**
- Handles billions of small objects: Iceberg and Hudi both generate large numbers of small metadata files (manifest lists, manifests, timeline files). Ozone's distributed SCM/OM metadata avoids the HDFS NameNode saturation that these workloads cause.
- Native S3 compatibility (`s3a://`): Iceberg's multi-engine clients all use S3A; Ozone's S3-compatible API means any engine can connect without filesystem-specific drivers.
- Erasure coding for data files reduces storage capacity requirements by ~67% vs 3x replication.
- Cloudera uses Ozone as the physical storage layer in their production Iceberg lakehouse reference architecture.

### `ceph`

Ceph (RadosGW S3 interface) is the right physical storage for lakehouse deployments that require **shared access across multiple independent compute clusters**.

**Lakehouse-specific caveats:**
- Ceph's ~2x read slowdown vs HDFS/Ozone is magnified in lakehouse patterns where metadata files are small and numerous (manifest lookups, footer reads). Iceberg's 10x query planning speedup over Hive partitioned tables partially compensates.
- Ceph works better with Hudi than Iceberg for write-heavy workloads: Hudi's record-level index reduces the number of files rewritten per upsert, partially compensating for Ceph's write penalty.
- Alluxio Community Edition as a caching layer on executor-local SSDs can recover Ceph's read performance gap for frequently accessed hot data.

### `nessie`

Project Nessie (Apache 2.0) is the recommended on-premise catalog for Iceberg-based lakehouses.

**Why Nessie over Hive Metastore:**
- Cross-table ACID transactions: multiple Iceberg tables can be updated atomically in a single Nessie commit. Hive Metastore has no cross-table consistency.
- Git-like data versioning: branch the catalog for `dev`/`staging`/`prod` promotion workflows. Experiment with schema changes on a branch without touching production.
- Implements Iceberg REST Catalog spec: any engine connects with standard config.
- Self-hosted on PostgreSQL — full data sovereignty, no cloud dependency.
- 28.6% adoption in 2025 Iceberg ecosystem survey.

**Why not Apache Polaris (incubating):** Still in ASF incubation, operational maturity lower than Nessie. Watch for graduation.

**Why not Apache Gravitino:** Broader scope (data + AI asset management) but less mature for pure Iceberg catalog use. Better positioned as a future unified governance layer.

---

## Findings

### 1. The lakehouse stack is three independent decisions, not one

Evidence: [Iceberg/Ozone architecture](../resources/iceberg/iceberg-ozone-cloudera-lakehouse.summary.md), [format comparison](../resources/iceberg/iceberg-hudi-delta-lakehouse-comparison.summary.md). Confidence: **MEDIUM** — architecture principle is sound; the specific combination validations come from vendor-adjacent sources only.

Physical storage, table format, and catalog must be chosen separately and verified for composition. The validated on-premise reference architecture is: **Ozone + Iceberg + Nessie** (Cloudera production evidence, vendor-adjacent) and **HDFS + Hudi** (community-confirmed production at scale, no independent source cited here).

### 2. Iceberg is the right table format for analytical-dominant enterprise lakehouses

Evidence: [format comparison](../resources/iceberg/iceberg-hudi-delta-lakehouse-comparison.summary.md) (vendor-adjacent, Hudi-biased), [Delta governance](../resources/delta-lake/delta-lake-governance-analysis.summary.md) (press). Confidence: **MEDIUM** — multi-engine superiority and ASF governance are well-documented; the performance gap vs Hudi rests on a vendor-biased benchmark only.

Its performance gap vs Hudi is stated in a Hudi-backer comparison and has no neutral corroboration. Treat the magnitude as uncertain. The compaction gap is solved with a scheduled Spark maintenance job.

### 3. Hudi is the right table format when streaming CDC ingestion dominates

Evidence: [Hudi strengths](../resources/hudi/hudi-lakehouse-streaming-strengths.summary.md) (vendor), [format comparison](../resources/iceberg/iceberg-hudi-delta-lakehouse-comparison.summary.md) (vendor-adjacent). Confidence: **LOW** — both sources have a financial stake in Hudi. Technical claims (NBCC, DeltaStreamer, record-level index) are architecturally verifiable from project docs, but no neutral benchmark confirms their advantage under real enterprise workloads.

Hudi's NBCC and DeltaStreamer are unique and open-source; the framing of their superiority comes from Hudi-aligned sources and should be independently verified before committing to Hudi as a platform.

### 4. Ozone is the correct physical layer for both formats

Evidence: [Ozone 2.0 release](../resources/ozone/apache-ozone-2-release.summary.md) (official), [Iceberg+Ozone architecture](../resources/iceberg/iceberg-ozone-cloudera-lakehouse.summary.md) (vendor-adjacent). Confidence: **MEDIUM** — official ASF release confirms scalability claims; Cloudera production validation is vendor-adjacent.

Ozone's billion-object metadata scalability and S3-compatible interface are confirmed by official ASF documentation. The claim that Iceberg+Ozone performs well at lakehouse scale comes from Cloudera only.

### 5. Delta Lake is eliminated under these constraints

Evidence: [Delta governance analysis](../resources/delta-lake/delta-lake-governance-analysis.summary.md) (press), [format comparison](../resources/iceberg/iceberg-hudi-delta-lakehouse-comparison.summary.md) (vendor-adjacent). Confidence: **HIGH** — proprietary feature stratification and lock-provider requirements are verifiable directly from Delta's own documentation and are corroborated by independent press.

The proprietary feature list (auto-optimize, Autoloader, Bloom filters) is documented by Databricks itself. The DynamoDB lock provider default is in Delta's own configuration docs. These are not contested claims.

### 6. Compaction is the operational hidden cost for Iceberg

Evidence: [format comparison](../resources/iceberg/iceberg-hudi-delta-lakehouse-comparison.summary.md) (vendor-adjacent). Confidence: **MEDIUM** — the absence of built-in compaction in Iceberg is a documented architectural fact; the operational burden assessment is a qualitative judgment, not a measured cost.

---

## Decision Matrix

| Workload profile | Recommendation | Eliminated options and why |
|---|---|---|
| Analytical reads, multi-engine, new deployment | Iceberg + Ozone + Nessie | Hudi (weaker multi-engine); Delta (proprietary tooling, lock-provider gap) |
| Streaming CDC, continuous upserts, Spark-centric | Hudi + Ozone | Iceberg (no native CDC, manual compaction); Delta (eliminated) |
| Multi-cluster shared lake, cost-sensitive | Iceberg + Ceph (S3A) + Alluxio + Nessie | Ozone (single-cluster optimised, no multi-cluster cost advantage) |
| Existing HDFS, incremental modernisation | Hudi + HDFS → Ozone migration | Delta (eliminated); Iceberg (requires catalog uplift before HDFS migration) |
| Pure Databricks on-premise (commercial) | Delta Lake + Unity Catalog | Not applicable under open-source constraint |

---

## References

- [Iceberg vs Hudi vs Delta Lake: Feature Comparison](../resources/iceberg/iceberg-hudi-delta-lakehouse-comparison.summary.md)
- [Iceberg on Ozone: Cloudera Production Architecture](../resources/iceberg/iceberg-ozone-cloudera-lakehouse.summary.md)
- [Delta Lake Governance Analysis](../resources/delta-lake/delta-lake-governance-analysis.summary.md)
- [Hudi Lakehouse Streaming Strengths](../resources/hudi/hudi-lakehouse-streaming-strengths.summary.md)
- [Project Nessie On-Premise Catalog](../resources/nessie/nessie-on-premise-catalog.summary.md)
- [Apache Ozone 2.0 Release](../resources/ozone/apache-ozone-2-release.summary.md)
- [Ceph/Spark Performance Analysis](../resources/ceph/spark-ceph-performance-analysis.summary.md)
- [Prior: Open-Source On-Premise Storage for Spark](spark-on-premise-open-source-storage.report.md)

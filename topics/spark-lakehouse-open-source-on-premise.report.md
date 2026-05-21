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
| Onehouse: Iceberg vs Hudi vs Delta comparison | article | independent (Hudi-biased) | Yes |
| Cloudera: Iceberg on Ozone production architecture | article | independent | Yes |
| Delta Lake governance analysis (Dremio + trade press) | article | independent | Yes |
| Hudi lakehouse strengths (Apache Hudi blog + Onehouse) | article | vendor | Yes |
| Project Nessie: on-premise catalog (projectnessie.org + LakeFS) | article | official + independent | Yes |
| Apache Ozone 2.0 release (from prior research) | article | official | Yes |
| Ceph/Spark performance benchmarks (from prior research) | article | independent | Yes |

**Gaps**: No independent TPC-DS benchmark directly comparing Iceberg+Ozone vs Hudi+Ozone in an on-premise environment was found. Nessie production scale data at 10B+ objects is limited. Hudi read performance data on Ozone (vs HDFS) is not independently published.

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

Physical storage, table format, and catalog must be chosen separately and verified for composition. The best independently chosen options may not be the best combination. The validated on-premise reference architecture is: **Ozone + Iceberg + Nessie** (Cloudera production evidence) and **Ozone/HDFS + Hudi** (long-standing production at Uber, LinkedIn scale).

### 2. Iceberg is the right table format for analytical-dominant enterprise lakehouses

If the primary workload is analytical reads (SQL queries, BI, ad-hoc), multi-engine access (Spark + Trino + Flink on the same tables), and the data team values ecosystem longevity and future optionality, Iceberg is the correct choice. Its performance gap vs Hudi is real but acceptable when weighed against its multi-engine interoperability and ASF governance. The compaction gap is solved with a scheduled Spark maintenance job.

### 3. Hudi is the right table format when streaming CDC ingestion dominates

If the primary ingestion pattern is continuous CDC from Debezium/Kafka into mutable tables with high upsert rates, Hudi's NBCC, DeltaStreamer, and automated compaction make it operationally superior. The write infrastructure is entirely self-managing in open-source. The multi-engine interoperability sacrifice is acceptable when Spark is the primary (or only) query engine.

### 4. Ozone is the correct physical layer for both formats

Ozone's billion-object metadata scalability, S3-compatible interface, and ASF governance make it the best physical storage for any new on-premise lakehouse. The Iceberg+Ozone combination has Cloudera production validation. Hudi+Ozone follows naturally from Hudi's HDFS-compatible FileSystem API support.

### 5. Delta Lake is eliminated under these constraints

The best Delta features require Databricks Runtime. Multi-writer on-premise needs a custom lock provider. De facto Databricks roadmap control is an open-source governance risk. No recommendation for on-premise open-source deployments.

### 6. Compaction is the operational hidden cost for Iceberg

Iceberg requires explicit compaction scheduling. For production lakehouses with continuous ingestion, this means a scheduled Spark job running `rewrite_data_files` and `expire_snapshots`. This is manageable but is operational overhead that Hudi eliminates with its async table services.

---

## Decision Matrix

| Workload profile | Table format | Physical storage | Catalog |
|---|---|---|---|
| Analytical reads, multi-engine, new deployment | **Apache Iceberg** | **Apache Ozone** | **Project Nessie** |
| Streaming CDC, continuous upserts, Spark-centric | **Apache Hudi** | **Apache Ozone** | Hive Metastore (optional) |
| Multi-cluster shared lake, cost-sensitive | **Apache Iceberg** | **Ceph (S3A)** + Alluxio caching | **Project Nessie** |
| Existing HDFS, incremental modernisation | **Apache Hudi** | **HDFS** (migrate to Ozone later) | Hive Metastore |
| Pure Databricks stack on-premise | Delta Lake | — | Unity Catalog |

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

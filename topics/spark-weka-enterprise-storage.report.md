---
title: Apache Spark with Weka Storage for Enterprise — Is It the Best Choice?
date: 2026-05-21
status: complete
components: [spark, weka, storage]
---

## Overview

Weka (weka.io) markets its NeuralMesh parallel filesystem as a high-performance storage backend for Apache Spark, publishing a TPC-DS benchmark showing that storage quality directly determines Spark job throughput — particularly for small Parquet files. This report evaluates that claim critically, assesses Weka's fit for enterprise Spark deployments, and compares it against realistic alternatives.

**Verdict: Weka is a technically credible option for specific enterprise profiles, but it is not the best default choice for most Spark deployments.** The decision depends heavily on workload type, existing infrastructure, budget, and whether AI/ML workloads are collocated with Spark.

---

## Evidence Quality

| Source | Type | Tier | Accessible |
|---|---|---|---|
| Weka TPC-DS Performance Brief | pdf | vendor | No (gated) |
| Weka NeuralMesh Architecture Blog | article | vendor | Yes |
| Apache Spark Hardware Provisioning Docs | article | official | Yes |
| Ceph vs VAST vs Weka Comparison | article | independent | Yes |
| Enterprise Storage Market Landscape | article | independent | Yes |

**Gaps**: The primary evidence for Weka's Spark performance (TPC-DS brief) was inaccessible. No independent reproduction of that benchmark was found. Gartner reviews behind auth walls could not be read directly. Vendor-side evidence dominates Weka-specific claims; treat those findings as directional, not authoritative.

---

## Components

### `spark`

Apache Spark is an in-memory distributed compute engine. Its relationship with storage is nuanced:

- **Storage is only on the critical path at job start (input read) and end (output write).** Intermediate operations live in memory or spill to local disks.
- **Shuffle is the true performance bottleneck** in join-heavy and aggregation-heavy workloads. Shuffle uses local disks on each executor node — not the distributed storage backend. An ultra-fast shared storage layer does not help shuffle at all.
- **Data locality matters**: Spark performs best when compute runs on the same nodes as data. Pure remote storage (NFS, S3, Weka over network) sacrifices locality unless Weka clients run on every Spark executor node.
- **Small file problem is real but addressable**: Many small Parquet files (Weka's main benchmark scenario) cause excessive metadata operations and reduce scan throughput. This is commonly solved by compaction (`OPTIMIZE` in Delta Lake/Iceberg, `coalesce`/`repartition` in Spark), without requiring premium storage.
- Official Spark docs recommend 10 GbE minimum networking and 4–8 local NVMe disks per node regardless of the distributed storage choice.

### `weka`

Weka's NeuralMesh is a software-defined parallel filesystem built for NVMe SSDs:

- **Architecture**: Software layer on standard x86 hardware; requires a proprietary client on every compute node. Supports POSIX file access over Ethernet or InfiniBand with sub-millisecond latency.
- **Performance claims**: >600 GB/s sustained throughput, 5M IOPS. These figures are vendor-stated and apply to optimized HPC configurations; real-world Spark throughput will be lower and highly topology-dependent.
- **NeuralMesh deduplication**: Similarity-based dedup useful for AI checkpoints and versioned datasets. Benefit for Spark Parquet data is workload-dependent and not benchmarked independently.
- **Primary market**: AI/ML training, genomics, EDA, autonomous vehicle data pipelines. Spark analytics is a secondary use case.
- **Gartner rating**: 4.9/5 (119 verified reviews, 2025 Customers' Choice). However, reviewer base skews toward AI/HPC users, not traditional analytics.
- **Pricing**: Opaque. No public per-TB pricing. Custom enterprise quotes only. Premium-tier cost assumed based on market positioning.
- **Lock-in risk**: Proprietary client requirement creates operational dependency. Migrating off Weka requires reinstalling the client layer across all nodes and re-validating Spark configurations.

### `storage`

Enterprise storage options for Spark range from commodity to premium:

| Option | Throughput | Cost | Spark fit | Complexity |
|---|---|---|---|---|
| HDFS (co-located) | High (local) | Low (commodity HW) | Best for on-prem locality | High (NameNode ops) |
| S3 / object storage | Medium | Low–Medium (cloud) | Good for cloud-native, batch | Low |
| IBM Storage Scale | High (parallel FS) | Medium–High | Excellent — HDFS-API compatible | Medium |
| Dell PowerScale + pNFS | Medium–High | Medium | Good — scale-out NFS, Project Lightning | Medium |
| VAST Data | High | Premium | Very good — unified NFS+S3 namespace | Medium |
| **Weka NeuralMesh** | **Very high** | **Premium** | **Good — best when AI/Spark are collocated** | **Medium** |
| Ceph | Medium | Low (commodity) | Acceptable for batch | High (admin expertise) |

---

## Findings

### 1. Weka's benchmark is directionally valid but commercially motivated

The TPC-DS brief correctly identifies that small Parquet files degrade performance on S3 and traditional NFS. The benchmark is not independently reproducible — the PDF is gated, competitor configurations are not disclosed, and no third-party validation exists. The scenario benchmarked (1 MB Parquet files) represents a worst-case tuning failure, not a properly configured enterprise Spark deployment. A well-operated Spark environment using file compaction would substantially close the gap with a cheaper storage option.

### 2. Shuffle is the dominant bottleneck — not input/output storage

For join-heavy and aggregation-heavy Spark workloads (the majority of enterprise analytics), shuffle performance is the binding constraint. Shuffle uses local executor disks, not the distributed storage layer. Investing in local NVMe SSDs on executor nodes yields more performance per dollar than upgrading shared storage to Weka for these workloads. Weka's advantage only materializes when input reads (scan-heavy, read-intensive, large Parquet datasets with high concurrency) dominate the job profile.

### 3. IBM Storage Scale is the strongest competitor for on-premise Spark

IBM Storage Scale (formerly GPFS/Spectrum Scale) is a mature, POSIX-compliant parallel filesystem with native HDFS API compatibility. It allows Spark to interact with it exactly as with HDFS, preserving data locality patterns, and it has a 17-year track record in enterprise HPC. It holds 19% market share vs Weka's growing but undisclosed share. For purely Spark-driven enterprises without AI/ML colocation needs, IBM Storage Scale is the more defensible choice with better Spark ecosystem integration.

### 4. VAST Data is the strongest alternative if a modern all-flash platform is preferred

VAST Data's DASE architecture provides unified NFS, S3, and SMB access from a single namespace at high throughput (>140 GB/s measured, vs Weka's >600 GB/s claimed). The unified namespace means Spark, Python-based ML tools, and object-store-native tools can all read the same data without copy. VAST is gaining enterprise traction rapidly ($200M ARR, CoreWeave partnership). It is a compelling alternative to Weka for mixed analytics and AI environments.

### 5. Weka makes most sense in a narrow enterprise profile

Weka delivers its full value when:
- Spark is collocated with GPU AI/ML training workloads on the same cluster
- The workload is read-scan-intensive with many small files and no opportunity for compaction
- Budget allows premium storage and the operational cost of the proprietary client layer
- The organization needs ultra-low latency for interactive queries on large datasets (Spark SQL, BI tools hitting Spark directly)

Weka is a poor fit when:
- Spark workloads are primarily shuffle-heavy (ETL, join-heavy pipelines)
- The organization has existing HDFS, IBM Spectrum Scale, or PowerScale infrastructure
- Cost per TB is a constraint
- Teams lack HPC storage operational expertise
- Portability and avoiding vendor lock-in are priorities

### 6. Cloud-native alternative often wins on total cost

For enterprises not committed to on-premise, Amazon EMR with optimized Spark + Iceberg on S3 delivers 4.3x the performance of open-source Spark on standard S3, at a fraction of the operational overhead of managing a Weka cluster. For Spark specifically, cloud-managed services sidestep the entire storage architecture question.

---

## References

- [Weka Spark TPC-DS Performance Brief (vendor, gated)](../resources/weka/spark-tpcds-performance-brief.summary.md)
- [Weka NeuralMesh Architecture](../resources/weka/neuralmesh-architecture.summary.md)
- [Apache Spark Official Hardware Provisioning](../resources/spark/hardware-provisioning-official.summary.md)
- [Storage Comparison: Ceph vs VAST Data vs Weka](../resources/storage/ai-storage-ceph-vast-weka-comparison.summary.md)
- [Enterprise Storage Market Landscape 2025](../resources/storage/enterprise-storage-market-comparison.summary.md)

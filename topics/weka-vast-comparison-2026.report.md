---
title: Weka NeuralMesh vs VAST Data — Lakehouse Storage Comparison, On-Premise 2026
date: 2026-05-21
status: complete
components: [weka, vast, spark, trino]
constraints:
  - deployment: on-premise
  - compute-engines: [spark, trino]
  - focus: physical storage service layer only
  - open-source: false  ← both candidates are proprietary; this report extends the open-source series
related-reports:
  - weka-lakehouse-2026.report.md
  - spark-trino-lakehouse-storage-service.report.md
---

## Overview

This report compares Weka NeuralMesh and VAST Data as storage backends for an on-premise Spark + Trino lakehouse in 2026. Both are proprietary, AI/HPC-first, all-NVMe storage systems that enter lakehouse evaluation only when maximum performance is required alongside AI/ML workloads or when open-source alternatives (Ceph, Apache Ozone) are ruled out. The open-source comparison report ([spark-trino-lakehouse-storage-service](spark-trino-lakehouse-storage-service.report.md)) eliminated both on proprietary grounds; this report answers: *if you have decided to evaluate a premium proprietary option, which one is better suited for a Spark + Trino lakehouse?*

**Verdict: VAST Data is the stronger choice for a Spark + Trino lakehouse deployment.** VAST has documented Trino integration (two modes: S3 DataStore and native DataBase connector), confirmed Spark + Parquet + Iceberg support, a 2024 Gartner Magic Quadrant Leader position, and an active product roadmap that explicitly includes data analytics workloads. Weka has zero documented Trino integration, no lakehouse-specific product roadmap items, and a smaller customer base with recent leadership instability. Both products are AI/HPC-first and carry the same fundamental risk: neither vendor optimizes for pure SQL analytics, and neither integration has been independently validated at lakehouse production scale.

---

## Evidence Quality

| Source | File | Tier | Accessible |
|---|---|---|---|
| VAST Platform Whitepaper | [link](../resources/vast/vast-platform-whitepaper.summary.md) | vendor | Yes |
| VAST Trino Integration KB | [link](../resources/vast/vast-trino-integration-kb.summary.md) | official | Yes |
| VAST Trino Workloads Blog | [link](../resources/vast/vast-trino-workloads-blog.summary.md) | vendor | Yes |
| VAST Lakehouse Strategy (TechTarget) | [link](../resources/vast/vast-lakehouse-strategy-techtarget.summary.md) | press | Yes |
| VAST Gartner MQ 2024 (Blocks & Files) | [link](../resources/vast/vast-gartner-mq-2024-blocksandfiles.summary.md) | press | Yes |
| VAST Business Context (datagravity.dev) | [link](../resources/vast/vast-business-context-datagravity.summary.md) | press | Yes |
| VAST vs Weka Memory Bottleneck (Blocks & Files) | [link](../resources/vast/vast-weka-memory-bottleneck-blocksandfiles.summary.md) | press | Yes |
| Storage Comparison: Ceph vs VAST vs Weka (WhiteFiber) | [link](../resources/storage/ai-storage-ceph-vast-weka-comparison.summary.md) | press | Yes |
| Weka NeuralMesh Architecture (prior research) | [link](../resources/weka/neuralmesh-architecture.summary.md) | vendor | Yes |
| Weka S3 Protocol Docs (prior research) | [link](../resources/weka/weka-s3-protocol-docs.summary.md) | official | Yes |
| Weka Business Context 2025 (prior research) | [link](../resources/weka/weka-business-context-blocksandfiles-2025.summary.md) | press | Yes |
| Trino S3 Native Client Docs (prior research) | [link](../resources/weka/trino-s3-native-client-docs.summary.md) | official | Yes |

**Gaps and confidence limits:**

1. **No independent head-to-head benchmark for analytics**: No third-party benchmark comparing VAST DataStore vs Weka S3 for SQL analytics (TPC-DS, Spark query throughput) was found. All performance claims are vendor-sourced.
2. **VAST TPC-DS benchmark is vendor-only and gated**: The VAST DataBase white paper with TPC-DS results is behind a form gate; methodology, Trino/Spark version, and configuration are not independently verifiable.
3. **VAST DataStore S3 path-style compatibility**: Confirmed as supported (`hive.s3.path-style-access=true`) but not explicitly tested by the Trino project — only AWS S3 and MinIO are Trino-certified S3 targets.
4. **Weka S3 Trino compatibility**: Zero documentation from either Weka or Trino project. Must be treated as untested configuration.
5. **Gartner MQ for Weka**: Weka's inclusion or exclusion from the 2024 Gartner Magic Quadrant could not be confirmed from available sources — the Blocks & Files MQ article does not mention Weka.
6. **Write performance (VAST)**: User reviews report VAST write performance at less than half of read performance — relevant for Spark ingestion and compaction workloads. This is practitioner-sourced (PeerSpot), not independently benchmarked.

---

## Architecture Comparison

### Weka NeuralMesh

Software-defined parallel filesystem on NVMe SSDs. Runs on standard x86 bare-metal servers. POSIX-native at the WekaFS layer; S3 and NFS are gateway protocols layered on top. Metadata is distributed across all cluster nodes. Minimum: 8 nodes for on-premise bare-metal. Requires proprietary kernel module on every Spark executor/Trino worker for POSIX access; S3 and NFS are clientless. [Source](../resources/weka/neuralmesh-architecture.summary.md)

### VAST Data (DASE)

Disaggregated Shared Everything architecture. Stateless CNodes (compute) + DBoxes (NVMe SSD storage enclosures). All CNodes access all DBoxes simultaneously over NVMe-over-Fabrics. NFS, SMB, S3, NVMe-over-TCP, and Kafka are all native peers. No translation layer between protocols. Minimum: single DBox for basic deployment; 8 DBoxes for HA. No client installation required on Spark executors or Trino workers for any access mode. [Source](../resources/vast/vast-platform-whitepaper.summary.md)

**Key architectural difference**: VAST's DASE avoids any notion of "primary" vs "gateway" protocol — S3 is native, not a gateway bolted onto a filesystem. Weka's S3 is explicitly an "additional protocol" gateway on top of WekaFS. For a lakehouse workload that uses S3 as its primary interface, VAST's architecture is more natural.

---

## Trino Integration

### VAST — Two documented integration modes

**Mode 1: VAST DataStore + Hive/Iceberg connector (S3)**
- Standard Trino `hive.s3.path-style-access=true` configuration with VAST endpoint
- No proprietary software on Trino workers
- Works with Iceberg-formatted tables stored in VAST DataStore as Parquet files
- Configuration identical to MinIO or Ceph RGW setup
- [Source: Trino Workloads Blog](../resources/vast/vast-trino-workloads-blog.summary.md)

**Mode 2: VAST DataBase native connector (VastDB)**
- Proprietary Trino plugin from VAST GitHub (`vastdataorg/trino-vast`)
- Listed on Trino's official ecosystem page as a community connector
- Query pushdown to VAST DataBase; 500x less data transfer vs full scan (vendor benchmark, internal comparison only — not against Ceph/Ozone)
- Version must match Trino version exactly; requires maintenance on every Trino upgrade
- Data must be imported from S3 into VastDB before the native connector applies
- NOT compatible with Iceberg table format — VastDB is VAST's own table format
- [Source: Trino KB](../resources/vast/vast-trino-integration-kb.summary.md)

**Caveat**: Mode 1 (DataStore S3) is analogous to Weka S3. The Trino project only officially tests AWS S3 and MinIO. VAST DataStore's S3 API compatibility is documented and path-style access is confirmed, but runtime edge cases (multipart, ETag handling, LIST pagination) are not independently verified at scale.

### Weka — No documented integration

Zero documentation from Weka or Trino project covering Trino + Weka S3. Not listed on Trino's ecosystem page. Any Trino + Weka deployment is a customer-owned integration with no vendor support baseline. [Source: Weka Trino gap finding](../resources/weka/trino-s3-native-client-docs.summary.md)

---

## Spark Integration

Both VAST DataStore and Weka S3 support Spark via the standard S3A connector (`s3a://` with endpoint override and `path.style.access=true`). No meaningful difference at the Spark layer for S3-based access.

VAST DataBase additionally supports native Spark integration with Parquet import. Weka has no equivalent native database layer.

For POSIX access (highest throughput): Weka requires proprietary WekaFS client on every executor. VAST NFS (NFSoRDMA) is clientless and provides RDMA acceleration — a meaningful operational advantage.

---

## Benchmarks

| Benchmark | VAST | Weka | Source | Confidence |
|---|---|---|---|---|
| Peak throughput | >140 GB/s (vendor claim) | >600 GB/s (vendor claim) | WhiteFiber comparison | LOW — vendor claims, no methodology |
| SPECstorage 2025 (HPC workloads) | Not found | No. 1 across all 5 workloads (123 GB/s EDA, etc.) | spec.org (verified) | HIGH for Weka HPC; not applicable to lakehouse |
| TPC-DS (analytics) | 25% faster than Iceberg (vendor, gated) | Gated brief; no public numbers | Vendor benchmarks | LOW — vendor-only, gated, no methodology |
| NVMe IOPS | Not published | 5M+ IOPS (vendor claim) | Vendor | LOW |
| Write vs read ratio | Write < 50% of read (practitioner reports) | Not reported | PeerSpot | MEDIUM — practitioner source |

**Staleness**: Weka's only independently verified benchmark (SPECstorage 2025) uses HPC workloads (EDA, AI_IMAGE, GENOMICS) — not applicable to SQL analytics. No independently verified 2024–2026 benchmark for either VAST or Weka for lakehouse/SQL analytics workloads exists.

---

## Findings

### 1. VAST has documented Trino integration; Weka has none

Evidence: [VAST KB](../resources/vast/vast-trino-integration-kb.summary.md), [VAST Blog](../resources/vast/vast-trino-workloads-blog.summary.md), [Trino docs](../resources/weka/trino-s3-native-client-docs.summary.md). Confidence: **HIGH**.

VAST provides both an S3-compatible path (DataStore + Hive connector) and a native VastDB connector (community plugin on Trino's ecosystem page). Weka has zero documentation from either Weka or the Trino project. For a Spark + Trino lakehouse deployment, this gap requires enterprises choosing Weka to own the entire integration validation themselves with no vendor baseline.

### 2. VAST's native DataBase connector is a different product category from an Iceberg lakehouse

Evidence: [VAST KB](../resources/vast/vast-trino-integration-kb.summary.md), [VAST Blog](../resources/vast/vast-trino-workloads-blog.summary.md). Confidence: **HIGH**.

The VAST DataBase native connector is not for Iceberg tables in object storage — it is for VastDB, VAST's proprietary database format. Enterprises choosing the native connector path must migrate data from S3 into VastDB, which locks data into VAST's format. This eliminates Iceberg's portability guarantee. For an open standard Iceberg lakehouse on VAST, the DataStore S3 path (Mode 1) is the correct choice — and that path is effectively equivalent to using Ceph or Ozone as the storage backend.

### 3. VAST is a 2024 Gartner Magic Quadrant Leader; Weka is absent from the MQ

Evidence: [Blocks & Files MQ coverage](../resources/vast/vast-gartner-mq-2024-blocksandfiles.summary.md). Confidence: **MEDIUM** (Weka absence is based on non-mention, not explicit exclusion statement).

VAST moved from Challenger to Leader in the 2024 Gartner Magic Quadrant for File and Object Storage Platforms. The evaluation criterion emphasizes unified file-and-object platforms — a profile VAST fits well with its NFS+S3+SMB native protocol support. Weka was not mentioned in Blocks & Files' coverage of the MQ. For enterprise procurement teams using Gartner criteria, VAST has a meaningful evaluation advantage.

### 4. VAST is more financially stable and analytically invested than Weka in 2025–2026

Evidence: [datagravity.dev](../resources/vast/vast-business-context-datagravity.summary.md), [Weka Business Context](../resources/weka/weka-business-context-blocksandfiles-2025.summary.md). Confidence: **HIGH** (from press sources).

VAST: $9.1B valuation, $200M+ ARR, 3.3x YoY growth, positive cash flow 12 consecutive quarters, 107+ customers including Fortune 1000 and GPU cloud providers, NVIDIA SuperPOD certification. Weka: "raft of executive departures" in early 2025 (Blocks & Files), 57 customers, primarily HPC/genomics/AI focus with no announced analytics product roadmap. The relative business stability favors VAST for multi-year enterprise commitments.

### 5. Both products are AI/HPC-first; neither optimizes for pure SQL analytics

Evidence: [VAST vs Weka memory bottlenecks](../resources/vast/vast-weka-memory-bottleneck-blocksandfiles.summary.md), [Weka Business Context](../resources/weka/weka-business-context-blocksandfiles-2025.summary.md), [TechTarget VAST lakehouse](../resources/vast/vast-lakehouse-strategy-techtarget.summary.md). Confidence: **HIGH**.

Both VAST and Weka's 2025 product investments target AI inference (token loading, GPU memory optimization) — not SQL analytics or Iceberg table operations. VAST DataBase's analytics story targets VAST's own proprietary format, not the open Iceberg ecosystem. The analyst quoted in TechTarget noted VAST "needs to be two times better" to compete in mature data management markets. Neither vendor will provide enterprise analytics support comparable to Cloudera (Ozone) or Red Hat (Ceph).

### 6. VAST write performance asymmetry is a risk for Spark compaction and ingestion

Evidence: PeerSpot user reviews. Confidence: **MEDIUM** (practitioner-sourced; not independently benchmarked).

Practitioner reviews report VAST write performance at less than half of read performance. Spark lakehouse workloads are write-intensive: initial ingestion, compaction (`rewrite_data_files`), and streaming micro-batch writes all generate sustained write pressure. Weka has no equivalent write performance complaint in the literature, though this may reflect Weka's smaller practitioner review base rather than a genuine advantage.

---

## Head-to-Head Summary

| Dimension | VAST Data | Weka NeuralMesh |
|---|---|---|
| **Architecture** | DASE: native S3/NFS/SMB, stateless compute | NeuralMesh: parallel filesystem, S3 is gateway |
| **S3 native vs gateway** | S3 is a native protocol peer | S3 is an "additional protocol" on WekaFS |
| **Trino integration** | Two documented modes (S3 + native connector) | Zero documentation; untested configuration |
| **Spark S3A** | Standard; identical to Ceph/MinIO config | Standard; identical to Ceph/MinIO config |
| **Iceberg support** | DataStore: full S3-based Iceberg | S3 gateway: Iceberg should work; not tested |
| **Client on compute nodes** | Not required for any access mode | Required for WekaFS POSIX (highest perf only) |
| **Analytics roadmap** | DataBase targets Spark/Trino/Parquet | No announced analytics roadmap |
| **Lakehouse references** | Dremio partnership; DataBase positioning | None found publicly |
| **Peak throughput claim** | >140 GB/s | >600 GB/s |
| **SPECstorage 2025** | Not found | No. 1 all 5 HPC workloads (verified) |
| **TPC-DS benchmark** | 25% faster than Iceberg (vendor, gated) | Gated; inaccessible |
| **Write performance** | Reported asymmetry (< 50% of read) | Not reported |
| **Minimum cluster (HA)** | 8 DBoxes | 8 nodes |
| **Pricing tier** | Premium; avg $1.2M new customer | Premium; opaque pricing |
| **Gartner MQ 2024** | **Leader** (File & Object Storage) | Not mentioned |
| **Gartner Peer Insights** | 4.9/5, 98% recommend | 4.9/5, 119 reviews (AI/HPC users) |
| **Business stability** | $9.1B, 3.3x growth, positive cash flow | Executive departures 2025, smaller scale |
| **License** | Proprietary | Proprietary |
| **Open-source component** | None | None |

---

## Decision Matrix

| Profile | Recommendation | Rationale |
|---|---|---|
| Spark + Trino lakehouse, Iceberg, no AI/ML co-location | **Neither — use Ceph or Apache Ozone** | Both are proprietary, AI-first, and lack verified lakehouse production references |
| Spark + Trino lakehouse + heavy AI/ML GPU training on same cluster | **VAST Data (DataStore + S3 path)** | Documented Trino integration, Gartner Leader, better business stability |
| Maximum raw throughput priority (AI training dominates, analytics secondary) | **Weka** (if SPECstorage HPC numbers matter) | Only verifiable independent benchmark; Weka holds No. 1 HPC SPECstorage 2025 |
| Analytics-first with proprietary database preferred over Iceberg | **VAST DataBase** (with native connector) | Native Trino/Spark pushdown, proprietary but documented; note: not Iceberg |
| Budget-unconstrained, best long-term vendor relationship for mixed HPC+analytics | **VAST Data** | Gartner Leader, larger customer base, more stable business, explicit analytics roadmap |
| Existing HDFS or open-source infrastructure, open-source mandate | **Ceph or Apache Ozone** | Neither VAST nor Weka qualifies |

---

## References

- [VAST Platform Whitepaper](../resources/vast/vast-platform-whitepaper.summary.md)
- [VAST Trino Integration KB](../resources/vast/vast-trino-integration-kb.summary.md)
- [VAST Trino Workloads Blog (vendor)](../resources/vast/vast-trino-workloads-blog.summary.md)
- [VAST Lakehouse Strategy — TechTarget](../resources/vast/vast-lakehouse-strategy-techtarget.summary.md)
- [VAST Gartner MQ 2024 — Blocks & Files](../resources/vast/vast-gartner-mq-2024-blocksandfiles.summary.md)
- [VAST Business Context — datagravity.dev](../resources/vast/vast-business-context-datagravity.summary.md)
- [VAST vs Weka Memory Bottleneck — Blocks & Files](../resources/vast/vast-weka-memory-bottleneck-blocksandfiles.summary.md)
- [Storage Comparison: Ceph vs VAST vs Weka](../resources/storage/ai-storage-ceph-vast-weka-comparison.summary.md)
- [Weka NeuralMesh Architecture](../resources/weka/neuralmesh-architecture.summary.md)
- [Weka S3 Protocol Docs](../resources/weka/weka-s3-protocol-docs.summary.md)
- [Weka Business Context 2025](../resources/weka/weka-business-context-blocksandfiles-2025.summary.md)
- [Trino Native S3 Client Docs](../resources/weka/trino-s3-native-client-docs.summary.md)
- [Prior: Weka Lakehouse 2026](weka-lakehouse-2026.report.md)
- [Prior: Spark + Trino Lakehouse Storage Service](spark-trino-lakehouse-storage-service.report.md)

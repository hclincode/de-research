---
title: Weka NeuralMesh as Lakehouse Storage — On-Premise Enterprise 2026
date: 2026-05-21
status: complete
components: [weka, spark, trino]
constraints:
  - open-source: false  ← proprietary; included at user request as extension to open-source analysis
  - deployment: on-premise
---

## Overview

This report evaluates Weka (weka.io) NeuralMesh as a physical storage service for a Spark + Trino lakehouse deployed on-premise in 2026. Weka is a proprietary, software-defined parallel filesystem built on NVMe SSDs. It is NOT open-source. This report is explicitly scoped to Weka as a storage layer only — table formats (Iceberg, Hudi, Delta Lake) and catalogs (Nessie, Polaris) are mentioned only where they constrain Weka's behaviour.

**Verdict: Weka NeuralMesh is technically capable of serving a Spark + Trino lakehouse through its S3-compatible gateway, but it is a poor strategic fit for most enterprise analytics deployments.** Its product investment, benchmark profile, reviewer base, and pricing model are all oriented toward AI/ML training and HPC workloads. Enterprises building a lakehouse will incur premium storage cost, a significant minimum cluster size (8 nodes), and an untested Trino integration — with limited independent evidence that the performance premium over open-source alternatives (Ceph, Ozone) justifies the expense for SQL analytics workloads specifically.

---

## Evidence Quality

| Source | File | Tier | Accessible |
|---|---|---|---|
| Weka S3 Protocol Documentation | [link](../resources/weka/weka-s3-protocol-docs.summary.md) | official | Yes |
| Weka S3 API Limitations | [link](../resources/weka/weka-s3-limitations-docs.summary.md) | official | Yes |
| Weka Client and Mount Modes | [link](../resources/weka/weka-client-mount-modes-docs.summary.md) | official | Yes |
| Trino Native S3 Client Docs (Trino 481) | [link](../resources/weka/trino-s3-native-client-docs.summary.md) | official | Yes |
| Weka TPC-DS Performance Brief | [link](../resources/weka/spark-tpcds-performance-brief.summary.md) | vendor | No (gated) |
| Weka NeuralMesh Architecture Blog | [link](../resources/weka/neuralmesh-architecture.summary.md) | vendor | Yes |
| Weka Gartner Customers' Choice 2025 | [link](../resources/weka/weka-gartner-customers-choice-2025.summary.md) | vendor | Yes |
| Weka + HPE SPECstorage 2025 (Blocks & Files) | [link](../resources/weka/weka-specstorage-hpe-2025.summary.md) | press | Yes |
| Weka MinIO License Dispute 2023 (Blocks & Files) | [link](../resources/weka/weka-minio-license-dispute-2023.summary.md) | press | Yes |
| Weka NeuralMesh Axon 2025 (HPCwire) | [link](../resources/weka/weka-neuralmesh-axon-2025.summary.md) | press | Yes |
| Weka Business Context 2025 (Blocks & Files) | [link](../resources/weka/weka-business-context-blocksandfiles-2025.summary.md) | press | Yes |
| Storage Comparison: Ceph vs VAST vs Weka | [link](../resources/storage/ai-storage-ceph-vast-weka-comparison.summary.md) | press | Yes |
| Enterprise Storage Market 2025 | [link](../resources/storage/enterprise-storage-market-comparison.summary.md) | press | Yes |

**Gaps and confidence limits:**

1. **Gated benchmark**: The Weka TPC-DS Performance Brief (the only Weka benchmark explicitly covering Spark SQL analytics) remains gated. No public numbers, no competitor configuration, no independent reproduction. All Weka-specific Spark/Trino performance claims are LOW confidence.
2. **No Trino + Weka integration documentation exists** from either Weka or the Trino project. The Trino project explicitly states only AWS S3 and MinIO are tested as S3-compatible targets. Any Trino + Weka deployment is an untested configuration.
3. **SPECstorage workload mismatch**: The only publicly verifiable 2025 Weka benchmark (SPECstorage, spec.org) covers AI_IMAGE, EDA, GENOMICS, SWBUILD, VDA workloads — none of which are SQL analytics or lakehouse patterns. These results cannot be directly extrapolated to TPC-DS performance.
4. **Reviewer base skew**: Weka's 4.9/5 Gartner Peer Insights rating (119 reviews) comes overwhelmingly from AI/HPC users. There is no evidence of significant enterprise analytics/lakehouse deployments in the review base.
5. **Pricing**: No public pricing. All cost claims are based on market positioning (premium tier) and indirect reports.
6. **MinIO license dispute (2023)**: Resolved at the product level (Weka built its own S3 implementation) but not through legal adjudication. Some compliance risk remains unresolved.
7. **Leadership instability**: Blocks & Files reported a "raft of executive departures" in early 2025. Impact on product roadmap and support continuity is unknown.

---

## Components

### `weka`

Weka NeuralMesh is a software-defined parallel filesystem that runs on standard x86 servers with NVMe SSDs. It exposes three primary access modes: POSIX/WekaFS (requires proprietary client on each compute node), S3-compatible API (clientless for compute nodes), and NFS v3/v4 (clientless for compute nodes). For lakehouse deployments, the S3 gateway is the most practical interface because it avoids mandatory client installation on every Spark executor and Trino worker.

Core architecture characteristics per official documentation ([Client Modes](../resources/weka/weka-client-mount-modes-docs.summary.md), [NeuralMesh Architecture](../resources/weka/neuralmesh-architecture.summary.md)):
- Distributed metadata striped across all cluster nodes — eliminates NameNode-style single-point bottleneck
- POSIX-compliant at the WekaFS layer; S3 is an "additional protocol" gateway layer on top
- Minimum 8-node cluster for on-premise bare-metal deployment
- Similarity-aware deduplication reduces storage footprint for near-duplicate data (AI checkpoints, versioned files); benefit for Parquet data depends on dataset similarity profile
- Vendor-stated peak: >600 GB/s, 5M+ IOPS at sub-millisecond latency (not independently verified for lakehouse workloads)

Primary market is AI/ML training, genomics, EDA, and HPC. Data analytics/lakehouse is a secondary use case not prominently featured in Weka's product positioning, documentation, or benchmark portfolio as of 2026 ([Business Context](../resources/weka/weka-business-context-blocksandfiles-2025.summary.md), [NeuralMesh Axon](../resources/weka/weka-neuralmesh-axon-2025.summary.md)).

### `spark`

Spark accesses Weka via one of three paths:
1. **S3A connector** (`s3a://`) pointing to Weka's S3 gateway — no Weka client needed on executor nodes. Standard Hadoop S3A configuration with `fs.s3a.endpoint` and `fs.s3a.path.style.access=true`.
2. **WekaFS POSIX mount** — requires Weka client installed on every executor node; highest throughput, lowest latency for read-heavy scans.
3. **NFS mount** — no Weka client; moderate performance; simplest operationally.

For shuffle performance: Spark shuffle uses local executor disks, not the distributed storage layer. Weka's storage performance advantage does not apply to shuffle-heavy workloads (join-heavy ETL, aggregation pipelines).

### `trino`

Trino accesses Weka exclusively via the S3 gateway — Trino has no POSIX or WekaFS connector. Configuration uses the native S3 client (`fs.s3.enabled=true`, `s3.path-style-access=true`, custom `s3.endpoint`) per [Trino S3 docs](../resources/weka/trino-s3-native-client-docs.summary.md). No proprietary Weka software is required on Trino worker nodes.

**Critical limitation**: The Trino project's official documentation explicitly states that only AWS S3 and MinIO are tested as S3-compatible storage targets. Weka S3 is untested. This means:
- S3 API edge cases (specific multipart behaviors, ETag handling, LIST pagination) may behave differently than expected
- Any behavioral gap will manifest as a runtime error, not a configuration error
- Enterprises must validate their specific Trino + Iceberg query patterns against Weka S3 before production deployment

---

## Findings

### 1. Weka exposes S3-compatible API that satisfies Trino's configuration requirements

Evidence: [Weka S3 Protocol Docs](../resources/weka/weka-s3-protocol-docs.summary.md), [Trino S3 Docs](../resources/weka/trino-s3-native-client-docs.summary.md). Confidence: **MEDIUM**.

Weka's S3 gateway supports both path-style and virtual-hosted-style access. Trino's native S3 client requires `fs.s3.enabled=true` and `s3.path-style-access=true`, both of which are compatible with Weka S3. Authentication uses AWS-style access key / secret key, supported by Weka's S3 user model. The configuration is mechanically straightforward — the same pattern used for MinIO. Confidence is MEDIUM (not HIGH) because the Trino project has not tested or certified Weka S3 compatibility, and S3 API edge cases may require workarounds. At least one test deployment validating Iceberg reads and writes is required before treating this as production-ready.

### 2. Weka provides POSIX and NFS interfaces in addition to S3, but POSIX requires proprietary client on every compute node

Evidence: [Weka Client Mount Modes](../resources/weka/weka-client-mount-modes-docs.summary.md). Confidence: **HIGH**.

Official documentation confirms: WekaFS POSIX access requires a proprietary kernel module or FUSE driver on every compute node (Spark executor, Trino worker). S3 and NFS access do NOT require the client on compute nodes. For a lakehouse deployment that uses Trino (S3-only access) and Spark (via S3A or NFS), the client installation dependency can be avoided entirely. However, if POSIX performance is required for Spark (for maximum throughput), every executor node must install and maintain the Weka client — adding OS-level dependency management, kernel version compatibility constraints, and operational overhead to cluster provisioning and upgrades.

### 3. No documented Trino + Weka integration exists from either party

Evidence: [Trino S3 Docs](../resources/weka/trino-s3-native-client-docs.summary.md), [Weka Docs site](https://docs.weka.io). Confidence: **HIGH**.

Searches across Weka's documentation, Trino's documentation, and trade press found zero Trino + Weka integration guides, blog posts, or reference architectures. Trino explicitly tests only AWS S3 and MinIO. Weka's documentation covers Spark integration (through S3 and POSIX) but does not mention Trino. This is not evidence that the integration does not work — it means enterprises must own the integration validation themselves. Compare this to Ceph (RGW), which has documented Trino compatibility because MinIO's API surface (which Trino tests) closely matches Ceph RGW's.

### 4. Weka handles small-file workloads natively at the filesystem layer; S3 gateway behavior for billions of objects is less documented

Evidence: [Weka LOSF Whitepaper](https://www.weka.io/wp-content/uploads/files/2022/10/weka-solving-challenge-lots-of-small-files-losf.pdf), [Weka Architecture](../resources/weka/neuralmesh-architecture.summary.md). Confidence: **LOW** (vendor sources only).

Weka's documentation claims distributed metadata striped across all nodes, supporting up to 6.4 trillion files and 6.4 billion files in a single directory. At the POSIX/WekaFS layer, Weka's architecture is explicitly designed to handle the "lots of small files" (LOSF) problem — a real bottleneck for Iceberg manifests (many small JSON/Avro metadata files), Parquet footers, and snapshot files. However, the small-file claim applies at the POSIX layer. For Iceberg over S3, the relevant interface is the S3 gateway — and the S3 gateway's LIST performance for billions of objects (a common pattern for Iceberg snapshot housekeeping and metadata reads) is not independently benchmarked. Confidence is LOW because all claims about small-file performance come from Weka's own documentation, and no independent validation of the S3 gateway's behavior for billions of objects exists in the sources found.

### 5. The only verifiable 2025 benchmark (SPECstorage) covers HPC workloads, not SQL analytics

Evidence: [Weka SPECstorage 2025](../resources/weka/weka-specstorage-hpe-2025.summary.md). Confidence: **MEDIUM** for HPC I/O claims; **LOW** for lakehouse/SQL analytics claims.

In January 2025, Weka and HPE achieved No. 1 rankings across all five SPECstorage Solution 2020 workloads on spec.org (publicly verifiable): EDA_BLENDED at 17,000 job sets, 7.65M ops/sec, 123 GB/s; GENOMICS at 5,600 jobs; AI_IMAGE, SWBUILD, VDA also leading. These are genuine third-party-validated numbers — the only such numbers in this research. However, SPECstorage does not include a SQL analytics or data warehouse workload. The TPC-DS brief remains gated. The inference that "very high HPC I/O performance implies very high lakehouse query throughput" is directionally plausible but not directly validated.

### 6. Weka has no open-core, community, or free tier for on-premise deployment

Evidence: [Weka pricing search], [Weka Architecture](../resources/weka/neuralmesh-architecture.summary.md). Confidence: **HIGH**.

Weka is fully proprietary with no open-source or community tier. On-premise licensing requires a custom enterprise quote with a minimum 8-node cluster. A cloud-based AWS free trial exists but is irrelevant to on-premise deployments. The pricing model is capacity-based subscription (per usable TB) — no list prices are published. Weka has confirmed a minimum cluster size of 8 nodes for bare-metal on-premise, setting a significant minimum capital commitment before any workloads can run. This contrasts sharply with Ceph (zero license cost, 3-node minimum) and Apache Ozone (zero license cost, embedded in existing Hadoop infrastructure).

### 7. Weka's S3 implementation was built partially on MinIO code; a 2023 licensing dispute revealed compliance risks

Evidence: [MinIO License Dispute 2023](../resources/weka/weka-minio-license-dispute-2023.summary.md). Confidence: **HIGH** (the dispute happened; its legal resolution is unresolved).

In March 2023, MinIO formally accused Weka of distributing MinIO AGPL v3 binaries without source disclosure, violating the AGPL license. MinIO revoked all Weka licenses. Weka contested this, asserting it used only Apache 2.0-licensed MinIO code. The dispute was not resolved through litigation. Weka subsequently built its own S3 implementation to reduce this dependency. The practical risk today (2026) is low — Weka's current S3 gateway is documented as its own implementation — but the incident signals a historical pattern of open-source license hygiene issues that enterprises with strict legal/compliance requirements should be aware of.

### 8. Weka's product direction is AI-first, not analytics-first — lakehouse is not on the primary roadmap

Evidence: [NeuralMesh Axon 2025](../resources/weka/weka-neuralmesh-axon-2025.summary.md), [Business Context 2025](../resources/weka/weka-business-context-blocksandfiles-2025.summary.md). Confidence: **HIGH**.

Weka's 2025 product launches (NeuralMesh Axon, WEKApod) and named adopters (Cohere, CoreWeave, NVIDIA, Memorial Sloan Kettering) are all AI/HPC-focused. Lakehouse, Trino, or SQL analytics are not mentioned in any Weka press release, product announcement, or technical brief from 2024–2026. This is not evidence the product cannot serve lakehouse workloads, but it does mean: (a) lakehouse-specific features and integrations will not be on Weka's roadmap priority list, (b) support teams will have limited experience with Trino + Iceberg + Weka production deployments, and (c) the integration burden falls entirely on the customer.

### 9. Weka compared to Ceph and Apache Ozone for lakehouse workloads

Evidence: [Storage Comparison](../resources/storage/ai-storage-ceph-vast-weka-comparison.summary.md), [Enterprise Storage Market 2025](../resources/storage/enterprise-storage-market-comparison.summary.md). Confidence: **MEDIUM** (comparisons are high-level; no head-to-head lakehouse benchmark exists).

| Dimension | Weka NeuralMesh | Ceph (RGW) | Apache Ozone |
|---|---|---|---|
| License | Proprietary (subscription) | Open-source (LGPL/MIT) | Open-source (Apache 2.0) |
| S3 API completeness | Good; own implementation | Excellent; AWS S3-compatible, Trino-tested via MinIO proximity | Good; native S3 gateway |
| Trino tested | No | Via MinIO compatibility | Limited; community-tested |
| Minimum cluster | 8 nodes (on-prem) | 3 nodes (production: 5+) | 3 nodes (production: 5+) |
| Metadata at scale | Distributed (vendor claim: 6.4T files) | Distributed (RADOS); well-documented at PB scale | Optimized for Hadoop metadata; improving |
| Cost | Premium (opaque pricing) | Very low (commodity HW) | Zero license (HW only) |
| Lakehouse references | None found publicly | IBM Ceph for watsonx.data (2024); Red Hat Ceph on OpenShift | Cloudera-backed; growing |
| Admin complexity | Medium (proprietary tooling) | High (Ceph expertise required) | Medium (Hadoop-familiar) |
| AI/ML co-location | Excellent (primary use case) | Poor (latency, POSIX) | Poor |

Ceph and Ozone have a meaningful advantage over Weka for pure analytics/lakehouse use cases: documented Trino compatibility, zero license cost, and larger community of practitioners with lakehouse experience. Weka's advantage is raw I/O performance at scale — which matters when AI/ML and lakehouse workloads coexist on the same infrastructure.

---

## Decision Matrix

| Profile | Recommendation | Eliminated options and why |
|---|---|---|
| Pure Spark + Trino lakehouse, no AI/ML colocation, cost-sensitive | Ceph (RGW) or Apache Ozone | Weka: premium cost, no Trino validation, analytics not on roadmap |
| Spark + Trino lakehouse with GPU AI/ML training on same cluster | Weka NeuralMesh (with S3 gateway for Trino) | Ceph: insufficient IOPS for AI training; Ozone: same |
| Existing Hadoop/HDFS infrastructure migrating to lakehouse | Apache Ozone or stay on HDFS | Weka: requires full new cluster (8 nodes min), proprietary |
| Budget-unconstrained, maximum performance, mixed HPC+analytics | Weka or VAST Data | Both viable; VAST has more documented Trino/S3 compatibility |
| Greenfield on-premise lakehouse, open-source mandate | Ceph or Apache Ozone | Weka: proprietary, violates constraint |

---

## References

- [Weka S3 Protocol Docs](../resources/weka/weka-s3-protocol-docs.summary.md)
- [Weka S3 API Limitations](../resources/weka/weka-s3-limitations-docs.summary.md)
- [Weka Client Mount Modes](../resources/weka/weka-client-mount-modes-docs.summary.md)
- [Trino Native S3 Client Docs (Trino 481)](../resources/weka/trino-s3-native-client-docs.summary.md)
- [Weka TPC-DS Performance Brief (vendor, gated)](../resources/weka/spark-tpcds-performance-brief.summary.md)
- [Weka NeuralMesh Architecture (vendor)](../resources/weka/neuralmesh-architecture.summary.md)
- [Weka Gartner Customers' Choice 2025 (vendor)](../resources/weka/weka-gartner-customers-choice-2025.summary.md)
- [Weka + HPE SPECstorage 2025 (Blocks & Files)](../resources/weka/weka-specstorage-hpe-2025.summary.md)
- [Weka MinIO License Dispute 2023 (Blocks & Files)](../resources/weka/weka-minio-license-dispute-2023.summary.md)
- [Weka NeuralMesh Axon 2025 (HPCwire)](../resources/weka/weka-neuralmesh-axon-2025.summary.md)
- [Weka Business Context 2025 (Blocks & Files)](../resources/weka/weka-business-context-blocksandfiles-2025.summary.md)
- [Storage Comparison: Ceph vs VAST vs Weka (WhiteFiber)](../resources/storage/ai-storage-ceph-vast-weka-comparison.summary.md)
- [Enterprise Storage Market 2025 (Blocks & Files / HPCwire)](../resources/storage/enterprise-storage-market-comparison.summary.md)

---
title: Ceph (RadosGW) as Lakehouse Storage — On-Premise Enterprise 2026
date: 2026-05-21
status: complete
components: [ceph, spark, trino]
constraints:
  - open-source: true
  - deployment: on-premise
---

## Overview

This report evaluates Ceph (specifically its RadosGW S3-compatible object storage interface) as the physical storage service for an on-premise Spark + Trino lakehouse in 2026. Scope is limited to Ceph as the storage layer; table formats (Iceberg, Hudi, Delta Lake) and catalogs (Nessie, Polaris) are mentioned only where they constrain Ceph's behaviour. The open-source constraint is satisfied (LGPL 2.1). The on-premise constraint eliminates cloud-managed services.

**Verdict: Ceph RadosGW is the strongest open-source, on-premise S3-compatible storage option available in 2026, following MinIO Community Edition's archival in early 2026, and is a viable — though operationally demanding — physical storage layer for a Spark + Trino lakehouse. Teams should plan for significant operational investment: Ceph requires specialized expertise, careful hardware sizing (minimum 25 Gb/s networking, all-NVMe recommended), and deliberate tuning of bucket index sharding and RGW scaling to achieve acceptable performance for the small-file-heavy access patterns that Iceberg metadata imposes.**

## Evidence Quality

| Source | File | Tier | Accessible |
|---|---|---|---|
| Ceph Squid v19.2.0 release notes | [link](../resources/ceph/ceph-squid-v19-release-notes.summary.md) | official | Yes |
| Ceph Tentacle v20.2.0 release notes | [link](../resources/ceph/ceph-tentacle-v20-release-notes.summary.md) | official | Yes |
| Ceph RadosGW S3 API documentation | [link](../resources/ceph/ceph-rgw-s3-api-documentation.summary.md) | official | Yes |
| Ceph hardware recommendations (official docs) | [link](../resources/ceph/ceph-hardware-recommendations-official.summary.md) | official | Yes |
| Ceph RGW dynamic bucket sharding (official docs) | [link](../resources/ceph/ceph-rgw-dynamic-bucket-sharding.summary.md) | official | Yes |
| Ceph FastEC erasure coding (Tentacle) | [link](../resources/ceph/ceph-tentacle-fastec-performance-2025.summary.md) | official | Yes |
| Ceph slow OSD failure modes (official docs) | [link](../resources/ceph/ceph-slow-osd-failure-modes.summary.md) | official | Yes |
| Ceph: A Journey to 1 TiB/s (Clyso, 2024) | [link](../resources/ceph/ceph-journey-to-1tibps-2024.summary.md) | press | Yes |
| Ceph vs. MinIO / open-source landscape (onidel, 2025) | [link](../resources/ceph/ceph-vs-minio-comparison-2025.summary.md) | press | Yes |
| Ceph license and commercial support (TechTarget) | [link](../resources/ceph/ceph-license-commercial-support-2025.summary.md) | press | Yes |
| Trino S3 filesystem documentation | [link](../resources/ceph/trino-s3-filesystem-documentation.summary.md) | official | Yes |
| IBM RGW benchmarking 2025 (Parts 1–3) | [link](../resources/ceph/ceph-rgw-benchmarking-2025-ibm-part1-3.summary.md) | vendor-adjacent | Yes |
| TPC-DS Trino + Ceph S3 Select benchmark (IBM, 2025) | [link](../resources/ceph/ceph-trino-tpcds-s3select-benchmark-2025.summary.md) | vendor-adjacent | Yes |
| IOMETE lakehouse storage evaluation (2025) | [link](../resources/ceph/ceph-evaluating-s3-lakehouse-iomete-2025.summary.md) | vendor-adjacent | Yes |
| Red Hat / Spark on Ceph benchmark (2019, STALE) | [link](../resources/ceph/spark-ceph-performance-analysis.summary.md) | vendor-adjacent | Yes |

**Gaps and confidence limits:**

1. **Staleness gap (partially closed)**: The only dedicated Spark-on-Ceph concurrent workload benchmark (Red Hat, 2019) remains stale. The 2025 IBM RGW benchmarks cover single-engine (RGW-only) object throughput and do not test simultaneous Spark + Trino concurrency. No independent 2024–2026 benchmark of concurrent Spark + Trino on Ceph was found. This gap is explicitly flagged in Finding 3.

2. **Small-object benchmark floor**: The 2025 IBM benchmarks use 64 KiB as the smallest test object. Iceberg manifest files (typically 1–50 KB) and Parquet footer reads (8–32 KB) are smaller. No benchmark with sub-16 KB objects was found for Ceph RadosGW. Confidence in small-object latency claims is MEDIUM.

3. **Trino formal compatibility**: Trino's official documentation explicitly tests only AWS S3 and MinIO. MinIO Community Edition is now archived. Ceph RGW Trino compatibility is community-supported, not formally certified. No independent comprehensive Trino + Ceph integration test report was found.

4. **FastEC RGW benefit unquantified**: Ceph Tentacle's FastEC partial-read improvement for RGW object workloads is stated but not separately benchmarked for object storage. The official blog reports 2–3× improvement for block/file workloads; RGW benefit is described as secondary and unquantified.

5. **Counter-evidence search**: No credible published evidence was found recommending against Ceph RadosGW for lakehouse specifically (as opposed to general operational complexity warnings). Operational complexity concerns are well-documented by independent sources.

## Components

### `ceph`

Ceph is an open-source, software-defined, distributed storage platform (LGPL 2.1) that provides object, block, and file storage from a single cluster. The RADOS (Reliable Autonomic Distributed Object Store) layer is the underlying distributed storage engine. RadosGW (RGW) provides an S3-compatible HTTP interface on top of RADOS.

**Current release**: Ceph Tentacle (v20.2.0, November 2025) is the current stable release. Ceph Squid (v19.2.3, July 2025) is the prior stable with EOL in September 2026. [Tentacle release notes](../resources/ceph/ceph-tentacle-v20-release-notes.summary.md).

**Governance**: Ceph Foundation (Linux Foundation umbrella); multi-vendor contributors including IBM, Bloomberg, DigitalOcean, Intel, Samsung. Production deployments include CERN (50+ PB) and Bloomberg (100+ PB). [License and support](../resources/ceph/ceph-license-commercial-support-2025.summary.md).

### `spark`

Apache Spark connects to Ceph RadosGW via the S3A connector (`hadoop-aws`). The S3A connector uses the AWS SDK's S3 client, which supports path-style access and multipart uploads — both supported by Ceph RGW. The primary performance caveat is that Spark reading Iceberg tables issues many small parallel GetObject requests (manifests, footers); RGW's per-instance connection limit and EC read amplification both limit throughput under high executor concurrency.

### `trino`

Trino connects to Ceph RadosGW via its native S3 file system implementation. Path-style access is configurable via `s3.path-style-access=true`. Trino's official compatibility testing covers only AWS S3 and MinIO; Ceph RGW is user-supported. [Trino S3 docs](../resources/ceph/trino-s3-filesystem-documentation.summary.md). The TPC-DS benchmark with Ceph S3 Select demonstrated 2.5× average speedup when S3 Select predicate pushdown is enabled. [TPC-DS benchmark](../resources/ceph/ceph-trino-tpcds-s3select-benchmark-2025.summary.md).

## Findings

### 1. S3 Compatibility: RadosGW covers the full lakehouse S3 feature set

Evidence: [Ceph RadosGW S3 API docs](../resources/ceph/ceph-rgw-s3-api-documentation.summary.md), [Squid release notes](../resources/ceph/ceph-squid-v19-release-notes.summary.md), [Trino S3 docs](../resources/ceph/trino-s3-filesystem-documentation.summary.md). Confidence: **HIGH**.

Ceph RadosGW supports all S3 features that Spark's S3A connector and Trino's native S3 implementation require for lakehouse operation:

- **Path-style access**: Fully supported; configured via `rgw_dns_name` or zonegroup hostnames. Trino's `s3.path-style-access=true` configuration works without wildcard DNS.
- **Multipart uploads**: Fully supported; Squid added per-part reads via `partNumber` query parameter.
- **Object tagging**: Fully supported (PutObjectTagging, GetObjectTagging, DeleteObjectTagging); usable as Iceberg lifecycle filter predicates.
- **Bucket lifecycle policies**: Supported — Expiration, NoncurrentVersionExpiration, expired delete marker cleanup; filterable by prefix and/or tags; relevant for Iceberg snapshot expiration and orphan file cleanup workflows.
- **Bucket versioning**: Fully supported (enable/suspend). Required by some Iceberg locking strategies.
- **S3 Object Lock**: Supported in compliance and governance modes. As of Squid, `PutObjectLockConfiguration` works on existing versioned buckets.
- **Server-side encryption**: SSE-S3, SSE-KMS (HashiCorp Vault integration), and SSE-C.
- **S3 Select**: Supported; enables predicate pushdown from Trino/Spark into the Ceph cluster, reducing network transfer.
- **IAM APIs**: AWS-compatible IAM (users, groups, roles, policies) via User Accounts feature (Squid+).

**Gap**: Trino formally tests only AWS S3 and MinIO. Ceph RGW compatibility is community-verified, not officially certified. No specific Trino S3 API call has been documented as failing on Ceph RGW.

### 2. Current release (Tentacle) and roadmap

Evidence: [Tentacle release notes](../resources/ceph/ceph-tentacle-v20-release-notes.summary.md), [Squid release notes](../resources/ceph/ceph-squid-v19-release-notes.summary.md). Confidence: **HIGH**.

- **Ceph Tentacle (v20.2.0, November 2025)** is the current stable release.
- **Ceph Squid (v19.2.3, July 2025)** is in active maintenance with EOL September 2026.
- Teams deploying new clusters in 2026 should target Tentacle for FastEC support.
- Teams on Squid have approximately 16 months of supported life from the research date; plan for Tentacle upgrade before EOL.

Tentacle RGW improvements relevant to lakehouse:
- Faster OMAP iteration → faster bucket listing (critical for Iceberg table discovery and manifest reads)
- `GetObjectAttributes` S3 API (used by some AWS SDK integrations)
- `LastModified` timestamps truncated to the second (AWS S3 compatibility fix)
- OAuth 2.0 integration, multi-site automation, tiering, lifecycle, and notifications improvements

### 3. Performance under concurrent Spark + Trino access

Evidence: [IBM RGW benchmarks 2025](../resources/ceph/ceph-rgw-benchmarking-2025-ibm-part1-3.summary.md), [TPC-DS Trino benchmark 2025](../resources/ceph/ceph-trino-tpcds-s3select-benchmark-2025.summary.md), [Red Hat Spark/Ceph 2019 STALE](../resources/ceph/spark-ceph-performance-analysis.summary.md), [IOMETE evaluation](../resources/ceph/ceph-evaluating-s3-lakehouse-iomete-2025.summary.md). Confidence: **MEDIUM**.

**What is known (2025 data)**:
- All-NVMe, 12-node cluster with 100 GE networking: 391K GET IOPS at 64 KiB objects (~24.4 GiB/s), 2 ms average latency. This is the highest concurrency Ceph RGW benchmark in the current evidence set.
- Trino + Ceph S3 Select: 2.5× average TPC-DS speedup, up to 9× on individual queries, 144 TB less network transfer across the query set. (Vendor-adjacent IBM result; independent corroboration not found.)
- Large-object (≥32 MiB) throughput scales near-linearly: 115 GiB/s GET, 65 GiB/s PUT at 12 nodes.

**Why confidence is MEDIUM (not HIGH)**:
- No 2024–2026 benchmark tests simultaneous Spark + Trino reads on the same Ceph cluster.
- The 2019 Red Hat benchmark found that under 10-concurrent-user Spark workloads with EC 4:2, Ceph was ~90% slower than HDFS 3× for mixed workloads (benchmark is STALE but the architectural reason — EC read amplification — remains relevant).
- IBM benchmarks are vendor-adjacent; independent reproduction not found.
- The 32-concurrent-connections per RGW instance limit means high Spark executor counts (100+ executors) saturate individual RGW instances; operators must deploy multiple RGW instances horizontally.

**Architectural implication for concurrent access**:
Spark (hundreds of executors) and Trino (dozens of threads) will issue concurrent object reads simultaneously. Ceph's RADOS layer handles this via PG-level parallelism, but EC pools incur read amplification (a 4:2 EC read touches 4 OSDs per object; 2+2 EC touches 2 data shards). With FastEC in Tentacle, partial reads reduce this amplification for sub-stripe reads (Iceberg manifests, Parquet footers). The cluster must be sized with enough RGW instances and OSD throughput to absorb simultaneous load from both engines.

### 4. Small-file read patterns (Iceberg manifests, Parquet footers)

Evidence: [FastEC blog 2025](../resources/ceph/ceph-tentacle-fastec-performance-2025.summary.md), [IBM benchmarks 2025](../resources/ceph/ceph-rgw-benchmarking-2025-ibm-part1-3.summary.md), [RGW dynamic resharding](../resources/ceph/ceph-rgw-dynamic-bucket-sharding.summary.md). Confidence: **MEDIUM**.

Iceberg imposes a characteristic small-file read pattern: every query reads manifest list files (~1–5 KB), manifest files (~5–100 KB), and Parquet file footers (~8–64 KB) before reading any actual data. These are random, highly concurrent, sub-64 KB object reads — the worst case for erasure-coded storage and RADOS bucket index overhead.

**EC read amplification (the core problem)**:
- In a 4:2 EC pool, a read of any part of an object requires reading 4 OSD shards.
- For a 10 KB Iceberg manifest, the actual I/O on the OSDs is 4× larger than necessary.
- In a 2+2 EC pool: 2 data shard reads; better but still worse than replication.
- FastEC (Tentacle) adds partial reads, directly reducing this amplification for sub-stripe reads. Quantified benefit for RGW object workloads: not yet independently benchmarked.

**Mitigation options**:
1. Use 3× replication pools for the Iceberg metadata prefix (manifest lists, manifests, statistics files) and EC pools for large data files. Ceph supports per-pool policies; Iceberg prefix-based layout makes this feasible.
2. Enable Tentacle's FastEC for EC pools.
3. Enable S3 Select pushdown to reduce round-trips for query-selective access patterns.
4. Pre-shard bucket indexes for tables with millions of objects (>100,000 objects per shard threshold).

**Bucket index as a bottleneck**:
A lakehouse bucket holding millions of Parquet files across many Iceberg table partitions will hit the bucket index sharding threshold. Dynamic resharding handles this automatically but causes brief write stalls. Pre-sharding at bucket creation is recommended for high-churn tables. [RGW dynamic resharding docs](../resources/ceph/ceph-rgw-dynamic-bucket-sharding.summary.md).

### 5. Operational complexity: OSD tuning, BlueStore, NVMe vs. HDD

Evidence: [Hardware recommendations (official)](../resources/ceph/ceph-hardware-recommendations-official.summary.md), [1 TiB/s case study (press, 2024)](../resources/ceph/ceph-journey-to-1tibps-2024.summary.md). Confidence: **HIGH**.

Ceph requires substantially more operational expertise than single-purpose object stores (e.g., the now-archived MinIO Community Edition). Key operational requirements:

**Network**:
- 10 Gb/s Ethernet is the absolute minimum (official); 25 Gb/s or 100 Gb/s for production lakehouse throughput.
- Mandatory separation: public network (client-to-OSD) and cluster network (OSD-to-OSD replication/recovery). Without separation, recovery storms degrade client I/O.

**BlueStore / NVMe**:
- BlueStore is the only current backend (FileStore removed).
- All-NVMe is strongly preferred for lakehouse (low read latency critical for Iceberg small-file pattern).
- OSD count: 2 OSDs per NVMe device for BlueStore; 14–16 CPU threads per OSD is the saturation point; plan CPU count accordingly.
- Memory: 8 GB per OSD minimum for BlueStore metadata cache; under-provisioning causes cache thrashing.
- Kernel IOMMU must be disabled for maximum NVMe performance (2024 Clyso finding; often overlooked).
- RocksDB compilation flags: verify Ceph packages are built with proper optimization flags; Ubuntu/Debian packages historically lacked them (doubles 4K random write performance when fixed).

**HDD alternative**:
- HDD clusters are viable for archive/cold tiers; NVMe WAL + NVMe DB (RocksDB metadata) greatly improves HDD cluster latency.
- HDD is not recommended for the hot lakehouse tier (Iceberg manifest latency requirements).

**Minimum viable cluster**:
- 3 OSD nodes for 3× replication; 4 OSD nodes for 2+2 EC; 3 monitor nodes (quorum).
- For a production lakehouse: ≥6 OSD nodes recommended (failure domain isolation for EC).

**RGW horizontal scaling**:
- Multiple RGW instances required for high concurrency (Spark 100+ executors).
- Each RGW instance handles ~32 concurrent connections at saturation; deploy behind a load balancer (HAProxy, Nginx, or DNS round-robin).

### 6. Known failure modes: slow OSD cascade, recovery storms, bucket index contention

Evidence: [Ceph slow OSD failure modes](../resources/ceph/ceph-slow-osd-failure-modes.summary.md), [Hardware recommendations](../resources/ceph/ceph-hardware-recommendations-official.summary.md). Confidence: **HIGH** (documented in official sources).

**Slow OSD cascade**:
A single slow OSD delays PG peering and can stall client I/O across the entire cluster. Root causes: disk I/O saturation, memory pressure (BlueStore cache), excessive scrub activity, network congestion. The Ceph health system alerts on OSDs breaching the 30-second slow request threshold; active monitoring is essential.

**Recovery storms**:
OSD failure triggers data recovery. Recovery I/O competes with client (RGW) I/O on the cluster network. Without a separate cluster network, a large recovery event (e.g., a 15 TB NVMe OSD failure) will measurably degrade Spark/Trino query throughput for the duration of recovery. The mclock scheduler is designed to throttle recovery in favour of client I/O, but a 2024 production incident showed mclock misconfiguration caused an OSD to overload disks during scrub operations. Cluster network separation is non-negotiable for production lakehouse deployments.

**Bucket index contention**:
High write concurrency from Spark (INSERT/OVERWRITE operations writing many new Parquet files simultaneously) can cause bucket index RADOS object contention if the bucket is under-sharded. Dynamic resharding addresses this automatically but causes brief write stalls. Pre-sharding is the correct approach for known high-write-rate table prefixes.

**RGW memory growth**:
High object-count RGW workloads cause OSD memory to grow due to RADOS metadata. Monitor OSD memory usage; ensure the 8 GB OSD memory target is set and respected.

**PG lock contention**:
Under very high concurrency (approaching network-speed limits), PG lock contention inside OSDs becomes a performance bottleneck. This is relevant at the top end of performance (NVMe clusters nearing line rate). For most lakehouse deployments this is not the primary bottleneck.

### 7. Erasure coding interaction with lakehouse workloads

Evidence: [FastEC blog (official, 2025)](../resources/ceph/ceph-tentacle-fastec-performance-2025.summary.md), [Red Hat Spark/Ceph 2019 (STALE, vendor-adjacent)](../resources/ceph/spark-ceph-performance-analysis.summary.md). Confidence: **MEDIUM**.

EC in Ceph provides significant storage efficiency: EC 4:2 uses ~33% of the raw capacity required by 3× replication for the same usable data. This is the primary motivation for using EC in a lakehouse. The trade-off is read performance, especially for small or partial-object reads.

**Pre-Tentacle (Squid and earlier)**:
- Reading any part of an EC-pooled object requires reading the full stripe across K data shards.
- For EC 4:2: 4 OSD reads per object read, regardless of how small the read is.
- Under high concurrency (10 Spark users): the 2019 Red Hat benchmark found ~90% throughput degradation vs. HDFS 3× for mixed workloads. (STALE data; cited for architectural context only.)

**Tentacle FastEC**:
- Partial reads are now supported: reading part of an EC stripe no longer requires full-stripe I/O.
- For Iceberg manifests and Parquet footers (which are at the beginning or end of an object), FastEC directly reduces OSD I/O amplification.
- Quantified RGW-specific benefit: not yet independently published (Tentacle released November 2025; as of May 2026, no third-party benchmark found).

**Recommended configuration**:
- EC 2+2 (not 4:2) for the hot lakehouse tier: lower read amplification, acceptable storage efficiency (~50% overhead vs. ~33% for 4:2).
- 3× replication pools for the Iceberg metadata prefix (manifest files, statistics); these are small objects with high read frequency.
- EC pools for large Parquet data files (>1 MB): storage efficiency gains are material here; FastEC partial reads address footer access patterns.

### 8. License and commercial support

Evidence: [License and commercial support (TechTarget, press)](../resources/ceph/ceph-license-commercial-support-2025.summary.md). Confidence: **HIGH**.

- **License**: LGPL 2.1 or later. Permissive for enterprise use — Ceph can be used in proprietary applications; only modifications to Ceph itself must be shared under LGPL. Fully satisfies the open-source constraint.
- **Commercial support**:
  - **IBM Storage Ceph** (successor to Red Hat Ceph Storage): Primary commercial vendor; Premium and Pro editions; perpetual/annual/monthly licensing; SLA-backed support. IBM acquired Red Hat in 2019 and continues both product lines.
  - **Red Hat Ceph Storage**: Continues under IBM/Red Hat branding.
  - **SUSE Enterprise Storage**: Discontinued March 2021; SUSE now focuses on Longhorn.
  - **Canonical**: Commercial support through Ubuntu Advantage contracts (Ceph is included in Ubuntu cluster tooling); not a standalone Ceph product.
  - **Community support**: Active; ceph-users mailing list, Slack, IRC, quarterly releases.
- **Governance**: Ceph Foundation under the Linux Foundation; multi-vendor (IBM, Bloomberg, DigitalOcean, Intel, Samsung, Fujitsu, CERN). This distributed governance model is explicitly cited as protection against the single-company abandonment risk that affected MinIO (Community Edition archived early 2026).

## Decision Matrix

| Profile | Recommendation | Notes |
|---|---|---|
| New on-premise lakehouse, open-source constraint, team has Ceph expertise, >50 TB target | **Ceph RadosGW** | LGPL, multi-vendor governance, full S3 API, proven at scale. Target Tentacle (v20) for FastEC. Use EC 2+2 for data, 3× replication for Iceberg metadata. Pre-shard buckets. Horizontal RGW scaling behind load balancer. |
| New on-premise lakehouse, open-source constraint, no Ceph expertise, <50 TB | **Ceph RadosGW with IBM or Red Hat support contract**, or evaluate SeaweedFS | MinIO Community Edition is archived (early 2026). Without Ceph expertise, a support contract is strongly recommended. SeaweedFS is simpler to operate but less mature at scale. |
| Existing Ceph cluster (block/file), adding lakehouse | **Add RadosGW to existing Ceph cluster** | Lowest incremental cost; existing expertise; unified storage. Ensure separate EC pool for data and replication pool for metadata. |
| Lakehouse with heavy streaming writes (CDC/ETL) | **Ceph RadosGW** with pre-sharded buckets and separate cluster network | High write concurrency from Spark requires pre-sharded bucket indexes and multiple RGW instances. Recovery storms during streaming must be isolated by separate cluster network. |
| Interactive analytics, sub-100ms query latency requirements | **Ceph RadosGW (all-NVMe) + Tentacle FastEC** | EC read amplification for Iceberg metadata is the primary latency risk. Mitigate with 3× replication for metadata prefix and FastEC for data. If latency SLAs cannot be met, consider a dedicated NVMe caching tier. |

**Eliminated options**:
- **MinIO Community Edition**: Archived (repository read-only) in early 2026. Not viable for new open-source deployments. [Source: onidel.com 2025 comparison](../resources/ceph/ceph-vs-minio-comparison-2025.summary.md).
- **HDFS**: Not S3-compatible; requires Hadoop FileSystem API rather than S3 API; eliminates Trino native S3 integration. Data locality advantage is lost when compute and storage are decoupled.
- **Cloud-managed S3 (AWS, GCS, Azure)**: Eliminated by on-premise constraint.

## References

- [Ceph Squid v19 Release Notes](../resources/ceph/ceph-squid-v19-release-notes.summary.md)
- [Ceph Tentacle v20 Release Notes](../resources/ceph/ceph-tentacle-v20-release-notes.summary.md)
- [Ceph RadosGW S3 API Documentation](../resources/ceph/ceph-rgw-s3-api-documentation.summary.md)
- [Ceph Hardware Recommendations (Official)](../resources/ceph/ceph-hardware-recommendations-official.summary.md)
- [Ceph RGW Dynamic Bucket Index Resharding](../resources/ceph/ceph-rgw-dynamic-bucket-sharding.summary.md)
- [Ceph Tentacle FastEC Performance (2025)](../resources/ceph/ceph-tentacle-fastec-performance-2025.summary.md)
- [Ceph Slow OSD Failure Modes](../resources/ceph/ceph-slow-osd-failure-modes.summary.md)
- [Ceph: A Journey to 1 TiB/s (Clyso, 2024)](../resources/ceph/ceph-journey-to-1tibps-2024.summary.md)
- [Ceph vs. MinIO / Open-Source Landscape (onidel, 2025)](../resources/ceph/ceph-vs-minio-comparison-2025.summary.md)
- [Ceph License and Commercial Support (TechTarget)](../resources/ceph/ceph-license-commercial-support-2025.summary.md)
- [Trino S3 Filesystem Documentation](../resources/ceph/trino-s3-filesystem-documentation.summary.md)
- [IBM RGW Benchmarking 2025 (Parts 1–3)](../resources/ceph/ceph-rgw-benchmarking-2025-ibm-part1-3.summary.md)
- [TPC-DS Trino + Ceph S3 Select Benchmark (IBM, 2025)](../resources/ceph/ceph-trino-tpcds-s3select-benchmark-2025.summary.md)
- [IOMETE Lakehouse Storage Evaluation (2025)](../resources/ceph/ceph-evaluating-s3-lakehouse-iomete-2025.summary.md)
- [Red Hat Spark on Ceph Benchmark (2019, STALE)](../resources/ceph/spark-ceph-performance-analysis.summary.md)

---
title: SeaweedFS as Lakehouse Storage — On-Premise Enterprise 2026
date: 2026-05-21
status: complete
components: [seaweedfs, spark, trino]
constraints:
  - open-source: true
  - deployment: on-premise
---

## Overview

This report evaluates SeaweedFS (github.com/seaweedfs/seaweedfs) as the physical object storage layer for a Spark + Trino lakehouse deployed on-premise. The evaluation covers S3 API compatibility, small-file performance, architecture and failure domains, production maturity, operational pain points, and licensing. Table formats (Iceberg, Hudi, Delta Lake) and catalogs (Nessie, Polaris) are mentioned only where they constrain SeaweedFS behavior.

**Verdict: SeaweedFS is a viable, actively developed open-source object store that has crossed the threshold from "watch item" to "conditional production candidate" as of mid-2026 — primarily because MinIO Community Edition was archived in February 2026 and SeaweedFS is now the most mature Apache 2.0 alternative with documented production deployments at scale. However, it is not yet enterprise-grade in the way Ceph RGW is: its backup/restore story requires cluster downtime, filer consistency under network partition is acknowledged as "not planned" to fix, and its aggressive release cadence (2–3 releases/week) introduces regression risk if teams track latest. Enterprise adopters should run SeaweedFS on a pinned, internally validated version, use a distributed filer store (not LevelDB), deploy 3-master Raft quorum, and allocate dedicated ops capacity for compaction and GC monitoring.**

---

## Evidence Quality

| Source | File | Tier | Accessible |
|---|---|---|---|
| SeaweedFS GitHub repository (v4.27) | [link](../resources/seaweedfs/github-repo-seaweedfs.summary.md) | official | Yes |
| SeaweedFS Production Setup wiki | [link](../resources/seaweedfs/production-setup-wiki.summary.md) | official | Yes |
| SeaweedFS Amazon S3 API wiki | [link](../resources/seaweedfs/s3-api-wiki.summary.md) | official | Yes |
| SeaweedFS S3 Configuration wiki | [link](../resources/seaweedfs/trino-s3-native-docs.summary.md) | official | Yes |
| Trino native S3 file system docs | [link](../resources/seaweedfs/trino-s3-native-docs.summary.md) | official | Yes |
| SeaweedFS Failover Master Server wiki | [link](../resources/seaweedfs/failover-master-wiki.summary.md) | official | Yes |
| SeaweedFS FAQ wiki | [link](../resources/seaweedfs/faq-wiki.summary.md) | official | Yes |
| SeaweedFS Data Backup wiki | [link](../resources/seaweedfs/data-backup-wiki.summary.md) | official | Yes |
| SeaweedFS Components wiki | [link](../resources/seaweedfs/seaweedfs-components-wiki.summary.md) | official | Yes |
| SeaweedFS Release Cadence (GitHub releases) | [link](../resources/seaweedfs/seaweedfs-release-cadence-2026.summary.md) | official | Yes |
| SeaweedFS Benchmarks wiki | [link](../resources/seaweedfs/seaweedfs-benchmarks-wiki.summary.md) | official | Yes |
| GitHub Issue #8151 — data corruption bug (encryptVolumeData) | [link](../resources/seaweedfs/github-issue-8151-data-corruption.summary.md) | official | Yes |
| GitHub Issue #6186 — filer consistency under network partition | [link](../resources/seaweedfs/github-issue-6186-filer-consistency.summary.md) | official | Yes |
| GitHub Discussion #5826 — GC / disk space not reclaimed | [link](../resources/seaweedfs/github-discussion-5826-gc.summary.md) | official | Yes |
| Sentry self-hosted issue #4071 — production readiness | [link](../resources/seaweedfs/sentry-issue-4071-production-readiness.summary.md) | press | Yes |
| Kubeflow Pipelines adopts SeaweedFS (Medium, Sep 2025) | [link](../resources/seaweedfs/kubeflow-seaweedfs-adoption-2025.summary.md) | press | Yes |
| Trino Iceberg CI integration test (Feb 6, 2026) | [link](../resources/seaweedfs/trino-iceberg-integration-test-2026.summary.md) | official | Yes |
| Software Heritage production deployment | [link](../resources/seaweedfs/software-heritage-deployment.summary.md) | press | Yes |
| RustFS benchmark comparison (Jan 14, 2026) | [link](../resources/seaweedfs/rustfs-benchmark-comparison-2026.summary.md) | vendor-adjacent | Yes |
| SeaweedFS official benchmarks wiki | [link](../resources/seaweedfs/seaweedfs-benchmarks-wiki.summary.md) | official | Yes |

**Gaps and confidence limits:**

1. **No independent lakehouse-scale benchmark exists.** The only multi-node performance data (RustFS discussion, Jan 2026) is vendor-adjacent; independent press benchmarks for SeaweedFS in a Spark/Trino/Iceberg context were not found. Confidence on performance at lakehouse scale is MEDIUM at best.

2. **Trino native S3 compatibility is not officially tested.** Trino's own documentation states only AWS S3 and MinIO are tested. SeaweedFS CI tests Trino integration in CI but does not document full API parity. Confidence on `fs.s3.enabled=true` compatibility is MEDIUM (functional for core operations, edge cases untested).

3. **No independently verified petabyte-scale lakehouse deployment.** SmartMore's petabyte-scale SeaweedFS deployment is reported only via JuiceFS's blog (vendor-adjacent). The Software Heritage deployment (independent) shows billions-of-objects scale but was measured in 2023.

4. **Filer metadata consistency under network partition is a known, unplanned gap.** Issue #6186 was closed "not planned." This is an accepted limitation, not a bug being fixed.

5. **Data corruption bug (#8151) affecting `-encryptVolumeData` in v4.05–4.07 has an open PR but no confirmed resolution in retrieved content.** Teams using encryption at rest should validate their specific version.

6. **Performance benchmark from official wiki is stale** (single-machine MacBook, no date specified); 2026 multi-node WARP data provides more relevant context but is vendor-adjacent.

---

## Components

### `seaweedfs`

SeaweedFS is an Apache 2.0 distributed object store written in Go, inspired by Facebook's Haystack paper. It separates concerns across four layers: master servers (cluster coordination via Raft), volume servers (data persistence), filer servers (filesystem abstraction), and an S3 gateway (AWS API compatibility). The separation allows independent scaling of each layer. As of v4.27 (May 20, 2026), it has 32,400+ GitHub stars, 301 releases, and an active release cycle. See [GitHub repository](../resources/seaweedfs/github-repo-seaweedfs.summary.md).

The architecture packs many small files into large volume blocks (default 32 GB, configurable to 8 TB in large-disk mode), eliminating per-file disk seeks — O(1) lookup regardless of object count. Memory overhead is 16 bytes per object in RAM and 40 bytes on disk. This design directly addresses the Iceberg small-file problem (manifest files, Parquet footers, snapshot JSON). See [Components wiki](../resources/seaweedfs/seaweedfs-components-wiki.summary.md).

### `spark`

Spark accesses SeaweedFS as an S3-compatible object store via the `s3a://` Hadoop FileSystem connector or via SeaweedFS's built-in Iceberg REST Catalog. Spark requires the endpoint to be set (`fs.s3a.endpoint=http://seaweedfs-host:8333`) and path-style access enabled (`fs.s3a.path.style.access=true`). SeaweedFS's CI pipeline tests Spark Iceberg integration. No independent benchmark for Spark on SeaweedFS was found.

### `trino`

Trino accesses SeaweedFS via `fs.s3.enabled=true` (the native S3 implementation) with `s3.path-style-access=true` and `s3.endpoint=http://seaweedfs-host:8333`. Path-style access is required because SeaweedFS does not provision per-bucket DNS subdomains. See [Trino S3 docs](../resources/seaweedfs/trino-s3-native-docs.summary.md). Trino only officially tests AWS S3 and MinIO; SeaweedFS compatibility relies on the project's own CI integration tests, which passed in February 2026. See [Trino Iceberg CI test](../resources/seaweedfs/trino-iceberg-integration-test-2026.summary.md).

---

## Findings

### 1. S3 API Compatibility: Functional for Core Lakehouse Operations, Untested at Full Parity

Evidence: [Trino S3 docs](../resources/seaweedfs/trino-s3-native-docs.summary.md), [SeaweedFS S3 API wiki](../resources/seaweedfs/s3-api-wiki.summary.md), [Trino Iceberg CI test](../resources/seaweedfs/trino-iceberg-integration-test-2026.summary.md). Confidence: **MEDIUM**.

SeaweedFS implements the core S3 operations required by Spark and Trino for Iceberg workloads: GetObject, PutObject, CopyObject, DeleteObject, DeleteObjects (multi-object, critical for Iceberg snapshot expiry), ListObjects/V2, HeadObject, CreateMultipartUpload, UploadPart, CompleteMultipartUpload, AbortMultipartUpload, and versioning. These cover the full Iceberg read/write/maintenance surface.

Trino's native S3 client (`fs.s3.enabled=true`) requires three configuration properties for SeaweedFS: `s3.endpoint=http://seaweedfs-host:8333`, `s3.path-style-access=true`, and `s3.region=us-east-1` (placeholder). Path-style is mandatory — SeaweedFS does not provision per-bucket DNS subdomains. This is the same configuration pattern used by Apache Doris (documented in official Doris docs for SeaweedFS integration).

Confidence is MEDIUM rather than HIGH because: (1) Trino's own documentation explicitly states only AWS S3 and MinIO are tested; (2) SeaweedFS CI tests Trino integration but the scope of that test covers "S3 over HTTPS using AWS CLI," not comprehensive API surface validation; (3) no independent third-party report of running Trino in production against SeaweedFS with full feature validation was found.

Missing S3 features that do not affect lakehouse operations: MFA Delete, ListDirectoryBuckets (S3 Express), SelectObjectContent, Object Lambda, lifecycle transition rules, website hosting. None of these are needed for Spark/Trino/Iceberg.

### 2. Small File Handling: Architecture Is Purpose-Built for the Iceberg Small-File Problem

Evidence: [GitHub repository](../resources/seaweedfs/github-repo-seaweedfs.summary.md), [Components wiki](../resources/seaweedfs/seaweedfs-components-wiki.summary.md), [FAQ wiki](../resources/seaweedfs/faq-wiki.summary.md), [RustFS benchmark](../resources/seaweedfs/rustfs-benchmark-comparison-2026.summary.md). Confidence: **HIGH** (architecture design); **MEDIUM** (lakehouse-scale benchmark).

SeaweedFS's core design differentiator is efficient small file handling. Files are packed into volume blocks; the master stores only volume locations (not per-file metadata), and each volume server stores per-file metadata in memory (16 bytes) with one disk seek per read regardless of cluster size. This architecture directly benefits Iceberg manifest files (typically 1–100 KB), Parquet footer files, and snapshot JSON objects.

In the January 2026 WARP benchmark (vendor-adjacent, RustFS discussion), SeaweedFS outperformed both MinIO and RustFS for small file objects (4 KiB and 32 KiB). For large files (5 MiB+), MinIO was faster. The benchmark summary conclusion was: "Small files: seaweedfs > minio > rustfs; Large files: minio > rustfs > seaweedfs." A typical Iceberg lakehouse has a mixed small/large profile: many small manifest and metadata files, with Parquet data files ranging from tens of MB to several GB. SeaweedFS's advantage in small-file operations is architecturally sound and confirmed by independent benchmark evidence, though the benchmark source is vendor-adjacent.

Software Heritage achieves 2,000 objects/second at ~200 MB/s on a modest 4-node cluster — an independently documented deployment at archive scale. Benchmark data is from 2023 and is therefore stale, but it establishes that billions-of-objects scale is operationally viable with SeaweedFS.

### 3. Architecture and Failure Domains

Evidence: [Components wiki](../resources/seaweedfs/seaweedfs-components-wiki.summary.md), [Failover Master wiki](../resources/seaweedfs/failover-master-wiki.summary.md), [Production Setup wiki](../resources/seaweedfs/production-setup-wiki.summary.md). Confidence: **HIGH**.

SeaweedFS has four distinct failure domains:

**Master cluster (Raft group):** 3-master deployment is standard HA. Raft ensures leader election on failure; the election window temporarily makes some volume servers unavailable for writes (until heartbeat reconnects). The master stores only soft state (volume locations) — if all masters fail, data is intact but the cluster is inaccessible until a master recovers. Single-master is explicitly acceptable per official docs for clusters where simplicity is prioritized over write availability.

**Volume servers:** Data is stored in append-only volume files. Replication is configured at volume level (`001` = one extra replica in same rack, `100` = one extra replica in different rack). After adding new volume servers, data rebalancing is NOT automatic — operators must use `weed shell` commands during off-hours. This means capacity expansion does not immediately improve data distribution.

**Filer servers:** Optional stateless component providing POSIX and bucket path semantics. For multi-filer HA, a shared distributed filer store (Redis, PostgreSQL, Cassandra) is required. The default LevelDB store supports only one filer instance and causes permanent metadata divergence if that filer experiences a network partition (Issue #6186, closed "not planned").

**S3 gateway:** Stateless; runs co-located with filer or standalone. Multiple S3 gateways behind a load balancer are supported.

**Critical operational requirement:** For lakehouse use, deploy a shared filer store (Redis or PostgreSQL recommended) rather than LevelDB. Without this, multi-filer deployments cannot guarantee metadata consistency after any network instability, and Iceberg snapshot operations involve frequent file deletions that would trigger the Issue #6186 divergence scenario.

### 4. Production Maturity: Crossed the Threshold, Not Yet Ceph-Grade

Evidence: [Kubeflow adoption 2025](../resources/seaweedfs/kubeflow-seaweedfs-adoption-2025.summary.md), [Software Heritage deployment](../resources/seaweedfs/software-heritage-deployment.summary.md), [Release cadence 2026](../resources/seaweedfs/seaweedfs-release-cadence-2026.summary.md), [Sentry issue #4071](../resources/seaweedfs/sentry-issue-4071-production-readiness.summary.md). Confidence: **MEDIUM**.

Documented production deployments exist: Software Heritage (independent nonprofit, billions of objects, 2,000 objects/s at 200 MB/s on 4 nodes, 2023); Kubeflow Pipelines officially replaced MinIO with SeaweedFS as default storage backend (September 2025). SmartMore's petabyte-scale deployment uses SeaweedFS under JuiceFS as a storage layer, but this is reported only through JuiceFS's own blog (vendor-adjacent; cannot assign HIGH confidence).

SeaweedFS has 12+ years of development history, 301 releases, and 32,000+ GitHub stars. Active development is confirmed by the 4.x release series in 2025–2026. The Kubeflow adoption represents the most significant independent production validation: it replaces MinIO in a CNCF-tracked ML platform used by enterprise teams.

However, the aggressive release cadence (2–3 releases/week in May 2026) is a double-edged signal. Version 4.23 was explicitly marked unsafe for erasure coding on multi-disk servers and corrected in 4.24 — a critical regression that self-corrected in days but demonstrates that tracking latest is risky for production infrastructure. An enterprise deployment must implement: version pinning, internal validation pipeline before upgrades, and monitoring for regression notices in release notes.

Compared to Ceph RGW, SeaweedFS lacks: a documented enterprise support SLA from its open-source community (the enterprise edition at seaweedfs.com provides this commercially), a mature multi-year track record in petabyte-scale financial or telecommunications deployments with published case studies, and a governance structure equivalent to the Ceph Foundation.

Confidence is MEDIUM because: no independently verified lakehouse-scale (multi-TB Iceberg) production deployment was found; the strongest independent evidence (Software Heritage) is from 2023; and the Kubeflow validation is ML pipeline artifacts, not Iceberg table storage.

### 5. Performance Benchmarks: Competitive with MinIO for Mixed Small/Large Workloads

Evidence: [RustFS benchmark Jan 2026](../resources/seaweedfs/rustfs-benchmark-comparison-2026.summary.md), [Official benchmarks wiki](../resources/seaweedfs/seaweedfs-benchmarks-wiki.summary.md). Confidence: **MEDIUM** (single vendor-adjacent source for multi-node data).

The only available multi-node benchmark with specific numbers is the January 2026 WARP comparison (RustFS organization, vendor-adjacent). On 4 nodes (32 cores, 128 GB RAM, SATA SSD), SeaweedFS 4.02 achieved:
- GET: ~1,800–1,900 MiB/s (MinIO: ~2,000 MiB/s; ~5–10% lower)
- PUT: ~630–640 MiB/s (MinIO: ~650–675 MiB/s; ~2–3% lower)
- Small files (4 KiB, 32 KiB): SeaweedFS outperformed MinIO and RustFS

The performance gap vs MinIO is small for large objects and reversed for small objects — an ideal profile for Iceberg lakehouses, which are read-heavy with mixed object sizes (small manifests + large Parquet files).

The official SeaweedFS benchmarks wiki shows single-machine numbers (5,747 KB/s write, 12,988 KB/s read for 1 million 1 KB files) — stale and not representative of multi-node deployments; cited here only for baseline context.

No independent press benchmark (InfoQ, The New Stack) for SeaweedFS in a lakehouse context was found. Confidence is MEDIUM — one vendor-adjacent benchmark aligns with architectural expectations but independent corroboration is absent.

### 6. License: Apache 2.0 Confirmed and Stable

Evidence: [GitHub repository](../resources/seaweedfs/github-repo-seaweedfs.summary.md). Confidence: **HIGH**.

SeaweedFS's open-source edition is Apache 2.0 licensed — confirmed via the GitHub LICENSE file. No licensing changes or AGPL conversion were found. A separate commercial enterprise edition exists at seaweedfs.com under a proprietary EULA, providing additional features (self-healing storage, automated EC shard rebuild, time-travel undelete, enterprise support SLA). The open-source Apache 2.0 license satisfies the on-premise, open-source constraint and imposes no network-service obligation (unlike AGPL used by alternatives such as Garage).

The MinIO Community Edition was archived in February 2026 — removing it as a viable open-source option and directly elevating SeaweedFS as the primary Apache 2.0 candidate. This licensing shift is the primary driver of SeaweedFS's rising adoption in 2026.

### 7. Operational Pain Points: GC, Compaction, Backup, and Filer Consistency

Evidence: [Data Backup wiki](../resources/seaweedfs/data-backup-wiki.summary.md), [Discussion #5826](../resources/seaweedfs/github-discussion-5826-gc.summary.md), [Issue #6186](../resources/seaweedfs/github-issue-6186-filer-consistency.summary.md), [Issue #8151](../resources/seaweedfs/github-issue-8151-data-corruption.summary.md), [FAQ wiki](../resources/seaweedfs/faq-wiki.summary.md). Confidence: **HIGH** (these gaps are documented in official sources).

Four operational pain points are confirmed by official documentation or open issues:

**Garbage collection / compaction:** Default vacuum threshold (30% garbage) is often too conservative — documented user experience (Discussion #5826) shows 60 GB consumed for effectively 3.3 GB of live data. Operators must actively tune `volume.vacuum -garbageThreshold` and monitor compaction status. For Iceberg lakehouses, which generate constant deletes during snapshot expiry, compaction overhead is a routine operational concern, not an edge case.

**Backup and restore:** The documented full-cluster backup procedure requires stopping all writes. The async filer backup (`weed backup`) supports online incremental backup by scanning filer change logs, but it does not provide consistent point-in-time restore for distributed deployments. The official wiki self-describes the procedure as "not bullet proof" and "tested on a small deployment." This is the most significant operational gap for enterprise production.

**Filer metadata consistency (multi-filer, local store):** Issue #6186 documents permanent metadata divergence under network partition when using LevelDB filer stores. Closed "not planned." Mitigation: use shared filer store (Redis, PostgreSQL) and treat LevelDB as single-filer-only. For lakehouse Iceberg workloads involving frequent object deletions (snapshot maintenance, compaction), this divergence risk is elevated.

**Data corruption (encryptVolumeData, v4.05–4.07):** Issue #8151 documents silent data corruption with `-encryptVolumeData` flag in versions 4.05–4.07. A PR was created. Teams using encryption at rest must validate their specific version is not affected and should track the fix. Version 4.04 is confirmed clean; versions above 4.07 should be tested before production use of this feature.

**Release regression risk:** Version 4.23 introduced a regression for erasure coding on multi-disk servers (corrected in 4.24). Rapid release cadence requires active version governance.

### 8. Enterprise Lakehouse Readiness in 2026: Conditional Yes, Not Unconditional

Evidence: all sources above. Confidence: **MEDIUM**.

The prior assessment of SeaweedFS as "too immature for enterprise production" was reasonable in early 2025 but is outdated by mid-2026. SeaweedFS has meaningfully matured: Kubeflow Pipelines adoption (September 2025), active Trino Iceberg CI integration, built-in Iceberg REST Catalog, and the January 2026 multi-node benchmark confirming competitive performance. The MinIO Community Edition archival in February 2026 removes the primary alternative and elevates SeaweedFS as the default Apache 2.0 choice.

However, "enterprise-grade" has specific implications for lakehouse infrastructure:
- **Point-in-time backup:** Not solved without cluster downtime or async-only incremental backup
- **Filer consistency guarantee:** Not provided for multi-filer + local-store configurations
- **Release governance:** 2–3 releases/week is too fast to track in production without internal validation
- **Support SLA:** Available only via commercial enterprise edition (seaweedfs.com), not from the open-source community

For teams willing to invest ops capacity in: pinned versioning, shared filer store deployment, monitored compaction, and async backup strategy — SeaweedFS is a viable enterprise lakehouse storage layer in 2026. For teams expecting MinIO-level "install and forget" simplicity or Ceph-level enterprise support from the open-source edition, it is not yet ready.

Confidence is MEDIUM because: the prior "too immature" assessment lacked dedicated evidence; this report reverses it based on concrete production validations (Kubeflow, Software Heritage), but no independently verified lakehouse-scale Iceberg production deployment was found.

---

## Decision Matrix

| Profile | Recommendation | Eliminated options and why |
|---|---|---|
| New on-premise lakehouse, open-source required, willing to manage ops | **SeaweedFS** (pinned version, Redis filer store, 3-master HA) | MinIO Community: archived Feb 2026, no security patches; Garage: AGPL license, geo-distribution focus not applicable; RustFS: too new (alpha, documented stability crashes in Jan 2026 benchmark) |
| Existing Ceph deployment, team expertise present | **Ceph RGW** — not SeaweedFS | Switching cost not justified; Ceph provides stronger S3 parity, petabyte-scale track record, and multi-protocol (block, file, object) under one system |
| Team needs battle-tested platform with commercial support from open-source community | **Ceph RGW** (Ceph Foundation) or **SeaweedFS Enterprise** (seaweedfs.com) | Open-source SeaweedFS has no community support SLA |
| Encryption at rest required, no commercial support budget | **SeaweedFS** — but validate version (avoid 4.05–4.07 with `-encryptVolumeData`); monitor Issue #8151 resolution | Bug in encrypted volume path requires version validation |
| Strictly zero-downtime backup required, no ops investment | **Not SeaweedFS** — use Ceph RGW or a cloud-backed solution | SeaweedFS full-cluster backup requires write pause; async backup lacks consistent point-in-time restore |

---

## References

- [SeaweedFS GitHub Repository (v4.27)](../resources/seaweedfs/github-repo-seaweedfs.summary.md)
- [SeaweedFS Production Setup Wiki](../resources/seaweedfs/production-setup-wiki.summary.md)
- [SeaweedFS Amazon S3 API Wiki](../resources/seaweedfs/s3-api-wiki.summary.md)
- [Trino Native S3 File System Documentation](../resources/seaweedfs/trino-s3-native-docs.summary.md)
- [SeaweedFS Failover Master Server Wiki](../resources/seaweedfs/failover-master-wiki.summary.md)
- [SeaweedFS FAQ Wiki](../resources/seaweedfs/faq-wiki.summary.md)
- [SeaweedFS Data Backup Wiki](../resources/seaweedfs/data-backup-wiki.summary.md)
- [SeaweedFS Components Wiki](../resources/seaweedfs/seaweedfs-components-wiki.summary.md)
- [SeaweedFS Release Cadence (GitHub Releases)](../resources/seaweedfs/seaweedfs-release-cadence-2026.summary.md)
- [SeaweedFS Official Benchmarks Wiki](../resources/seaweedfs/seaweedfs-benchmarks-wiki.summary.md)
- [GitHub Issue #8151 — Data Corruption Bug (encryptVolumeData)](../resources/seaweedfs/github-issue-8151-data-corruption.summary.md)
- [GitHub Issue #6186 — Filer Consistency Under Network Partition](../resources/seaweedfs/github-issue-6186-filer-consistency.summary.md)
- [GitHub Discussion #5826 — GC Disk Space Not Reclaimed](../resources/seaweedfs/github-discussion-5826-gc.summary.md)
- [Sentry Self-Hosted Issue #4071 — Production Readiness](../resources/seaweedfs/sentry-issue-4071-production-readiness.summary.md)
- [Kubeflow Pipelines Adopts SeaweedFS (Sep 2025)](../resources/seaweedfs/kubeflow-seaweedfs-adoption-2025.summary.md)
- [Trino Iceberg CI Integration Test (Feb 6, 2026)](../resources/seaweedfs/trino-iceberg-integration-test-2026.summary.md)
- [Software Heritage Production Deployment](../resources/seaweedfs/software-heritage-deployment.summary.md)
- [RustFS Benchmark Comparison Jan 2026](../resources/seaweedfs/rustfs-benchmark-comparison-2026.summary.md)

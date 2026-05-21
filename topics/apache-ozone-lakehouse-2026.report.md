---
title: Apache Ozone as Lakehouse Storage — On-Premise Enterprise 2026
date: 2026-05-21
status: complete
components: [ozone, spark, trino]
constraints:
  - open-source: true
  - deployment: on-premise
---

## Overview

Apache Ozone is a distributed object store built into the Hadoop ecosystem, designed as HDFS's long-term successor at scale. This report evaluates Ozone as the physical storage layer for a Spark + Trino lakehouse, under strict open-source and on-premise constraints, as of May 2026.

**Verdict: Apache Ozone is viable as on-premise lakehouse storage in 2026 for organizations already operating in the Hadoop ecosystem, but it carries three concrete operational risks that must be accepted before adoption: (1) Trino on open-source Trino cannot use OFS natively — it must go through the S3 Gateway with static credentials because no STS endpoint exists yet (planned for 2.2.0, not yet released); (2) small-object performance under high LIST concurrency was a documented failure mode in 1.4.x that required hardware and configuration workarounds and has not been independently benchmarked at the 2.x level; and (3) operational tooling (multi-tenancy, disk balancing, cross-cluster DR) requires Ranger, Kerberos, and upcoming 2.2 features not yet generally available. For greenfield deployments with no existing Hadoop footprint, the operational complexity of Ozone vs. MinIO (simpler, lighter) should be weighed carefully — though MinIO's AGPL license creates a compliance risk for enterprises that would otherwise qualify it.**

---

## Evidence Quality

| Source | File | Tier | Accessible |
|---|---|---|---|
| ASF: Apache Ozone 2.0.0 Announcement | [link](../resources/ozone/apache-ozone-2-release.summary.md) | official | Yes |
| Apache Ozone: Comparison Table (HDFS/Ceph/MinIO) | [link](../resources/ozone/ozone-comparison-hdfs-ceph-minio.summary.md) | official | Yes |
| Apache Polaris Blog: Trino+Ozone+Polaris Lakehouse (Apr 2026) | [link](../resources/ozone/trino-ozone-polaris-lakehouse-guide.summary.md) | official | Yes |
| GitHub Ozone Discussion #5004: Trino OFS Configuration | [link](../resources/ozone/trino-ozone-ofs-configuration-discussion.summary.md) | official | Yes |
| GitHub Trino Discussion #18026: Ozone Iceberg Protobuf Failures | [link](../resources/ozone/trino-ozone-iceberg-protobuf-issue.summary.md) | official | Yes |
| GitHub Ozone Discussion #7501: S3 LIST Performance Issues | [link](../resources/ozone/ozone-s3-list-performance-issues.summary.md) | official | Yes |
| ASF Blog: DiDi Scales to Hundreds of PB (Jan 2026) | [link](../resources/ozone/ozone-didi-production-deployment.summary.md) | official | Yes |
| Apache Ozone 2.1.0 Release Notes (Dec 2025) | [link](../resources/ozone/ozone-2-1-release-notes.summary.md) | official | Yes |
| Apache Ozone: S3 Gateway Architecture | [link](../resources/ozone/ozone-s3-gateway-architecture.summary.md) | official | Yes |
| Apache Ozone: S3A Configuration Guide | [link](../resources/ozone/ozone-s3a-configuration.summary.md) | official | Yes |
| Apache Ozone: OFS Interface (2.0 docs) | [link](../resources/ozone/ozone-ofs-filesystem-interface.summary.md) | official | Yes |
| Apache Ozone: FSO Bucket Layout | [link](../resources/ozone/ozone-fso-bucket-layout.summary.md) | official | Yes |
| Apache Ozone: Roadmap (retrieved May 2026) | [link](../resources/ozone/ozone-roadmap-2026.summary.md) | official | Yes |
| Apache Ozone: Disk Balancer Preview (Jan 2026) | [link](../resources/ozone/ozone-disk-balancer-2-2.summary.md) | official | Yes |
| Apache Ozone: S3 Multi-Tenancy Documentation | [link](../resources/ozone/ozone-multi-tenancy-s3.summary.md) | official | Yes |
| Cloudera Engineering: Iceberg on Ozone (Feb 2023) | [link](../resources/ozone/cloudera-iceberg-ozone-lakehouse.summary.md) | vendor-adjacent | Yes |

**Gaps and confidence limits:**

1. **No independent benchmark for Ozone 2.x**: The only available independent small-object / LIST performance data dates to Ozone 1.4.x (2024, stale relative to 2026-05-21). The TPC-DS result (+3.5% vs. HDFS) in the 2.0 announcement is an official source but was not independently replicated. The DiDi metrics (P90 latency, read throughput) are production measurements but come from an ASF-managed blog post based on DiDi's own internal data — not third-party verified.
2. **Cloudera CDP source is vendor-adjacent**: Cloudera is the primary commercial vendor for Ozone. All Cloudera-sourced claims (licensing, support, performance) are vendor-adjacent and excluded from HIGH confidence findings.
3. **STS development status**: STS is confirmed in the roadmap for 2.2.0 (HDDS-13323, active as of Nov 2025) but as of May 2026, no 2.2.0 release exists. Current state is static credentials only.
4. **Ceph license clarification needed**: The comparison page lists Ceph as LGPLv2.1, which is permissive and would not create the same compliance concern as AGPL. MinIO's SSPL/AGPL license is the stronger constraint argument for Ozone over MinIO.
5. **No independent press coverage**: No InfoQ, The New Stack, or HPCwire articles were found covering Ozone in 2025–2026 production deployments. All supporting evidence is either official (ASF) or vendor-adjacent (Cloudera). This limits some findings to MEDIUM confidence despite strong official sourcing.
6. **Cross-datacenter DR**: No official documentation found for cross-site replication or disaster recovery. Ozone OM HA runs within a single site. This is a confirmed gap.

---

## Components

### `ozone`

Apache Ozone (current stable: 2.1.0, released December 31, 2025) is a distributed object store for the Hadoop ecosystem. It replaces HDFS's single NameNode with distributed metadata using RocksDB, removing the file-count scalability ceiling. Ozone exposes three client interfaces: the S3 Gateway (S3G), the OFS Hadoop-compatible filesystem (`ofs://`), and the legacy single-bucket `o3fs://`. The storage model is based on Storage Containers (groups of blocks) managed by the Storage Container Manager (SCM), with Ozone Manager (OM) handling namespace metadata. OM HA runs on a 3-node Raft quorum; SCM HA runs similarly. [Release Notes](../resources/ozone/ozone-2-1-release-notes.summary.md), [ASF 2.0 Announcement](../resources/ozone/apache-ozone-2-release.summary.md)

License: Apache 2.0. No single-vendor risk; ASF-governed. Commercial support available through Cloudera CDP Private Cloud Base. [Comparison](../resources/ozone/ozone-comparison-hdfs-ceph-minio.summary.md)

### `spark`

Apache Spark connects to Ozone via two paths: (a) the S3A connector (`s3a://`) pointing at the Ozone S3 Gateway, or (b) the OFS filesystem (`ofs://`) directly. For Iceberg + Spark workloads, OFS via FSO buckets is the recommended path for atomic rename semantics. S3A via S3G is the alternative when simpler S3-compatible access is preferred. [S3A Guide](../resources/ozone/ozone-s3a-configuration.summary.md), [FSO Layout](../resources/ozone/ozone-fso-bucket-layout.summary.md)

### `trino`

Trino (open-source) does not natively support the `ofs://` scheme. Attempts to add Ozone filesystem JARs to Trino plugin directories fail with Protobuf class conflicts. The production-supported integration path for open-source Trino is via the Ozone S3 Gateway with static S3 credentials. Starburst Enterprise adds native OFS support via `fs.hadoop.enabled=true` — not available in open-source Trino. [OFS Discussion](../resources/ozone/trino-ozone-ofs-configuration-discussion.summary.md), [Protobuf Issue](../resources/ozone/trino-ozone-iceberg-protobuf-issue.summary.md), [Polaris Guide](../resources/ozone/trino-ozone-polaris-lakehouse-guide.summary.md)

---

## Findings

### 1. Ozone exposes three interfaces; Trino uses S3 Gateway exclusively in open-source deployments

Evidence: [OFS Discussion](../resources/ozone/trino-ozone-ofs-configuration-discussion.summary.md), [Trino Protobuf Issue](../resources/ozone/trino-ozone-iceberg-protobuf-issue.summary.md), [Polaris Guide](../resources/ozone/trino-ozone-polaris-lakehouse-guide.summary.md), [OFS Interface](../resources/ozone/ozone-ofs-filesystem-interface.summary.md). Confidence: **HIGH**

The three interfaces are:
- **S3 Gateway (S3G)**: Stateless, horizontally scalable, AWS Signature V4, path-style required. Works with Trino's native-S3 filesystem (`fs.native-s3.enabled=true`). Adds one network hop vs. direct access.
- **OFS (`ofs://`)**: Full Hadoop HCFS view across all volumes and buckets. Supported natively by Spark, Hive, Flink, and MapReduce. Not natively supported in open-source Trino — produces "No FileSystem for scheme 'ofs'" errors. Starburst Enterprise adds OFS support via `fs.hadoop.enabled=true`.
- **o3fs (`o3fs://`)**: Legacy single-bucket scope. Deprecated in favor of OFS; not recommended for new deployments.

For Trino, the integration path is: deploy S3G behind a load balancer (Nginx or similar), configure Trino catalog with `s3.endpoint`, `s3.path-style-access=true`, `s3.aws-access-key`, `s3.aws-secret-key`, `s3.region=us-east-1`.

### 2. No STS endpoint in 2026 — static credentials required for Trino; workaround confirmed and documented

Evidence: [Polaris Guide](../resources/ozone/trino-ozone-polaris-lakehouse-guide.summary.md), [Roadmap](../resources/ozone/ozone-roadmap-2026.summary.md), [S3 Gateway Architecture](../resources/ozone/ozone-s3-gateway-architecture.summary.md). Confidence: **HIGH**

As of May 2026, Apache Ozone has no STS (Security Token Service) endpoint. This prevents credential vending — the production pattern where a catalog service (like Polaris or Nessie) issues short-lived, scoped credentials to query engines for each table access.

The documented workaround (confirmed in an April 2026 Apache Polaris official blog post):
1. Set `stsUnavailable: true` in the Polaris catalog storage configuration.
2. Provide static Ozone S3 credentials (`s3.aws-access-key` / `s3.aws-secret-key`) directly to Trino.
3. All Trino data I/O goes directly to Ozone using these static credentials; Polaris handles only metadata routing.

STS support (HDDS-13323) is planned for Ozone 2.2.0 ("Katmai") — the next minor release. Active development was confirmed via JIRA updates in November 2025 (HDDS-13926: IAM policy → OzoneObj/Acl conversion). No 2.2.0 release date has been announced as of May 2026.

**Security implication**: Static S3 credentials in Trino configuration are cluster-wide — all queries use the same credentials, with no per-table or per-user credential scoping. This is weaker than the STS model used with AWS S3. Mitigation: Kerberos + Ranger ACLs on the Ozone side can enforce per-user access control even with shared S3 credentials, but this requires deploying and operating Ranger.

### 3. Small object performance: FSO layout resolves the metadata layer; concurrent LIST under high load was a failure mode in 1.4.x with no independent 2.x validation

Evidence: [S3 LIST Issues](../resources/ozone/ozone-s3-list-performance-issues.summary.md), [FSO Layout](../resources/ozone/ozone-fso-bucket-layout.summary.md), [DiDi Deployment](../resources/ozone/ozone-didi-production-deployment.summary.md). Confidence: **MEDIUM**

Ozone's FSO (File System Optimized) bucket layout handles Iceberg's metadata pattern well at the namespace layer:
- Directory renames are O(1) metadata pointer updates — Iceberg's atomic commit relies on this.
- Small file creation (manifests, snapshots, Parquet footers) is efficient because children reference parent by unique ID, not by name string.
- FSO is the default bucket layout in Ozone 2.x and is strongly recommended for lakehouse workloads.

However, a documented failure mode exists: In Ozone 1.4.0 (data from ~2024), S3 LIST operations failed catastrophically on HDD-only datanodes (23,503 errors in one benchmark run), and small-object performance was poor compared to MinIO on equivalent hardware. The fix required: (a) moving Ratis metadata to SSD, (b) increasing heap to 31 GB, (c) adopting FSO bucket layout. These changes stabilized LIST but degraded READ throughput by ~75% in that configuration.

Confidence is MEDIUM because: the failure data is from 1.4.x (stale — Ozone 2.0 includes 1,700+ fixes, 2.1.0 adds 805 more). The 2.1.0 release includes OzoneManagerLock refactoring (hierarchical pool-based locking replacing global spin lock) that directly addresses the contention root cause. The DiDi deployment (January 2026, production data from 2025) shows P90 latency at 17ms and >20% read throughput improvement — but DiDi's workload characteristics differ from Iceberg-heavy small manifest queries. No independent 2025–2026 benchmark covering high-concurrency S3 LIST with small objects on Ozone 2.x has been found.

**Recommendation**: Before production deployment, validate LIST throughput under Iceberg workload (manifest scans, table discovery) on target hardware and Ozone 2.1.0. Require NVMe or SSD for Ratis/RocksDB metadata volumes.

### 4. Operational maturity: improving rapidly in 2025–2026, but key features require upcoming releases or complex dependencies

Evidence: [2.1.0 Release Notes](../resources/ozone/ozone-2-1-release-notes.summary.md), [Disk Balancer](../resources/ozone/ozone-disk-balancer-2-2.summary.md), [Multi-Tenancy](../resources/ozone/ozone-multi-tenancy-s3.summary.md), [Roadmap](../resources/ozone/ozone-roadmap-2026.summary.md). Confidence: **MEDIUM**

**Available now (Ozone 2.1.0, Dec 2025)**:
- Quota: Volume- and bucket-level storage quotas. Space usage metrics exposed.
- Monitoring: OpenTelemetry distributed tracing (migrated from OpenTracing in 2.1.0). New Grafana dashboards for RocksDB operations and deletion progress. StorageVolumeScannerMetrics and VolumeInfoMetrics on DataNodes. Deletion Progress tracking in OM Web UI.
- Snapshot/backup: Up to 10,000 snapshots per cluster; enhanced scalability (Phase 3). Backup SST pruning defaulted to 10-minute interval.
- ACLs: Per-volume, per-bucket, per-key ACLs. Apache Ranger integration available for enterprise policy management.
- OM HA: 3-node Raft quorum for OM; Listener OMs (read-only non-voting nodes) added in 2.1.0 for horizontal read scaling.
- Container reconciliation: New protocol (2.1.0) for detecting and resolving mismatched container states.

**Requires Ozone 2.2.0 (not yet released)**:
- Disk Balancer: Automatic intra-node disk rebalancing.
- S3 Lifecycle Configurations: Object expiration policies.
- STS temporary credentials: Per-session credential vending.

**Requires Ranger + Kerberos (significant operational overhead)**:
- S3 Multi-tenancy: True namespace isolation per team/org. Requires Apache Ranger and Kerberos infrastructure — substantially more complex than MinIO's simpler access model.

**Confirmed gap — no cross-datacenter DR**: No official cross-site replication feature is documented for Apache Ozone 2.x. OM HA is intra-cluster only. Organizations requiring geographic DR must implement their own data replication (e.g., DistCp-based batch replication, as DiDi uses for migration — not real-time DR). This is a significant gap for enterprises with RPO/RTO requirements.

Confidence is MEDIUM (not HIGH) because the operational tooling assessment relies primarily on official documentation without independent operational review from the trade press.

### 5. Performance: TPC-DS +3.5% vs. HDFS (Aug 2025); DiDi production P90 17ms metadata latency (Jan 2026)

Evidence: [ASF 2.0 Announcement](../resources/ozone/apache-ozone-2-release.summary.md), [DiDi Deployment](../resources/ozone/ozone-didi-production-deployment.summary.md). Confidence: **MEDIUM**

Two data points are available as of May 2026:

**TPC-DS benchmark (Aug 2025, from Ozone 2.0.0 ASF announcement)**:
- Ozone outperforms HDFS by an average of 3.5% across 99 TPC-DS queries.
- In >70% of individual queries, Ozone is faster than HDFS.
- Source: official ASF announcement; benchmark methodology not independently verified.
- Benchmark age: 2025 — within the 2-year window, not stale.

**DiDi production metrics (published Jan 2026, data from ~2024–2025)**:
- P90 GetMetaLatency: 90ms → 17ms (81% reduction after lock contention fixes and OM follower reads).
- Read throughput: >20% improvement with OM follower reads + NVMe caching.
- Scale: ~500+ PB, tens of billions of files.
- These are self-reported production metrics from an ASF-published case study; not independently audited.

**No independent Ozone 2.x benchmark** (Ceph, MinIO, HDFS comparison) was found from 2025 or 2026. The Cloudera benchmarking blog post is vendor-adjacent and not cited as a primary performance source. The 1.4.x S3 LIST benchmark (stale, from ~2024) showed severe performance degradation on HDD-only clusters — but predates the OzoneManagerLock refactor in 2.1.0.

Confidence is MEDIUM: two independent data points (TPC-DS from official announcement + DiDi from ASF case study) support adequate performance claims, but neither is from a financially neutral third party.

### 6. License (Apache 2.0) and commercial support eliminate compliance risk; MinIO AGPL is the key differentiator

Evidence: [Comparison](../resources/ozone/ozone-comparison-hdfs-ceph-minio.summary.md), [ASF 2.0 Announcement](../resources/ozone/apache-ozone-2-release.summary.md). Confidence: **HIGH**

Apache Ozone is licensed under Apache 2.0 — permissive, ASF-governed, no single-vendor lock-in. This satisfies the open-source constraint without any network-use or AGPL concerns.

MinIO uses AGPLv3 (described in their terms as SSPL-like), which requires any network-served derivative work to be open-sourced. For enterprises that cannot comply with AGPL's copyleft provisions, MinIO requires a commercial license — effectively removing it from a "free open-source" deployment. This is the strongest practical argument for Ozone over MinIO under the on-premise + open-source constraint.

Ceph is licensed under LGPLv2.1, which is permissive for linking but still requires careful legal review. Ceph also introduces significantly more operational complexity (RADOS, RGW, CephFS, MONs) and has weaker native big data ecosystem integration than Ozone.

Commercial support: Cloudera CDP Private Cloud Base includes Apache Ozone with enterprise support, Ranger integration, Atlas data governance, and long-term support releases (up to 4-year support periods). This is a vendor-adjacent support option, not a dependency — Ozone is fully usable without Cloudera.

### 7. Known failure modes and production incidents

Evidence: [S3 LIST Issues](../resources/ozone/ozone-s3-list-performance-issues.summary.md), [2.1.0 Release Notes](../resources/ozone/ozone-2-1-release-notes.summary.md), [DiDi Deployment](../resources/ozone/ozone-didi-production-deployment.summary.md), [Disk Balancer](../resources/ozone/ozone-disk-balancer-2-2.summary.md). Confidence: **MEDIUM**

Documented failure modes (with status):

| Failure Mode | Version Affected | Status in 2.1.0 |
|---|---|---|
| S3 LIST failures on HDD-only datanodes | 1.4.0–1.4.1 | Likely improved (OzoneManagerLock refactor, Derby→RocksDB migration); not independently re-benchmarked |
| Mismatched container states (data integrity risk) | Pre-2.1.0 | Fixed: container reconciliation protocol added in 2.1.0 |
| Disk imbalance causing I/O hotspots | All versions pre-2.2 | Partially manual workaround; Disk Balancer ships in 2.2.0 |
| OM RPC congestion at high concurrency | Pre-2.1 fixes | Mitigated: Listener OMs in 2.1.0; follower reads available |
| Out-of-space failures from block deletion backlog | Pre-2.1.0 | Fixed: SCM throttling mechanism + improved disk space management in 2.1.0 |
| Trino OFS Protobuf class conflict | Trino + Ozone 1.3 | Partial: Ozone PR #7729 adds shading; S3G path avoids entirely |

DiDi's migration story confirms that production stability required: dual-write during migration for rollback safety, checksums verification, NVMe caching for hot data, and cluster-level file count caps (~5B per cluster with ViewFs routing for spillover).

No public reports of data loss or catastrophic production incidents at scale were found. The counter-evidence search found no "not recommended" or "failure" incident reports in 2025–2026.

---

## Decision Matrix

| Profile | Recommendation | Eliminated options and why |
|---|---|---|
| On-premise, existing Hadoop footprint, Spark + Trino, Apache 2.0 license required | **Use Ozone 2.1.0 with S3 Gateway for Trino, OFS for Spark; accept static credentials until 2.2.0 STS** | MinIO: AGPL license non-compliant with open-source-only constraint without commercial license. HDFS: No S3 API; file-count ceiling unresolved without federation. Ceph: Higher operational complexity; weaker Hadoop ecosystem integration. |
| Greenfield, no Hadoop footprint, S3-only workload | **Evaluate MinIO first (simpler, lighter, faster small-object performance) — but verify AGPL compliance.** If AGPL is a blocker, use Ozone. | MinIO AGPL: If acceptable, MinIO is operationally simpler. If not acceptable, Ozone with S3G is the only open-source path. Ceph: Overkill for pure object store; requires RGW for S3. |
| Enterprise multi-tenancy required (per-team namespace isolation) | **Ozone with Ranger + Kerberos for full S3 multi-tenancy.** Accept significant operational overhead. | MinIO: Has tenant isolation but AGPL license concern. Ceph RGW: Supports user isolation but not Hadoop-native. |
| Cross-datacenter DR required | **Ozone alone is insufficient — no built-in cross-site replication.** Supplement with DistCp-based batch replication or evaluate Ceph's native cross-datacenter RGW replication. | Ozone OM HA is intra-cluster only; no active-active geo-replication exists. |

---

## References

- [Apache Ozone 2.0.0 ASF Announcement](../resources/ozone/apache-ozone-2-release.summary.md)
- [Ozone Comparison: HDFS, Ceph, MinIO](../resources/ozone/ozone-comparison-hdfs-ceph-minio.summary.md)
- [Trino + Ozone + Polaris Lakehouse Guide (Apr 2026)](../resources/ozone/trino-ozone-polaris-lakehouse-guide.summary.md)
- [GitHub Discussion: Trino OFS Configuration](../resources/ozone/trino-ozone-ofs-configuration-discussion.summary.md)
- [GitHub Discussion: Trino Ozone Iceberg Protobuf Failures](../resources/ozone/trino-ozone-iceberg-protobuf-issue.summary.md)
- [GitHub Discussion: S3 LIST Performance Issues](../resources/ozone/ozone-s3-list-performance-issues.summary.md)
- [ASF Blog: DiDi Scales to Hundreds of PB with Ozone (Jan 2026)](../resources/ozone/ozone-didi-production-deployment.summary.md)
- [Apache Ozone 2.1.0 Release Notes (Dec 2025)](../resources/ozone/ozone-2-1-release-notes.summary.md)
- [Ozone S3 Gateway Architecture](../resources/ozone/ozone-s3-gateway-architecture.summary.md)
- [Ozone S3A Configuration Guide](../resources/ozone/ozone-s3a-configuration.summary.md)
- [Ozone OFS Interface (2.0 docs)](../resources/ozone/ozone-ofs-filesystem-interface.summary.md)
- [Ozone FSO Bucket Layout](../resources/ozone/ozone-fso-bucket-layout.summary.md)
- [Apache Ozone Roadmap (retrieved May 2026)](../resources/ozone/ozone-roadmap-2026.summary.md)
- [Disk Balancer Preview (Jan 2026)](../resources/ozone/ozone-disk-balancer-2-2.summary.md)
- [Ozone S3 Multi-Tenancy Documentation](../resources/ozone/ozone-multi-tenancy-s3.summary.md)
- [Cloudera Engineering: Iceberg on Ozone (Feb 2023 — vendor-adjacent, stale)](../resources/ozone/cloudera-iceberg-ozone-lakehouse.summary.md)

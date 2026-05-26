---
title: MinIO as Lakehouse Storage — On-Premise Enterprise 2026
date: 2026-05-21
status: complete
components: [minio, spark, trino]
constraints:
  - open-source: true
  - deployment: on-premise
---

## Overview

This report evaluates MinIO as a physical S3-compatible object storage service for an on-premise enterprise Spark + Trino lakehouse in 2026. Scope is limited to the storage layer; table formats (Iceberg, Hudi, Delta Lake) and catalogs are referenced only where they constrain storage behaviour.

**Verdict: MinIO Community Edition is eliminated. The GitHub repository was archived on April 25, 2026 — it is read-only, receives no security patches, and carries known unpatched data-integrity vulnerabilities. MinIO AIStor is the only actively maintained MinIO-codebase product, but it requires a proprietary commercial subscription and violates the open-source constraint. For a new on-premise lakehouse in 2026, the open-source path leads to SeaweedFS (primary) or Ceph RGW (large-scale), not MinIO in any form.**

---

## Evidence Quality

| Source | File | Tier | Accessible |
|---|---|---|---|
| Hacker News — MinIO repo no longer maintained | [link](../resources/minio/minio-repo-archived-2026.summary.md) | press | Yes |
| It's FOSS — MinIO moves away from open source | [link](../resources/minio/minio-open-source-ends-itsfoss.summary.md) | press | Yes |
| Blocks & Files — Admin UI removed from CE | [link](../resources/minio/minio-admin-ui-removed-blocksandfiles.summary.md) | press | Yes |
| StorageMath — AIStor vs OSS: 13,061 commits | [link](../resources/minio/minio-aistor-oss-divergence-storagemath.summary.md) | press | Yes |
| MinIO pricing page (AIStor tiers) | [link](../resources/minio/minio-aistor-free-tier-license.summary.md) | vendor | Yes |
| onidel.com — MinIO vs Ceph vs SeaweedFS vs Garage 2025 | [link](../resources/minio/minio-vs-ceph-seaweedfs-garage-2025.summary.md) | press | Yes |
| Medium (Rost Glukhov) — MinIO CE dead in 2026 | [link](../resources/minio/minio-ce-dead-2026-alternatives.summary.md) | press | Yes |
| MinIO GitHub discussion — AGPL and clients | [link](../resources/minio/minio-agpl-license-analysis.summary.md) | official | Yes |
| Trino docs — S3 filesystem support | [link](../resources/minio/trino-s3-minio-documentation.summary.md) | official | Yes |
| MinIO blog — Small object architecture | [link](../resources/minio/minio-small-objects-metadata.summary.md) | vendor | Yes |
| Dev.to — MinIO alternatives on-prem 2026 | [link](../resources/minio/minio-alternatives-open-source-2026.summary.md) | press | Yes |
| MinIO blog — AIStor vs OSS technical comparison | [link](../resources/minio/minio-aistor-technical-comparison-vendor.summary.md) | vendor | Yes |
| IOMETE — S3-compatible storage for lakehouse | [link](../resources/minio/seaweedfs-lakehouse-iceberg-iomete.summary.md) | vendor-adjacent | Yes |
| InfoQ — MinIO maintenance mode and S3 alternatives | [link](../resources/minio/minio-maintenance-mode-risk.summary.md) | press | Yes |

**Gaps and confidence limits:**

1. **Vendor-only claims**: Small object performance superiority and the 13,061-commit divergence statistics originate from or are amplified by MinIO Inc. materials. The divergence count is cross-checked by StorageMath (independent), which raises it to MEDIUM confidence. Performance claims at scale for AIStor are vendor-only (LOW confidence).
2. **Benchmark currency**: The onidel.com MinIO vs Ceph vs SeaweedFS benchmark is from 2025 (within the two-year threshold). No 2026 independent benchmarks were found.
3. **RustFS production readiness**: Could not be independently corroborated — still in alpha as of this research date. Not assessed further.
4. **AIStor pricing**: List price of ~$0.02/GB/month (~$245,760/PB/year) comes from the vendor pricing page; volume discounts are not publicly disclosed. This is accepted as a directional data point only.
5. **Expected conclusion that could not be corroborated**: Whether MinIO's S3 API is still the "gold standard" in isolation from the broader community ecosystem cannot be assessed for CE specifically, as the reference implementation is now frozen and unmaintained. AIStor's S3 compatibility is only testable under a commercial subscription.

---

## Components

### `minio`

MinIO is a high-performance, S3-compatible object store written in Go, originally released under Apache 2.0 and later relicensed to AGPLv3 in 2021. Its design is purpose-built for object storage (no block or file storage), with a simple erasure-coding architecture, no external metadata database, and inline storage for objects below 128 KiB. As of April 25, 2026, the community edition GitHub repository is fully archived ([Hacker News thread](../resources/minio/minio-repo-archived-2026.summary.md)).

### `spark`

Apache Spark uses the `S3FileIO` adapter (or legacy `HadoopFileIO`) to read and write Iceberg table data on S3-compatible storage. Spark reads and writes Parquet/ORC data files; Iceberg handles the metadata layer (manifests, manifest lists, metadata JSON). MinIO's architecture — particularly inline storage for objects below 128 KiB and the absence of an external metadata DB — is architecturally aligned with Iceberg's many-small-files metadata access pattern ([MinIO small objects blog](../resources/minio/minio-small-objects-metadata.summary.md)). However, these performance claims are vendor-published and are not independently benchmarked.

### `trino`

Official Trino documentation (release 481) explicitly lists MinIO as one of only two tested S3-compatible storage backends, alongside AWS S3 ([Trino S3 docs](../resources/minio/trino-s3-minio-documentation.summary.md)). Configuration requires path-style access (`s3.path-style-access=true`) and a custom endpoint URL. As of Trino 470 (February 2025), legacy `hive.s3.*` properties are deprecated; new deployments should use `fs.s3.*`. Trino imposes no hard dependency on MinIO specifically — any S3-compatible store with the same API surface is a drop-in replacement.

---

## Findings

### 1. MinIO Community Edition is archived and carries unpatched data-integrity vulnerabilities

Evidence: [Hacker News thread](../resources/minio/minio-repo-archived-2026.summary.md), [It's FOSS](../resources/minio/minio-open-source-ends-itsfoss.summary.md), [StorageMath](../resources/minio/minio-aistor-oss-divergence-storagemath.summary.md). Confidence: **HIGH**.

The MinIO GitHub repository was archived on April 25, 2026. It is read-only. The preceding steps were: feature stripping (March–June 2025), binary and Docker image cessation (October 2025), maintenance mode declaration (December 3, 2025), and "no longer maintained" status (February 12, 2026). MinIO Inc. has explicitly documented that 47 critical and 85 high-severity fixes exist in AIStor that were never backported to OSS, including data loss and split-brain fixes ([StorageMath analysis](../resources/minio/minio-aistor-oss-divergence-storagemath.summary.md)). Running any version of MinIO CE in production after October 2025 means running software with known, unpatched data-integrity issues.

### 2. AIStor is not open-source and violates the on-premise open-source constraint

Evidence: [MinIO pricing page](../resources/minio/minio-aistor-free-tier-license.summary.md), [AIStor vs OSS technical comparison](../resources/minio/minio-aistor-technical-comparison-vendor.summary.md). Confidence: **HIGH**.

AIStor is governed by a proprietary "MinIO AIStor Free Tier License Agreement" (effective December 22, 2025) for single-node deployments, or a commercial subscription for distributed HA deployments. Neither option qualifies as open-source under any OSI-approved license. AIStor Free is explicitly restricted to standalone (single-node) mode — no distributed clustering, no high availability — making it ineligible for any production enterprise lakehouse. AIStor Enterprise carries a list price of approximately $0.02/GB/month (~$245,760/PB/year), with a minimum engagement of approximately $96,000/year. This eliminates AIStor as a path for organisations with an open-source constraint.

### 3. The AGPLv3 license of the archived OSS version is a hard blocker for most enterprises

Evidence: [MinIO AGPL GitHub discussion](../resources/minio/minio-agpl-license-analysis.summary.md), [InfoQ maintenance mode article](../resources/minio/minio-maintenance-mode-risk.summary.md). Confidence: **HIGH**.

Even before archival, MinIO CE's AGPLv3 license created compliance risk. The network-use clause of AGPLv3 requires that if an organisation modifies MinIO and provides it as a networked service — even internally — those modifications must be made available. MinIO Inc.'s own interpretation is that client applications accessing the server via the S3 API are not derivative works, but enterprise legal departments in regulated industries (financial services, healthcare, defence) routinely reject AGPL dependencies categorically. The repository archival makes this point largely academic: the CE cannot receive security patches, and the only commercially supported path is the proprietary AIStor subscription.

### 4. No viable community fork of MinIO CE exists

Evidence: [It's FOSS](../resources/minio/minio-open-source-ends-itsfoss.summary.md), [MinIO CE dead — May 2026](../resources/minio/minio-ce-dead-2026-alternatives.summary.md). Confidence: **HIGH**.

OpenMaxIO appeared in May 2025 to restore the removed admin console. Its last commit was June 24, 2025; the project is effectively dormant. No other fork has gained community traction. The codebase's complexity (13,061 commits of divergence from AIStor, an actively developed commercial product) makes independent maintenance impractical without MinIO Inc.'s resources.

### 5. MinIO's S3 API compatibility remains the Trino reference implementation, but the CE reference is frozen

Evidence: [Trino official documentation](../resources/minio/trino-s3-minio-documentation.summary.md), [InfoQ](../resources/minio/minio-maintenance-mode-risk.summary.md). Confidence: **MEDIUM**.

Trino's official documentation (release 481) lists MinIO as one of only two explicitly tested S3-compatible backends. This establishes that the S3 API surface — path-style access, multipart uploads, bucket policies — was well-tested against MinIO. However, the CE reference implementation is now frozen at its October 2025 state. Confidence is MEDIUM because: (a) the official Trino test covers the S3 API contract, not MinIO CE specifically; (b) SeaweedFS, Ceph RGW, and other alternatives that implement the same S3 API surface are drop-in replacements at the Trino configuration level; (c) no independent post-2025 benchmark confirms that AIStor (the only active MinIO product) retains the same compatibility characteristics.

### 6. MinIO's small-object architecture is technically sound but performance claims are vendor-only

Evidence: [MinIO small objects blog](../resources/minio/minio-small-objects-metadata.summary.md), [AIStor vs OSS comparison](../resources/minio/minio-aistor-technical-comparison-vendor.summary.md). Confidence: **LOW**.

MinIO's inline storage design (objects below 128 KiB stored inline with metadata, no external metadata database) is architecturally well-suited to Iceberg's metadata access pattern — manifest files and manifest list files typically fall within this threshold. The performance claim (scaling to billions of objects with no metadata bottleneck) is plausible given the design, but is supported only by vendor blog posts and case studies. No independent benchmark of MinIO CE at billion-object scale against Ceph or SeaweedFS was found. Confidence is LOW because the evidence rests only on vendor sources.

### 7. SeaweedFS and Ceph RGW are the production-viable open-source alternatives in 2026

Evidence: [onidel.com 2025 benchmark](../resources/minio/minio-vs-ceph-seaweedfs-garage-2025.summary.md), [Dev.to alternatives evaluation](../resources/minio/minio-alternatives-open-source-2026.summary.md), [IOMETE lakehouse storage evaluation](../resources/minio/seaweedfs-lakehouse-iceberg-iomete.summary.md). Confidence: **HIGH**.

Multiple independent practitioner evaluations converge on the same conclusion: for on-premise open-source lakehouse deployments in 2026, SeaweedFS is the closest drop-in replacement for MinIO CE at typical enterprise scale (<100 TB), and Ceph RGW is the correct choice for large-scale (100+ TB) multi-tenant, multi-site deployments.

**SeaweedFS** (Apache 2.0):
- O(1) disk access regardless of object count — directly addresses Iceberg's many-small-files access pattern for manifests.
- Ships a built-in S3-compatible API — Trino, Spark, and DuckDB connect by changing the endpoint URL.
- Optional embedded Iceberg REST Catalog, eliminating the need for a separate catalog service.
- Migration from MinIO CE is a configuration change (endpoint URL and credentials).
- Production-proven at scale; mature community since 2015.

**Ceph RGW** (LGPL):
- Battle-tested at petabyte scale in multi-tenant deployments.
- Unified block, file, and object storage in a single cluster — reduces operational surface if block storage is also needed.
- Higher operational complexity; requires dedicated Ceph expertise.
- S3-compatible via RadosGW — the same Trino configuration applies.

**RustFS** (Apache 2.0): Alpha as of 2026. Not production-ready. Monitor for future evaluation.

### 8. Migration from MinIO CE to SeaweedFS or Ceph is technically low-risk

Evidence: [Trino official documentation](../resources/minio/trino-s3-minio-documentation.summary.md), [Dev.to alternatives](../resources/minio/minio-alternatives-open-source-2026.summary.md). Confidence: **HIGH**.

Trino, Spark's `S3FileIO`, and the broader S3 API ecosystem are storage-agnostic. Migration from MinIO CE to a S3-compatible alternative requires:
1. Migrating data (object copy between buckets/clusters — tools: `aws s3 sync`, `rclone`, SeaweedFS native import).
2. Updating endpoint URLs and credentials in Trino catalog properties (`s3.endpoint`, `s3.aws-access-key`, `s3.aws-secret-key`).
3. Verifying path-style access is enabled (`s3.path-style-access=true`) — this setting is the same across MinIO, SeaweedFS, and Ceph RGW.
No application-layer or Iceberg metadata changes are required.

---

## Decision Matrix

| Profile | Recommendation | Eliminated options and why |
|---|---|---|
| New on-premise lakehouse, open-source, any scale | SeaweedFS (primary) | MinIO CE: archived, known unpatched CVEs. MinIO AIStor: proprietary, violates open-source constraint. Garage: AGPLv3, single-site throughput not optimised. RustFS: alpha, not production-ready. |
| New on-premise lakehouse, open-source, >100 TB multi-tenant | Ceph RGW | Same eliminations as above. SeaweedFS viable but Ceph preferred at this scale for maturity and multi-tenancy. |
| Existing MinIO CE deployment — migrate | SeaweedFS first; Ceph if already running block storage | MinIO CE: no security patches, known data-integrity bugs. Staying is not a viable option. |
| Open-source constraint waived, commercial budget available | MinIO AIStor (evaluate alongside Weka, VAST) | Outside scope of this report. |

---

## References

- [InfoQ — MinIO maintenance mode and S3 alternatives](../resources/minio/minio-maintenance-mode-risk.summary.md)
- [Hacker News — MinIO repository no longer maintained](../resources/minio/minio-repo-archived-2026.summary.md)
- [It's FOSS — MinIO moves away from open source](../resources/minio/minio-open-source-ends-itsfoss.summary.md)
- [Blocks & Files — Admin UI removed from MinIO CE](../resources/minio/minio-admin-ui-removed-blocksandfiles.summary.md)
- [StorageMath — AIStor vs OSS: 13,061 commits of divergence](../resources/minio/minio-aistor-oss-divergence-storagemath.summary.md)
- [MinIO pricing page — AIStor tiers and licensing](../resources/minio/minio-aistor-free-tier-license.summary.md)
- [onidel.com — MinIO vs Ceph vs SeaweedFS vs Garage 2025 benchmark](../resources/minio/minio-vs-ceph-seaweedfs-garage-2025.summary.md)
- [Medium (Rost Glukhov) — MinIO CE effectively dead in 2026](../resources/minio/minio-ce-dead-2026-alternatives.summary.md)
- [MinIO GitHub discussion — AGPL and client applications](../resources/minio/minio-agpl-license-analysis.summary.md)
- [Trino official documentation — S3 filesystem support](../resources/minio/trino-s3-minio-documentation.summary.md)
- [MinIO blog — Small object architecture](../resources/minio/minio-small-objects-metadata.summary.md)
- [Dev.to — Open-source on-prem MinIO alternatives 2026](../resources/minio/minio-alternatives-open-source-2026.summary.md)
- [MinIO blog — AIStor vs OSS technical comparison (vendor)](../resources/minio/minio-aistor-technical-comparison-vendor.summary.md)
- [IOMETE — Evaluating S3-compatible storage for lakehouse](../resources/minio/seaweedfs-lakehouse-iceberg-iomete.summary.md)

---
source: https://iomete.com/resources/blog/evaluating-s3-compatible-storage-for-lakehouse
component: ceph
type: article
evidence-tier: vendor-adjacent
accessible: true
benchmark-age: n/a
date-retrieved: 2026-05-21
security:
  malicious-code: none
  suspicious-urls: none
  quality: high
---

## Summary

IOMETE is a commercial lakehouse platform vendor (vendor-adjacent, with a stake in recommending storage that works with their product). Their evaluation of S3-compatible object storage options for on-premise lakehouses includes Ceph RGW as a key option. The article provides practitioner-oriented guidance on when to choose Ceph vs. alternatives. IOMETE's assessment is broadly corroborated by independent sources on architecture but carries vendor-adjacent bias in framing. Published circa 2025.

## Key Points

**Ceph RGW strengths identified**:
- Battle-tested at massive scale: 1+ exabyte deployed globally across 2,500+ production clusters
- Reference deployments: CERN (50+ PB), Bloomberg (100+ PB), DigitalOcean
- Unified storage platform (block + file + object in one cluster) — unique among S3-compatible options
- Best multi-tenancy and multi-site replication among open-source options
- LGPL license: permissive for enterprise use
- Multi-vendor governance reduces single-company lock-in risk

**Ceph RGW weaknesses / trade-offs**:
- Large-object performance: community benchmarks show MinIO ~2× faster for 20 MB+ objects (note: MinIO Community Edition now archived)
- Lakehouse workloads mix small metadata files (Iceberg manifests: ~1–100 KB) with large Parquet files (1 MB–1 GB+); both need to be fast
- Heavyweight operationally: requires Ceph expertise for deployment and tuning
- Recommended only when teams have Ceph experience or access to support

**When Ceph RGW is recommended (IOMETE's view)**:
- Enterprise scale 100+ TB deployments
- Multi-tenant environments (multiple analytics clusters sharing storage)
- Multi-site / DR requirements
- Existing Ceph expertise on team
- Need for unified block+file+object in a single cluster

**When alternatives are preferred**:
- Simpler deployments: SeaweedFS or similar for smaller scale, Kubernetes-native setups
- Pure object storage with maximum simplicity: was MinIO (now archived as open-source)

**Trino compatibility**: IOMETE's platform (Spark/Trino-based) works with Ceph RGW as an S3-compatible backend; no Ceph-specific integration issues reported

## Security Notes

No issues detected.
Checks performed:
- Malicious or obfuscated code: none
- Suspicious URLs or redirects: none
- Content quality / AI-generated: structured evaluation article; IOMETE is a commercial vendor (vendor-adjacent label applied)

---
source: https://www.min.io/blog/minio-aistor-vs-minio-oss-technical-comparison
component: minio
type: article
evidence-tier: vendor
accessible: true
benchmark-age: n/a
date-retrieved: 2026-05-21
security:
  malicious-code: none
  suspicious-urls: none
  quality: high — vendor blog, technically detailed, cross-referenced against independent analysis
---

## Summary

MinIO Inc.'s official technical comparison of AIStor versus the archived OSS edition, published February 2026. The document quantifies the divergence and lists enterprise features exclusive to AIStor. Confidence on all quantitative claims is LOW (vendor source), but many specifics are independently corroborated by StorageMath's commit-level analysis. This document confirms AIStor is not open-source and requires a commercial subscription.

## Key Points

- **AIStor is not open-source**: Licensed under the proprietary "MinIO AIStor Free Tier License Agreement" (single-node only) or commercial subscription. Not OSI-approved.
- **Divergence statistics** (per vendor, cross-checked by StorageMath): 13,061 commits ahead of OSS, 245 unique source files, 24 new internal packages, 130+ critical/high CVE and bug fixes exclusive to AIStor.
- **Subsystems absent from OSS**:
  - AIStor Tables: Native Apache Iceberg REST Catalog V3, embedded in the object store — warehouses, namespaces, tables, views, multi-table ACID transactions, schema evolution, time travel.
  - Delta Sharing protocol support.
  - Rolling updates (zero-downtime upgrades).
  - QoS / traffic shaping.
  - FIPS 140-3 compliant cryptography.
  - GPUDirect support (NVMe-over-Fabrics for AI inference).
- **Data integrity risk in OSS**: Vendor explicitly states that data loss bugs, split-brain scenarios, and corruption issues were identified in the shared codebase and patched only in AIStor.
- **Upgrade path**: MinIO provides a documented migration from OSS to AIStor, but this requires accepting the proprietary license.

## Security Notes

No issues detected.
Checks performed:
- Malicious or obfuscated code: none — vendor blog
- Suspicious URLs or redirects: none
- Content quality / AI-generated: technically substantive, consistent with independent commit-level analysis

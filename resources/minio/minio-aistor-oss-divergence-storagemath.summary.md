---
source: https://storagemath.com/posts/minio-aistor-oss-divergence-open-source-strategy/
component: minio
type: article
evidence-tier: press
accessible: true
benchmark-age: n/a
date-retrieved: 2026-05-21
security:
  malicious-code: none
  suspicious-urls: none
  quality: high — StorageMath, independent storage analysis blog with disclosed methodology
---

## Summary

StorageMath performed a commit-level analysis of the divergence between MinIO AIStor and MinIO OSS, quantifying the gap at 13,061 commits. The analysis, which cross-references the co-founder's own public GitHub Gist, finds that AIStor has accumulated 245 unique source files, 24 new internal packages, and over 130 critical/high-severity CVE and bug fixes that do not exist in the community edition. Entire subsystems including Iceberg catalog, Delta Sharing, rolling updates, and QoS exist only in AIStor.

## Key Points

- **Commit divergence**: 13,061 commits separate AIStor from the frozen OSS codebase as of February 2026 analysis.
- **Unique source files in AIStor**: 245 files absent from OSS.
- **New internal packages**: 24 packages added to AIStor not present in OSS.
- **Security fixes exclusive to AIStor**: 47+ critical fixes and 85+ high-severity fixes — 130+ total critical/high not backported to OSS.
- **Missing subsystems in OSS**: Iceberg REST Catalog (V3), Delta Sharing, rolling updates, QoS, FIPS 140-3 cryptography.
- **Data integrity risk**: MinIO identified data loss bugs and split-brain scenarios in the shared codebase and fixed them only in AIStor — every MinIO OSS production deployment carries known, unpatched data integrity vulnerabilities.
- **Source**: Co-founder Harshavardhana published the divergence statistics himself at https://gist.github.com/harshavardhana/0f44addfcafd1778bb503078834c74d1

## Security Notes

No issues detected.
Checks performed:
- Malicious or obfuscated code: none — analytical article, no executable content
- Suspicious URLs or redirects: none
- Content quality / AI-generated: technical analysis with cited methodology, credible

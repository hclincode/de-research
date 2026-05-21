---
source: https://github.com/seaweedfs/seaweedfs
component: seaweedfs
type: github-repo
evidence-tier: official
accessible: true
benchmark-age: n/a
date-retrieved: 2026-05-21
security:
  malicious-code: none
  suspicious-urls: none
  quality: high
---

## Summary

SeaweedFS is an Apache 2.0 distributed storage system written in Go, designed for object storage (S3), file systems, and Iceberg tables. It targets billions of files with O(1) disk access via a Haystack-inspired architecture. As of May 20, 2026, version 4.27 is current, with 32,400+ GitHub stars and 301 total releases. The project added a built-in Iceberg REST Catalog enabling direct Spark, Trino, Dremio, DuckDB, RisingWave, and Apache Doris integration without an external metastore.

## Key Points

- License: Apache 2.0 (confirmed; separate enterprise edition at seaweedfs.com under EULA)
- Architecture: master servers (Raft consensus, odd count), volume servers (32 GB or large-disk variant volumes, append-only), filer (optional, with 12+ metadata backends), S3 gateway (port 8333)
- Small file optimization: 40 bytes disk overhead per file, 16 bytes in-memory metadata; packs many small objects into larger volumes eliminating per-file disk seeks
- Inspired by: Facebook Haystack, f4, Tectonic Filesystem, Google Colossus
- Rack and datacenter-aware replication strategies
- TTL expiration, erasure coding for warm/cold tiering, cloud tiering to AWS/GCP/Azure
- Iceberg REST Catalog built-in (no external catalog service required); Trino integration tests pass in CI (v4.x, Feb 2026)
- S3 gateway supports: buckets, multipart upload, versioning, lifecycle policies, object locking, CORS, IAM, OIDC/STS
- Latest active release cadence: multiple releases per week in May 2026 (4.23 through 4.27 in three weeks)

## Security Notes

No issues detected.
Checks performed:
- Malicious or obfuscated code: None — public Apache 2.0 Go project with active community review
- Suspicious URLs or redirects: None
- Content quality / AI-generated: High — official project repository

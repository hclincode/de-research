---
source: https://ceph.io/en/news/blog/2024/v19-2-0-squid-released/
component: ceph
type: article
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

Ceph Squid (v19.2.0) is the 19th stable release of Ceph, released in September 2024. It follows Reef (v18) and introduces significant RGW enhancements including AWS-compatible IAM APIs via User Accounts, redesigned S3 Bucket Notifications metadata layout, improved S3 multipart upload handling in multi-site SSE scenarios, new bucket versioning index tools, and an RGW S3 Analytics Grafana dashboard. The most recent point release is v19.2.3, released July 28, 2025, with an estimated end-of-life in September 2026.

## Key Points

- **Release date**: September 2024 (v19.2.0); latest point release v19.2.3 on 2025-07-28
- **EOL**: Estimated September 2026 for the Squid branch
- **RGW / S3 improvements**:
  - User Accounts feature unlocks AWS-compatible IAM APIs (users, keys, groups, roles, policy) for self-service management
  - Redesigned S3 Bucket Notifications: each Topic stored as separate RADOS object; bucket notification config stored in bucket attribute; supports multisite replication via metadata sync; scales to many topics
  - S3 multipart uploads using Server-Side Encryption now replicate correctly in multi-site (previously corrupted on decryption)
  - `radosgw-admin bucket resync encrypted multipart` tool to repair legacy SSE multipart objects
  - `GetObject`/`HeadObject` now return `x-rgw-replicated-at` header for multisite replication tracking
  - `GetObject`/`HeadObject` support `partNumber` query parameter to read a specific part of a completed multipart upload
  - `PutObjectLockConfiguration` can now enable Object Lock on existing versioning-enabled buckets (not just at creation time)
  - S3 policy now enforces ARN-based conditionals
  - CVE-2023-43040 (STS authentication bypass) fixed in v19.2.3
  - Copying an object to itself no longer causes data loss (bug fix)
- **RGW S3 Analytics Dashboard**: new Grafana dashboard for per-bucket and per-user analytics (GETs, PUTs, Deletes, Copies, list metrics)
- **Versioned bucket tools**: `radosgw-admin bucket check olh --fix` to remove unnecessary OLH entries from versioned bucket indexes
- **Successor**: Ceph Tentacle (v20) released November 2025

## Security Notes

No issues detected.
Checks performed:
- Malicious or obfuscated code: none
- Suspicious URLs or redirects: none
- Content quality / AI-generated: official Ceph Foundation blog post, release announcement

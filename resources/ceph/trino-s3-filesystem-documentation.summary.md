---
source: https://trino.io/docs/current/object-storage/file-system-s3.html
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

Official Trino documentation for native S3 filesystem integration. Specifies the S3 features that Trino's native implementation relies on, including configuration options for path-style access, multipart uploads, and server-side encryption. Critically, the documentation explicitly states that only AWS S3 and MinIO are formally tested for S3 compatibility with Trino; all other S3-compatible storage systems (including Ceph RadosGW) are user-supported. MinIO Community Edition has since been archived (early 2026), leaving Ceph RGW as the principal open-source S3-compatible backend without formal Trino compatibility testing.

## Key Points

**S3 features Trino uses**:
- **Path-style access**: Configurable via `s3.path-style-access=true` property; must be explicitly enabled for backends that do not support virtual-hosted-style (or where wildcard DNS is not available)
- **Multipart uploads**: Controlled via `s3.streaming.part-size` (5 MB–256 MB); used for large file uploads from Trino to object storage
- **Server-side encryption**: SSE-S3, SSE-KMS, and SSE-C all configurable
- **Canned ACLs**: Configurable for uploaded objects
- **Cross-region access**: Optional

**Compatibility statement**:
- "While Trino is designed to support S3-compatible storage systems, **only AWS S3 and MinIO are tested for compatibility**. For other storage systems, perform your own testing and consult your vendor for more information."
- Legacy Trino S3 implementation has been removed; all configurations must use the current native implementation
- No known Trino-specific technical blockers for Ceph RGW reported in community, but no formal certification exists

**Path-style access importance for Ceph**:
- Ceph RadosGW supports both path-style and virtual-hosted-style
- Path-style is simpler to configure (no wildcard DNS, no per-bucket certificates)
- Trino's `s3.path-style-access=true` setting is the correct configuration when using Ceph RGW without wildcard DNS

**Practical interoperability**:
- Community reports of Trino + Ceph RGW working in production exist (no formal documentation)
- S3 Select support in Trino can be leveraged with Ceph's S3 Select implementation for predicate pushdown
- No documented Trino features that Ceph RGW is known to be missing (beyond the formal testing gap)

## Security Notes

No issues detected.
Checks performed:
- Malicious or obfuscated code: none
- Suspicious URLs or redirects: none
- Content quality / AI-generated: official Trino project documentation (trino.io)

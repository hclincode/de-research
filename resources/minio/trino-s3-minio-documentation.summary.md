---
source: https://trino.io/docs/current/object-storage/file-system-s3.html
component: minio
type: article
evidence-tier: official
accessible: true
benchmark-age: n/a
date-retrieved: 2026-05-21
security:
  malicious-code: none
  suspicious-urls: none
  quality: high — official Trino project documentation
---

## Summary

Official Trino documentation (release 481, current as of 2026) confirms that only AWS S3 and MinIO are explicitly tested for S3-compatible filesystem compatibility. The documentation includes a MinIO-specific configuration example using path-style access and a custom endpoint. As of Trino 470 (February 2025), the legacy S3 file system properties are deprecated in favour of the native `fs.s3.*` implementation.

## Key Points

- **Explicit MinIO support**: Trino documentation states "only AWS S3 and MinIO are tested for compatibility" for the S3 filesystem connector.
- **Configuration example**: `fs.s3.enabled=true`, `s3.endpoint=http://minio:9080/`, `s3.path-style-access=true` — path-style access is required for MinIO (virtual-hosted style is not the default).
- **Deprecation of legacy S3 config (Trino 470, Feb 2025)**: All legacy `hive.s3.*` catalog properties are deprecated. New deployments should use `fs.s3.*` native implementation.
- **Compatibility gap risk**: While the S3 API surface tested against MinIO is stable, the AIStor S3 API documentation diverges from the archived OSS documentation. Operators should verify endpoint compatibility against AIStor or an alternative when migrating from CE.
- **Practical implication for CE migration**: SeaweedFS, Ceph RGW, and other S3-compatible stores can be substituted by changing the endpoint URL and credentials — Trino itself imposes no hard dependency on MinIO specifically.
- **Native Iceberg REST Catalog**: Trino supports Iceberg REST Catalog natively, decoupling catalog selection from storage selection.

## Security Notes

No issues detected.
Checks performed:
- Malicious or obfuscated code: none — official project documentation
- Suspicious URLs or redirects: none
- Content quality / AI-generated: official Trino documentation, authoritative

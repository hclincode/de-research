---
source: https://trino.io/docs/current/object-storage/file-system-s3.html
component: weka
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

Official Trino documentation for the native S3 file system implementation (as of Trino 481). Trino's native S3 client is enabled with `fs.s3.enabled=true` and supports path-style access via `s3.path-style-access=true`. The documentation explicitly states that "only AWS S3 and MinIO are tested for compatibility" — Weka S3 is not listed as a tested or certified S3-compatible target. However, the configuration mechanism (endpoint URL + path-style access + access/secret key) is the same for any S3-compatible storage, and Weka's S3 gateway supports the same addressing modes. The documentation includes an explicit MinIO configuration example using path-style access and a custom endpoint, which is directly analogous to how Weka S3 would be configured.

## Key Points

- **Native S3 client config key**: `fs.s3.enabled=true` (NOT the legacy `hive.s3.*` namespace).
- **Path-style access**: `s3.path-style-access=true` — required for Weka S3 because Weka S3 buckets are not hosted at DNS-resolvable subdomains. This flag must be set in Trino catalog properties.
- **Endpoint configuration**: `s3.endpoint=http(s)://<weka-s3-gateway-host>:<port>/` — analogous to MinIO configuration.
- **Tested targets**: Official docs say only AWS S3 and MinIO are tested. Weka is NOT a tested integration. This means S3 API edge cases or lesser-used operations may behave differently and would require user validation.
- **Credential model**: Standard AWS-style access key / secret key, compatible with Weka's S3 user authentication model.
- **Legacy support**: The legacy `hive.s3.*` namespace is still supported but deprecated. Weka S3 would work with either the native or legacy client, but the native client (`fs.s3.enabled=true`) is the current recommended path.
- **Important implication**: No official Trino+Weka integration exists. Users configure Trino to treat Weka as a generic S3-compatible endpoint. Any gaps in Weka's S3 API completeness will manifest as runtime errors, not configuration-time errors.

## Security Notes

No issues detected. Official project documentation.
Checks performed:
- Malicious or obfuscated code: n/a
- Suspicious URLs or redirects: none
- Content quality / AI-generated: high — official Trino project documentation with version-tagged releases

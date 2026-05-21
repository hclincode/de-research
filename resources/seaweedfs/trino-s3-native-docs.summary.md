---
source: https://trino.io/docs/current/object-storage/file-system-s3.html
component: trino
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

The Trino official S3 file system documentation covers native S3 support configuration. The native S3 implementation is enabled with `fs.s3.enabled=true` (not `fs.native-s3.enabled` as sometimes referenced). Path-style access is supported via `s3.path-style-access=true`. Trino explicitly states that only AWS S3 and MinIO are tested for compatibility; all other S3-compatible systems require independent validation.

## Key Points

- Native S3 property: `fs.s3.enabled=true` (set in catalog properties file)
- Path-style access: `s3.path-style-access=true` — enables path-style for all requests (required for SeaweedFS since it does not provision per-bucket DNS subdomains)
- Custom endpoint: `s3.endpoint=<url>` — required for SeaweedFS (e.g., `http://seaweedfs-host:8333`)
- Region: `s3.region=us-east-1` — typically set to a placeholder for on-premise deployments
- Security mapping: `s3.endpoint` and `s3.region` can be overridden per-bucket via security mapping configuration
- Explicit compatibility statement: "While Trino is designed to support S3-compatible storage systems, only AWS S3 and MinIO are tested for compatibility. For other storage systems, perform your own testing and consult your vendor for more information."
- No documented SeaweedFS-specific incompatibilities in the official docs, but the absence of testing means edge cases may exist

## Security Notes

No issues detected.
Checks performed:
- Malicious or obfuscated code: None — official Trino documentation
- Suspicious URLs or redirects: None
- Content quality / AI-generated: High — official Apache-licensed project documentation

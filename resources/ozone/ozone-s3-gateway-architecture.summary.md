---
source: https://ozone.apache.org/docs/core-concepts/architecture/s3-gateway/
component: ozone
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

This official Ozone documentation page describes the S3 Gateway (S3G) as a stateless protocol adapter that translates REST S3 operations into native Ozone calls. Metadata routes through the Ozone Manager; bulk data streams directly between S3G and Datanodes. Multiple S3G instances can run behind a load balancer for horizontal scaling.

## Key Points

- **Architecture**: Stateless service; multiple instances supported behind load balancers. Metadata via OM; data directly with Datanodes.
- **S3 mapping**: All S3 buckets stored under `/s3v` volume by default (configurable via `ozone.s3g.volume.name`). S3 objects map to Ozone keys.
- **Supported operations**: Bucket create/delete, object PUT/GET/DELETE, object listing, multipart uploads, ETags for multipart operations.
- **Authentication**: AWS Signature V4 supported; Signature V2 explicitly NOT supported. Kerberos supported for secure clusters. Anonymous/dummy access in unsecured mode. OM validates access/secret keys.
- **STS/IAM**: Not mentioned in documentation — no STS endpoint documented.
- **Path style**: Default is path-style addressing (`host/bucket/key`); virtual-hosted style configurable via `ozone.s3g.domain.name`.
- **Trino compatibility**: Not addressed in documentation — requires separate configuration.

## Security Notes

No issues detected.
Checks performed:
- Malicious or obfuscated code: none (official Apache documentation)
- Suspicious URLs or redirects: none
- Content quality / AI-generated: official architecture documentation — authoritative

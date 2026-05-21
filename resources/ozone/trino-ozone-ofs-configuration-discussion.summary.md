---
source: https://github.com/apache/ozone/discussions/5004
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

This GitHub Discussion on the Apache Ozone repository documents practitioners attempting and failing to configure Trino with OFS (Ozone File System) native protocol. The core finding is that open-source Trino does not natively support the `ofs://` scheme, and the recommended production path is the S3 Gateway.

## Key Points

- **OFS not natively supported in Trino**: Adding Ozone filesystem JARs to Trino plugin directories does not enable `ofs://` scheme recognition. Users get "No FileSystem for scheme 'ofs'" errors.
- **S3 Gateway workaround**: Deploy S3G instances behind a load balancer (e.g., Nginx), then configure Trino Iceberg catalog with `hive.s3.endpoint`, `hive.s3.aws-access-key`, `hive.s3.aws-secret-key`, and `hive.s3.path-style-access=true`.
- **Performance caveat**: S3G introduces an additional network hop compared to direct OFS access; multiple S3G instances behind a load balancer mitigate the impact.
- **Starburst Enterprise exception**: Starburst's commercial distribution supports `ofs://` natively via `fs.hadoop.enabled=true` — this is not available in open-source Trino.

## Security Notes

No issues detected.
Checks performed:
- Malicious or obfuscated code: none (GitHub community discussion)
- Suspicious URLs or redirects: none
- Content quality / AI-generated: practitioner troubleshooting discussion — reliable for documenting known gaps

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

Official Trino documentation for the native S3 filesystem implementation (replacing the legacy Hive S3 connector). Trino's native S3 implementation (`fs.native-s3.enabled=true`) is the recommended approach for accessing S3-compatible object storage in Trino 480+. The documentation explicitly states that only AWS S3 and MinIO are tested for compatibility. JuiceFS S3 Gateway is not listed as tested but implements the S3 API; path-style access configuration should allow it to work.

## Key Points

- Enable native S3: `fs.native-s3.enabled=true` in catalog properties file
- Path-style access: `s3.path-style-access=true` (required for non-AWS S3-compatible systems that don't support virtual-host-style)
- Endpoint: `s3.endpoint=http://<host>:<port>` for non-AWS S3-compatible systems
- Credentials: `s3.aws-access-key` and `s3.aws-secret-key`
- Region: `s3.region=us-east-1` (or any value for non-AWS systems that ignore region)
- Officially tested: AWS S3, MinIO — JuiceFS gateway NOT explicitly listed as tested
- Legacy Hive connector (`hive.s3.*` properties): still supported but deprecated; native S3 preferred
- Alternative for JuiceFS: use Hadoop Java SDK (jfs:// scheme) with Trino's legacy Hive connector — avoids S3 gateway entirely; this is the more tested integration path
- Trino version context: Trino 481 (current at research date May 2026)

## Security Notes

No issues detected.
Checks performed:
- Malicious or obfuscated code: Official Trino documentation
- Suspicious URLs or redirects: None
- Content quality / AI-generated: High quality; official project documentation

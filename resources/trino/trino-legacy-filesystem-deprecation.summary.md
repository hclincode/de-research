---
source: https://trino.io/blog/2025/02/10/old-file-system.html
component: trino
type: article
evidence-tier: official
accessible: true
benchmark-age: n/a
date-retrieved: 2026-05-21
security:
  malicious-code: none
  suspicious-urls: none
  quality: high — official Trino engineering blog
---

## Summary

Trino 470 (February 2025) deprecated its legacy Hadoop-based file system support — the `hive.s3.*`, `hive.azure.*`, `hive.gcs.*` property namespaces — and is replacing them with native file system implementations (`fs.native-s3.enabled=true`, etc.). HDFS can still be enabled via `fs.hadoop.enabled=true` but this is a compatibility shim slated for removal. The direction of travel is clear: Trino is moving away from Hadoop dependencies and toward native S3 as the primary on-premise storage interface.

## Key Points

- **Deprecated (Trino 470+)**: `hive.s3.*`, `hive.azure.*`, `hive.gcs.*` configuration properties. These generate deprecation warnings and will be removed within "weeks or months."
- **Replacement**: `fs.native-s3.enabled=true` for S3 and S3-compatible storage. Uses Trino's own S3 implementation — no Hadoop dependencies.
- **HDFS path**: `fs.hadoop.enabled=true` — works but is the compatibility shim, not the forward path.
- **Impact on storage selection**:
  - **Ozone via S3 gateway**: Fully aligned with Trino's native S3 direction. Use `fs.native-s3.enabled=true` with Ozone's S3 endpoint and `s3.path-style-access=true`.
  - **Ceph via RadosGW**: Fully aligned. Same S3-native configuration.
  - **HDFS directly**: Supported via shim but not the forward path. Operational debt accumulates as Trino evolves.
- **No removal date given** for `fs.hadoop.enabled`, but "within weeks or months" for Hive S3 properties means the Hadoop dependency is being actively unwound.

**Key conclusion**: New Trino deployments should use S3-compatible storage backends. Choosing HDFS as the primary storage for a Spark+Trino lakehouse creates Trino-side technical debt.

## Security Notes

No issues detected.
Checks performed:
- Malicious or obfuscated code: none
- Suspicious URLs or redirects: none
- Content quality / AI-generated: official engineering blog — authoritative

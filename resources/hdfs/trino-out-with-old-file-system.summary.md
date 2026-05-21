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
  quality: high
---

## Summary

This Trino project blog post (February 10, 2025) announces the deprecation of all legacy file system properties as of Trino 470. The post is the authoritative statement of Trino's intent to decouple from Hadoop libraries. It distinguishes between legacy cloud-storage access (via Hadoop FS shims for S3, Azure, GCS) — which is deprecated and will be removed — and genuine HDFS access via `fs.hadoop.enabled=true`, which is preserved. The removal timeline for the deprecated cloud-storage properties was stated as "within the next weeks or months" from February 2025, with no specific version cited.

## Key Points

- Trino 470 (February 2025): deprecated all `hive.azure`, `hive.cos`, `hive.gcs`, and `hive.s3` properties — these are the legacy Hadoop FS shims for cloud storage.
- Trino 458 (September 2024): legacy Hadoop FS layer first declared legacy; users advised to migrate.
- `fs.hadoop.enabled=true` is preserved specifically for genuine HDFS access — it is NOT deprecated in 470.
- For cloud storage (S3, Azure, GCS), users must migrate to native implementations: `fs.native-s3.enabled=true`, `fs.native-azure.enabled=true`, `fs.native-gcs.enabled=true`.
- The post states removal of deprecated cloud-storage code would come "within the next weeks or months."
- Users continuing to use HDFS are explicitly told they "can use `fs.hadoop.enabled=true`" — this is the supported path.
- The broader decoupling effort is tracked in GitHub issue #15921 ("Decouple Trino from Hadoop and Hive codebases").
- Starburst Enterprise (commercial Trino distribution) continues to support HDFS for enterprise customers.

## Security Notes

No issues detected.
Checks performed:
- Malicious or obfuscated code: n/a (blog post)
- Suspicious URLs or redirects: none; official trino.io domain
- Content quality / AI-generated: high; official Trino project blog

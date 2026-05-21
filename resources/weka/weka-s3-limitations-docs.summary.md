---
source: https://docs.weka.io/additional-protocols/s3/s3-limitations
component: weka
type: article
evidence-tier: official
accessible: true
benchmark-age: n/a
date-retrieved: 2026-05-21
security:
  malicious-code: none
  suspicious-urls: none
  quality: high — official product documentation
---

## Summary

Official Weka documentation page enumerating S3 API limitations. The key constraints relevant to lakehouse workloads include: multipart upload requires 2x object size in free bucket capacity, only the S3 `Expiration` lifecycle action is supported (no Transition actions for tiered storage), object keys are capped at 1024 characters, and directory segments in keys are limited to 255 characters. The S3 layer does not support the full AWS S3 API surface — it covers core operations (GET, PUT, DELETE, LIST, multipart upload, bucket policy, object tagging) but some less-common S3 features may be absent.

## Key Points

- **Multipart capacity constraint**: During multipart upload, the bucket must have free space equal to at least 2x the upload object size — this is a relevant operational constraint for large Parquet file writes or compaction operations.
- **Lifecycle limitation**: Only `Expiration` lifecycle action is supported; `Transition` (storage class tiering) is not supported via the S3 interface. Iceberg's S3-based lifecycle management for expired snapshots will work (expiration), but automated storage tiering via S3 lifecycle will not.
- **Object key limits**: 1024-character key limit and 255-character directory segment limit. These are unlikely to be binding for standard Iceberg or Parquet file paths but are documented constraints.
- **S3 API completeness**: Core CRUD operations, multipart uploads, bucket policies, object tagging, and versioning are supported. The exact set of unsupported APIs is not enumerated in publicly accessible search snippets — consult docs.weka.io directly for the full list.
- **S3 checksum validation**: End-to-end checksum validation with AWS-aligned algorithms is supported, including trailer-based signed chunked uploads — ensuring data integrity for Spark/Trino writes.

## Security Notes

No issues detected. Official vendor documentation.
Checks performed:
- Malicious or obfuscated code: n/a
- Suspicious URLs or redirects: none
- Content quality / AI-generated: high — structured technical documentation

---
source: https://docs.weka.io/additional-protocols/s3
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

Weka's official documentation for its S3 protocol implementation. Weka exposes an S3-compatible API as an additional protocol layer on top of WekaFS, enabling object-store clients (including Spark and Trino S3 connectors) to access Weka storage without the proprietary Weka client installed. Both path-style and virtual-hosted-style URL formats are supported. Authentication uses local Weka S3 users with IAM policy attachment, STS AssumeRole for temporary tokens, and service accounts with restricted permissions. End-to-end checksum validation is aligned with AWS algorithms. S3 bucket notifications can be sent to Kafka targets. The S3 gateway is documented as part of the "additional protocols" layer, meaning it runs alongside the core WekaFS POSIX interface.

## Key Points

- **Path-style access**: Supported — `s3.path-style-access=true` is a valid configuration mode, enabling Trino's native S3 client (`fs.native-s3.enabled=true`) to connect to Weka S3.
- **Virtual-hosted-style**: Also supported, giving flexibility on how Trino or Spark resolves bucket URLs.
- **Authentication**: IAM-compatible policy model; STS AssumeRole tokens supported — compatible with how Trino and Spark pass credentials to S3 endpoints.
- **Multipart uploads**: Supported with MD5 ETag per part. Bucket must have free capacity of at least 2x the object size during multipart uploads (capacity planning constraint for large writes).
- **Lifecycle policies**: Only the `Expiration` action is supported — no `Transition` (to tiered storage class) support in the S3 layer.
- **Bucket notifications**: S3 event notifications delivered to Kafka — relevant for change data capture and streaming ingest patterns.
- **Object key limit**: 1024 characters; directory path segments limited to 255 characters.
- **S3 gateway position**: The S3 API is an "additional protocol" — the core system is still POSIX/WekaFS. S3 performance depends on gateway throughput, not raw WekaFS POSIX throughput.

## Security Notes

No issues detected. Official vendor documentation.
Checks performed:
- Malicious or obfuscated code: n/a (documentation page)
- Suspicious URLs or redirects: none
- Content quality / AI-generated: high — structured technical documentation with version history

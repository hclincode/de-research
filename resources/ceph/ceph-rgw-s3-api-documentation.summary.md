---
source: https://docs.ceph.com/en/latest/radosgw/s3/
component: ceph
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

The official Ceph RadosGW S3 API documentation describes a comprehensive S3-compatible interface supporting the majority of the AWS S3 REST API surface used by analytics engines. RadosGW supports both path-style and virtual-hosted-style bucket access, multipart uploads (including per-part reads via `partNumber`), object tagging, bucket lifecycle policies, bucket versioning, S3 Object Lock, server-side encryption (SSE-S3, SSE-KMS, SSE-C), presigned URLs, and bucket notifications. Path-style access is the simpler deployment mode (no wildcard DNS required); virtual-hosted-style requires wildcard DNS or per-bucket certificate configuration.

## Key Points

- **Access styles**: Both path-style (`/bucket/key`) and virtual-hosted-style (`bucket.hostname/key`) supported; configured via `rgw_dns_name` or zonegroup hostnames
- **Multipart uploads**: Fully supported; upload parts, complete/abort operations; `partNumber` query param added in Squid for per-part reads
- **Object tagging**: Supported (PutObjectTagging, GetObjectTagging, DeleteObjectTagging); usable as lifecycle filter predicate
- **Bucket lifecycle policies**: Supported — Expiration, NoncurrentVersionExpiration, expired delete marker cleanup; filterable by prefix and/or tags
- **Bucket versioning**: Fully supported; includes suspend/enable
- **S3 Object Lock**: Supported — compliance and governance modes; PutObjectLockConfiguration now works on existing versioned buckets (Squid+)
- **Server-side encryption**: SSE-S3, SSE-KMS (Vault integration), SSE-C all supported
- **Presigned URLs**: Supported
- **Bucket notifications**: S3-compatible notifications to HTTP endpoints, AMQP, Kafka; topic-based; multisite-replicable (Squid+)
- **S3 Select**: Supported (pushdown of CSV/Parquet/JSON filtering to cluster)
- **IAM APIs**: AWS-compatible IAM users, groups, roles, policies via User Accounts feature (Squid+)
- **Known gaps / not fully supported**:
  - Not all AWS S3 API operations are implemented; some advanced features (e.g., replication configuration, analytics, inventory) are absent or partially implemented
  - Trino's official compatibility testing covers only AWS S3 and MinIO; Ceph is user-supported
- **Bucket index sharding**: Dynamic resharding (automatic) detects large buckets and increases shard count; target is ≤100,000 entries per shard; resharding blocks writes briefly but not reads

## Security Notes

No issues detected.
Checks performed:
- Malicious or obfuscated code: none
- Suspicious URLs or redirects: none
- Content quality / AI-generated: official Ceph project documentation (docs.ceph.com)

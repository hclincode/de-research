---
source: https://blog.min.io/small_objects/
component: minio
type: article
evidence-tier: vendor
accessible: true
benchmark-age: n/a
date-retrieved: 2026-05-21
security:
  malicious-code: none
  suspicious-urls: none
  quality: high — vendor blog, technically detailed
---

## Summary

MinIO's official blog describes its architecture for handling small object workloads, relevant to Iceberg manifest files and metadata objects. Objects smaller than 128 KiB are stored inline with metadata, reducing the IOPS overhead for read/write operations. MinIO claims to avoid the external metadata database bottleneck that affects competing systems, allowing it to scale to billions of small objects without centralized chokepoints. These claims are vendor-published and should be treated as LOW confidence without independent corroboration.

## Key Points

- **Inline storage below 128 KiB**: Small objects are stored inline with their metadata, reducing disk seeks per operation.
- **No external metadata database**: MinIO's architecture does not depend on a separate metadata database. This is a design differentiator against systems that rely on external stores (e.g. RocksDB-backed metadata servers).
- **Iceberg relevance**: Iceberg manifest files (typically 1–100 KB) and manifest list files fall within the inline threshold — MinIO's architecture is theoretically well-suited to the Iceberg metadata access pattern.
- **Scaling claim**: Vendor claims production deployments have scaled to billions of objects with no observable metadata performance degradation.
- **Confidence note**: This content is vendor-published. The architectural claims (inline storage, no external metadata DB) are verifiable design facts; the performance claims at scale are not independently benchmarked in this document.
- **Status caveat**: This blog post describes AIStor/historical MinIO architecture. The archived OSS version shares the same underlying storage engine, but AIStor continues to receive performance improvements not available to OSS users.

## Security Notes

No issues detected.
Checks performed:
- Malicious or obfuscated code: none — vendor blog
- Suspicious URLs or redirects: none
- Content quality / AI-generated: vendor marketing with technical substance

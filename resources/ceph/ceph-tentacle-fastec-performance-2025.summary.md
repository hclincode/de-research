---
source: https://ceph.io/en/news/blog/2025/tentacle-fastec-performance-updates/
component: ceph
type: article
evidence-tier: official
accessible: true
benchmark-age: 2025
date-retrieved: 2026-05-21
security:
  malicious-code: none
  suspicious-urls: none
  quality: high
---

## Summary

Published on ceph.io in 2025, this post describes FastEC — a new erasure coding I/O implementation introduced in Ceph Tentacle (v20). FastEC adds support for partial reads and partial writes in erasure-coded pools, which previously required reading/writing full stripes. The optimizations primarily benefit block (RBD) and file (CephFS) workloads, with secondary benefits for object storage (RGW) when handling smaller-sized objects. The default EC plugin was changed from Jerasure to ISA-L (Intel ISA-L) for improved CPU-accelerated encoding/decoding.

## Key Points

- **FastEC**: New erasure coding I/O code path in Tentacle (v20.2.0, November 2025)
- **Partial reads**: EC reads no longer require reading an entire stripe when only part of it is needed — directly reduces read amplification for small/partial-object reads
- **Partial writes**: Allows writing to part of a stripe without a full read-modify-write cycle
- **Performance improvement**: 2–3× erasure coding performance improvement cited (primarily for RBD/CephFS); RGW benefits for small objects stated but not specifically quantified separately
- **Default plugin changed**: Jerasure → ISA-L (Intel ISA-L leverages hardware instruction sets for faster encode/decode)
- **Relevance to lakehouse**:
  - Iceberg manifest files and Parquet footers (typically 1–128 KB) are partial-read patterns that historically triggered full-stripe I/O in Ceph EC pools
  - FastEC partial reads specifically address this; measured benefit for RGW object workloads not yet independently confirmed
  - EC 2+2 (or similar) pools remain viable for large Parquet file bodies; the partial-read gap narrows with FastEC
- **Limitation**: FastEC is new in Tentacle (Nov 2025); production track record is limited as of May 2026. Teams on Squid (v19, EOL Sep 2026) do not have FastEC.

## Security Notes

No issues detected.
Checks performed:
- Malicious or obfuscated code: none
- Suspicious URLs or redirects: none
- Content quality / AI-generated: official Ceph Foundation blog; technical description of engineering improvements

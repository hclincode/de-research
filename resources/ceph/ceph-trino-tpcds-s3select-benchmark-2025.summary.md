---
source: https://ceph.io/en/news/blog/2025/tpc-bench-trino/
component: ceph
type: article
evidence-tier: vendor-adjacent
accessible: true
benchmark-age: 2025
date-retrieved: 2026-05-21
security:
  malicious-code: none
  suspicious-urls: none
  quality: high
---

## Summary

Published on ceph.io in 2025, this post presents TPC-DS benchmark results for Trino running against Ceph Object Storage with the S3 Select pushdown feature enabled. The benchmark was conducted by the IBM Storage Ceph team (vendor-adjacent). It demonstrates significant query acceleration via S3 Select — an S3 server-side query capability that filters data within the object store before returning it to the client, reducing network transfer. Results should be treated as vendor-adjacent (IBM) and independently corroborated before relying on the magnitude of speedup.

## Key Points

**Benchmark setup**:
- Engine: Trino with S3 Select feature enabled against Ceph RadosGW
- Scale: TPC-DS at 1 TB, 2 TB, and 3 TB scale factors
- 72 TPC-DS queries executed at each scale factor
- Published on ceph.io (February 2025 on IBM community, 2025 on ceph.io)

**Performance results (S3 Select enabled vs. disabled)**:
- Average query speedup: **2.5× faster** with S3 Select
- Best case: **9× improvement** on individual queries
- Network data transfer reduction: 144 TB less data moved compared to baseline (without S3 Select)

**How S3 Select works in this context**:
- Trino pushes predicate/projection evaluation into the Ceph cluster
- Ceph filters Parquet/CSV data server-side before returning results
- Reduces client-side CPU and network I/O, especially for selective queries

**Caveats**:
- Speedup is heavily query-dependent; aggregate or full-scan queries will see less benefit
- S3 Select requires Trino to be configured for Ceph's S3 Select implementation
- IBM is vendor-adjacent (sells IBM Storage Ceph); independent reproduction of these numbers not found
- S3 Select is a pushdown optimization, not a replacement for proper storage-level performance tuning

**Implication for lakehouse**: S3 Select can partially mitigate Ceph's small-file overhead by filtering at the storage tier, but this is an optional optimization layer, not a substitute for baseline performance.

## Security Notes

No issues detected.
Checks performed:
- Malicious or obfuscated code: none
- Suspicious URLs or redirects: none
- Content quality / AI-generated: structured benchmark blog, IBM authorship disclosed, methodology partially described

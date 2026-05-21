---
source: https://ceph.io/en/news/blog/2025/benchmarking-object-part1/
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

A three-part benchmarking series (Parts 1–3) published on ceph.io in 2025, conducted by the IBM Storage Ceph Performance and Interoperability team on production-grade all-NVMe IBM Storage Ceph Ready Nodes with 100 GE networking. The series covers large-object throughput, small-object IOPS, horizontal scaling (4→8→12 nodes), and the performance impact of encryption (TLS at RGW, Messenger v2 Secure Mode, SSE-S3). IBM is a vendor-adjacent source because IBM sells IBM Storage Ceph commercially. Benchmark data is current (2025).

## Key Points

**Hardware setup**:
- All-NVMe IBM Storage Ceph Ready Nodes (Dell-based)
- 100 GE leaf-spine network
- 4, 8, and 12-node cluster configurations tested
- EC 2+2 erasure coding for object storage pools
- RGW with and without SSL

**Large-object performance (≥32 MiB objects, EC 2+2, 12 nodes)**:
- Peak GET throughput: ~115 GiB/s aggregate (NIC saturation reached)
- Peak PUT throughput: ~65 GiB/s aggregate
- Near-linear scaling from 4→12 nodes: GET grew ~3x (39→113 GiB/s), PUT ~3x (15.5→50 GiB/s)

**Small-object performance (64 KiB objects, EC 2+2, 12 nodes, 1024 client threads)**:
- GET: ~391K IOPS at ~2 ms average latency (~24.4 GiB/s)
- PUT: ~86.6K IOPS (with SSL)
- CPU availability (not disk I/O) is the dominant performance bottleneck for small objects at high concurrency

**Horizontal scalability**:
- GET latency improved (not degraded) as cluster scaled 8→12 nodes: 52 ms → 36 ms
- Near-linear throughput scaling confirmed

**Security / encryption overhead**:
- TLS at RGW: near-baseline performance impact
- Messenger v2 Secure Mode (daemon-to-daemon): negligible overhead
- SSE-S3: size-dependent cost, tunable

**Critical caveat**: IBM Storage Ceph is a commercial product built on Ceph; IBM has financial interest in strong benchmark results. No independent corroboration of these specific numbers found. Benchmark uses large cluster (12 nodes, 100 GE) not representative of smaller on-premise deployments.

**Lakehouse-specific note**: Benchmark uses 64 KiB as the smallest object size. Iceberg manifest files (typically 1–100 KB) and Parquet footer reads (typically 8–64 KB) fall within this range. No sub-4KB or sub-16KB specific tests published.

## Security Notes

No issues detected.
Checks performed:
- Malicious or obfuscated code: none
- Suspicious URLs or redirects: none
- Content quality / AI-generated: structured benchmark blog post with methodology; authored by IBM Performance team; vendor-adjacent bias noted

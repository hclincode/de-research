---
source: https://github.com/orgs/rustfs/discussions/1500
component: seaweedfs
type: article
evidence-tier: vendor-adjacent
accessible: true
benchmark-age: 2026
date-retrieved: 2026-05-21
security:
  malicious-code: none
  suspicious-urls: none
  quality: high
---

## Summary

A January 14, 2026 benchmark published in the RustFS GitHub Discussions compares MinIO, SeaweedFS, and RustFS using WARP on a 4-node cluster (32 cores, 128 GB RAM, single SATA SSD per node). SeaweedFS (v4.02) achieved GET throughput of ~1,800–1,900 MiB/s and PUT throughput of ~630–640 MiB/s for large objects, competitive with MinIO (~2,000 MiB/s GET, ~650–675 MiB/s PUT). For small files, SeaweedFS outperformed both MinIO and RustFS. Note: this benchmark was published by the RustFS organization, making it vendor-adjacent — independent corroboration of specific numbers is lacking.

## Key Points

- Publication date: January 14, 2026
- Publisher: RustFS organization (vendor-adjacent — RustFS competes with SeaweedFS and MinIO)
- Tool: WARP S3 benchmark tool
- Hardware: 4 nodes, 32 cores, 128 GB RAM, single SATA SSD per node
- Versions: MinIO RELEASE.2024-10-29T16-01-48Z; SeaweedFS 4.02 (dual-replica, EC for cold data); RustFS 1.0.0-alpha.79
- Object sizes tested: 4 KiB, 32 KiB, 256 KiB, 1 MiB, 5 MiB, 10 MiB, 32 MiB
- Concurrency levels: 32, 64, 96, 128, 256, 512 threads
- Results summary: "Small files: seaweedfs > minio > rustfs; Large files: minio > rustfs > seaweedfs"
- SeaweedFS GET: ~1,800–1,900 MiB/s (vs MinIO ~2,000 MiB/s); PUT: ~630–640 MiB/s (vs MinIO ~650–675 MiB/s)
- RustFS showed stability crashes at 512 threads + 1 MiB objects during testing
- Benchmark age: 2026 — recent data, not stale

## Security Notes

No issues detected.
Checks performed:
- Malicious or obfuscated code: None — GitHub discussion page
- Suspicious URLs or redirects: None
- Content quality / AI-generated: High — specific methodology and numbers provided, though source is vendor-adjacent

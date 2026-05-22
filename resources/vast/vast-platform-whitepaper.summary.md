---
source: https://www.vastdata.com/whitepaper
component: vast
type: article
evidence-tier: vendor
accessible: true
benchmark-age: n/a
date-retrieved: 2026-05-21
security:
  malicious-code: none
  suspicious-urls: none
  quality: high
---

## Summary

VAST Data's official platform whitepaper describing the Disaggregated Shared Everything (DASE) architecture. Stateless CNodes (compute) access all storage DBoxes (NVMe SSDs) simultaneously via NVMe-over-Fabrics. Does not mention Spark, Trino, Iceberg, or lakehouse workloads; no throughput/IOPS benchmarks are included. Focuses on erasure coding efficiency, flash endurance, and AI/HPC positioning.

## Key Points

- **DASE architecture**: Separates stateless compute (CNodes) from stateful storage (DBoxes); all CNodes mount all SSDs at boot for shared access
- **Protocols**: NFS (including NFSoRDMA), SMB, S3, NVMe-over-TCP, Kafka-compatible event broker — all native peers, no translation layer
- **Erasure coding**: VAST Classic: 2.7% overhead; DBox-HA: 10% overhead (vs. 3x replication = 200% overhead)
- **Minimum cluster**: Single DBox for basic VAST Classic; 8 DBoxes for HA deployments
- **Flash endurance**: Claims 20x improvement vs. 4KB random writes via write aggregation
- **Lakehouse/analytics gap**: Zero mention of Spark, Trino, Iceberg, Delta Lake, Parquet, or SQL analytics use cases
- **No performance numbers**: Whitepaper makes no throughput or IOPS claims — these appear only in product-specific briefs

## Security Notes

No issues detected.
Checks performed:
- Malicious or obfuscated code: none
- Suspicious URLs or redirects: none
- Content quality / AI-generated: appears human-authored vendor documentation

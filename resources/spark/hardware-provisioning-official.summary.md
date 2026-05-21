---
source: https://spark.apache.org/docs/latest/hardware-provisioning.html
component: spark
type: article
date-retrieved: 2026-05-21
security:
  malicious-code: none
  suspicious-urls: none
  quality: high — official Apache Spark documentation
---

## Summary

The official Apache Spark hardware provisioning guide defines minimum and recommended configurations for storage, memory, network, and CPU in Spark clusters. The guidance is storage-agnostic — Spark does not mandate HDFS and works with any distributed filesystem. The guide emphasizes data locality as the single most important storage-related performance factor.

## Key Points

- **Storage**: 4–8 local disks per node, no RAID, separate mount points. Mount with `noatime`. Local disks are used for shuffle spill and temporary data — this is separate from the distributed storage tier.
- **Data locality**: Spark should ideally run on the same nodes as the data store. If that is not feasible, both should be on the same LAN segment. This is a strong argument against purely remote storage.
- **Network**: 10 GbE minimum; critical for shuffle-heavy operations (joins, group-bys). High-bandwidth shuffle = better performance regardless of which storage backend is used.
- **Memory**: 8 GiB to hundreds of GiB per node; allocate at most 75% to Spark JVM, leave remainder for OS/buffer cache. The OS page cache absorbs repeated reads and can mask storage latency differences.
- **CPU**: 8–16 cores minimum per node; Spark scales linearly to tens of cores.
- **Key implication for storage selection**: Spark's in-memory model means the storage tier is only hit for initial reads and final writes. Shuffle (the most performance-sensitive operation) uses local disks, not the distributed storage. This limits the performance advantage of an ultra-fast storage backend for many workload types.

## Security Notes

No issues detected.
Checks performed:
- Malicious or obfuscated code: none
- Suspicious URLs or redirects: none
- Content quality / AI-generated: authoritative official documentation

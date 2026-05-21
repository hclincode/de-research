---
source: https://ceph.io/en/news/blog/2024/ceph-a-journey-to-1tibps/
component: ceph
type: article
evidence-tier: press
accessible: true
benchmark-age: 2024
date-retrieved: 2026-05-21
security:
  malicious-code: none
  suspicious-urls: none
  quality: high
---

## Summary

Published in January 2024 on ceph.io by the Clyso team (a Ceph consultancy), this post documents the first publicly reported Ceph cluster to break the 1 TiB/s barrier for large sequential reads. The work originated from a customer project building a 10 PB all-NVMe Ceph cluster on Reef. Clyso is a Ceph consultancy (vendor-adjacent, not an IBM/Red Hat commercial vendor), and the methodology is disclosed with specific hardware. The result demonstrates the performance ceiling achievable with careful tuning on modern hardware. Benchmark data is from 2024.

## Key Points

**Hardware**:
- 68 Dell PowerEdge R6615 servers, 63 nodes active
- 48-core AMD EPYC 9454P CPU per server
- 192 GiB DDR5 RAM per server
- 2× 100 GbE Mellanox NICs per server
- 10× 15.36 TB Dell enterprise NVMe drives per server
- 630 OSDs total

**Performance results**:
- Large sequential reads (4 MB objects, 3× replication): **1,025 GiB/s** (first published 1 TiB/s barrier breach)
- Large sequential writes (erasure coding): 387 GiB/s
- Small random reads (4 KB, 3× replication): **25.5 million IOPS**
- Object storage not directly benchmarked in this test (focus was block/RADOS level)

**Critical tuning discoveries**:
- Disabling IOMMU in the kernel produced a substantial performance boost (especially as NVMe count increased); a known but often overlooked optimization
- Ubuntu/Debian Ceph packages compiled without proper RocksDB optimization flags — fixing this halved RocksDB compaction time and doubled 4K random write performance
- 14–16 CPU threads per OSD is the saturation point for a single OSD; more cores are needed to run multiple OSDs per server efficiently
- 2 OSDs per NVMe drive is the recommended ratio for BlueStore/NVMe (vs. 4 OSDs/NVMe for legacy FileStore)

**Operational insight**: Achieving top performance requires deep system-level tuning (kernel parameters, compiler flags, IOMMU, RocksDB, OSD count) — significant expertise and time investment.

## Security Notes

No issues detected.
Checks performed:
- Malicious or obfuscated code: none
- Suspicious URLs or redirects: none
- Content quality / AI-generated: detailed technical post with disclosed methodology, hardware specs, and tuning steps; Clyso is a Ceph specialist consultancy

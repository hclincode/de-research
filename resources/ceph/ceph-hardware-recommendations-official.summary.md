---
source: https://docs.ceph.com/en/latest/start/hardware-recommendations/
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

Official Ceph hardware recommendations documentation covering OSD, monitor, RGW, and network requirements for production clusters. Provides the definitive guidance on BlueStore/NVMe configuration, network requirements, and minimum cluster sizing. Key finding: 10 Gb/s Ethernet is the minimum for production; 25 Gb/s or 100 Gb/s recommended for high-throughput workloads. OSD-to-NVMe ratio guidance and memory requirements are also specified.

## Key Points

**Network requirements**:
- 10 Gb/s Ethernet is the minimum for production (1 Gb/s explicitly stated as not suitable for production storage clusters)
- Separate public network (client traffic) and cluster network (replication/recovery traffic) strongly recommended
- 25 Gb/s or 100 Gb/s for high-throughput workloads (especially all-NVMe clusters)

**OSD configuration (BlueStore + NVMe)**:
- BlueStore is the current default and only supported backend (FileStore removed)
- 2 OSDs per NVMe device is the recommended ratio for BlueStore (vs. 4 OSDs/NVMe for legacy FileStore)
- OSD memory target: 8 GB minimum per OSD (for BlueStore metadata cache)
- CPU: 14–16 threads per OSD is the saturation point; high-core-count CPUs needed for multi-OSD-per-server configurations
- NVMe WAL/DB on separate fast media when using HDDs improves random write performance significantly

**BlueStore tuning for HDD vs. NVMe**:
- HDD + NVMe DB/WAL: NVMe WAL greatly reduces write latency; NVMe DB (RocksDB) prevents HDD seek overhead for metadata
- All-NVMe: highest performance; requires kernel IOMMU disabled for full speed (known optimization)
- RocksDB compilation flags matter: Ubuntu/Debian packages historically lacked optimization flags; verify package provenance

**RGW (RadosGW) hardware**:
- RGW is CPU-bound for small-object workloads at high concurrency
- Multiple RGW instances (horizontal scaling) recommended for high-request-rate deployments
- 32 concurrent connections per RGW instance is a practical limit under high concurrency (documented community observation)

**Monitor nodes**:
- 3 or 5 monitor nodes for quorum; separate from OSD nodes in production
- SSDs for monitor storage (RADOS journal/WAL)

**Minimum cluster size**:
- Minimum 3 OSD nodes for a replicated pool (3× replication)
- Minimum 4 OSD nodes for a 2+2 EC pool; more nodes for larger EC profiles

**Operational complexity factors**:
- Requires careful planning: OSD placement, MON quorum, network separation, BlueStore tuning
- Specialized Ceph expertise required for production deployments
- Recovery operations (OSD failure/replacement) are resource-intensive; recovery traffic competes with client I/O on the cluster network if not separated

## Security Notes

No issues detected.
Checks performed:
- Malicious or obfuscated code: none
- Suspicious URLs or redirects: none
- Content quality / AI-generated: official Ceph project documentation (docs.ceph.com)

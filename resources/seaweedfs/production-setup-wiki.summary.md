---
source: https://github.com/seaweedfs/seaweedfs/wiki/Production-Setup
component: seaweedfs
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

The official SeaweedFS Production Setup wiki covers master HA configuration, volume server replication, multi-disk setups, and maintenance strategy. It documents that a single master is acceptable for large clusters (load is light; soft state only), while 3-master Raft quorum is the HA option. The guide has several important operational caveats that affect production reliability planning.

## Key Points

- Single master is explicitly acceptable for large clusters per official docs ("load on master is very light," "only has soft states")
- Multi-master: odd count required (1 or 3 typical); 2-master consensus is impossible; use `-peers=ip1:9333,ip2:9333,ip3:9333`
- Replication warning: "Multiple volume servers on the same physical host count as separate servers for replication purposes" — rack-aware replication (`001`) does NOT guarantee separate physical hosts without explicit physical separation
- Data rebalancing: NOT automatic after adding volume servers — new data written to new servers once new volumes are created, but existing data is not moved
- Multi-disk: do not use multiple directories on the same physical disk (automatic volume count double-counts capacity)
- Filer default (`leveldb2`): supports only ONE filer; production multi-filer requires shared filer store (Redis, Cassandra, etc.)
- Compaction: configure `-compactionMBps` to throttle background jobs; automatic rebalancing is "problematic" — use manual `weed shell` commands during off-hours instead
- Volume size default: 30 GB; large-disk variant available for petabyte-scale (8 TB volumes, mutually incompatible with standard 30 GB volumes in same cluster)

## Security Notes

No issues detected.
Checks performed:
- Malicious or obfuscated code: None — official wiki documentation
- Suspicious URLs or redirects: None
- Content quality / AI-generated: High — official maintainer documentation

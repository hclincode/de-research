---
source: https://github.com/seaweedfs/seaweedfs/wiki/FAQ
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

The official SeaweedFS FAQ covers durability, consistency, compaction, and operational configuration. It documents CRC-based bitrot detection, append-only design for SSD longevity, garbage collection trigger thresholds, and per-file replication configuration. The FAQ acknowledges that replica sizes may differ across nodes due to asynchronous compaction and selective write failures.

## Key Points

- Bitrot protection: CRC checked on volume servers; accessible via ETag; can also enable erasure coding tolerating up to 4 shard losses
- Garbage collection: disk space released only when volume is vacuumed; default threshold is 30% garbage; manual trigger via `volume.vacuum -garbageThreshold=0.0001`
- Compaction: recommend smaller volumes for high-update/deletion workloads to reduce compaction time
- Replication consistency: "volumes are consistent, but not necessarily the same size or the same number of files" — size discrepancies from selective replica writes (treated as failures) and async compaction timing across replicas; this is acknowledged expected behavior
- Per-file replication: each file can have independent replication level (flexible but operationally complex)
- Memory footprint: high volume counts with many small files increase memory; mitigation is converting older volumes to erasure-coded (EC) format with binary-searchable sorted indexes
- Large-disk incompatibility: standard 30 GB and large-disk (8 TB) volume builds cannot coexist in same cluster; migration requires manual `.dat` file copying and `.idx` regeneration
- Volume sizing: leave capacity for several volume sizes to enable compaction (in-place compaction needs free space)

## Security Notes

No issues detected.
Checks performed:
- Malicious or obfuscated code: None — official wiki documentation
- Suspicious URLs or redirects: None
- Content quality / AI-generated: High — official maintainer documentation

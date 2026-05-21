---
source: https://docs.ceph.com/en/reef/rados/troubleshooting/troubleshooting-osd/
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

Official Ceph OSD troubleshooting documentation covering slow OSD diagnosis, recovery storms, and cluster-wide performance degradation patterns. Supplemented by community mailing list observations and the Ceph SLOW_OPS diagnostic documentation. Covers the critical failure modes relevant to a lakehouse deployment: slow OSD cascade, recovery I/O competing with client I/O, PG lock contention, and RGW memory growth under high object-count workloads.

## Key Points

**Slow OSD definition and detection**:
- Ceph defines a slow request threshold (default: 30 seconds); OSDs exceeding this log warnings and trigger cluster health alerts
- Metrics to watch: `apply_latency_ms`, `commit_latency_ms` per OSD
- Slow OSDs cascade: a single slow OSD can delay PG peering and client I/O across the entire cluster

**Root causes of slow OSDs**:
1. Disk I/O saturation (OSD device throughput exceeded)
2. Excessive recovery or scrub activity competing with client I/O
3. Memory pressure (OSD memory target too low; BlueStore cache thrashing)
4. Network congestion between cluster nodes
5. PG lock contention under very high concurrency (documented in high-speed clusters)
6. mclock scheduler misconfiguration: OSD may obtain inaccurate disk performance benchmarks causing overload during routine scrub operations (2024 production incident reported)

**Recovery storms**:
- When an OSD fails, Ceph initiates data recovery across remaining OSDs
- Recovery I/O competes with client (RGW) I/O on the cluster network
- Without a separate cluster network, client throughput degrades significantly during recovery
- Large clusters (many OSDs) have longer recovery times, increasing the window of degraded performance
- mclock scheduler (current default) is designed to throttle recovery I/O to protect client performance; misconfiguration can reverse this

**RGW-specific failure modes**:
- High object-count workloads cause OSD memory explosion (RGW metadata growth in RADOS)
- Bucket index RADOS objects can become hotspots under concurrent RGW writes (addressed by dynamic sharding)
- RGW multisite sync error log can grow unboundedly during transient sync failures if not monitored
- 32 concurrent connections per RGW instance limit: under high Spark executor concurrency (100+ executors), RGW queues fill and requests back up; mitigation is horizontal RGW scaling

**PG lock contention**:
- Concurrent reads/writes to the same PG serialize at the PG lock
- EC 4:2 pools: a single read requires 4 OSD accesses (vs. 1 for 3× replication); under high concurrency this increases contention on all participating OSDs
- FastEC (Tentacle) partial reads reduce the number of OSD accesses for small/partial reads, mitigating this

**Lakehouse-specific risk**:
- Spark jobs reading Iceberg tables may issue thousands of concurrent GetObject requests; if concurrent requests exceed RGW capacity, the entire pipeline backs up
- A single failed OSD during a Spark job can degrade read throughput across the cluster for the job's duration
- Mitigation: adequate RGW instances (horizontal scale), separate cluster/public networks, pre-sharded bucket indexes, health monitoring with alerting

## Security Notes

No issues detected.
Checks performed:
- Malicious or obfuscated code: none
- Suspicious URLs or redirects: none
- Content quality / AI-generated: official Ceph project documentation; supplemented by community mailing list and practitioner observations

---
source: https://news.ycombinator.com/item?id=39476609
component: juicefs
type: article
evidence-tier: press
accessible: true
benchmark-age: n/a
date-retrieved: 2026-05-21
security:
  malicious-code: none
  suspicious-urls: none
  quality: high
---

## Summary

Hacker News community discussion thread raising architecture concerns about JuiceFS. Community members noted that JuiceFS introduces multiple stacked layers of failure: the underlying object storage (S3/MinIO/Ceph) has its own reliability concerns, and adding a separate metadata engine (Redis/TiKV) on top creates an additional independent failure domain. Some users questioned the operational complexity of running and maintaining both systems. The discussion also raised questions about the consistency/durability/availability guarantees when Redis metadata fails.

## Key Points

- Community concern 1: JuiceFS requires two separate systems (object store + metadata engine), each with its own failure modes; two independent points of failure vs. a single self-contained system like Ceph or Ozone
- Community concern 2: Redis metadata store failure blocks all filesystem operations; no read-only degraded mode documented
- Community concern 3: Questions about what happens during metadata engine downtime — whether operations block or fail immediately
- Community concern 4: Some asked why benchmarks highlight Redis when Redis has scalability limits; asked for benchmarks with PostgreSQL/TiKV
- Counterpoint from community: JuiceFS's architecture allows independent scaling of metadata and data; simpler than Ceph to operate for teams not already running Ceph
- Historical note: when first open-sourced, Redis was the only metadata option; the project has since added SQL and TiKV, but Redis remains the default recommendation for performance
- Key architectural trade-off: operational simplicity of each individual component (Redis, MinIO) vs. complexity of coordinating two separate systems with their own HA requirements

## Security Notes

No issues detected.
Checks performed:
- Malicious or obfuscated code: Hacker News discussion; no executable content
- Suspicious URLs or redirects: None
- Content quality / AI-generated: Community discussion; human-authored

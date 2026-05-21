---
source: https://juicefs.com/en/blog/usage-tips/juicefs-metadata-engine-selection-guide
component: juicefs
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

JuiceData blog post (vendor) providing a decision guide for choosing between JuiceFS metadata engines. The guide explicitly discusses the tradeoffs and is candid about each engine's weaknesses. It recommends Redis for performance-sensitive workloads where memory capacity is sufficient, PostgreSQL/MySQL for moderate scale with stronger durability requirements, and TiKV for very large scale (billions of files) despite its operational complexity.

## Key Points

- Redis: recommended when raw metadata performance is the priority and dataset fits in memory (≤100M files per typical 64 GiB Redis)
- PostgreSQL/MySQL: recommended as durability-first option for tens of millions of files; 2-13x slower than Redis on metadata-heavy workloads
- TiKV: recommended only for >1 billion files; requires 3-node PD cluster + 3-node TiKV cluster (minimum 6 nodes for HA); 44x slower than Redis on small-file intensive workloads in one real-world production test
- BadgerDB: embedded, single-node only; suitable for development/testing only
- etcd: not recommended for production JuiceFS use; performance similar to TiKV but adds operational complexity without proportional benefit
- Vendor recommendation matrix (simplified):
  - <10M files: any engine; SQLite or PostgreSQL for simple on-premise setups
  - 10M–100M files: Redis+Sentinel or PostgreSQL
  - 100M–1B files: Redis with 64 GiB+ instance or PostgreSQL/MySQL with read replicas
  - >1B files: TiKV or Enterprise Edition
- Key caveat: TiKV production deployment involves significantly higher operational burden than Redis

## Security Notes

No issues detected.
Checks performed:
- Malicious or obfuscated code: Vendor blog; no code artifacts
- Suspicious URLs or redirects: None
- Content quality / AI-generated: High quality; appears human-authored

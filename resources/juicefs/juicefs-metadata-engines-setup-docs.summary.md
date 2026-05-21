---
source: https://juicefs.com/docs/community/databases_for_metadata/
component: juicefs
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

Official JuiceFS documentation describing all supported metadata engine options for the Community Edition. Metadata engines fall into three categories: Redis (in-memory), SQL relational databases (MySQL, PostgreSQL, SQLite, MariaDB), and transactional key-value stores (TiKV, BadgerDB, FoundationDB, etcd). Each engine has distinct tradeoffs in performance, reliability, scalability, and operational complexity. The documentation provides connection string formats and configuration guidance per engine.

## Key Points

- Three categories of metadata engines:
  1. **Redis**: Fastest, fully in-memory; supports Redis standalone, Sentinel, Cluster; asynchronous replication means potential data loss on failover
  2. **SQL**: MySQL, PostgreSQL, SQLite, MariaDB; strong durability; 2-13x slower than Redis for metadata-heavy workloads
  3. **TKV (transactional KV)**: TiKV (distributed, Raft-based), BadgerDB (single-node embedded), FoundationDB; TiKV offers horizontal scalability; performance similar to PostgreSQL not Redis
- Redis memory requirement: ~300 bytes per inode; 100 million files ≈ 32 GiB RAM; max recommended instance: 64 GiB
- Redis Cluster: supported since v1.0.0 Beta3, but limited to one hash slot per JuiceFS volume
- Redis Sentinel: supported for HA; asynchronous replication means acknowledged writes may be lost on master failure
- PostgreSQL/MySQL: recommended for smaller deployments (tens of millions of files) where durability > raw speed
- TiKV: requires separate 3-node PD + 3-node TiKV cluster; significant operational overhead; recommended for >1 billion files
- SQLite: suitable only for single-host, non-critical use; not for production
- Official recommendation: Redis with Sentinel for performance; TiKV for very large scale; PostgreSQL as durability-first alternative

## Security Notes

No issues detected.
Checks performed:
- Malicious or obfuscated code: Official documentation; no code artifacts
- Suspicious URLs or redirects: None
- Content quality / AI-generated: High quality; official project documentation

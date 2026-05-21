---
source: https://juicefs.com/docs/community/metadata_engines_benchmark/
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

Official JuiceFS benchmarks comparing metadata engine performance across Redis, MySQL, TiKV, and etcd. Tests use mdtest and measure pure metadata operations as well as combined I/O workloads at different file sizes. The benchmark is published by JuiceData (vendor) in their official docs but tests their own product's internals, making it a reasonable basis for relative comparisons between metadata engines even if absolute numbers are optimistic. No benchmark publication date is given in the search-derived summary; treat as vendor-sourced data.

## Key Points

- Performance hierarchy (pure metadata ops): Redis >> TiKV ≈ MySQL > etcd
- Specific ratios (pure metadata): MySQL costs 2-4x more than Redis; TiKV slightly better than MySQL; etcd ≈ 1.5x TiKV
- At 100 KiB file size: MySQL total time ≈ 13x Redis; TiKV and etcd similar to MySQL
- At large I/O (≈4 MiB files): no significant difference between engines — object storage becomes the bottleneck
- Test setup note: Redis and MySQL are single-replica local deployments; TiKV and etcd are 3-node Raft clusters — the comparison is not apples-to-apples in durability
- Real-world user report (GitHub Discussion #884): JuiceFS+TiKV production mode was 448 seconds vs <10 seconds for JuiceFS+Redis on the same distributed write workload (100 workers writing files)
- Redis memory bound: 100 million inodes ≈ 32 GiB RAM; max recommended Redis instance = 64 GiB
- Benchmark does not include PostgreSQL separate results; MySQL is used as the SQL proxy

## Security Notes

No issues detected.
Checks performed:
- Malicious or obfuscated code: Official documentation; no code artifacts
- Suspicious URLs or redirects: None
- Content quality / AI-generated: High quality; official project documentation

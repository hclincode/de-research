---
source: https://juicefs.com/docs/community/guide/cache/
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

Official JuiceFS documentation describing the client-side caching architecture, including the writeback cache (--writeback flag). The documentation explicitly warns that writeback mode can cause data loss, particularly in containerized environments. It also describes close-to-open consistency trade-offs when --open-cache is enabled. These are vendor-documented known risks, not third-party criticisms.

## Key Points

- JuiceFS client-side cache has two modes: read cache (safe) and write cache / writeback (risky)
- **Writeback mode (--writeback)**: data is written to local disk cache first and uploaded to object storage asynchronously
  - Risk: if node crashes or container restarts before upload completes, data is lost permanently (no recovery)
  - Documentation says: "strongly advised against" for --writeback in containers
  - Write cache data from one node is not readable by other nodes until async upload completes
  - If object storage upload is too slow, write cache can accumulate indefinitely; other nodes get I/O timeout errors on reads
- **Read cache**: data blocks fetched from object storage are cached locally; safe; evicted under LRU
- **Close-to-open consistency (default)**: writes from client A visible to client B only after client A closes the file
  - This is sufficient for most analytics workloads (Spark writers close files; Trino readers see them)
  - Iceberg's atomic commit pattern (file written, manifest updated) aligns well with close-to-open semantics
- **--open-cache flag**: disables close-to-open consistency in favor of local kernel metadata cache for reads; improves throughput for read-heavy workloads but allows stale reads across clients
- Distributed cache: Community Edition only supports local per-node cache; Enterprise Edition adds distributed peer cache

## Security Notes

No issues detected.
Checks performed:
- Malicious or obfuscated code: Official documentation
- Suspicious URLs or redirects: None
- Content quality / AI-generated: High quality; official project documentation

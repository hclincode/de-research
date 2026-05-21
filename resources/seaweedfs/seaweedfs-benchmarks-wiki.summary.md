---
source: https://github.com/seaweedfs/seaweedfs/wiki/Benchmarks
component: seaweedfs
type: article
evidence-tier: official
accessible: true
benchmark-age: 2023
date-retrieved: 2026-05-21
security:
  malicious-code: none
  suspicious-urls: none
  quality: high
---

## Summary

The SeaweedFS official benchmark wiki shows internal benchmarks using the `weed benchmark` command on a MacBook (i7 2.2 GHz, SSD). Write throughput for 1 million 1 KB files: 5,747 KB/s at 64 concurrent threads (avg 10.9 ms latency). Read throughput: 12,988 KB/s (avg 4.7 ms latency). The wiki explicitly cautions that "benchmarks are misleading" and that real-world results depend heavily on CPU, disk, network, and topology. These are single-machine benchmarks and are not representative of multi-node production deployments. Benchmark data is stale (no year specified but test hardware suggests early period; independent sources from 2025-2026 show far higher multi-node throughput).

## Key Points

- Test tool: `weed benchmark` — writes 1 million 1 KB files, then randomly reads them back
- Hardware: single MacBook i7 2.2 GHz SSD
- Write: 5,747 KB/s, 5,747 req/s, 64 concurrent, avg 10.9 ms, p95 14.9 ms, p99 17.3 ms
- Read: 12,988 KB/s, 12,988 req/s, 64 concurrent, avg 4.7 ms, p95 16.6 ms, p99 34.8 ms
- With replication (001): write 5,994 KB/s; read 19,423 KB/s
- Official caveat: "benchmarks are misleading" — CPU, disk, memory, network significantly impact results
- Single-machine results are not representative of multi-node production performance
- Multi-node WARP benchmarks (2026, via RustFS discussion) show 1,800–1,900 MiB/s GET for SeaweedFS 4.02 — orders of magnitude higher
- Benchmark age: stale for single-machine numbers; context of 2025-2026 multi-node WARP results provides more relevant data

## Security Notes

No issues detected.
Checks performed:
- Malicious or obfuscated code: None — official wiki page
- Suspicious URLs or redirects: None
- Content quality / AI-generated: High — official documentation with explicit methodology

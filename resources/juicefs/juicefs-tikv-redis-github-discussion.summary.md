---
source: https://github.com/juicedata/juicefs/discussions/884
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

GitHub community discussion on the JuiceData repository comparing Redis vs TiKV as JuiceFS metadata engines in a production context. A user reported a dramatically worse result with TiKV in production mode than expected: the same distributed workload (100 workers writing files) completed in under 10 seconds with JuiceFS+Redis but took 448 seconds with JuiceFS+TiKV in production mode (3-node cluster). This is significantly worse than the 2-3x ratio suggested by the official benchmark, indicating that the TiKV deployment had additional overhead (Raft consensus latency, network RTT across 3 nodes) in a real workload vs. a synthetic benchmark.

## Key Points

- Real-world production test: 100 worker processes, distributed write workload
- JuiceFS+Redis (with AOF, 8 io-threads): completed in <10 seconds
- JuiceFS+TiKV (production mode, 3-node cluster): 448 seconds for the same workload
- TiKV production mode = distributed 3-node Raft consensus; every metadata write requires quorum
- The 44x slower result is attributed to Raft write amplification under concurrent small-file metadata ops
- Community discussion suggests TiKV playground mode (single-node) performs closer to Redis
- Implication: TiKV's durability guarantee comes at a very high metadata latency cost for workloads with many small concurrent metadata operations
- This issue was raised and acknowledged; JuiceData did not dispute the measurement

## Security Notes

No issues detected.
Checks performed:
- Malicious or obfuscated code: GitHub discussion; no executable content
- Suspicious URLs or redirects: None
- Content quality / AI-generated: Community discussion; human-authored

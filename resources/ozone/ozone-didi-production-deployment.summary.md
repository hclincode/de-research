---
source: https://news.apache.org/foundation/entry/how-didi-scaled-to-hundreds-of-petabytes-with-apache-ozone
component: ozone
type: article
evidence-tier: official
accessible: true
benchmark-age: 2025
date-retrieved: 2026-05-21
security:
  malicious-code: none
  suspicious-urls: none
  quality: high
---

## Summary

Published January 20, 2026, this ASF blog post documents DiDi's large-scale Apache Ozone production deployment — the most comprehensive public evidence of Ozone operational maturity at exabyte-class scale. DiDi runs hundreds of petabytes with tens of billions of files across multiple Ozone clusters, having migrated from HDFS. This constitutes the most current large-scale production evidence (data collected 2025, published January 2026).

## Key Points

- **Scale**: >500 PB total, >1 PB daily ingestion, ~5 billion files per cluster, scalable to tens of billions.
- **Metadata latency**: P90 GetMetaLatency improved from 90ms to 17ms (81% reduction) through lock contention fixes and OM follower reads.
- **Read throughput**: >20% improvement with Object Manager (OM) follower reads.
- **Storage efficiency**: Transition from 3x replication (3.0x overhead) to Erasure Coding RS-6-3 (1.5x overhead) — saved hundreds of petabytes.
- **Small file handling**: Deployed heterogeneous caching: first 1 MB chunk of each block cached on NVMe SSDs; EC first stripe (9 MB) also cached. LRU-managed hot data on datanodes.
- **Cluster scaling**: Individual clusters capped at ~5 billion files; ViewFs routing used to distribute load across multiple clusters.
- **Migration approach**: Dual-write for rollback safety; DistCp with COMPOSITE_CRC checksums for data consistency verification.
- **Production duration**: Over two years of stable operation before the January 2026 publication.
- **No Trino/Spark integration specifics**: This article covers infrastructure operations, not query engine integration.

## Security Notes

No issues detected.
Checks performed:
- Malicious or obfuscated code: none (official ASF blog post)
- Suspicious URLs or redirects: none
- Content quality / AI-generated: first-party production case study with specific metrics — high quality

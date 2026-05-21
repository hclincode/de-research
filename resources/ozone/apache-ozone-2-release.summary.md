---
source: https://news.apache.org/foundation/entry/the-apache-software-foundation-announces-apache-ozone-2-0-0
component: ozone
type: article
evidence-tier: official
accessible: true
benchmark-age: 2025
date-retrieved: 2026-05-21
security:
  malicious-code: none
  suspicious-urls: none
  quality: high — official ASF announcement
---

## Summary

Apache Ozone 2.0.0 was released August 19, 2025, marking the maturation of the Hadoop-native distributed object store. The release incorporates production lessons from large-scale deployments and 1,700+ new features and bug fixes over 1.4. Ozone is designed as the long-term successor to HDFS at scale, removing the NameNode bottleneck through a distributed metadata model.

## Key Points

- **License**: Apache 2.0. Fully open-source, foundation-governed, no single-vendor risk.
- **Spark integration**: Implements the Hadoop Compatible FileSystem API — Spark, Hive, and MapReduce work against Ozone without any code changes, using the same patterns as HDFS.
- **Performance vs HDFS**: In TPC-DS benchmarks, Ozone outperforms HDFS by an average of 3.5% across 99 queries; in >70% of cases individual queries are faster on Ozone.
- **Scalability advantage**: Eliminates the HDFS NameNode bottleneck through distributed metadata. Scales to billions of objects without federation complexity.
- **New in 2.0**: Atomic key operations (write consistency), HBase support, JDK 17/21, ARM64 builds, zero-copy datanode replication, improved streaming write throughput, lower client heap usage.
- **Production signals**: Adopted at scale by Cloudera (CDP Private Cloud Base). Multiple production deployments reported in the community.
- **Maturity concern**: Ozone's ecosystem tooling (audit, policy, ACL management) is less mature than HDFS, which has a decade of operational tooling built around it.

## Security Notes

No issues detected.
Checks performed:
- Malicious or obfuscated code: none
- Suspicious URLs or redirects: none
- Content quality / AI-generated: official ASF release announcement — authoritative

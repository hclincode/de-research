---
source: https://www.redhat.com/en/blog/why-spark-ceph-part-3-3
component: ceph
type: article
evidence-tier: vendor-adjacent
accessible: true
benchmark-age: 2019
date-retrieved: 2026-05-21
security:
  malicious-code: none
  suspicious-urls: none
  quality: low — Red Hat is a major Ceph contributor with commercial interest in Ceph adoption; benchmark data is from 2019 (stale for 2025/2026 decisions)
---

## Summary

Part 3 of a 3-part Red Hat series benchmarking Apache Spark on Ceph (via S3A/RadosGW) versus native HDFS. Covers single-user SparkSQL, multi-user Impala, and multi-user Spark workloads with erasure-coded (EC 4:2) Ceph clusters. The benchmark uses realistic data sizes and concurrency levels relevant to enterprise deployments.

## Key Points

**Performance numbers (Ceph EC4:2 vs HDFS 3x replication):**

| Workload | Ceph EC4:2 vs HDFS |
|---|---|
| Single-user SparkSQL reads | ~2x slower |
| Multi-user Impala (10 concurrent) | 57% slower |
| Multi-user Spark (10 concurrent) | Comparable |
| Write jobs | 37–200%+ slower |
| Mixed workloads | ~90% slower |

**Cost advantage:**
- Ceph EC4:2 uses ~33% of the raw capacity that HDFS 3x replication requires for the same data volume.
- At equal hardware cost, Ceph delivers up to 50% lower cost per usable TB.

**When Ceph makes sense for Spark:**
- Primary driver: multiple independent analytics clusters sharing a single data store (no data silos/duplication)
- Read-dominated, multi-tenant batch workloads at moderate concurrency
- Storage cost reduction is a hard constraint

**When Ceph does not make sense:**
- Write-intensive Spark ETL pipelines
- Latency-sensitive interactive SparkSQL or BI use cases
- Single-cluster deployments (HDFS data locality advantage is not recouped)

**Operational note**: Ceph RadosGW (RGW) S3 interface requires careful tuning for high object-count scenarios. Performance degrades with large numbers of small objects (50,000+ objects in a prefix causes significant slowdown regardless of RGW scaling).

## Security Notes

No issues detected.
Checks performed:
- Malicious or obfuscated code: none
- Suspicious URLs or redirects: none
- Content quality / AI-generated: well-structured benchmark article, methodology disclosed

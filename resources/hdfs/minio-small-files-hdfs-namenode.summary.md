---
source: https://blog.min.io/challenge-big-data-small-files/
component: hdfs
type: article
evidence-tier: vendor-adjacent
accessible: true
benchmark-age: n/a
date-retrieved: 2026-05-21
security:
  malicious-code: none
  suspicious-urls: none
  quality: high
---

## Summary

This MinIO blog post analyzes the small-files problem in HDFS and large-scale data systems. MinIO is a competing object storage vendor, making this a vendor-adjacent source requiring corroboration. The technical claims align with widely-documented HDFS behavior: every file, directory, and block in HDFS occupies 150–300 bytes of NameNode heap memory; 100 million small files can consume hundreds of GB of NameNode RAM and waste storage in undersized blocks. The article advocates for object stores (including MinIO) as the solution.

## Key Points

- Every HDFS file/directory/block consumes ~150–300 bytes of NameNode heap memory.
- 100 million small files → hundreds of GB of NameNode RAM consumed.
- 100 million small files also wastes >10 TB of storage capacity (undersized 128 MB blocks mostly empty).
- NameNode OutOfMemoryError documented at under 30 million files on low-RAM deployments.
- Small files increase NameNode CPU load via more complex heartbeat tracking and more frequent access requests.
- Lakehouse workloads that generate many Iceberg/Hudi manifest files and snapshot files are especially susceptible.
- Object stores inherently avoid this problem by separating metadata storage from the object storage layer.
- Claim: "100 million files consumes hundreds of GB NameNode memory" — corroborated by Cloudera docs (150 bytes/object × 100M = ~14 GB heap minimum, but with JVM overhead, block replicas, and inode overhead the practical figure is much higher).

## Security Notes

No issues detected. Source is a vendor-adjacent blog (MinIO sells a competing product); technical claims are corroborated by official Hadoop/Cloudera documentation.
Checks performed:
- Malicious or obfuscated code: n/a (blog post)
- Suspicious URLs or redirects: none; blog.min.io domain
- Content quality / AI-generated: high; technically specific with verifiable claims

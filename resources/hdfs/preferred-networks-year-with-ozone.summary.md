---
source: https://tech.preferred.jp/en/blog/a-year-with-apache-ozone/
component: hdfs
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

Preferred Networks (PFN), a Japanese AI/ML research company, published this practitioner account of migrating from HDFS (~9 PB across clusters, largest cluster ~10 PB capacity) to Apache Ozone. The article provides an independent, non-vendor account of the migration challenges and improvements observed after one year. PFN is not a Hadoop/Ozone vendor or significant commercial contributor, making this a press-tier source. Key finding: NameNode scalability improved significantly when responsibilities were divided between Ozone's OzoneManager (OM) and StorageContainerManager (SCM), with metadata moved to RocksDB.

## Key Points

- PFN migrated ~9 PB of data from HDFS to Apache Ozone; largest HDFS cluster was ~10 PB capacity on Ubuntu 16.04.
- Primary motivation: operational issues with HDFS NameNode at scale; improved metadata scalability with Ozone.
- Migration tool: Hadoop DistCp — works between same-KDC clusters; S3A protocol used for cross-KDC clusters.
- Critical migration blocker: HDFS has no file size limit but S3 API has a 5 TB maximum per object — some HDFS files exceeded 5 TB, requiring special handling.
- After migration: NameNode scalability improved as OzoneManager + StorageContainerManager replaced single NameNode.
- Ozone metadata stored in RocksDB (vs. NameNode in-memory fsImage) — persistent, not RAM-bound.
- Confirms DistCp as the standard migration tool; notes checksum type mismatch (HDFS CRC32C vs. Ozone CRC32) requires explicit `dfs.checksum.combine.mode=COMPOSITE_CRC` parameter.
- Real-world production validation of migration path: HDFS → Ozone is operationally feasible at petabyte scale.

## Security Notes

No issues detected.
Checks performed:
- Malicious or obfuscated code: n/a (practitioner blog post)
- Suspicious URLs or redirects: none; tech.preferred.jp official company tech blog
- Content quality / AI-generated: high; specific operational details with disclosed methodology and cluster scale

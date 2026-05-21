---
source: https://medium.com/engineering-cloudera/open-data-lakehouse-powered-by-apache-iceberg-on-apache-ozone-a225d5dcfe98
component: iceberg
type: article
evidence-tier: independent
accessible: true
date-retrieved: 2026-05-21
security:
  malicious-code: none
  suspicious-urls: none
  quality: high — Cloudera engineering blog; practitioner experience, not vendor marketing
---

## Summary

Cloudera engineering documents their production reference architecture for an open lakehouse using Apache Iceberg as the table format on top of Apache Ozone as the storage layer. This is the primary independent production evidence for the Iceberg + Ozone combination.

## Key Points

**Architecture:**
- Ozone exposes both `ofs://` (Hadoop FileSystem API) and `s3a://` (S3-compatible) interfaces simultaneously. Iceberg can use either; `s3a://` is preferred for multi-engine portability.
- Iceberg metadata files (table state, snapshot manifests, data file references) stored directly in Ozone object buckets. No external metadata store for file location tracking beyond the catalog.
- Ozone scales to billions of objects — critical for lakehouse workloads where metadata growth (snapshots, manifests) is intrinsic and continuous.

**Ozone capabilities that enable lakehouse:**
- Handles large numbers of small objects without NameNode-equivalent saturation (distributed SCM + OM metadata).
- Erasure coding for storage efficiency on data files.
- Built-in encryption and ACLs for multi-tenant data isolation.
- Native Hadoop FileSystem compatibility means Spark, Hive, and Impala read/write without modification.

**Iceberg advantages demonstrated on this stack:**
- Query planning 10x faster than Hive partitioned tables (avoids metastore partition listing calls).
- Hidden partitioning: queries filter on logical columns; physical partition layout transparent to writers and readers.
- Schema evolution without rewrites. Time travel for late-arriving data and snapshot delta processing.

**Limitations noted:**
- No benchmark data comparing this stack against HDFS+Hudi or Ceph+Iceberg alternatives.
- Concurrent multi-writer conflict resolution details not addressed — depends on catalog implementation.
- Operational tooling for Ozone (backup, disaster recovery, quota management) less mature than HDFS equivalents.

## Security Notes

No issues detected.
Checks performed:
- Malicious or obfuscated code: none
- Suspicious URLs or redirects: none
- Content quality / AI-generated: practitioner-written engineering article — credible

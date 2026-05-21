---
source: https://trino.io/docs/current/connector/iceberg.html
component: trino
type: article
evidence-tier: official
accessible: true
benchmark-age: n/a
date-retrieved: 2026-05-21
security:
  malicious-code: none
  suspicious-urls: none
  quality: high — official Trino documentation
---

## Summary

Trino's Iceberg connector provides full read/write access to Iceberg tables. It supports six catalog backends including REST and Nessie, and can access storage over S3 (native), HDFS, Azure, and GCS. Storage access via S3 is the recommended path going forward as Trino is deprecating its Hadoop file system libraries.

## Key Points

**Supported catalogs**: Hive Metastore, AWS Glue, JDBC, REST Catalog, Nessie, Snowflake Catalog. REST Catalog is the recommended type for new deployments — engine-agnostic and compatible with Nessie and Apache Polaris.

**Write support**: Full DML — INSERT, UPDATE, DELETE, MERGE, TRUNCATE. Schema DDL — CREATE/DROP/ALTER TABLE. Row-level deletes via Iceberg v2 position delete files.

**Storage backends**: S3 (native, recommended), HDFS (via `fs.hadoop.enabled=true`, deprecated path), Azure, GCS. For S3-compatible storage (Ozone, Ceph RadosGW), uses native S3 with `s3.path-style-access=true` and endpoint override.

**Performance features**: Metadata caching in coordinator memory by default; projection pushdown; dynamic filtering; partition pruning.

**Known limitations**:
- `table_changes` function not supported on tables with delete files.
- Iceberg format v3 support is experimental; row-level updates/deletes on v3 not stable.
- MERGE INTO on large partitioned tables can trigger full table scans (known performance regression, tracked upstream).
- Wide tables (>100 columns) require manual ANALYZE for statistics.
- Iceberg write concurrency ceiling via OCC: ~15 transactions per minute maximum — not a bottleneck for interactive SQL but can affect high-frequency Spark+Trino concurrent writes.

**Trino's role in dual-engine architecture**: Trino handles interactive analytics and ad-hoc SQL. Spark handles batch ingestion, CDC, and compaction. Both access the same Iceberg tables through a shared catalog.

## Security Notes

No issues detected.
Checks performed:
- Malicious or obfuscated code: none
- Suspicious URLs or redirects: none
- Content quality / AI-generated: official documentation — authoritative

---
source: https://medium.com/engineering-cloudera/open-data-lakehouse-powered-by-apache-iceberg-on-apache-ozone-a225d5dcfe98
component: ozone
type: article
evidence-tier: vendor-adjacent
accessible: true
benchmark-age: 2023
date-retrieved: 2026-05-21
security:
  malicious-code: none
  suspicious-urls: none
  quality: high
---

## Summary

Published February 28, 2023 by Cloudera Engineering (a vendor-adjacent source — Cloudera commercially supports Ozone in CDP Private Cloud). This article describes how Apache Iceberg runs on Ozone via OFS or S3A. The benchmark data is from 2023 and is stale relative to the 2026-05-21 research date (>2 years old). The claim that Iceberg query planning is "10x faster" vs. traditional Hive rests on Iceberg architecture, not Ozone-specific benchmarks.

## Key Points

- **Two access paths**: OFS (`ofs://ozone1/tenant1/warehouse/...`) or S3A via linked buckets — both supported for Iceberg on Ozone.
- **Small file claim**: Article states Ozone "can handle billions of files without sacrificing performance" — but provides no Ozone-specific benchmark data; this is a general capability claim.
- **Iceberg query planning**: "10x faster query planning" claim refers to Iceberg's hidden partitioning eliminating HMS calls, not to Ozone-specific performance — applicable regardless of storage backend.
- **Spark configuration**: Requires `iceberg-spark-runtime-3.2_2.12:1.1.0`, Iceberg Spark extensions, and OFS/S3A configuration in core-site.xml.
- **Trino not covered**: Only Hive, Impala, and Spark are mentioned.
- **No limitations disclosed**: Article focuses on capabilities without discussing known issues or constraints.
- **Benchmark staleness**: Publication date February 2023; data is >2 years old and predates Ozone 2.0 and 2.1. Do not use for current performance assessment.

## Security Notes

No issues detected.
Checks performed:
- Malicious or obfuscated code: none (Medium/Cloudera Engineering blog)
- Suspicious URLs or redirects: none
- Content quality / AI-generated: technical blog with configuration examples — likely accurate but vendor-adjacent and stale

---
source: https://www.vastdata.com/blog/accelerate-your-trino-workloads-with-vast
component: vast
type: article
evidence-tier: vendor
accessible: true
benchmark-age: 2025
date-retrieved: 2026-05-21
security:
  malicious-code: none
  suspicious-urls: none
  quality: high
---

## Summary

VAST vendor blog describing two distinct Trino integration paths: (1) standard S3 access via Hive connector against VAST DataStore for Iceberg/Parquet files, and (2) native parallel connector against VAST DataBase with query pushdown. The vendor benchmark comparing DataStore vs DataBase is internal and unverified.

## Key Points

- **Two integration modes**:
  - *DataStore + Hive S3*: `hive.s3.path-style-access=true`, standard endpoint override — works with Iceberg/Parquet files; configuration identical to MinIO or Ceph RGW
  - *DataBase native connector*: parallel connector with query pushdown; requires VAST DataBase (not DataStore); proprietary plugin installation
- **Vendor benchmark (1.4B rows comparison)**:
  - DataStore (S3 scan): 1m 24s, ~30GB transferred
  - DataBase (native pushdown): 56s, ~60MB transferred ("500x less data")
  - This is NOT a comparison against Ceph/Ozone — it compares two VAST products against each other
- **Ingestion claim**: 19 million rows/second (lab environment, unverified)
- **DataBase path requires CTAS**: Data must be imported from S3 into VAST DataBase before the native connector can be used
- **No Iceberg metadata in DataBase path**: The native DataBase connector is for VastDB tables, not Iceberg-format files; enterprises must choose between Iceberg portability and VAST-native performance

## Security Notes

No issues detected.
Checks performed:
- Malicious or obfuscated code: none
- Suspicious URLs or redirects: none
- Content quality / AI-generated: vendor blog, internally consistent

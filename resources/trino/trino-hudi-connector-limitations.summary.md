---
source: https://trino.io/docs/current/connector/hudi.html
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

Trino's Hudi connector is read-only. It cannot write to Hudi tables in any form. This is a hard architectural constraint, not a roadmap gap — it is documented in the current Trino release. For a dual Spark+Trino lakehouse, using Hudi means Trino can never ingest, update, or delete data: all writes must go through Spark.

## Key Points

- **No writes**: Connector provides read access only. No INSERT, UPDATE, DELETE, MERGE, CREATE TABLE, or DDL.
- **Metastore dependency**: Tables must be synced to Hive Metastore via Hudi's sync tool (`HoodieHiveSyncTool`). This is an operational requirement above and beyond what Iceberg needs.
- **CoW tables**: Snapshot queries only.
- **MoR tables**: Read-optimized queries only — Trino cannot query MoR tables in snapshot mode, meaning it reads compacted base files only, not the most recent log deltas. Query results may lag behind the actual table state.
- **Incremental queries**: Not supported through Trino. Downstream applications needing change feeds from Trino cannot use Hudi's incremental pull.
- **Point-in-time queries**: Not supported.
- **Format restriction**: Data must be in Parquet format.

**Implication for storage selection**: If Trino is a query engine in your lakehouse, Hudi forces Trino into a read-only role with stale MoR reads. This is acceptable only if Trino is strictly a BI/analytics surface with no mutation requirements. Any architecture that needs Trino writes must use Iceberg.

## Security Notes

No issues detected.
Checks performed:
- Malicious or obfuscated code: none
- Suspicious URLs or redirects: none
- Content quality / AI-generated: official documentation — authoritative

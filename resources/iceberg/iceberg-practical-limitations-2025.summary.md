---
source: https://quesma.com/blog/apache-iceberg-practical-limitations-2025/
component: iceberg
type: article
evidence-tier: press
accessible: true
benchmark-age: 2025
date-retrieved: 2026-05-21
security:
  malicious-code: none
  suspicious-urls: none
  quality: high — independent engineering blog, well-sourced, names specific GitHub issues and Adobe production data
---

## Summary

A practitioner analysis of Apache Iceberg's real-world limitations in 2025. This is counter-evidence to Iceberg's dominant market position. Key issues: write amplification on object storage, compaction dependency, OCC concurrency ceiling, planning stage slowness in Trino, and weak support for semi-structured data.

## Key Points

**Write amplification**: A single-row update requires writing a new Parquet file + manifest + manifest-list + metadata JSON. Each operation that takes milliseconds in a traditional database takes seconds in Iceberg due to object storage round-trip latency for metadata.

**Compaction is mandatory, not optional**: Without routine `rewrite_data_files` + `expire_snapshots` maintenance, file explosion degrades query performance progressively. These jobs require a running Spark session (or Trino/Flink OPTIMIZE calls). Not self-managing like Hudi.

**OCC concurrency ceiling**: Optimistic concurrency control limits Iceberg to "a few commits per minute." Adobe engineers reported a hard ceiling of ~15 transactions per minute. OLTP-style high-frequency writes are incompatible with Iceberg.

**Trino planning stage slowness**: In production, 5–10% of Iceberg queries on Trino take 1–10 minutes in the planning stage alone (GitHub issue #26563, documented 2025). Root cause: statistics collection overhead on large tables.

**MERGE INTO full-table scan (Trino)**: Trino's MERGE INTO on Iceberg tables can scan the entire target table even when the source data covers a single partition, causing massive read overhead and potential query failures.

**Parquet performance penalty**: Parquet+Iceberg is 2–3x slower than native columnar formats in ClickHouse and DuckDB. Snowflake reports 20% performance penalty vs native tables even with full optimization.

**No built-in row-level security**: Column masks, row filters, and dynamic views must be implemented in the catalog or query engine layer — not in Iceberg itself.

**Real-time visibility gap**: New rows are invisible until the writer completes the full Parquet upload and atomic metadata swap. Not suitable for sub-second latency monitoring.

**Semi-structured data**: VARIANT type is only being introduced in 2025; JSON-heavy schemas have poor optimization options.

## Security Notes

No issues detected.
Checks performed:
- Malicious or obfuscated code: none
- Suspicious URLs or redirects: none
- Content quality / AI-generated: practitioner blog, references specific GitHub issues and named company data (Adobe) — credible

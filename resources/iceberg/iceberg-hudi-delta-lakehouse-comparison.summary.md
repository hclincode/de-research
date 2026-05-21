---
source: https://www.onehouse.ai/blog/apache-hudi-vs-delta-lake-vs-apache-iceberg-lakehouse-feature-comparison
component: iceberg
type: article
evidence-tier: vendor-adjacent
accessible: true
benchmark-age: 2025
date-retrieved: 2026-05-21
security:
  malicious-code: none
  suspicious-urls: none
  quality: low — published by Onehouse (Hudi's commercial backer); structurally favours Hudi. Claims are verifiable but framing should be read critically.
---

## Summary

The most detailed publicly available feature matrix comparing Apache Iceberg, Delta Lake, and Apache Hudi across ACID transactions, schema evolution, concurrency, streaming/CDC, table management, and ecosystem support. Published by Onehouse (Hudi commercial backer) — treat Hudi-favourable framing with appropriate scepticism, but individual technical facts are largely verifiable against project documentation.

## Key Points

**ACID and concurrency:**
- All three support ACID transactions.
- Hudi uniquely offers non-blocking concurrency control (NBCC) — writers do not fail or retry on conflict. Delta and Iceberg use optimistic concurrency control (OCC) — conflicts cause writer retries. For continuous streaming ingestion on-premise, NBCC eliminates ingestion stalls.
- Delta requires an external lock provider for multi-writer (DynamoDB by default — not easily available on-premise). Hudi offers Zookeeper or HBase as alternatives.
- Iceberg delegates conflict resolution to the catalog.

**Write operations:**
- Hudi: full Merge-on-Read, partial column updates, bulk insert, 8+ index types (including record-level global index for key-based upserts).
- Delta: Copy-on-Write primary; deletion vectors experimental (limited MoR). No partial updates.
- Iceberg: deletion vectors with one-per-file constraint; no partial updates; no native ingestion service.

**Streaming and CDC:**
- Hudi: full CDC (before/after images + change type); DeltaStreamer for managed ingestion from Kafka/Debezium; incremental queries with deletes.
- Delta: Change Data Feed (experimental); Autoloader is proprietary Databricks-only.
- Iceberg: incremental append queries only; no native CDC primitives; no ingestion service.

**Table management (compaction, clustering, cleaning):**
- Hudi: fully automated async compaction, file size management, Z-order/Hilbert curve clustering, snapshot cleaning — all open-source and self-managing.
- Delta: auto-compaction for file sizing; Z-order is open-source but auto-optimize is Databricks-only.
- Iceberg: all compaction and maintenance is manual; no built-in services.

**Performance (TPC-DS):**
- Delta ≈ Hudi (when Hudi configured to bulk-insert mode, not upsert).
- Iceberg consistently slowest of the three.
- Hudi default config optimises for upsert (slower on pure reads); requires tuning for append-only analytical workloads.

**Catalog dependency:**
- Hudi: catalog optional. Can run without any external catalog.
- Delta: catalog optional.
- Iceberg: catalog required. No catalog = no consistent reads across multiple writers.

**Governance:**
- Hudi: Apache Software Foundation (ASF). No single commercial controller.
- Iceberg: Apache Software Foundation (ASF).
- Delta Lake: Linux Foundation. Databricks employees dominate TSC and commit history despite stated open governance.

## Security Notes

No issues detected.
Checks performed:
- Malicious or obfuscated code: none
- Suspicious URLs or redirects: none
- Content quality / AI-generated: structurally written, factual claims are verifiable, but framing favours Hudi as author's commercial product

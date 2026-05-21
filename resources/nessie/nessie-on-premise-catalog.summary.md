---
source: https://projectnessie.org + https://lakefs.io/blog/nessie-catalog/
component: nessie
type: article
evidence-tier: independent
accessible: true
date-retrieved: 2026-05-21
security:
  malicious-code: none
  suspicious-urls: none
  quality: high — project documentation + independent editorial
---

## Summary

Project Nessie is an open-source (Apache 2.0) transactional catalog for Apache Iceberg that provides Git-like versioning semantics (branches, tags, commits) over table metadata. It is the strongest self-hosted catalog option for on-premise Iceberg deployments, using PostgreSQL as a durable backend. Nessie reached 28.6% adoption in the 2025 Iceberg ecosystem survey — second only to AWS Glue (39.3%) in overall usage.

## Key Points

**License**: Apache 2.0. Dremio is the primary backer but does not commercially gate the open-source project.

**Deployment model for on-premise:**
- Lightweight Java REST API server.
- Production backend: PostgreSQL (JDBC). Also supports RocksDB (embedded, simpler but single-node).
- Runs in Docker or Kubernetes. No cloud dependency.
- Implements the Iceberg REST Catalog specification — any Iceberg-compatible engine (Spark, Trino, Flink, Presto) connects using standard config.

**Git-like data versioning:**
- Create named branches of the catalog (e.g., `dev`, `staging`, `prod`).
- Experiment with schema changes or table rewrites on a branch without affecting production readers.
- Merge branches with conflict detection.
- Tag specific catalog states for reproducible point-in-time queries.
- Commit history provides full audit trail of all table changes.

**ACID transactions:**
- Nessie uses an optimistic concurrency control model with compare-and-swap (CAS) operations on the PostgreSQL backend.
- Provides cross-table transactional consistency — multiple Iceberg tables can be updated atomically within a single Nessie commit.
- This is a key advantage over Hive Metastore (no cross-table transactions).

**On-premise fit:**
- Full data sovereignty — metadata never leaves the data center.
- PostgreSQL backend can be deployed on existing on-premise database infrastructure.
- OIDC-compatible authentication (integrates with enterprise LDAP/AD via Keycloak or similar).

**Limitations:**
- Nessie is Iceberg-native. Does not support Hudi or Delta Lake table management (those formats have their own catalog mechanisms).
- Operational burden of PostgreSQL availability and backup falls on the data engineering team.
- Branching/merging semantics require training for teams unfamiliar with Git-style data management.

## Security Notes

No issues detected.
Checks performed:
- Malicious or obfuscated code: none
- Suspicious URLs or redirects: none
- Content quality / AI-generated: project documentation + independent editorial, both credible

# Storage Service Debate — Iteration 4: HMS Constraint Re-evaluation

**Date:** 2026-05-22  
**Trigger:** Data team switches from Apache Polaris REST catalog to Iceberg + Hive Metastore (HMS) backed by MySQL/PostgreSQL.  
**Question:** Does this flip the storage recommendation from Ozone 2.1 to Ceph?  
**Answer: No. Recommendation unchanged. Operational sign-off conditions tighten.**

---

## Task A — HMS commit path and HDDS-13117: still a non-blocker (both TLs verified independently)

**Finding:** Iceberg's `HiveCatalog` writes `metadata.json` to object storage with an **unconditional PUT** and performs CAS at the HMS database level via the Thrift `alter_table` call, protected by an HMS lock and backed by the metastore RDBMS transaction. This is structurally identical to the Polaris pattern (Postgres CAS at the catalog DB layer). HDDS-13117 (Ozone S3G's incomplete `If-None-Match` conditional PUT, targeted for 2.2) is not on the HMS commit path and remains a non-blocker.

**Evidence chain (independently verified by both TLs from source code):**

1. `HiveTableOperations.doCommit()` writes the new metadata file first — at a fresh UUID-suffixed path (`metadata/NNNNN-<uuid>.metadata.json`) — before acquiring any HMS lock. The write uses `BaseMetastoreTableOperations.writeNewMetadata()` which calls `TableMetadataParser.overwrite()` with the explicit comment: *"use overwrite to avoid negative caching in S3. this is safe because the metadata location is always unique because it includes a UUID."* No `If-None-Match` header. ([HiveTableOperations.java, apache/iceberg main](https://github.com/apache/iceberg/blob/main/hive-metastore/src/main/java/org/apache/iceberg/hive/HiveTableOperations.java))

2. After writing, Iceberg acquires an HMS lock (row in `HIVE_LOCKS`), re-reads `metadata_location` to verify it still matches the base, then issues Thrift `alter_table` to CAS the pointer. Lock is released after confirmation. CAS = HMS RDBMS transaction. ([Iceberg PR #2547](https://github.com/apache/iceberg/pull/2547), [Iceberg HiveTableOperations Javadoc](https://iceberg.apache.org/javadoc/master/org/apache/iceberg/hive/HiveTableOperations.html))

3. "Lock-free" HMS path (HIVE-26882 / HIVE-28121): uses `environmentContext` on `alterTable` with `expected_parameter_key=metadata_location, expected_parameter_value=<prior>` — still pure HMS DB-level CAS, no S3 conditional required. Requires HMS 2.3.10 or 4.0.0-beta-1+ plus HIVE-28121 for MySQL/MariaDB. ([Trino #22182](https://github.com/trinodb/trino/issues/22182), [Trino PR #25445](https://github.com/trinodb/trino/pull/25445))

4. Iceberg metadata naming design: metadata files land at UUID-suffixed paths by construction — no name collision possible between concurrent writers. Iceberg's design predates S3 conditional writes precisely because the catalog provides CAS. ([iceberg-file-name-convention, tomtan.dev 2025](https://tomtan.dev/blog/2025-01-12-iceberg-file-name-convention/), [Iceberg reliability docs](https://iceberg.apache.org/docs/1.5.0/reliability/))

**Sign-off condition #3 update (from iter3):** "Polaris + Postgres catalog confirmed" rewrites to: **"HMS-backed Iceberg catalog with HMS locks enabled (default), OR lock-free commits with HIVE-26882/HIVE-28121 patches present. NOT HadoopCatalog or filesystem catalog."**

**New HMS-specific risk (storage-agnostic — affects Ceph equally):**
- HMS lock leaks if a writer dies mid-commit ([Iceberg #2301](https://github.com/apache/iceberg/issues/2301))
- HMS 4.0 had a lock-creation regression ([Iceberg #11784](https://github.com/apache/iceberg/issues/11784)) — pin HMS 3.1.3 or 4.0.x post-fix
- Trino concurrency bug with lock-free mode ([Trino #27942](https://github.com/trinodb/trino/issues/27942)): `CommitStateUnknownException` under concurrent commits with `iceberg.hive-catalog.locking-enabled=false`. **Mitigation: leave HMS locks enabled (the default) unless HIVE-26882 patches are confirmed present.** This is a metastore-tier problem, identical on Ceph.

---

## Task B — Security model without Polaris

**The InfoSec posture is materially weaker under HMS.** This is the most significant operational change.

**What HMS removes:**
- Polaris's native per-table, per-operation short-lived credential vending (no equivalent exists in HMS or in any Iceberg+HMS implementation as of 2026 — [Dremio on credential vending](https://www.dremio.com/blog/iceberg-credential-vending/), [Snowflake vended creds docs](https://docs.snowflake.com/en/user-guide/tables-iceberg-configure-catalog-integration-vended-credentials))
- The iter3 InfoSec fallback: "switch to Polaris vended-credentials" — **this fallback no longer exists**

**Blast radius under HMS:** A compromised `spark-prod` or `trino-prod` static S3 key gives access to the entire lakehouse volume prefix. Ranger denials at the HMS metadata layer do not prevent direct S3G access by an attacker who holds the raw credential, because they bypass the catalog entirely.

**Critical:** this weakness is **identical on Ceph**. Ceph RGW uses the same AWS-style static keypairs. Neither storage backend provides Polaris-equivalent vended creds under HMS. Choosing Ceph does not improve the InfoSec posture.

**Compensating controls (accepted by both TLs):**
- Per-service-account key separation (spark-prod, trino-prod, ingest, polaris-svc, ops-admin)
- **30-day rotation** (reduced from 90-day in iter3) to shorten exposure window
- S3G on internal-only VLAN, not routable from corporate/production VPCs
- Ranger policies on Ozone OM keyed to Kerberos principals for HMS-tier table access
- Ranger audit logs → SIEM with anomaly alerting (new addition under HMS)
- Explicit roadmap commitment to revisit credential vending when Iceberg/HMS or Ozone STS (HDDS-13323) provides it

**InfoSec sign-off status:** upgraded from "tracked execution item" to **hard go/no-go gate before PoC PO is signed** (storage-tl). The prior Polaris vended-creds fallback no longer exists; if InfoSec rejects the static-creds posture, the only remaining paths are (a) wait for Ozone 2.2 STS — no release date, (b) implement an external STS proxy — significant engineering effort. Both TLs must present the updated compensating controls to InfoSec before hardware procurement.

**Ranger + Kerberos timeline:** Must compress from "Q2 2027 ML onboarding milestone" to **production-ready at mid-December 2026 lakehouse cutover**. Under HMS, Ranger+Kerberos is the only authorization layer between compute service accounts and Ozone — it cannot be deferred to ML onboarding. Kickoff deadline: **2026-06-30** (parallel to PoC hardware procurement). This is the biggest concrete schedule change from the HMS switch.

**HMS operational overhead (storage-agnostic):** HMS+MySQL/Postgres HA pair + Thrift service cluster + ZooKeeper = new HA dependency on the data team side. If HMS goes down: Iceberg commits fail, catalog reads fail, but raw S3G access to already-known Ozone objects continues unaffected. Storage-side independence is preserved. However, **HMS RDBMS must be provisioned as a dedicated HA pair**, isolated from any other RDBMS tenants, to prevent cross-tenant blast radius. HMS lock-leak and Thrift latency SLAs should be added to the PoC monitoring scope.

---

## Task C — Does Ceph get relatively more attractive under HMS?

**No. The Ozone advantage widens slightly.**

**C1 — Reference architecture:** Cloudera CDP has shipped **Ozone + HMS + Iceberg + Hive/Spark** as a combined production stack for enterprise customers for multiple years ([Cloudera Iceberg on Ozone, 2023](https://medium.com/engineering-cloudera/open-data-lakehouse-powered-by-apache-iceberg-on-apache-ozone-a225d5dcfe98)). The Ozone+Polaris reference was the newer integration; HMS+Ozone is the older, more battle-tested one. By removing Polaris, the data team is moving to the stack that has the longer production track record with Ozone. Ceph+HMS+Iceberg has community deployments but no single-vendor production reference of comparable scale ([IOMETE self-hosted](https://iomete.com/resources/blog/self-hosted-data-lakehouse-kubernetes), [IBM Storage Ceph redbook](https://www.redbooks.ibm.com/redpieces/pdfs/sg248563.pdf)).

**C2 — Ceph's S3 conditional PUT support is now irrelevant:** Ceph RGW Tentacle does support `If-None-Match` / `If-Match` ([Ceph PR #65949](https://github.com/ceph/ceph/pull/65949), [PR #67425](https://github.com/ceph/ceph/pull/67425)), but since HMS CAS lives at the database layer, this capability is never exercised by the Iceberg commit path. Zero practical advantage to Ceph from this dimension under HMS.

**C3 — Multi-tenancy under HMS:** HMS supports namespace separation via `database` constructs (`lakehouse.*` / `ml.*`) with Ranger policies — identical pattern to what was planned under Polaris but without catalog-layer credential isolation. Bucket-prefix-scoped S3G keypairs + Ranger volume/bucket ACLs cover the gap. The ML Q2 2027 onboarding plan is unchanged in structure, just the authorization enforcement surface is different (Ranger ACLs only, no vended creds).

**C4 — Trino + HMS connector maturity:** Fully mature in Trino 481. The HMS Thrift catalog connector is the oldest and most-deployed Trino Iceberg catalog configuration. `fs.native-s3.enabled=true` is catalog-independent — it operates identically whether the catalog is HMS, REST, Nessie, or Glue. ([Trino 481 Iceberg connector](https://trino.io/docs/current/connector/iceberg.html))

---

## Iteration 4 Change Table

| Dimension | Iter3 (Polaris) | Iter4 (HMS) | Storage flip? |
|---|---|---|---|
| HDDS-13117 conditional PUT | Non-blocker — Polaris Postgres CAS | Non-blocker — HMS Thrift+RDBMS CAS | **No** |
| Per-table RBAC | Polaris native fine-grained | Ranger plugin, metadata-tier only | No — identical pain on Ceph |
| Vended short-lived creds | Yes (Polaris per-table per-op) | None (long-lived static keys) | No — identical on Ceph |
| InfoSec fallback | Polaris vended-creds | **None — hard gate now** | No — affects both backends |
| Ranger+Kerberos timeline | Q2 2027 (ML onboarding) | **Mid-Dec 2026 (at cutover)** | N/A |
| Credential rotation interval | 90 days | **30 days** | N/A |
| Reference architecture age | Newer (Ozone+Polaris, Apr 2026) | More mature (Ozone+HMS+CDP, years) | Favors Ozone |
| Trino connector | REST (newer) | HMS Thrift (most-deployed) | Neutral |
| Ceph conditional-PUT advantage | Moot (Polaris CAS) | Moot (HMS CAS) | **No advantage to Ceph** |
| New HMS risks | None | Lock leaks, lock-bug regression, Trino #27942 | Identical on Ceph |

---

## Updated Sign-Off Conditions

Replacing iter3 conditions:

1. **InfoSec hard gate (before PO signed):** Written approval of updated static-creds posture with compensating controls (30-day rotation, SIEM anomaly alerting, VLAN isolation, per-service keypairs, Ranger audit trail). No Polaris fallback. Deadline: before hardware PO — target 2026-06-12.

2. **PoC load test:** p95 GetObject ≤ 80 ms (formal), ≤ 60 ms (target) for sub-1 MB at 80% peak concurrency. No LIST failures. No OM Raft churn. **Add HMS lock-acquisition latency and HMS Thrift p95 to PoC SLA monitoring.** Add a concurrent-commit torture test (N Spark jobs + N Trino queries simultaneously committing to distinct tables) to validate HMS lock behavior under load.

3. **Catalog confirmed:** HMS-backed Iceberg with HMS locks enabled (default), OR lock-free mode with HIVE-26882/HIVE-28121 patches confirmed present. NOT HadoopCatalog. HMS RDBMS as a dedicated HA pair (not shared with other tenants).

4. **Hardware procurement:** within 16 weeks of PO, slip >20 weeks triggers re-scope.

5. **NEW — Ranger+Kerberos kickoff by 2026-06-30:** Must be production-ready at mid-December 2026 cutover. 4–6 month deployment window requires June start, not Q2 2027.

---

## Final Joint Verdict

**Storage recommendation: Apache Ozone 2.1 + S3G. Unchanged.**

The HMS switch does not flip the storage decision:
- HDDS-13117 is not on the HMS commit path (independently confirmed from `HiveTableOperations.java` by both TLs)
- The security posture weakens, but identically on Ceph — it is a catalog/credential architecture problem, not a storage problem
- Ozone+HMS+Iceberg is a more mature integration than Ozone+Polaris, not a less mature one
- Ceph's improved S3 conditional-PUT support (Tentacle) is irrelevant because HMS CAS lives at the RDBMS layer

The primary concrete changes from the Polaris→HMS switch are **operational, not architectural**: Ranger+Kerberos timeline compresses by 6 months, key rotation tightens from 90 days to 30 days, InfoSec sign-off becomes a hard gate, and HMS lock-leak/concurrency monitoring is added to the PoC scope.

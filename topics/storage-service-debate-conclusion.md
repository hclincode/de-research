# Joint Storage Service Recommendation
## Lakehouse Data Team TL + Storage Service Team TL

**Date:** 2026-05-22  
**Status:** Final — both tech leads signed off  
**Debate record:** iter1 → iter2 → iter3 → iter4 (this document)  
**Catalog trigger:** Data team switched from Apache Polaris REST catalog to Iceberg + Hive Metastore (HMS) backed by MySQL/PostgreSQL after iter3. Iter4 re-evaluated whether this flips the storage recommendation. **It does not.**

---

## Verdict

**Apache Ozone 2.1.0 with S3 Gateway (S3G), backed by Iceberg + Hive Metastore (HMS) on MySQL/PostgreSQL, is the recommended open-source on-premise storage service for the Spark + Trino enterprise lakehouse.**

The iter3 recommendation was made under Apache Polaris REST catalog. Iter4 re-evaluated the full recommendation after the data team switched to HMS. The storage recommendation is unchanged. HDDS-13117 (Ozone S3G's incomplete `If-None-Match` conditional PUT) is not on the HMS commit path; Ozone+HMS+Iceberg is a more mature integration than Ozone+Polaris; and HMS's weaker security posture (no per-table vended credentials) is identical on Ceph — it is not a storage problem.

Ceph RadosGW (Tentacle v20) is the credible plan B, activated only if the Ozone PoC fails the load test or if any of four defined trigger conditions is met (see below).

HDFS, MinIO, JuiceFS, and SeaweedFS are eliminated.

---

## Architecture

```
Spark executors  ──────► S3A (s3a://)  ────►┐
                                              ├──► HAProxy ──► 6× S3G instances ──► Ozone OM/SCM/Datanodes
Trino workers    ──────► native S3     ────►┘
                              │
                              └──► Iceberg HiveCatalog ──► HMS Thrift ──► HMS RDBMS (MySQL/Postgres HA pair)
                                       │ unconditional PUT (UUID-suffix) then Thrift alter_table CAS
```

- **Spark** uses S3A connector (`s3a://`) pointing at the Ozone S3G endpoint
- **Trino** uses `fs.native-s3.enabled=true`, same S3G endpoint
- **Iceberg atomic commits**: `HiveTableOperations.java` writes `metadata.json` to a fresh UUID-suffixed path via unconditional PUT, then acquires an HMS lock and calls Thrift `alter_table` with CAS backed by the HMS RDBMS transaction. S3 conditional PUT is not in the commit path (independently verified by both TLs, iter4 Task A). HMS locks must remain **enabled** (the default) unless HIVE-26882/HIVE-28121 patches are confirmed present.
- **S3G tier:** 6 instances behind HAProxy active/passive, sticky-by-bucket affinity
- **Ozone metadata tier:** 3-node OM Raft + Listener OMs (read-only followers for manifest lookups), 3-node SCM Raft, all-NVMe RocksDB + Ratis
- **Ozone data tier:** Datanodes with enterprise NVMe, RS-3-2 erasure coding (1.5× storage overhead)
- **Authorization:** Kerberos principals + Apache Ranger ACLs at OM (metadata tier only — no per-table vended credentials under HMS)
- **Credentials:** static AWS-style keypairs per service account (spark-prod, trino-prod, ingest, ops-admin), stored in HashiCorp Vault, **30-day rotation** (tightened from 90-day in iter3), S3G on internal-only VLAN not routable from corporate/production VPCs
- **HMS RDBMS:** dedicated HA pair (not shared with other tenants); HMS + Thrift + ZooKeeper is a new HA dependency owned by the data team

---

## Why Ozone over Ceph (primary rationale)

| Dimension | Ozone 2.1 | Ceph RadosGW (Tentacle) |
|---|---|---|
| **Trino integration** | Native — `fs.native-s3.enabled=true`, working Polaris reference architecture (Apr 2026) | Works — same native S3 config; no formal Trino certification |
| **Spark integration** | Native — S3A + OFS; Iceberg FSO buckets | Works — S3A via RadosGW |
| **Operational surface** | 3 daemon types (OM, SCM, DN), bounded tuning | 5+ daemon types, CRUSH, PG, BlueStore, EC profiles, mclock — non-linear interaction |
| **Metadata scalability** | Distributed SCM/OM with RocksDB; DiDi 5B files/cluster P90 17ms OM metadata | RGW bucket-index sharding required at >10M objects/bucket |
| **EC + metadata** | Small-file metadata tier (OM RocksDB) is separate from data EC — no amplification on manifests | Requires deliberate dual-pool design (replicated metadata pool + EC data pool) to avoid manifest EC amplification |
| **Failure recovery** | Per-container re-replication; no cluster-wide recovery cascade | 15 TB OSD failure triggers cluster-wide PG recovery I/O competing with client I/O |
| **License** | Apache 2.0 (ASF) | LGPL 2.1 (Linux Foundation) — permissive, no AGPL risk |
| **Cross-DC DR** | DistCp batch async, RPO ~hours | RGW multisite async, RPO ~minutes (still non-zero); RPO < 1h achievable, not RPO = 0 |
| **Operational experience** | HDFS-adjacent; Cloudera CDP production reference; DiDi 500+ PB | Broad community, CERN/Bloomberg production references |

**Single-cluster Spark+Trino deployment with RPO 24h and no immediate multi-protocol requirement:** Ozone wins on operational simplicity and metadata scale fit.

**Under HMS (iter4):** Ceph's improved S3 conditional-PUT support (RGW Tentacle) is irrelevant because HMS CAS lives at the RDBMS layer — it is never exercised by the Iceberg commit path. The security posture (static keypairs, no per-table vended credentials) is **identical on both backends**; it is a catalog architecture problem, not a storage problem. Ozone+HMS+Iceberg has a longer production track record (Cloudera CDP, years in enterprise deployments) than Ozone+Polaris. The Ozone advantage is unchanged or widens slightly under HMS.

---

## Why candidates were eliminated

| Candidate | Reason |
|---|---|
| **HDFS** | Trino 470 (Feb 2025) deprecated Hadoop filesystem libraries; `fs.hadoop.enabled=true` is a compatibility shim accumulating debt every release. Disqualified for new deployments. |
| **MinIO CE** | AGPL license + community edition archived in early 2026 (maintenance mode). Eliminated on license and sustainability grounds in prior research. |
| **JuiceFS CE** | Requires an external HA metadata engine (Redis/TiKV/PostgreSQL) as a critical dependency. Two HA systems instead of one. TiKV 44× slower than Redis on small-file concurrent workloads. Hard pass for primary lakehouse storage. |
| **SeaweedFS** | Full-cluster backup requires write pause. Filer metadata consistency under partition is "closed / not planned." 2–3 releases/week in 2026 with critical EC regressions. No independently verified petabyte-scale lakehouse deployment. Watch item for 2027+. |

---

## Conditions for Ozone to remain the recommendation

All five must hold. If any fails, both teams reconvene before further capex:

1. **InfoSec hard gate (before PO signed):** Written approval of updated static-creds posture with compensating controls: 30-day rotation, SIEM anomaly alerting, VLAN isolation, per-service keypairs (spark-prod, trino-prod, ingest, ops-admin), Ranger audit trail. No Polaris vended-credentials fallback exists under HMS. Deadline: **2026-06-12**. If InfoSec rejects: only remaining paths are (a) wait for Ozone 2.2 STS (HDDS-13323, no release date) or (b) implement an external STS proxy (significant engineering). Both are worse than the compensating controls.

2. **PoC load test passes** formal acceptance criterion: p95 GetObject ≤ 80 ms for sub-1 MB objects at 80% peak concurrency (320 Spark executors + 240 Trino workers), zero LIST failures, zero OM Raft leader churn, over a 30-minute soak. Target ("good") threshold is p95 ≤ 60 ms with the tuning levers committed by storage-tl. Reject Ozone if p95 > 100 ms or any LIST failures → activate Ceph plan B. **Add to PoC scope:** HMS lock-acquisition latency, HMS Thrift p95, and a concurrent-commit torture test (N Spark + N Trino jobs committing to distinct tables simultaneously).

3. **Catalog confirmed:** HMS-backed Iceberg with HMS locks enabled (default), OR lock-free commits with HIVE-26882/HIVE-28121 patches confirmed present. **NOT HadoopCatalog or any filesystem catalog.** HMS RDBMS must be a dedicated HA pair, not shared with other tenants. (Confirmed by both TLs in iter4.)

4. **Hardware procurement delivers within 16 weeks** of PO signed (slip beyond 20 weeks triggers re-scope).

5. **Ranger+Kerberos kickoff by 2026-06-30:** Must be production-ready at mid-December 2026 lakehouse cutover. Under HMS, Ranger+Kerberos is the only authorization layer between compute service accounts and Ozone — it cannot be deferred to ML onboarding. The iter3 Q2 2027 ML milestone is no longer the driver; the Dec 2026 cutover is.

---

## Ceph plan B — activation criteria and readiness

**Activate Ceph if any of:**
- (a) Ozone PoC fails load test (p95 > 100 ms or LIST failures)
- (b) Cross-DC active-active replication with RPO < 1h is required in future (not currently required; RPO 24h confirmed)
- (c) Multi-protocol storage (object + block + filesystem) required from day one (not currently required)
- (d) Storage team can hire two senior Ceph engineers and budget IBM Storage Ceph support (currently not the case)
- (e) Ozone 2.1 ships a production regression breaking the load-test SLA and the fix slips past Q4 2026

**Ceph plan B spec (dual-pool design):**
- All-NVMe 3× replication pool for Iceberg metadata prefix (`/metadata/*`) — eliminates EC amplification on small files
- All-NVMe EC 4:2 pool for Parquet data files (`/data/*`) — storage efficiency for large objects; FastEC (Tentacle) for partial reads
- 6 RGW instances behind HAProxy (mirrors S3G topology)
- Separate cluster network per OSD node (non-negotiable per official Ceph guidance)
- Stand-up delta vs. Ozone: +2 weeks (CRUSH map, dual-pool placement targets, BlueStore + mclock tuning)
- Additional PoC duration if triggered: 13–18 weeks (sequential, not parallel, due to team size and hardware constraints)

---

## PoC plan

**Purpose:** Produce the first independent Ozone 2.1 benchmark for Iceberg manifest-heavy Trino+Spark workloads at our concurrency profile. Also serves as team training for both TLs.

**Hardware spec (~20 rack units):**
| Component | Count | Spec |
|---|---|---|
| Datanodes | 6 | 32-core, 256 GB RAM, 8× 7.68 TB NVMe, 25 GbE dual-port |
| OM nodes (Raft + Listener) | 3 | 32-core, 256 GB RAM, 2× 3.84 TB NVMe, 25 GbE |
| SCM nodes | 3 | 16-core, 64 GB RAM, 2× 1.92 TB NVMe, 25 GbE |
| S3G nodes | 6 | 16-core, 64 GB RAM, 25 GbE (VMs acceptable) |
| HAProxy | 2 | 8-core, 32 GB RAM, 25 GbE (active/passive) |
| Load generators | 4 | 32-core, `warp` S3 + custom Iceberg-pattern script |

**Workload:** 10M small objects (4 KB / 16 KB / 64 KB uniform — manifest-list, manifest, stats files) + 100K Parquet @ 128 MB. Dataset generation script: lakehouse-tl, delivered by week 6.

**Timeline:**
| Week (from today) | Milestone | Owner |
|---|---|---|
| +2 (early Jun 2026) | PoC plan doc + success criteria + dataset schema | Joint |
| +4 (mid Jun) | Hardware in lab DC | Storage-tl |
| +6 (early Jul) | Ozone 2.1 + HMS + MySQL/Postgres + Trino + Spark deployed | Split |
| +8 (mid Jul) | Dataset loaded | Lakehouse-tl |
| +10 (end Jul) | Load test: 4-hour soak, 3 runs | Joint |
| +12 (mid Aug) | **Go/no-go signed by both TLs** | Joint |
| +24 (early Nov) | Production cluster build-out begins | Storage-tl |
| +~30 (mid Dec) | Production cutover | Joint |

---

## Multi-tenancy roadmap

ML training team joining storage cluster: **Q2 2027 target** (unchanged).

**Ranger+Kerberos timeline compressed (iter4 change):** Under iter3 (Polaris), Ranger+Kerberos was planned to be production-ready at Q2 2027 ML onboarding. Under HMS, it is the **only** authorization layer between compute service accounts and Ozone — it must be production-ready at mid-December 2026 lakehouse cutover instead. **Kickoff deadline: 2026-06-30** (parallel to PoC hardware procurement). Storage-tl estimates 4–6 months from kickoff; June start gives the deployment window.

Required by ML onboarding: volume-level ownership separation (`/lakehouse` vs `/ml`), bucket-level ACL policies for ML service accounts, separate S3 keypairs per ML service account, quota enforcement to prevent ML from starving lakehouse. Cross-team meeting scheduled 2026-05-28 to validate ML team's credential and access requirements.

**Blast-radius mitigation:** All Ranger policy changes via PR-reviewed Terraform/Ansible. No direct UI edits in prod.

---

## Open items tracked (not unresolved — execution only)

| Item | Owner | Deadline |
|---|---|---|
| InfoSec written approval of static-creds posture (updated for HMS — no Polaris fallback) | Both TLs jointly | 2026-06-12 |
| PoC plan doc finalized (add HMS lock torture test to scope) | Joint | 2026-06-04 |
| Hardware PO submitted | Storage-tl | 2026-06-04 |
| **Ranger+Kerberos kickoff** (new iter4 item — must start in June for Dec 2026 cutover) | Storage-tl | **2026-06-30** |
| Dataset generation script | Lakehouse-tl | 2026-07-04 |
| HMS RDBMS dedicated HA pair provisioned | Data team | Before PoC deploy |
| ML team credential expectations confirmed | Lakehouse-tl | 2026-05-28 |
| Go/no-go decision | Joint | 2026-08-14 (target) |

---

## Evidence base

| Claim | Source | Tier | Confidence |
|---|---|---|---|
| Trino deprecating Hadoop filesystem | [trino-legacy-filesystem-deprecation.summary.md](../resources/trino/trino-legacy-filesystem-deprecation.summary.md) | official | HIGH |
| Ozone S3G + Trino + Polaris working reference architecture | [trino-ozone-polaris-lakehouse-guide.summary.md](../resources/ozone/trino-ozone-polaris-lakehouse-guide.summary.md) | official | HIGH |
| Polaris REST catalog uses Postgres CAS (not S3 conditional PUT) | Iceberg REST OpenAPI spec, Polaris GitHub, independently verified by both TLs | official | HIGH |
| HMS HiveCatalog uses unconditional PUT + Thrift alter_table CAS (not S3 conditional PUT) | HiveTableOperations.java (apache/iceberg main), Iceberg PR #2547, independently verified by both TLs in iter4 | official | HIGH |
| Lock-free HMS path (HIVE-26882/HIVE-28121) uses DB-level CAS; Trino #27942 bug with lock-free mode | Trino #22182, Trino PR #25445, Trino #27942 | official | HIGH |
| HMS lock leaks on writer death; HMS 4.0 lock-creation regression | Iceberg #2301, Iceberg #11784 | official | HIGH |
| Ozone+HMS+Iceberg production reference (Cloudera CDP) | Cloudera Iceberg on Ozone (2023), medium.com/engineering-cloudera | vendor-adjacent | MEDIUM |
| No per-table vended credentials under HMS; identical on Ceph | Dremio credential vending blog, Snowflake vended creds docs | vendor-adjacent | HIGH (both agree) |
| DiDi 500+ PB, 5B files/cluster, P90 17ms OM metadata | [ozone-didi-production-deployment.summary.md](../resources/ozone/ozone-didi-production-deployment.summary.md) | official (ASF blog, DiDi data) | MEDIUM (not third-party audited) |
| Ceph RGW multisite is async, non-zero RPO | Ceph multisite docs (2025) | official | HIGH |
| Ceph 1 TiB/s sustained (tuned NVMe cluster) | [ceph-journey-to-1tibps-2024.summary.md](../resources/ceph/ceph-journey-to-1tibps-2024.summary.md) | press | HIGH |
| MinIO CE maintenance mode / archived | [minio-maintenance-mode-risk.summary.md](../resources/minio/minio-maintenance-mode-risk.summary.md), [minio-repo-archived-2026.summary.md](../resources/minio/minio-repo-archived-2026.summary.md) | press | HIGH |
| JuiceFS TiKV 44× slower than Redis | [juicefs-metadata-engines-benchmark-docs.summary.md](../resources/juicefs/juicefs-metadata-engines-benchmark-docs.summary.md) | official | HIGH |
| SeaweedFS backup requires write pause | [seaweedfs-lakehouse-2026 report](seaweedfs-lakehouse-2026.report.md) | official | HIGH |
| Ozone S3G conditional PUT (HDDS-13117) not GA in 2.1 | JIRA HDDS-13117, HDDS-14968 | official | HIGH |
| No independent Ozone 2.x Iceberg benchmark exists | Search by both TLs in iter2/iter3 | — | HIGH (absence confirmed) |

**Key gap:** No independent p95 latency benchmark for Ozone S3G at 200-worker concurrency. The PoC is the benchmark. Results will be published internally.

**Iter4 gap:** No independent benchmark for Ozone S3G + HMS commit latency under concurrent writers. HMS lock-acquisition latency and Thrift p95 are added to PoC monitoring scope; a concurrent-commit torture test is added to the workload plan.

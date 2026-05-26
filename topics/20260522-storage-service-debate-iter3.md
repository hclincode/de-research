# Storage Service Debate — Iteration 3: Final Blockers Resolved

**Date:** 2026-05-22  
**Status:** All 5 remaining blockers resolved. Both tech leads endorse the joint recommendation, conditional on PoC.

---

## Response A — Lakehouse Data Team Tech Lead

### Task 9 — Polaris conditional PUT path: RESOLVED, no S3 conditional PUT required

**Bottom line: the Apache Iceberg REST catalog protocol delegates commit atomicity entirely to the catalog server. Clients do NOT perform S3 conditional PUTs as part of the commit path.** HDDS-13117/HDDS-14968 (landing in Ozone 2.2) is not a blocker.

Evidence chain:

1. **The Iceberg REST OpenAPI spec** defines `POST /v1/{prefix}/namespaces/{namespace}/tables/{table}` as a server-side commit. The `UpdateTableRequest` carries `requirements` (CAS predicates: "assert current snapshot ID = X") and `updates` (metadata change descriptors). The client does NOT pre-write a finished `metadata.json` to S3 with `If-None-Match`. ([rest-catalog-open-api.yaml](https://github.com/apache/iceberg/blob/main/open-api/rest-catalog-open-api.yaml))

2. **The REST server writes the new metadata.json and performs CAS.** Per the Iceberg `RESTCatalog` Java client: "clients call `doCommit()` with REST-specific metadata management, delegating the actual metadata file writing to the catalog server." ([DeepWiki — REST Catalog](https://deepwiki.com/apache/iceberg/3.2-rest-catalog))

3. **In Polaris, CAS is a Postgres transaction.** The `UPDATE ... WHERE current_metadata_location = ?` atomic swap is at the Polaris metastore boundary. ([Apache Polaris metastores](https://polaris.apache.org/releases/1.0.0/metastores/))

4. **The S3 PUT of `metadata.json` is an ordinary PUT.** The new metadata file lands at a fresh UUID-suffixed path (`metadata/00012-<uuid>.metadata.json`) — by construction no name collision is possible. Iceberg's design predates S3 conditional writes precisely because the catalog provides atomicity. ([Iceberg commits — Alex Merced](https://iceberglakehouse.com/posts/2026-04-29-iceberg-masterclass-06/))

5. **S3 `If-None-Match` in Iceberg is opt-in belt-and-suspenders for the filesystem catalog path only** (HadoopCatalog, no REST server). Not the Polaris path. ([Iceberg issue #6514](https://github.com/apache/iceberg/issues/6514))

**Request to storage-tl for PoC:** validate that concurrent Spark/Trino metadata.json PUTs to different keys survive without S3G-side corruption. This is basic S3G durability under concurrent writers, not a conditional-PUT test.

### Task 10 — SLA adjusted to p95 <80 ms

No independent evidence found that Ozone S3G achieves p95 <50 ms at our concurrency. **Formal PoC acceptance criterion: p95 GetObject <80 ms for sub-1 MB objects at 80% of projected peak concurrency.** Math: 6 manifest/footer GETs per Trino worker × 80 ms = 480 ms metadata wait, leaving 2.5 s for data I/O and execution, meeting the 3 s end-to-end Trino SLA. 50 ms remains the aspirational "good" threshold; 80 ms is the formal pass/fail line. >80 ms triggers retune-then-Ceph.

### Task 11 — PoC timeline

| Week | Milestone | Owner |
|---|---|---|
| +2w (early Jun) | Joint PoC plan doc + success criteria finalized | Joint |
| +4w (mid Jun) | Hardware in lab DC | Storage-tl |
| +6w (early Jul) | Ozone 2.1 + Polaris + Postgres + Trino + Spark deployed | Split |
| +8w (mid Jul) | Synthetic dataset loaded (500K objects, 50 TB scaled-down) | Lakehouse |
| +10w (end Jul) | 4-hour soak load test, 3 runs | Joint |
| +12w (mid Aug) | Go/no-go decision document signed by both TLs | Joint |
| +24w (early Nov) | Production cluster build-out begins (assuming go) | Storage-tl |
| +~30w (mid Dec 2026) | Production cutover | Joint |

Lakehouse commitments: dataset generation script and Trino/Spark traffic profile replay harness by week 6.

### Task 12 — InfoSec engagement

Meeting requested for 2026-05-26. Written sign-off target: 2026-06-12 (3 weeks). Ask:

> "S3 Gateway endpoints for the lakehouse Ozone cluster will accept static long-lived AWS-style access key + secret credentials, scoped per service account (Spark, Trino, ingest). Network access to the S3G endpoints is restricted to the data-plane VLAN and is not routable from corporate or production VPCs. Per-table authorization is enforced at the Apache Polaris REST catalog layer using Polaris RBAC. Credential rotation is performed quarterly via a managed secret vault."

Fallback if InfoSec rejects: **Polaris vended-credentials flow** — Polaris issues short-lived per-table credentials from its own authority; underlying S3G keys remain static. This is a Polaris catalog change, not a storage change. Not a Ceph trigger. ([StarRocks + Polaris integration](https://polaris.apache.org/blog/2025/10/21/starrocks-and-apache-polaris-integration-building-a-unified-high-performance-data-lakehouse/))

### Task 13 — ML team credential expectations

Meeting with ML platform tech lead scheduled 2026-05-28. Assumed requirement: separate S3 keypairs per ML service account under `/ml` volume; Iceberg access via Polaris (separate namespace/principal); raw object access for training data/model artifacts; read-only access to select lakehouse tables. Standard Ozone multi-tenancy S3 pattern with Ranger volume/bucket ACLs. Ranger + Kerberos must be online before Q2 2027 ML onboarding.

### Lakehouse TL final sign-off

All evidence-level issues are closed. Remaining items are execution:
- PoC must pass (criteria written, fallback defined)
- InfoSec sign-off pending (deadline set, fallback ready)
- ML team confirmation pending (meeting scheduled, assumption documented)

**Lakehouse TL endorses the joint recommendation, conditional on PoC passing.**

---

## Response B — Storage Service Team Tech Lead

### Task 9 — Polaris conditional PUT path: RESOLVED, concern withdrawn

Independent research confirms: **Polaris uses catalog-side CAS (Postgres), not S3 conditional PUT.** HDDS-13117 is not a blocker.

1. Iceberg spec assigns CAS responsibility to the catalog, not FileIO. ([Olake, 2025](https://olake.io/blog/2025/10/03/iceberg-metadata/), [Polaris 1.4.0 docs](https://polaris.apache.org/releases/1.4.0/))
2. Polaris stores the metadata pointer in its own Postgres backend; the pointer swap is a database transaction. ([Polaris GitHub](https://github.com/apache/polaris), [Polaris-PG-MinIO walkthrough](https://medium.com/@georgelza/apache-iceberg-with-apache-polaris-as-catalog-with-postgresql-for-persistence-minio-object-store-dd7003d99723))
3. RESTCatalog client issues an unconditional PUT for `metadata.json` (fresh UUID-suffixed path), then POSTs to Polaris for the CAS pointer update.
4. S3 `If-None-Match` is opt-in belt-and-suspenders for filesystem catalog path only. ([4sysops](https://4sysops.com/archives/aws-s3-conditional-writes-the-if-none-match-parameter/), [MinIO](https://blog.min.io/leading-the-way-minios-conditional-write-feature-for-modern-data-workloads/))

**Concern withdrawn.** Watch set on HDDS-13117 and Iceberg REST spec evolution.

### Task 10 — p95 counter-offer: 60 ms

Engineering judgment on the production-spec cluster: p95 ~20–30 ms on a hot, well-tuned cluster; p99 80–150 ms with JVM GC and OM RocksDB compaction. **Counter-offer: PoC target p95 ≤ 60 ms, p99 ≤ 150 ms.** Tuning levers committed: G1GC with 200 ms pause cap on S3G/OM JVMs; OM RocksDB block cache ~150 GB; Listener OMs for read-only manifest lookups; HAProxy keep-alive. If lakehouse-tl accepts 80 ms as the formal pass/fail line, 60 ms becomes the "good" threshold — agreed.

### Task 11 — PoC hardware spec and timeline

**Minimum PoC cluster (~20 rack units):**
- 6 datanodes: 32-core, 256 GB RAM, 8× 7.68 TB NVMe, 25 GbE dual-port
- 3 OM nodes: 32-core, 256 GB RAM, 2× 3.84 TB NVMe (Ratis + RocksDB), 25 GbE
- 3 SCM nodes: 16-core, 64 GB RAM, 2× 1.92 TB NVMe, 25 GbE
- 6 S3G nodes: 16-core, 64 GB RAM, 25 GbE (VMs acceptable)
- 2 HAProxy: 8-core, 32 GB RAM, 25 GbE
- 4 load generator nodes: 32-core, `warp` + custom Iceberg-pattern script

Raw NVMe ~370 TB, ~125 TB usable at RS-3-2 EC.

**Timeline from PO signed:**
| Phase | Duration |
|---|---|
| Hardware procurement + delivery | 6–10 weeks (NVMe lead time is dominant variable) |
| Rack/cable/OS/network | 1 week |
| Ozone 2.1 cluster bring-up + OM/SCM HA validation | 1 week |
| Polaris + Postgres + Trino + Spark client config | 1 week |
| Dataset generation + load test + analysis | ~2 weeks |
| Go/no-go writeup | 3 days |
| **Total: 11–16 weeks** | |

Requires explicit ack from lakehouse-tl that PoC runs with Polaris + Postgres (confirmed above — not filesystem catalog).

### Task 12 — InfoSec posture statement

> "We propose deploying an on-premise Apache Ozone 2.1 object storage cluster as the lakehouse backing store. Client authentication to the Ozone S3 Gateway will use a small number of long-lived AWS-style access-key/secret-key pairs (one per service: spark-prod, trino-prod, polaris-svc, ops-admin). Credentials will be stored in HashiCorp Vault with 90-day rotation policy. The S3 Gateway sits on an internal-only VLAN reachable only from the lakehouse compute subnets; no internet egress, no DMZ exposure. Authorization is enforced at the Ozone Manager layer via Apache Ranger policies keyed to Kerberos principals (one-to-one with the access-key pairs). All API calls are audited via Ranger audit logs shipped to the SIEM. We will not vend per-table or per-user temporary credentials in v1; that capability (Ozone STS, HDDS-13323) is on the Ozone roadmap for 2.2 and we will revisit when GA. Compensating controls: network isolation, short rotation interval, per-service key separation, full audit trail."

Fallback if InfoSec rejects: Iceberg REST catalog with presigned-URL vending — Polaris issues short-lived presigned URLs per table operation; underlying S3G key remains static. This is a catalog change, not a storage change. Lakehouse-tl implements it.

### Task 13 — Ceph plan B readiness

**Minimum Ceph PoC cluster** (same footprint as Ozone PoC, +dual NIC per OSD node):
- 6 OSD nodes: 32-core, 256 GB RAM, 8× 7.68 TB NVMe, 25 GbE × 2 (public + cluster network)
- 3 MON+MGR nodes, 6 RGW nodes, 2 HAProxy — same spec as Ozone equivalents
- Dual-pool: all-NVMe 3x replication for `/metadata/*` prefix; EC 4:2 for `/data/*` Parquet

Stand-up delta vs. Ozone: +2 weeks (CRUSH map, dual-pool placement targets, BlueStore tuning, mclock, IOMMU disable, RocksDB rebuild check).

**Sequential, not parallel.** PoC sequence: Ozone first (11–16 weeks). If passes → Ceph never runs. If fails → Ceph PoC adds 13–18 weeks. Worst-case total: ~30 weeks. Parallel only if company commits separate Ceph specialist headcount and second hardware tranche (not requesting this now).

### Storage TL sign-off conditions (all must hold)

1. InfoSec written approval of the posture above within 2 weeks of presentation.
2. PoC passes: p95 ≤ 60 ms (target) / ≤ 80 ms (formal pass), no LIST failures, no OM Raft churn, 30-min soak minimum.
3. Lakehouse-tl confirms Polaris + Postgres catalog (confirmed in this response — not filesystem catalog).
4. Hardware procurement delivers within 16 weeks (slip >20 weeks triggers re-scope).

Named residual risk: my team has no personal Ozone 2.x production experience. HDFS experience is transferable; DiDi case study is the primary external reference. The PoC is also the team's training ground. Accepted because Ceph's expertise tax is worse for our current team shape.

**Storage TL endorses the joint recommendation, conditional on the 4 sign-off conditions above.**

---

## Iteration 3 Resolution Table

| # | Open Item | Status |
|---|---|---|
| 9 | Polaris conditional PUT path | **CLOSED** — Polaris uses Postgres CAS; S3 conditional PUT not in commit path; independently verified by both TLs |
| 10 | S3G p95 50 ms feasibility | **CLOSED** — Adjusted to p95 ≤ 80 ms (formal), ≤ 60 ms (target); storage-tl provides engineering path to 60 ms with tuning levers specified |
| 11 | PoC timeline | **CLOSED** — 11–16 weeks from PO to decision; 20-rack-unit hardware spec; critical path is NVMe lead time |
| 12 | InfoSec posture and sign-off | **CLOSED** — Posture statement written by both TLs; meeting set 2026-05-26; deadline 2026-06-12; Polaris vended-credentials fallback ready |
| 13 | ML team credential expectations | **CLOSED** — Meeting set 2026-05-28; assumption documented (separate S3 keypairs per ML service, Ozone multi-tenancy S3 pattern, Ranger + Kerberos online before Q2 2027) |

**All 13 issues across 3 iterations are now closed or execution-tracked. The debate is concluded.**

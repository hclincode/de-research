# Storage Service Debate — Iteration 2: Cross-Examination

**Date:** 2026-05-21  
**Status:** Both tech leads engaged all 8 contested points. Significant convergence. Two new critical findings surfaced.

---

## Response A — Lakehouse Data Team Tech Lead

### Answers to storage-tl's open questions

1. **Concurrency profile (year 1):** ~400 peak Spark executors (nightly batch), ~150 sustained / burst-to-300 Trino workers (business hours).
2. **Object size distribution:** 60% Parquet 128–256 MB; 25% Iceberg manifests/lists/snapshots <100 KB; 10% footers/dictionaries 8–64 KB; 5% raw landing 1 MB–1 GB.
3. **Table count / partition fanout:** Year 1: ~800 Iceberg tables, ~50 hot. Year 3: ~3,500 tables. Avg ~120 partitions/table, p99 ~5,000 (event-stream tables partitioned by hour+tenant).
4. **Read latency SLA:** Trino interactive p95 <3 s, p99 <10 s → translates to storage p95 GetObject <50 ms for <1 MB, <200 ms for 128 MB ranged Parquet reads.
5. **Write durability:** Zero data-loss window. Iceberg snapshot either visible or doesn't exist.
6. **Multi-tenancy:** **Confirmed — ML training team joins Q2 2027 (~12 months out).** Ranger + Kerberos required.
7. **Catalog / credential model:** Apache Polaris REST catalog. Static S3 creds acceptable at S3G if Polaris-level RBAC enforced; per-table credential vending is "nice to have" not hard requirement today.
8. **Cross-DC DR:** RPO 24h acceptable. Regulated/golden tables (~5% footprint): RPO 4h via async DistCp to secondary Ozone cluster. Sub-1h RPO **not required**.

---

### The 8 contested points

**1. STS gap — partial concession**  
No formal security review yet — only a hallway conversation. Will get written sign-off before go-live. If security demands STS-style vending: fallback is Polaris-level JWT brokering per-table while S3G credentials remain static cluster-wide. Polaris handles this today. **Ceph is not the solution — it also lacks STS.**

**2. S3G small-LIST latency — hard gate accepted**  
Pass/fail: at 80% peak concurrency (320 Spark + 240 Trino workers), synthetic Iceberg manifest pattern (8–80 KB random GETs against bucket with 50M objects), p95 GetObject <50 ms, zero LIST errors over 4-hour soak. Miss <20%: tune and retry once. Miss >20%: see point 7.

**3. S3G scaling ask**  
Opening ask: **6 S3G instances behind Nginx L7 LB**, handling ~2,500 req/s metadata + ~800 req/s ranged reads. Co-located with Trino coordinators on dedicated S3G subnet. Load test will reset this number.

**4. Cross-DC DR — RPO 24h confirmed**  
Confirmed with data platform leadership. **Ceph DR flip condition not triggered.**

**5. Multi-tenancy — confirmed, 12 months out**  
ML training joins Q2 2027. Ranger + Kerberos is now a must-do. Lakehouse TL will sponsor it, security team co-owns KDC integration. This is **not a Ceph flip condition**.

**6. Atomic commit semantics — KEY FINDING: Polaris REST catalog uses Postgres CAS, not S3 conditional PUT**  
- HDDS-13117 / HDDS-14968: Ozone S3G conditional PUT (`If-None-Match: *`) is **not GA in 2.1.0** — fix-version 2.2.0.
- **It does not matter for our deployment** because Apache Polaris REST catalog uses Postgres CAS for the catalog pointer swap, not object-storage conditional PUTs. S3FileIO writes new metadata via ordinary PUT; Polaris swaps atomically at the database layer. Spark and Trino races resolve at the catalog layer.
- **Caveat:** If we ever fall back to HadoopCatalog or a REST catalog implementation that uses S3 conditional PUT, this becomes critical. We are not doing that.
- Request to storage-tl: confirm no Ozone-side feature in our design also depends on conditional PUT semantics today.

**7. If load test fails — renegotiate first, then Ceph**  
If p95 is 50–80 ms: renegotiate internal SLA (meets Trino end-to-end SLA with margin). If p95 >100 ms or consistency failures: **Ceph becomes the alternative**, not a dead end.

**8. Ceph flip conditions — broadly agree, one lakehouse addition**  
Storage-tl's (a) cross-DC RPO < 1h: **not triggered**. (b) Multi-protocol from day one: **not triggered**. (c) Hire two senior Ceph engineers: storage-tl's call.  
**Adding (d):** Ozone 2.1 ships a regression that breaks p95 in load test and fix slips past Q4 2026 → Ceph on standby.

**On Ceph EC + replicated metadata pool — partial concession**  
Storage-tl is correct that Iceberg manifests in a 3x-replicated pool avoid EC amplification. That neutralizes the EC-kills-metadata-latency criticism. **Remaining concern:** bucket index bottleneck for high-cardinality LISTs at billions of objects even on replicated pool; RGW dynamic sharding has its own tuning regime. The real concern is operational surface area, which storage-tl already owns.

---

### Bottom line — iteration 2

**Recommendation unchanged: Apache Ozone 2.1.0 + Apache Polaris REST catalog + S3G.** Static creds + Polaris RBAC. RPO 24h. Multi-tenancy Q2 2027 milestone.

**Points resolved:** Load test hard gate agreed. Ranger+Kerberos roadmap confirmed. No sub-1h RPO. Ceph is credible runner-up. All-NVMe metadata path. Polaris uses DB CAS not S3 conditional PUT.

**Points still open:**  
- Concrete S3G topology (storage-tl owns)
- Load test plan document (joint)
- Security sign-off on static creds (lakehouse owns)
- ML team catalog/credential expectations (cross-team meeting needed)

---

## Response B — Storage Service Team Tech Lead

**1. STS gap — ownership corrected**  
I overstated it. "Security team may not accept" was hedging — I don't have a written position. Storage team owns authn/authz design; lakehouse owns catalog credential model; InfoSec signs off jointly. **Action: get written InfoSec sign-off on "static cluster-wide S3 creds + Kerberos+Ranger ACLs" before hardware procurement.** Ceph also lacks STS — this is a constraint both live with regardless of object store. Resolution lives at the catalog layer (Polaris/Nessie).

**2. S3G LIST load test — concrete criteria defined**
- **Hardware:** production-spec — all-NVMe datanodes, NVMe-backed Ratis, 25 GbE, 3-node OM HA, 3-node SCM HA, 4 S3G instances behind HAProxy.
- **Bucket layout:** FILE_SYSTEM_OPTIMIZED (FSO).
- **Workload:** 10M objects in one bucket, sizes {4 KB, 16 KB, 64 KB} + 100K Parquet @ 128 MB.
- **Concurrency:** 200 parallel clients, LIST + GET pattern (Trino manifest reads).
- **Pass thresholds:** p95 GetObject <1 MB ≤ 50 ms; p95 ListObjectsV2 ≤ 100 ms; zero LIST failures under 30-min soak; no OM Raft churn.
- **Reject Ozone if:** p95 GetObject >100 ms, OR any LIST failures, OR OM RocksDB compaction stalls >1 s → recommend Ceph dual-pool instead.

**3. S3G topology for 200 concurrent Trino workers**  
Concrete spec: **6 × S3G instances** (N+1 spare for maintenance), each on 16-core/64 GB RAM/25 GbE mid-tier server. **HAProxy** active/passive, sticky-by-bucket affinity. **OM sizing:** 3-node Raft, 32 cores, 256 GB RAM each, NVMe for Ratis + RocksDB. **Listener OMs** (2.1.0+) for read-only follower reads, directly supporting manifest-read pattern.

**4. Cross-DC DR — Ceph claim corrected**  
Ceph RGW multisite is **async by default — non-zero RPO**. Sync behavior requires stretch cluster (cross-DC latency penalty). Realistic RPO with proper bandwidth provisioning: **minutes-to-tens-of-minutes lag** (not zero). Ozone has DistCp only (hours RPO). So Ceph wins at RPO < 1h but the margin is smaller than I implied. RPO = 0 requires stretch Ceph or app-layer dual-write, neither of which is our requirement.  
Sources: Ceph Multisite Part 1 (2025), HTTP Sync Status Headers (2025).

**5. Multi-tenancy — what changes when ML joins**
- New Ranger policies per volume/bucket/prefix for `ml-team` group
- New Kerberos principals for ML compute services (Ray, PyTorch jobs), keytab distribution
- Volume + bucket quota enforcement (critical — prevents ML from starving lakehouse)
- S3G: separate access-key/secret-key per tenant → Ozone Multi-tenancy S3 with Ranger + Kerberos already deployed
- **Blast radius of Ranger misconfig:** permissive `*` allow on ml-team could expose lakehouse tables. Mitigation: policy changes via PR-reviewed Terraform/Ansible, never direct UI edits.
- **Honest gap:** I have not operated Ozone Multi-tenancy personally. Budget 4–6 months from kickoff.

**6. Atomic commit via S3G — HDDS-13117 is a real gap**  
Ozone S3G does not yet implement `If-None-Match: *` conditional PUT (tracked HDDS-13117, in progress HDDS-14968, not GA in 2.1.0). What Ozone provides: native atomic rename at OFS layer, strong read-after-write on S3G. **For catalog-based Iceberg (Polaris, Nessie, Hive)** this is fine. **For filesystem catalog or REST catalog implementations that use S3 conditional PUT directly** — Ozone 2.1 is not safe.  
**Must validate in PoC:** Does the Polaris REST catalog implementation lakehouse-tl plans to use actually hit S3 conditional PUT, or does it use Postgres CAS exclusively? The Ozone+Polaris reference architecture (April 2026) appears to work, but we need to confirm the commit path before hardware procurement.

**7. Independent Ozone 2.x benchmarks — concede the gap**  
Searched again: no independent Ozone 2.x benchmark against Trino+Iceberg with hard numbers exists. The PoC IS the benchmark. Will publish results internally.

**8. Ceph dual-pool — viable plan B with real operational cost**  
Architecture: all-NVMe 3x-replication pool for `/metadata/*` (no EC amplification); all-NVMe EC pool (4:2 or 2+2) for `/data/*` Parquet; Tentacle FastEC for partial Parquet footer reads. **Resolves the EC-manifest concern** lakehouse-tl raised.  
**Operational cost I'd own:** Two CRUSH rules + pool policies; routing logic (two buckets or RGW placement targets per prefix); cross-pool capacity planning. Roughly **+0.5 headcount ongoing Ceph attention** vs. single-pool Ozone.

**9. NEW CONCERN — S3G path hop count vs. p95 50 ms SLA**  
The S3G read path is Client → HAProxy → S3G → OM (metadata RPC) → Datanode (data RPC) — **4 network hops minimum, often 5**. At 0.1 ms intra-DC per hop baseline = ~0.5 ms healthy. Under contention (OM RocksDB read amplification, S3G GC pause, HAProxy queue depth) tail latency expands. The DiDi 17 ms P90 metric is **OM GetMetaLatency**, not full-path S3G GET latency. **No evidence Ozone S3G achieves p95 <50 ms for sub-1 MB GETs at 200-worker concurrency on production hardware exists.** The PoC proves or disproves this. I will not commit to the 50 ms SLA on paper pre-PoC.

---

### Bottom line — iteration 2

Five things materially change our plan:
1. STS gap jointly owned — InfoSec engagement this week. Ceph doesn't help.
2. **HDDS-13117 conditional PUT must be PoC-validated** — only risky if Polaris uses S3 conditional PUT (lakehouse-tl says it uses Postgres CAS; need to confirm in PoC).
3. Ceph multisite async — my iteration-1 framing was sloppy, corrected.
4. Load test pass criteria above — hard gate.
5. S3G topology: **6 instances + HAProxy + 3-node OM with Listener OMs, all-NVMe metadata.** Concrete spec ready for procurement.

**Still recommending Ozone.** The conditional-PUT gap is the only new risk that could flip me, and only if Polaris uses S3 conditional PUT for commits (which lakehouse-tl says it doesn't — must confirm in PoC).

---

## Iteration 2 Resolution Table

| # | Contested Point | Status |
|---|---|---|
| 1 | STS gap | **RESOLVED** — jointly owned, InfoSec engagement pending, Ceph doesn't solve it, Polaris brokers per-table at catalog layer |
| 2 | S3G LIST latency on Ozone 2.x | **RESOLVED** — hard gate, full acceptance criteria defined |
| 3 | S3G horizontal scaling plan | **RESOLVED** — 6 × S3G + HAProxy, 3-node OM Listener OMs, all-NVMe, concrete spec for procurement |
| 4 | Cross-DC DR | **RESOLVED** — RPO 24h accepted, no sub-1h requirement, Ceph flip condition not triggered; Ceph multisite claim corrected (minutes async, not zero) |
| 5 | Multi-tenancy timeline | **RESOLVED** — Q2 2027 confirmed, Ranger+Kerberos is a must-do milestone, 4–6 months to deploy |
| 6 | Atomic commit semantics | **PARTIALLY RESOLVED** — HDDS-13117 confirmed as real gap in 2.1; Polaris reportedly uses Postgres CAS not S3 conditional PUT; **must be confirmed in PoC** |
| 7 | Independent Ozone 2.x benchmarks | **RESOLVED** — gap accepted, PoC IS the benchmark; internal publication planned |
| 8 | Ceph trigger conditions | **RESOLVED** — all three storage-tl conditions evaluated; none triggered; lakehouse-tl adds (d) unresolved regression past Q4 2026 |

## New Open Items for Iteration 3

| # | Open Item | Owner |
|---|---|---|
| 9 | **Polaris conditional PUT path** — does production Polaris use S3 `If-None-Match` or Postgres CAS for catalog commit? Evidence required before PoC. | lakehouse-tl (research) |
| 10 | **S3G p95 50 ms at 200-worker concurrency** — no evidence exists pre-PoC; need to confirm whether this is achievable or SLA needs renegotiation | joint (PoC) |
| 11 | **PoC timeline and hardware procurement plan** | storage-tl (hardware) + lakehouse-tl (workload dataset) |
| 12 | **InfoSec sign-off on static creds** | jointly — both TLs engage InfoSec this week |
| 13 | **ML team catalog/credential expectations for Q2 2027** | lakehouse-tl (cross-team meeting) |

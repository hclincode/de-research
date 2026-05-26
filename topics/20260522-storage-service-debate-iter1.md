# Storage Service Debate — Iteration 1: Opening Positions

**Date:** 2026-05-21  
**Participants:**
- **Lakehouse Data Team Tech Lead** (lakehouse-tl) — owns Spark + Trino, Iceberg, data ingestion and OLAP query environment
- **Storage Service Team Tech Lead** (storage-tl) — owns bare-metal storage infrastructure, serves lakehouse as primary customer

**Summary:** Both tech leads independently arrive at **Apache Ozone with S3G** as their recommendation, but with different primary rationales, different risk tolerances, and several unresolved contested points that must be resolved in subsequent iterations.

---

## Position A — Lakehouse Data Team Tech Lead

**Recommendation:** Apache Ozone, accessed exclusively via its S3 Gateway (S3G), for both Spark and Trino.

---

### 1. The recommendation and the primary reason

I advocate for **Apache Ozone with S3G as the sole access path** for both engines. Spark uses the S3A client; Trino uses `fs.native-s3.enabled=true`. Both point at the same Ozone S3G endpoint, both read and write the same Iceberg tables, and neither engine touches a Hadoop filesystem JAR for the storage path.

My primary reason is not "Ozone is the fastest" — I will not claim that, the evidence does not support it. My reason is **architectural alignment with Trino's stated direction combined with metadata-scale headroom that HDFS cannot give us**.

Trino 470 (February 2025) deprecated the `hive.s3.*`, `hive.azure.*`, `hive.gcs.*` namespaces and is unwinding `fs.hadoop.enabled` over "weeks or months" for the Hive S3 path, with no removal date for the HDFS shim but an explicit "this is the compatibility layer" framing from the Trino team (`resources/trino/trino-legacy-filesystem-deprecation.summary.md`). Every release where we run on `fs.hadoop.enabled=true` we are accumulating debt we will eventually pay in an emergency. Native S3 is the only forward-compatible interface for Trino on-premises. That eliminates HDFS for greenfield and forces us to either Ozone S3G or Ceph RadosGW.

Between those two, Ozone wins on **single-cluster simplicity and metadata-scale fit for a lakehouse**. Iceberg generates billions of small objects (manifests, manifest lists, snapshots, position deletes). Ozone's SCM/OM split is purpose-built for that object cardinality, while Ceph's MON/RGW index-pool architecture is general-purpose and demands deep tuning to handle it. The Trino + Ozone + Apache Polaris reference architecture (April 2026) is a working production pattern; I have no equivalent reference for a Spark+Trino+Iceberg+Ceph stack at our scale.

### 2. Evidence

- **Trino's direction is settled.** `fs.native-s3.enabled=true` is the forward path. HDFS shim is compatibility-only. (`resources/trino/trino-legacy-filesystem-deprecation.summary.md`, official, HIGH confidence.) This single fact disqualifies HDFS for any new deployment with Trino in it.
- **Ozone OFS is a non-starter for open-source Trino.** Practitioners report "No FileSystem for scheme 'ofs'" errors; only Starburst Enterprise supports `ofs://` natively. The recommended path is S3G behind a load balancer. (`resources/ozone/trino-ozone-ofs-configuration-discussion.summary.md`, official.) Good — that's the path Trino is moving toward anyway. We commit to S3G and the OFS gap becomes irrelevant.
- **The Protobuf JAR conflict reinforces "S3G only."** Direct OFS JAR loading in Trino produces `NoClassDefFoundError: com/google/protobuf/ServiceException` and 500+ failover retries. Apache Ozone PR #7729 implements shading, but most practitioners gave up and shifted to S3G. (`resources/ozone/trino-ozone-iceberg-protobuf-issue.summary.md`, official.) We should not be the team chasing JAR shading in production.
- **Synthesis report agrees.** The verdict in `topics/spark-trino-lakehouse-storage-service.report.md` is Ozone for single-cluster Spark+Trino, Ceph only when multi-cluster shared storage or storage-cost reduction is the *primary* driver. Neither is our situation.

### 3. Strongest concerns about competing options

**Ceph (RadosGW)**
- The only Trino-on-Ceph performance evidence available is an IBM/Ceph S3 Select TPC-DS benchmark — vendor-adjacent (`resources/ceph/ceph-trino-tpcds-s3select-benchmark-2025.summary.md`). The headline 2.5x speedup is *with S3 Select enabled vs disabled on Ceph*, not Ceph vs Ozone or Ceph vs HDFS. It does not tell us what baseline Ceph latency looks like for an unpushed-down Iceberg manifest read.
- The 1 TiB/s Ceph benchmark is real and impressive (`resources/ceph/ceph-journey-to-1tibps-2024.summary.md`, press, 2024) — but read the tuning list: disable IOMMU, recompile Ceph with proper RocksDB flags, 2 OSDs per NVMe, 14–16 cores per OSD. That is the operating model for a Ceph performance cluster.
- Ceph's read path for small files (Iceberg manifests, Parquet footers) compounds latency at Trino concurrency. No current independent benchmark proves the 2x penalty is gone on NVMe Ceph in 2025/2026.
- Ceph is the right answer if we run **multiple** lakehouses sharing one pool. We don't — we run one.

**JuiceFS / SeaweedFS**
- JuiceFS is metadata-on-external-database. Adding an HA Redis/TiKV/Postgres to be the canonical metadata store introduces a failure mode we don't need. Hard pass for primary lakehouse storage.
- SeaweedFS has a passing Trino Iceberg CI test as of Feb 2026 (`resources/seaweedfs/trino-iceberg-integration-test-2026.summary.md`) — a basic smoke test, not production validation at petabyte scale with hundreds of concurrent Trino workers. Watch item for 2027.

**HDFS**
- Disqualified for new deployment by Trino's filesystem deprecation.

### 4. Open questions for the storage team

1. **Ozone S3G small-LIST performance on our hardware.** Can the storage team commit to Ozone >=2.1 with Ratis metadata on SSD and `FILE_SYSTEM_OPTIMIZED` bucket layout from day one?
2. **S3G horizontal scaling.** How many S3G instances behind what load balancer for projected Trino concurrency?
3. **Atomic rename / commit semantics through S3G.** Does Ozone S3G support strong read-after-write and conditional PUTs equivalent to AWS S3 in 2026?
4. **Backup, DR, and snapshot-expiry deletes.** What is Ozone's deletion throughput, and what is the DR posture?
5. **Operational maturity beyond Cloudera CDP.** Do we have non-CDP production experience, or are we among the early non-Cloudera large Ozone shops?
6. **STS gap acceptance.** Is the security team aligned to static credentials at the network perimeter?

### 5. What would change my mind

- An independent 2025/2026 benchmark of Ceph RGW Iceberg manifest read latency under Trino concurrency within 20% of Ozone S3G on comparable NVMe hardware.
- Ozone S3G cannot sustain projected Trino concurrency (200 concurrent worker connections, p95 GetObject <50 ms for objects <1 MB) in a proof-of-concept.
- Operationally: Ozone OM/SCM HA is a real burden the storage team cannot staff for, and Ceph MON/MGR they already know cold.

### Bottom line

Ozone with S3G, on NVMe-backed Ratis metadata, with a clear scale-out plan for the S3G tier.

---

## Position B — Storage Service Team Tech Lead

**Recommendation:** Apache Ozone 2.1.0 as the lakehouse storage service, with Ceph RGW as the runner-up.

---

### 1. Why Ozone — primary reasons from a storage-ops standpoint

**1.1 Bounded operational surface.** Ozone has three core daemons: Ozone Manager (OM), Storage Container Manager (SCM), and Datanode. OM HA is a 3-node Raft quorum; SCM HA is the same. Each datanode runs RocksDB for container metadata. Compare to Ceph: OSDs (BlueStore tuning), MONs, MGRs, RGWs, CRUSH map, placement groups, mclock scheduler, EC profiles per-pool, BlueStore cache sizing, RocksDB compaction inside the OSD, bucket index sharding — all interacting non-linearly (`resources/ceph/ceph-hardware-recommendations-official.summary.md`).

**1.2 Production-validated at scale.** DiDi runs **>500 PB, ~5B files per cluster, tens of billions across clusters**, with P90 GetMetaLatency of **17ms** after the OM lock work (`resources/ozone/ozone-didi-production-deployment.summary.md`). This is a published ASF case study with hard numbers. The architecture survives a metadata-heavy lakehouse workload at a scale we are unlikely to exceed in the next 5 years.

**1.3 Hardware efficiency is competitive.** DiDi uses RS-6-3 erasure coding (1.5x overhead vs 3x replication) and saved "hundreds of petabytes." Unlike Ceph, the small-file metadata layer is a separate concern (OM RocksDB) — so we don't pay EC read amplification on Iceberg manifests the way Ceph does on a poorly-segregated RGW deployment.

**1.4 Failure recovery has a simpler blast radius.** When a datanode fails, SCM re-replicates closed containers from surviving replicas. Recovery is per-container. No equivalent of Ceph's "slow OSD cascade" where one disk drags down PG peering across the whole cluster (`resources/ceph/ceph-slow-osd-failure-modes.summary.md`).

**1.5 License and governance.** Apache 2.0 under the ASF. No AGPL trap (MinIO), no single-vendor commercial entity (JuiceData), no LGPL legal review (Ceph).

### 2. Why not Ceph (honest runner-up)

- **Expertise tax.** Clyso's 1 TiB/s benchmark required: disabling kernel IOMMU, recompiling RocksDB with correct flags (Ubuntu/Debian packages ship without them), OSD-per-NVMe ratio tuning, 14–16 CPU threads per OSD. These are not knobs a typical SRE team turns.
- **RGW concurrency ceiling.** 32 concurrent connections per RGW instance; 100+ Spark executors saturate a single RGW. Horizontal RGW scaling adds another tier to operate.
- **EC read amplification on Iceberg metadata.** A 10 KB Iceberg manifest in a 4:2 EC pool reads 4 OSD shards. FastEC in Tentacle helps but is not independently benchmarked for RGW object workloads yet.
- **Recovery storms.** 15 TB NVMe OSD failure triggers cluster-wide recovery I/O competing with client I/O. Separate cluster network is non-negotiable — doubles NIC and switch budget.

Ceph is excellent. It is also a full-time job.

### 3. Why JuiceFS and SeaweedFS are eliminated

**JuiceFS:** Architecturally a filesystem layered on top of an object store plus an external metadata engine. We would need to operate either Ceph S3 or Ozone underneath JuiceFS, plus a Redis/PostgreSQL/TiKV cluster as the metadata plane — two HA systems instead of one. TiKV is **44× slower than Redis** under concurrent small-file workloads; Redis loses acknowledged writes on Sentinel failover. Hard pass.

**SeaweedFS:** Full-cluster backup **requires write pause**. Filer metadata consistency under network partition is **closed "not planned"** (Issue #6186). 2–3 releases per week in May 2026, with v4.23 shipping a critical EC regression fixed in v4.24. Data corruption bug #8151 on encrypted volumes. No verified petabyte-scale lakehouse deployment.

### 4. Operational concerns about Ozone (own pick)

- **No STS endpoint** (planned for 2.2, not released). Trino must use static S3 credentials against S3G. Mitigation: Kerberos + Ranger ACLs + internal network perimeter.
- **No built-in cross-datacenter DR.** DiDi uses DistCp for batch migration, not real-time DR. If the company requires geo-DR with RPO < 24h, we need to build it ourselves or accept fail-forward.
- **Small-object LIST performance on Ozone 2.x is not independently benchmarked.** The 1.4.x LIST failures required NVMe metadata volumes. 2.1.0 includes the OM lock refactor but no third-party benchmark confirms this for Iceberg manifest-heavy workloads. Pre-production load test required.
- **Disk balancer is in 2.2, not yet released.** Manual SCM rebalance until then.

### 5. Open questions for the Lakehouse team

1. **Concurrency profile.** Peak concurrent Spark executors and Trino workers? (Drives S3G instance count.)
2. **Object size distribution.** Mix of Iceberg manifests <100 KB, Parquet data files 64–256 MB, raw landing files?
3. **Table count and partition fanout.** How many Iceberg tables in year 1, year 3? Partitions per table?
4. **Read latency SLA.** p99 query latency target — <100ms interactive or batch-tolerant?
5. **Write durability requirement.** Acceptable data-loss window for in-flight Spark writes?
6. **Multi-tenancy timeline.** Only lakehouse, or ML training and other services in year 2?
7. **Catalog choice and credential model.** If using Polaris or Nessie, do you need STS-style credential vending?
8. **Cross-DC DR requirement.** RPO=0? RPO=24h? RPO=never?

### 6. Hard constraints (non-negotiable)

- Apache 2.0 or equivalent permissive license only
- No external metadata engine as a single point of architectural complexity (rules out JuiceFS CE)
- All-NVMe for hot metadata path (OM RocksDB / Ceph bucket index pool)
- Separate public and cluster networks if Ceph
- Pre-production Iceberg workload load test before go-live

### 7. What would change my mind to Ceph

- Cross-DC active-active replication required with RPO < 1h
- Commitment to multi-protocol storage (object + block + filesystem) from day one
- Can hire two senior Ceph engineers and budget IBM Storage Ceph support

---

## Contested Points Identified for Iteration 2

Both tech leads agree on Ozone S3G as the primary recommendation. The following specific points remain unresolved and require deeper analysis:

| # | Contested Point | Lakehouse TL Position | Storage TL Position |
|---|---|---|---|
| 1 | STS gap (no credential vending) | "Acceptable at network perimeter" | "Security team may not accept; need alignment" |
| 2 | S3G small-LIST latency on Ozone 2.x | "Need SSD-backed Ratis + FSO layout as prerequisite" | "No independent benchmark; pre-production test required" |
| 3 | S3G horizontal scaling plan | "Need a spec before committing" | "Need Lakehouse concurrency numbers first" |
| 4 | Cross-DC DR | "Not mentioned as a requirement" | "If RPO < 1h required, flips to Ceph" |
| 5 | Multi-tenancy timeline | "Not addressed" | "Ranger + Kerberos is a 6-month project; need timeline" |
| 6 | Atomic commit semantics through S3G | "Need confirmation of strong read-after-write" | "Not addressed" |
| 7 | Independent Ozone 2.x lakehouse benchmarks | "Gap acknowledged; would change mind if Ceph fills it" | "Load test required before go-live" |
| 8 | Ceph trigger conditions | "Would reconsider if Ozone PoC fails SLA" | "Only flip to Ceph if cross-DC DR or multi-protocol required" |

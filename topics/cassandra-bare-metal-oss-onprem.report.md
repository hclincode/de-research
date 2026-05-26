---
title: "Cassandra 5.x Bare-Metal — OSS-Only On-Premise (Container Required, Kubernetes Prohibited)"
date: 2026-05-26
status: complete
components:
  [cassandra, podman, ansible, reaper, medusa, seaweedfs, consul, prometheus, grafana, atlantis, opentofu]
constraints:
  deployment: on-premise
  os: ubuntu-22.04
  container: required
  kubernetes: prohibited
  commercial-tools: prohibited
prior-topics:
  - cassandra-bare-metal-no-k8s-container
  - cassandra-bare-metal-container-required
  - cassandra-bare-metal-nomad-stack
---

## Overview

This report refines the prior `cassandra-bare-metal-no-k8s-container` analysis under two new hard constraints: **OSS-only** (Apache 2.0 / MIT / LGPL-2.1; BSL and AGPL require explicit callout) and **on-premise data center** (physical bare-metal, no cloud APIs, private network). Under these constraints, AxonOps (commercial) and MinIO (AGPL, archived February 2026) are both eliminated, which collapses Architecture A.

**Verdict: Architecture B (Podman Quadlets + Ansible + Reaper + Medusa + SeaweedFS + Consul + Prometheus) is the only viable fully-qualified OSS path. Nomad remains a valid scheduler above ~100 nodes but is BSL 1.1, not strictly OSS — it must be evaluated separately if the license constraint is hard.**

---

## Evidence Quality

| Source | File | Tier | Accessible |
|---|---|---|---|
| instaclustr/cassandra-exporter (GitHub) | [instaclustr-cassandra-exporter-prometheus.summary.md](../resources/cassandra/instaclustr-cassandra-exporter-prometheus.summary.md) | official | Yes |
| MCAC Cassandra 5 deprecation | [mcac-cassandra5-deprecation-status.summary.md](../resources/cassandra/mcac-cassandra5-deprecation-status.summary.md) | press | Yes |
| Criteo exporter maintenance status | [criteo-cassandra-exporter-maintenance-status.summary.md](../resources/cassandra/criteo-cassandra-exporter-maintenance-status.summary.md) | press | Yes |
| Consul HA reference architecture | [consul-ha-reference-architecture.summary.md](../resources/consul/consul-ha-reference-architecture.summary.md) | vendor | Yes |
| etcd vs Consul KV lock comparison | [etcd-vs-consul-kv-lock-comparison.summary.md](../resources/consul/etcd-vs-consul-kv-lock-comparison.summary.md) | press | Yes |
| Atlantis GitOps on-premise | [atlantis-gitops-on-premise-terraform.summary.md](../resources/consul/atlantis-gitops-on-premise-terraform.summary.md) | official | Yes |
| Terraform state backend on-premise options | [terraform-state-backend-on-premise-options.summary.md](../resources/consul/terraform-state-backend-on-premise-options.summary.md) | press | Yes |
| SeaweedFS GitHub repo | [github-repo-seaweedfs.summary.md](../resources/seaweedfs/github-repo-seaweedfs.summary.md) | official | Yes |
| SeaweedFS production setup wiki | [production-setup-wiki.summary.md](../resources/seaweedfs/production-setup-wiki.summary.md) | official | Yes |
| Kubeflow SeaweedFS adoption 2025 | [kubeflow-seaweedfs-adoption-2025.summary.md](../resources/seaweedfs/kubeflow-seaweedfs-adoption-2025.summary.md) | press | Yes |
| MinIO repo archived 2026 | [minio-repo-archived-2026.summary.md](../resources/minio/minio-repo-archived-2026.summary.md) | press | Yes |
| MinIO CE dead 2026 alternatives | [minio-ce-dead-2026.summary.md](../resources/minio/minio-ce-dead-2026-alternatives.summary.md) | press | Yes |
| MinIO AGPL license analysis | [minio-agpl-license-analysis.summary.md](../resources/minio/minio-agpl-license-analysis.summary.md) | press | Yes |

**Gaps and confidence limits:**
- CEP-53 (Cassandra Sidecar rolling restart API) passed community vote October 2025 but is not yet shipped in any release; no independent benchmark of the feature exists. Timeline confidence LOW.
- Consul HA architecture guidance comes from a HashiCorp/vendor source. Independent corroboration via the etcd comparison paper raises confidence to MEDIUM.
- NIC bonding and BIOS settings are derived from practitioner synthesis; no single peer-reviewed source exists for the full set of recommendations. Confidence MEDIUM.
- SeaweedFS production readiness is supported by Kubeflow adoption (independent press) and GitHub issue analysis. Confidence MEDIUM — not yet in a major hyperscaler default stack.

---

## Why Architecture A (Nomad + AxonOps) Is Eliminated Under OSS-Only

Architecture A from the prior report depended on two non-OSS tools:

| Tool | Prior status | OSS-only verdict |
|---|---|---|
| AxonOps | Commercial (SaaS/self-hosted, paid) | **Eliminated** — no permissive license available |
| Nomad | BSL 1.1 (HashiCorp) | **Requires license review** — not Apache 2.0/MIT/LGPL |

Without AxonOps, Architecture A's operational advantage over Architecture B disappears: AxonOps provided the rolling restart coordinator, health-gated drain automation, and Cassandra-aware monitoring that Ansible + Prometheus replicate at higher implementation cost. Nomad's scheduler gains (bin-packing, multi-cluster dispatch) are real but irrelevant to the constraint: BSL 1.1 prohibits competitive hosting use cases and requires legal sign-off in many enterprises with strict OSS policies.

**Architecture A is eliminated. Nomad is retained as an optional BSL-licensed addition only when node count exceeds ~100 or multi-cluster co-location is required, with explicit license approval documented.**

---

## The Winning Stack: Podman Quadlets + Ansible + Reaper + SeaweedFS/Medusa + Consul + Prometheus

| Tool | License | Role |
|---|---|---|
| Podman (Quadlets) | Apache 2.0 | Rootless container runtime; systemd-native lifecycle |
| Ansible | GPL-3.0 / Apache 2.0 (modules) | Node provisioning, rolling restart orchestration |
| Apache Cassandra 5.x | Apache 2.0 | Distributed database |
| Reaper | Apache 2.0 | Repair scheduler, REST API for repair state gate |
| Medusa | Apache 2.0 | Backup/restore; S3-compatible or local backend |
| SeaweedFS | Apache 2.0 | On-premise S3-compatible object store (MinIO replacement) |
| Consul | BSL 1.1 | Service discovery, KV distributed lock, health checks |
| Prometheus | Apache 2.0 | Metrics collection |
| instaclustr/cassandra-exporter | Apache 2.0 | JVM agent; Cassandra 4.0+ native API metrics |
| prometheus/jmx_exporter | Apache 2.0 | JMX scraper fallback; actively maintained |
| Grafana | AGPL-3.0 | Dashboards (self-hosted; AGPL requires source publication if redistributed) |
| Alertmanager | Apache 2.0 | Alert routing; rolling restart gate enforcement |
| OpenTofu | Apache 2.0 | Infrastructure-as-code (API-compatible Terraform replacement) |
| Atlantis | Apache 2.0 | On-premise GitOps runner for OpenTofu PRs |

**Note:** Consul (BSL 1.1) and Grafana (AGPL-3.0) are the two tools in this stack that do not meet strict Apache 2.0/MIT/LGPL criteria. If the constraint is absolute, evaluate OpenBao (Apache 2.0, Vault fork) for secrets and Victoria Metrics (Apache 2.0) as a Grafana alternative. For Consul specifically, no drop-in OSS replacement covers all three functions (service discovery + health checks + KV lock + DNS) simultaneously — etcd covers KV lock only.

---

## Production Rolling Restart: The 5-Gate Ansible Playbook

Each node must pass all five gates sequentially before drain proceeds. These are non-negotiable in production; skipping any gate risks data loss on a cluster with RF=3 under concurrent failures.

### Gate definitions

| Gate | Command | Pass condition |
|---|---|---|
| 1. Node UN state | `nodetool status` | Target node shows `UN` |
| 2. Schema agreement | `nodetool describecluster` | Exactly one schema UUID returned |
| 3. No active streaming | `nodetool netstats` | Zero active streams / zero pending |
| 4. Hint queue depth | `nodetool tpstats \| grep HintedHandoff` | Pending = 0 on all nodes |
| 5. No active repair | `nodetool compactionstats \| grep VALIDATION` | Empty output |

### Compaction pressure check (additional gate)

```bash
PENDING=$(nodetool compactionstats | grep "pending tasks" | awk '{print $NF}')
[ "$PENDING" -gt 50 ] && echo "BLOCK: compaction backlog $PENDING, wait before draining"
```

Deploy this as an Ansible `assert` task with `retries: 60` and `delay: 30` (30 minutes max wait). A value above 50 means the node is under I/O pressure; draining while compaction is active causes cascading read latency across the replica set.

### Consul KV distributed lock recipe

Prevents concurrent restarts when multiple Ansible runs or clusters share the same Consul cluster. Acquire before drain, release after node returns UN.

```bash
# Acquire lock (returns 200 + true if acquired, 200 + false if held by another session)
SESSION=$(curl -s -X PUT http://consul:8500/v1/session/create \
  -d '{"TTL":"600s","Behavior":"release"}' | jq -r .ID)

LOCK=$(curl -s -X PUT \
  "http://consul:8500/v1/kv/cassandra/rolling-restart/lock?acquire=$SESSION" \
  -d "$(hostname)")

[ "$LOCK" != "true" ] && echo "BLOCK: another node is restarting" && exit 1

# ... drain, restart, wait for UN ...

# Release lock
curl -s -X PUT \
  "http://consul:8500/v1/kv/cassandra/rolling-restart/lock?release=$SESSION"
```

**CASSANDRA-8741 rule (mandatory):** Use `nodetool drain` only before a restart. Never run `nodetool drain` before `nodetool decommission` — drain disables the commit log listener, causing the decommission to hang indefinitely. Decommission proceeds without drain.

---

## CEP-53 Watch Item

**What it is:** CEP-53 (Cassandra Enhancement Proposal 53) defines a Cassandra Sidecar rolling restart REST API — a native, rack-aware, health-gated rolling restart coordinator built into the Cassandra project itself. It passed community vote in October 2025 and is under active development as of May 2026.

**Why it matters:** CEP-53 is the first native OSS answer to the structural gap identified in the prior reports: there is currently no rack-aware rolling restart coordinator outside Kubernetes operators. If shipped, it will replace the Ansible 5-gate playbook with a single API call, and the Consul KV lock with Cassandra-native cluster state.

**When to expect it:** Active development as of May 2026; no committed release date. Watch the [cassandra-dev mailing list](https://lists.apache.org/list.html?dev@cassandra.apache.org) and the `CEP-53` tag. Likely to appear in a Cassandra 5.x minor release (not a 4.x backport). **Do not block the current deployment on CEP-53 — adopt when GA.**

---

## Monitoring Stack

### MCAC is deprecated

MCAC (Metric Collector for Apache Cassandra) v0.3.6 (March 2025) supports Cassandra through 4.1 only. K8ssandra has formally announced removal in favor of MAAC + Vector. **Do not use MCAC for new deployments targeting Cassandra 5.x.**

### Replacement options

| Exporter | License | Cassandra support | Notes |
|---|---|---|---|
| `instaclustr/cassandra-exporter` | Apache 2.0 | 4.0+ native API | JVM agent; in-process scrape; no JMX required |
| `prometheus/jmx_exporter` | Apache 2.0 | No version restriction | Requires Cassandra JMX rules YAML; actively maintained by Prometheus project |

Recommendation: use `instaclustr/cassandra-exporter` as primary (native API, lower overhead) with `jmx_exporter` as fallback when JMX is already exposed for other tooling.

**Grafana dashboards:** K8ssandra project dashboards (Apache 2.0) or Grafana community dashboard ID 14070. Both work with the instaclustr exporter metric namespace.

### Alertmanager rolling restart gates

All four conditions must be clear before Ansible advances to the next node:

| Gate | Source | Alert expression (block if true) |
|---|---|---|
| Compaction queue empty | `cassandra_compaction_pending_tasks` | `cassandra_compaction_pending_tasks > 0` |
| Schema convergence | `cassandra_schema_versions` gauge | `count(cassandra_schema_versions) > 1` |
| No active repairs | Reaper REST API | `GET /repair_run` returns any RUNNING state |
| Gossip state UN | Ansible poll `nodetool status` | Node not in UN state (no Prometheus equivalent) |

---

## Backup: SeaweedFS On-Premise

### MinIO is archived — do not use

MinIO community edition repository was archived February 2026. Pre-built binaries stopped shipping October 2025. License: AGPLv3. Under the OSS-only constraint, AGPLv3 requires source publication of any derivative service — and the binary is no longer available in any case. **MinIO is a hard no.**

### SeaweedFS as the replacement

SeaweedFS (Apache 2.0) provides an S3-compatible API. Medusa supports it via `storage_provider = s3_compatible` with `host` and `port` pointing at the SeaweedFS filer S3 gateway.

**Minimum HA cluster layout:**

| Component | Minimum nodes | Notes |
|---|---|---|
| Master nodes | 3 | Raft consensus HA; tolerates 1 failure |
| Volume nodes | 4 | Erasure coding 4+2 minimum for durability |
| Filer nodes | 2 (active-passive) | S3 API gateway; stateless + metadata in volume store |

Deploy on dedicated hardware, **separate from Cassandra nodes** — SeaweedFS GC and rebalance operations compete for NVMe I/O bandwidth.

### Alternatives

| Option | License | When to choose |
|---|---|---|
| Ceph RGW | LGPL-2.1 | Ceph already deployed for other workloads; more operational complexity on greenfield |
| Medusa local backend (`storage_provider = local` + NFS) | Apache 2.0 | NAS/SAN already exists; zero additional infra; durability depends on NAS backend |

**Backup NIC rule:** All backup traffic (Medusa → SeaweedFS) must run over a dedicated out-of-band/management NIC bond, not the Cassandra data bond. Co-mingling backup and gossip/streaming traffic causes latency spikes during backup windows.

---

## Service Coordination

### Consul HA configuration

- **5 Consul server nodes minimum** for production (tolerates 2 simultaneous failures without quorum loss)
- Consul servers on **dedicated management nodes** — never co-located with Cassandra. GC pauses and compaction I/O cause Raft heartbeat timeouts, leading to leader elections during Cassandra stress events
- **etcd is not a Consul replacement** for this use case: etcd provides KV and leader election but lacks service discovery, health check TTLs, and DNS registration. The Consul KV lock recipe above requires all three capabilities

### OpenTofu state backend

```hcl
# Consul backend (zero additional infra if Consul is already deployed)
terraform {
  backend "consul" {
    address = "consul.mgmt.internal:8500"
    path    = "terraform/cassandra/<cluster-name>"
    lock    = true
  }
}
```

Alternative: PostgreSQL backend (`backend "pg"`) if a PostgreSQL instance is already operational.

### GitOps on-premise

- **OpenTofu** (Apache 2.0): API-compatible drop-in for Terraform (BSL 1.1). Use OpenTofu if the BSL constraint is hard
- **Atlantis** (Apache 2.0): runs on-premise; triggers `opentofu plan` on PR, `opentofu apply` on merge; pairs with self-hosted GitLab CE or Forgejo
- **Forgejo Actions**: CI-based alternative to Atlantis if a separate GitOps server is undesirable

---

## DC Bare-Metal Host Layer

### NIC bonding

| Mode | Use case | Risk |
|---|---|---|
| LACP 802.3ad (mode 4), `layer3+4` hash | ToR switch supports MLAG | Requires switch coordination |
| Active-backup (mode 1) | ToR uplinks land on independent switches | Lower throughput; fully fault-tolerant |
| Round-robin (mode 0) | **Never** | Reorders TCP packets; breaks Cassandra gossip and streaming |

```yaml
# /etc/netplan/cassandra-bond.yaml
bonds:
  bond0:
    mtu: 9000
    interfaces: [eno1, eno2]
    parameters:
      mode: 802.3ad
      transmit-hash-policy: layer3+4
      lacp-rate: fast
      mii-monitor-interval: 100
```

**Jumbo frames (MTU 9000):** Only deploy end-to-end. A single switch or NIC in the path that does not support MTU 9000 causes fragmentation black-holes that are worse than 1500 MTU. Validate with `ping -M do -s 8972 <peer>` across every hop before enabling.

### NUMA pinning

Pin the Cassandra JVM to a single NUMA node. Add to systemd `ExecStart`:

```bash
ExecStart=numactl --cpunodebind=0 --membind=0 /usr/bin/cassandra -f
```

For dual-socket co-location: cluster-1 → NUMA node 0, cluster-2 → NUMA node 1. Set `kernel.numa_balancing=0` in sysctl to prevent the kernel from migrating pages across NUMA domains.

### IRQ affinity (add to cassandra-tuning.service)

```bash
# Disable irqbalance for Cassandra NIC interfaces
systemctl stop irqbalance
# Pin NIC IRQs to 2–4 cores adjacent to but outside the Cassandra CPU pinset
for IRQ in $(grep eno /proc/interrupts | awk -F: '{print $1}'); do
  echo 6 > /proc/irq/$IRQ/smp_affinity  # cores 1+2 (bitmask 0x6)
done
```

Add as `ExecStartPost` in `cassandra-tuning.service`, after the NIC is confirmed up.

### Storage layout (NVMe-based server)

| Mount point | Device | Filesystem | Mount options |
|---|---|---|---|
| `/var/lib/cassandra/commitlog` | NVMe 0 (or dedicated partition) | XFS | `defaults,noatime,nodiratime,nobarrier,allocsize=64m` |
| `/var/lib/cassandra/data` | NVMe 1+ (JBOD) | XFS | `defaults,noatime,nodiratime,nobarrier,allocsize=64m` |
| Management tooling (Consul, etc.) | NVMe 0, separate partition | XFS | `defaults,noatime` |

**No hardware RAID on Cassandra data drives.** Cassandra RF=3 provides redundancy at the application layer. Multiple NVMe devices map to multiple `data_file_directories` in `cassandra.yaml` for JBOD round-robin.

`nobarrier` requires **enterprise NVMe with power-loss protection (PLP)**. Confirm PLP capability before enabling — consumer NVMe without PLP risks commit log corruption on sudden power loss.

### CPU governor

```bash
# In cassandra-tuning.service ExecStart
cpupower frequency-set -g performance
```

Use `performance` governor, not `schedutil`. `schedutil` scales clock speed based on load — this creates variable tail latency under mixed compaction + query workloads.

### BIOS settings (validate via Ansible + ipmitool/Redfish pre-flight)

| Setting | Required value | Reason |
|---|---|---|
| C-states | Disable C6/C7 (allow C1/C1E only) | C6 exit latency 100–500µs causes gossip tail latency spikes |
| CPU Power Management | Maximum Performance or OS Control | Prevents firmware from overriding `performance` governor |
| NUMA Interleave | Disable | Defeats single-NUMA-node pinning |
| Hyper-Threading | Enable | Compaction threads and G1GC concurrent marking benefit |
| Turbo Boost | Enable | Per-connection latency improvement |
| MMIO High (Above 4G) | Enable | Required for NVMe PCIe on dual-socket servers |

---

## License Landscape

| Tool | License | OSS status |
|---|---|---|
| Apache Cassandra 5.x | Apache 2.0 | Approved |
| Podman | Apache 2.0 | Approved |
| Ansible | GPL-3.0 (core) / Apache 2.0 (collections) | Approved (GPL runtime; not redistributed as SaaS) |
| Reaper | Apache 2.0 | Approved |
| Medusa | Apache 2.0 | Approved |
| SeaweedFS | Apache 2.0 | Approved |
| instaclustr/cassandra-exporter | Apache 2.0 | Approved |
| prometheus/jmx_exporter | Apache 2.0 | Approved |
| Prometheus | Apache 2.0 | Approved |
| Alertmanager | Apache 2.0 | Approved |
| OpenTofu | Apache 2.0 | Approved |
| Atlantis | Apache 2.0 | Approved |
| Ceph RGW (alternative) | LGPL-2.1 | Approved (LGPL) |
| Grafana | AGPL-3.0 | **Requires review** — AGPL requires source publication if the service is distributed externally |
| Consul | BSL 1.1 | **Requires review** — BSL prohibits competitive hosting use cases; legal sign-off needed |
| Nomad (optional) | BSL 1.1 | **Requires review** — same BSL concern as Consul |
| MinIO | AGPLv3 + archived | **Eliminated** — AGPL + no available binary |

---

## Known Operational Risks

| Risk | Trigger | Mitigation |
|---|---|---|
| CASSANDRA-8741 hang | `nodetool drain` run before `nodetool decommission` | Never drain before decommission; drain only before restart |
| Schema disagreement mid-restart | DDL change lands while rolling restart is running | Gate 2 (schema agreement) blocks advance; hold all DDL during restart windows |
| Compaction cascade | Restart during high compaction backlog causes I/O storm on remaining nodes | Gate 5 + compaction pressure check; wait until `PENDING < 50` |
| Consul leader election | GC/compaction on co-located Consul node causes Raft timeout | Consul servers on dedicated management nodes only |
| Jumbo frame black-hole | Single path device with MTU < 9000 | Validate end-to-end with `ping -M do -s 8972` before enabling MTU 9000 |
| `nobarrier` data loss | Consumer NVMe without PLP loses commit log on sudden power loss | Confirm NVMe PLP before enabling `nobarrier` |
| CEP-53 dependency | Teams wait for CEP-53 before deploying | Do not block; adopt current Ansible 5-gate playbook now; migrate when CEP-53 is GA |
| SeaweedFS filer single point | Single filer node S3 gateway outage blocks all backups | Deploy 2+ filer nodes behind a load balancer |

---

## Decision Matrix

| Profile | Recommendation | Eliminated options and why |
|---|---|---|
| OSS-only, on-prem, <100 nodes | **Architecture B** (Podman Quadlets + Ansible) | Nomad (BSL 1.1); K8ssandra (Kubernetes prohibited); AxonOps (commercial) |
| OSS-only, on-prem, >100 nodes, BSL approved | **Architecture B + Nomad** scheduler layer | AxonOps (commercial); K8ssandra (Kubernetes prohibited) |
| OSS-only, on-prem, Ceph already deployed | **Architecture B + Ceph RGW** instead of SeaweedFS | MinIO (archived + AGPL) |
| Cloud or managed OK (prior topics) | See `cassandra-bare-metal-no-k8s-container.report.md` | Out of scope for this report |
| Kubernetes allowed | See `cassandra-bare-metal-container-required.report.md` (K8ssandra) | Kubernetes prohibited in this context |

---

## What This Does Not Cover

- Cassandra 4.x to 5.x migration path (schema compatibility, SSTable format upgrade)
- Multi-DC / multi-region replication topology (NetworkTopologyStrategy across physical DCs)
- TLS mutual authentication between Cassandra nodes (client-to-node and inter-node encryption configuration)
- Secrets management (Vault / OpenBao integration for keystore passwords)
- Capacity planning and sizing (heap, compaction throughput, token assignment)
- Cassandra Sidecar (CEP-53) deployment once GA — revisit this report at that time

---

## References

- [instaclustr/cassandra-exporter Prometheus](../resources/cassandra/instaclustr-cassandra-exporter-prometheus.summary.md)
- [MCAC Cassandra 5 Deprecation Status](../resources/cassandra/mcac-cassandra5-deprecation-status.summary.md)
- [Criteo Cassandra Exporter Maintenance Status](../resources/cassandra/criteo-cassandra-exporter-maintenance-status.summary.md)
- [Consul HA Reference Architecture](../resources/consul/consul-ha-reference-architecture.summary.md)
- [etcd vs Consul KV Lock Comparison](../resources/consul/etcd-vs-consul-kv-lock-comparison.summary.md)
- [Atlantis GitOps On-Premise Terraform](../resources/consul/atlantis-gitops-on-premise-terraform.summary.md)
- [Terraform State Backend On-Premise Options](../resources/consul/terraform-state-backend-on-premise-options.summary.md)
- [SeaweedFS GitHub Repo](../resources/seaweedfs/github-repo-seaweedfs.summary.md)
- [SeaweedFS Production Setup Wiki](../resources/seaweedfs/production-setup-wiki.summary.md)
- [Kubeflow SeaweedFS Adoption 2025](../resources/seaweedfs/kubeflow-seaweedfs-adoption-2025.summary.md)
- [MinIO Repo Archived 2026](../resources/minio/minio-repo-archived-2026.summary.md)
- [MinIO CE Dead 2026 Alternatives](../resources/minio/minio-ce-dead-2026-alternatives.summary.md)
- [MinIO AGPL License Analysis](../resources/minio/minio-agpl-license-analysis.summary.md)
- Prior report: [cassandra-bare-metal-no-k8s-container.report.md](cassandra-bare-metal-no-k8s-container.report.md)
- Prior report: [cassandra-bare-metal-container-required.report.md](cassandra-bare-metal-container-required.report.md)
- Prior report: [cassandra-bare-metal-nomad-stack.report.md](cassandra-bare-metal-nomad-stack.report.md)

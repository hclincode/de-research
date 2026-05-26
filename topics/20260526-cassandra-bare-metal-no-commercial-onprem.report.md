---
title: "Cassandra 5.x Bare-Metal — No-Commercial-Tools On-Premise (Container Required, Kubernetes Prohibited)"
date: 2026-05-26
status: complete
components:
  [cassandra, podman, nomad, ansible, reaper, medusa, seaweedfs, consul, prometheus, grafana, terraform, opentofu, atlantis]
constraints:
  deployment: on-premise
  os: ubuntu-22.04
  container: required
  kubernetes: prohibited
  license: no-paid-commercial (BSL 1.1 and any zero-cost-for-internal-use license allowed; tools that charge for enterprise internal use prohibited)
prior-topics:
  - cassandra-bare-metal-oss-onprem (superseded by this report)
  - cassandra-bare-metal-no-k8s-container
  - cassandra-bare-metal-container-required
  - cassandra-bare-metal-nomad-stack
---

## Overview

This report revises `cassandra-bare-metal-oss-onprem` under updated license constraints: **BSL 1.1 is allowed, and any license that does not charge for enterprise internal use is allowed.** The prior "OSS-only" framing excluded BSL 1.1 tools (Nomad, Consul, Terraform); removing that restriction rehabilitates Nomad as a scheduler and Consul as a fully approved dependency without caveat. All other constraints carry forward: containers required, Kubernetes prohibited, on-premise bare-metal only, commercial paid tools prohibited.

**Verdict: Architecture splits by scale. Architecture B (Podman Quadlets + Ansible) remains the right choice for clusters under ~100 nodes or single-cluster deployments. Architecture A (Nomad + Ansible) is now the preferred choice for clusters over ~100 nodes, multi-cluster co-location, or teams already operating Nomad. AxonOps (commercial, charges for production tier) and MinIO (archived February 2026, no binary) remain eliminated.**

---

## License Constraint Definition

A tool is permitted if it satisfies **both**:
1. Does not charge money for enterprise internal production use.
2. Does not require a paid license or subscription to operate at scale.

| License | Verdict | Reason |
|---|---|---|
| Apache 2.0 | Approved | Permissive; no restrictions |
| MIT | Approved | Permissive; no restrictions |
| LGPL-2.1 | Approved | Internal use unrestricted |
| GPL-3.0 | Approved | No charge; source disclosure only if redistributed externally |
| BSL 1.1 (HashiCorp) | **Approved** | Zero cost for internal production use; restriction targets only competing commercial service providers |
| AGPL-3.0 | Approved | Zero cost; source disclosure only if the service is externally distributed; internal use unrestricted |
| Commercial / SaaS (e.g. AxonOps) | Eliminated | Charges for enterprise/production tier |
| AGPLv3 + archived binary (MinIO CE) | Eliminated | No available binary since October 2025; repository archived February 2026 |

**BSL 1.1 clarification:** The "Additional Use Grant" in BSL 1.1 prohibits using the software to offer a competing commercial hosted service. It does not restrict internal production use. An enterprise running Nomad or Consul internally to manage its own infrastructure is not competing with HashiCorp. Organizations with strict legal review processes should obtain formal sign-off before production deployment.

---

## Why Architecture A Is Rehabilitated

In `cassandra-bare-metal-oss-onprem`, Architecture A was eliminated because Nomad (BSL 1.1) failed the strict Apache 2.0 filter. Under the updated constraint:

| Tool | Prior status | Updated verdict |
|---|---|---|
| Nomad OSS | BSL 1.1 — eliminated | **Approved** — zero cost for internal use |
| Consul | BSL 1.1 — required review | **Approved** — zero cost for internal use |
| AxonOps | Commercial — eliminated | **Eliminated** — charges for production tier |

Architecture A's fundamental rolling restart gap remains: **without AxonOps, there is no rack-aware rolling restart coordinator outside Kubernetes.** Nomad's scheduler gains return (bin-packing, declarative job specs, static port enforcement, multi-cluster dispatch), but the rolling restart problem is still solved by the Ansible 5-gate playbook — the same solution used in Architecture B.

---

## Evidence Quality

| Source | File | Tier | Accessible |
|---|---|---|---|
| instaclustr/cassandra-exporter | [instaclustr-cassandra-exporter-prometheus.summary.md](../resources/cassandra/instaclustr-cassandra-exporter-prometheus.summary.md) | official | Yes |
| MCAC Cassandra 5 deprecation | [mcac-cassandra5-deprecation-status.summary.md](../resources/cassandra/mcac-cassandra5-deprecation-status.summary.md) | press | Yes |
| Criteo exporter maintenance status | [criteo-cassandra-exporter-maintenance-status.summary.md](../resources/cassandra/criteo-cassandra-exporter-maintenance-status.summary.md) | press | Yes |
| Consul HA reference architecture | [consul-ha-reference-architecture.summary.md](../resources/consul/consul-ha-reference-architecture.summary.md) | vendor | Yes |
| etcd vs Consul KV lock comparison | [etcd-vs-consul-kv-lock-comparison.summary.md](../resources/consul/etcd-vs-consul-kv-lock-comparison.summary.md) | press | Yes |
| Atlantis GitOps on-premise | [atlantis-gitops-on-premise-terraform.summary.md](../resources/nomad/atlantis-gitops-on-premise-terraform.summary.md) | official | Yes |
| Terraform state backend on-premise options | [terraform-state-backend-on-premise-options.summary.md](../resources/nomad/terraform-state-backend-on-premise-options.summary.md) | press | Yes |

**Gaps and confidence limits:**
- CEP-53 (Cassandra Sidecar rolling restart API) passed community vote October 2025 but is not yet shipped. Timeline confidence LOW.
- BSL 1.1 internal-use interpretation is derived from license text and HashiCorp's published FAQ; no independent legal opinion was obtained. Confidence MEDIUM — obtain legal sign-off for production use.
- Consul HA guidance comes from a HashiCorp/vendor source; corroborated by the etcd comparison paper (press tier). Confidence MEDIUM.
- NIC bonding and BIOS settings are practitioner synthesis; no single peer-reviewed source covers the full set. Confidence MEDIUM.
- SeaweedFS production readiness supported by Kubeflow adoption (press) and GitHub issue analysis. Confidence MEDIUM.

---

## Architecture A: Nomad + Ansible (large/multi-cluster)

**Choose when:** clusters exceed ~100 nodes; multiple Cassandra clusters share bare-metal hosts; team already operates Nomad for other workloads; declarative job specs and bin-packing improve operational efficiency.

| Tool | License | Role |
|---|---|---|
| Nomad OSS | BSL 1.1 | Container scheduler, placement, static port enforcement |
| containerd (nomad-driver-containerd) | Apache 2.0 | Runtime at scale; simpler cgroup model |
| Podman (nomad-driver-podman) | Apache 2.0 | Alternative runtime; rootless where required |
| Terraform or OpenTofu | BSL 1.1 / Apache 2.0 | Job spec templating; plan/diff/state |
| Consul | BSL 1.1 | KV distributed lock, service discovery, health checks |
| Ansible | GPL-3.0 / Apache 2.0 | 5-gate rolling restart orchestration |
| Reaper | Apache 2.0 | Repair scheduling |
| Medusa | Apache 2.0 | Backup/restore |
| SeaweedFS | Apache 2.0 | On-premise S3-compatible backup backend |
| Prometheus + instaclustr/cassandra-exporter | Apache 2.0 | Metrics |
| Grafana | AGPL-3.0 | Dashboards (internal self-hosted; approved) |

### Critical Nomad job spec

The following fields are safety-critical. Omitting any one creates a data availability risk.

```hcl
group "cassandra" {
  # Never reschedule to a different host — Cassandra owns its token range and data.
  reschedule {
    attempts  = 0
    unlimited = false
  }

  # Never auto-restart in place — all restarts go through the drain wrapper.
  restart {
    attempts = 0
    mode     = "fail"
  }

  task "cassandra" {
    # Allow nodetool drain to complete before SIGKILL. Raise to 900s under heavy compaction.
    kill_timeout = "600s"
    kill_signal  = "SIGTERM"

    volume_mount {
      volume      = "cassandra-data"
      destination = "/var/lib/cassandra"
      read_only   = false
    }
  }

  volume "cassandra-data" {
    type      = "host"
    read_only = false
    source    = "cassandra-data"
  }
}
```

The container entrypoint must trap SIGTERM, run `nodetool drain` (restart) or `nodetool decommission` (permanent removal — see CASSANDRA-8741 below), wait for completion, then exit.

### Nomad OSS vs Enterprise decision rule

The trigger is whether clusters with **different hardware profiles** share the same Nomad data-plane nodes.

| Condition | Decision |
|---|---|
| All nodes in a cluster are dedicated | Nomad OSS sufficient |
| Multiple clusters share nodes with the **same** hardware profile | Nomad OSS + ACL namespace isolation (access control only, not scheduling enforcement) |
| Multiple clusters share nodes with **different** hardware profiles | Nomad Enterprise required — hard node-pool binding prevents profile mismatches. Nomad Enterprise charges for production; requires separate license approval. If cost is a constraint, use dedicated nodes per hardware profile and stay on OSS. |

### Multi-cluster port-offset co-location

```hcl
network {
  port "cql"     { static = 9042 }  # cluster-1
  port "storage" { static = 7000 }
  port "jmx"     { static = 7199 }
}
# cluster-2: 9142 / 7100 / 7299   cluster-3: 9242 / 7200 / 7399
```

Register each cluster's port allocation in Consul KV at `cassandra/ports/<hostname>/<cluster-name>` at deploy time. Do not exceed 3 Cassandra clusters per host — beyond this, GC pressure and compaction I/O contention become unpredictable.

---

## Architecture B: Podman Quadlets + Ansible (small/single cluster)

**Choose when:** clusters are under ~100 nodes; "container required" is an image-packaging constraint rather than a runtime scheduling requirement; team already operates the Topic 1 Ansible toolchain.

| Tool | License | Role |
|---|---|---|
| Podman + systemd Quadlets | Apache 2.0 | Daemonless container lifecycle; `.container` unit files |
| Ansible | GPL-3.0 / Apache 2.0 | Provisioning + 5-gate rolling restart |
| Reaper | Apache 2.0 | Repair scheduling |
| Medusa | Apache 2.0 | Backup/restore |
| SeaweedFS | Apache 2.0 | On-premise S3-compatible backup backend |
| Consul | BSL 1.1 | KV distributed lock; seed inventory |
| Prometheus + instaclustr/cassandra-exporter | Apache 2.0 | Metrics |
| Grafana | AGPL-3.0 | Dashboards |

**ExecStopPost drain:** The systemd unit must include `ExecStopPost=/usr/bin/nodetool drain` and `TimeoutStopSec=600` to ensure drain completes before the container exits.

| Cluster size | Verdict |
|---|---|
| ≤30 nodes, single cluster | Fully viable |
| 30–100 nodes | Viable; rolling restart playbook becomes operational bottleneck |
| 100+ nodes | Degrades; migrate to Architecture A |
| Multiple clusters on shared hosts | Architecture A preferred |

---

## Production Rolling Restart: The 5-Gate Ansible Playbook

Applies to **both architectures.** Architecture A invokes it via Ansible triggered from Nomad batch jobs or CI; Architecture B invokes it directly.

### Gate definitions

| Gate | Command | Pass condition |
|---|---|---|
| 1. Node UN state | `nodetool status` | Target node shows `UN` |
| 2. Schema agreement | `nodetool describecluster` | Exactly one schema UUID returned |
| 3. No active streaming | `nodetool netstats` | Zero active streams and zero pending |
| 4. Hint queue depth | `nodetool tpstats \| grep HintedHandoff` | Pending = 0 on all nodes |
| 5. No active repair | `nodetool compactionstats \| grep VALIDATION` | Empty output |

### Compaction pressure check (additional gate)

```bash
PENDING=$(nodetool compactionstats | grep "pending tasks" | awk '{print $NF}')
[ "$PENDING" -gt 50 ] && echo "BLOCK: compaction backlog $PENDING, wait before draining"
```

Deploy as an Ansible `assert` with `retries: 60` and `delay: 30` (30-minute max wait per node).

### Consul KV distributed lock recipe

```bash
SESSION=$(curl -s -X PUT http://consul:8500/v1/session/create \
  -d '{"TTL":"600s","Behavior":"release"}' | jq -r .ID)

LOCK=$(curl -s -X PUT \
  "http://consul:8500/v1/kv/cassandra/rolling-restart/lock?acquire=$SESSION" \
  -d "$(hostname)")

[ "$LOCK" != "true" ] && echo "BLOCK: another node is restarting" && exit 1

# ... drain, restart, wait for UN ...

curl -s -X PUT \
  "http://consul:8500/v1/kv/cassandra/rolling-restart/lock?release=$SESSION"
```

**CASSANDRA-8741 rule (mandatory):** Use `nodetool drain` only before a restart. Never run `nodetool drain` before `nodetool decommission` — drain disables the commit log listener, causing decommission to hang indefinitely. Decommission proceeds without drain.

---

## CEP-53 Watch Item

CEP-53 (Cassandra Enhancement Proposal 53) defines a Cassandra Sidecar rolling restart REST API — a native, rack-aware, health-gated coordinator built into Cassandra itself. Passed community vote October 2025; under active development as of May 2026. When GA, it will replace the 5-gate Ansible playbook for both architectures. **Do not block deployment on CEP-53 — adopt when released.**

---

## Monitoring Stack

### MCAC is deprecated for Cassandra 5.x

MCAC v0.3.6 (March 2025) explicitly supports Cassandra through 4.1 only. Do not use for new Cassandra 5.x deployments.

### Replacement options

| Exporter | License | Cassandra support | Notes |
|---|---|---|---|
| `instaclustr/cassandra-exporter` | Apache 2.0 | 4.0+ native API | JVM agent; in-process scrape; no JMX required |
| `prometheus/jmx_exporter` | Apache 2.0 | No version restriction | Requires Cassandra JMX rules YAML; actively maintained by Prometheus project |

Use `instaclustr/cassandra-exporter` as primary; `jmx_exporter` as fallback when JMX is already exposed for other tooling.

**Grafana (AGPL-3.0)** is approved. Source disclosure is only required if the Grafana service is externally distributed — internal self-hosted deployment is unrestricted.

### Alertmanager rolling restart gates

| Gate | Source | Block expression |
|---|---|---|
| Compaction queue | `cassandra_compaction_pending_tasks` | `> 0` |
| Schema convergence | `cassandra_schema_versions` | `count() > 1` |
| No active repairs | Reaper REST API `GET /repair_run` | Any RUNNING state |
| Gossip state UN | Ansible `nodetool status` poll | Node not in UN |

---

## Backup: SeaweedFS On-Premise

MinIO is eliminated regardless of license stance: the community edition repository was archived February 2026 and pre-built binaries stopped shipping October 2025.

SeaweedFS (Apache 2.0) is the replacement. Medusa uses `storage_provider = s3_compatible` pointed at the SeaweedFS filer S3 gateway.

**Minimum HA cluster layout:**

| Component | Minimum nodes | Notes |
|---|---|---|
| Master nodes | 3 | Raft consensus HA; tolerates 1 failure |
| Volume nodes | 4 | Erasure coding 4+2 minimum for durability |
| Filer nodes | 2 (active-passive) | S3 API gateway; stateless |

Deploy on dedicated hardware separate from Cassandra nodes — SeaweedFS GC and rebalance compete for NVMe I/O.

**Alternatives:** Ceph RGW (LGPL-2.1, if Ceph already deployed); Medusa local backend + NFS (if NAS exists).

**Backup NIC rule:** All backup traffic (Medusa → SeaweedFS) must run over a dedicated out-of-band NIC bond, not the Cassandra data bond.

---

## Service Coordination

### Consul HA

Consul (BSL 1.1) is fully approved under the updated constraint — no caveat required.

- **5 Consul server nodes minimum** for production (tolerates 2 simultaneous failures)
- Consul servers on **dedicated management nodes** — never co-located with Cassandra; GC/compaction triggers Raft heartbeat timeouts
- etcd is not a Consul replacement: etcd covers KV and leader election only; lacks service discovery, health check TTLs, and DNS registration

### Infrastructure-as-Code

Both **Terraform (BSL 1.1)** and **OpenTofu (Apache 2.0)** are approved. Use whichever the team already operates.

```hcl
terraform {
  backend "consul" {
    address = "consul.mgmt.internal:8500"
    path    = "terraform/cassandra/<cluster-name>"
    lock    = true
  }
}
```

**Atlantis** (Apache 2.0): on-premise GitOps runner; triggers `plan` on PR, `apply` on merge; pairs with self-hosted GitLab CE or Forgejo.

---

## DC Bare-Metal Host Layer

### NIC bonding

| Mode | Use case | Risk |
|---|---|---|
| LACP 802.3ad (mode 4), `layer3+4` hash | ToR switch supports MLAG | Requires switch coordination |
| Active-backup (mode 1) | Independent switch uplinks | Lower throughput; fully fault-tolerant |
| Round-robin (mode 0) | **Never** | Reorders TCP; breaks gossip and streaming |

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

Validate jumbo frames end-to-end with `ping -M do -s 8972 <peer>` across every hop before enabling MTU 9000.

### NUMA pinning

```bash
ExecStart=numactl --cpunodebind=0 --membind=0 /usr/bin/cassandra -f
```

For dual-socket co-location: cluster-1 → NUMA node 0, cluster-2 → NUMA node 1. Set `kernel.numa_balancing=0`.

### IRQ affinity (add to cassandra-tuning.service)

```bash
systemctl stop irqbalance
for IRQ in $(grep eno /proc/interrupts | awk -F: '{print $1}'); do
  echo 6 > /proc/irq/$IRQ/smp_affinity  # cores 1+2 (bitmask 0x6)
done
```

### Storage layout (NVMe)

| Mount point | Filesystem | Mount options |
|---|---|---|
| `/var/lib/cassandra/commitlog` | XFS | `defaults,noatime,nodiratime,nobarrier,allocsize=64m` |
| `/var/lib/cassandra/data` | XFS | `defaults,noatime,nodiratime,nobarrier,allocsize=64m` |

No hardware RAID on Cassandra data drives — RF=3 provides redundancy. Map multiple NVMe devices to multiple `data_file_directories` in `cassandra.yaml` (JBOD). `nobarrier` requires **enterprise NVMe with power-loss protection (PLP)** — confirm before enabling.

### CPU governor

```bash
cpupower frequency-set -g performance
```

Use `performance`, not `schedutil`. `schedutil` creates variable tail latency under mixed compaction + query workloads.

### BIOS settings

| Setting | Required value | Reason |
|---|---|---|
| C-states | Disable C6/C7 (allow C1/C1E) | C6 exit latency 100–500µs causes gossip tail latency spikes |
| CPU Power Management | Maximum Performance or OS Control | Prevents firmware overriding `performance` governor |
| NUMA Interleave | Disable | Defeats single-NUMA-node pinning |
| Hyper-Threading | Enable | Compaction threads and G1GC concurrent marking benefit |
| Turbo Boost | Enable | Per-connection latency improvement |
| MMIO High (Above 4G) | Enable | Required for NVMe PCIe on dual-socket servers |

---

## License Landscape

| Tool | License | Status |
|---|---|---|
| Apache Cassandra 5.x | Apache 2.0 | Approved |
| Podman | Apache 2.0 | Approved |
| Ansible | GPL-3.0 / Apache 2.0 | Approved |
| Reaper | Apache 2.0 | Approved |
| Medusa | Apache 2.0 | Approved |
| SeaweedFS | Apache 2.0 | Approved |
| instaclustr/cassandra-exporter | Apache 2.0 | Approved |
| prometheus/jmx_exporter | Apache 2.0 | Approved |
| Prometheus | Apache 2.0 | Approved |
| Alertmanager | Apache 2.0 | Approved |
| OpenTofu | Apache 2.0 | Approved |
| Atlantis | Apache 2.0 | Approved |
| Ceph RGW (alternative) | LGPL-2.1 | Approved |
| Nomad OSS | BSL 1.1 | **Approved** — zero cost for internal production use |
| Consul | BSL 1.1 | **Approved** — zero cost for internal production use |
| Terraform | BSL 1.1 | **Approved** — zero cost for internal production use |
| Grafana | AGPL-3.0 | **Approved** — zero cost; source disclosure only if externally distributed |
| Nomad Enterprise | Commercial | **Requires separate approval** — charges for production; only needed for different-hardware-profile multi-cluster co-location on shared nodes |
| AxonOps | Commercial | **Eliminated** — charges for enterprise/production tier |
| MinIO CE | AGPLv3 + archived | **Eliminated** — no available binary; repository archived February 2026 |

---

## Known Operational Risks

| Risk | Trigger | Mitigation |
|---|---|---|
| CASSANDRA-8741 hang | `nodetool drain` before `nodetool decommission` | Never drain before decommission; drain only before restart |
| Cassandra ghost node (Architecture A) | Nomad reschedules failed task to different host | `reschedule { attempts = 0 }` mandatory in job spec |
| Schema disagreement mid-restart | DDL change during rolling restart window | Gate 2 blocks advance; hold all DDL during restart windows |
| Compaction cascade | Restart during high compaction backlog | Gate 5 + pressure check; wait until PENDING < 50 |
| `host_volume` path absent (Architecture A) | Node rebooted, mount not re-attached before Nomad schedules | node-exporter mount-presence alert; Nomad placement fails if volume not declared |
| Consul leader election | GC/compaction on co-located Consul server causes Raft timeout | Consul servers on dedicated management nodes only |
| Port collision (Architecture A, multi-cluster) | Static port not declared in Nomad spec | Nomad static port reservation; Consul KV registry for ops audit |
| Jumbo frame black-hole | Single device in path with MTU < 9000 | Validate end-to-end with `ping -M do -s 8972` before enabling |
| `nobarrier` data loss | Consumer NVMe without PLP loses commit log on power loss | Confirm NVMe PLP before enabling `nobarrier` |
| BSL 1.1 interpretation | Legal team interprets BSL more restrictively | Obtain formal sign-off before production deployment |

---

## Decision Matrix

| Profile | Recommendation | Eliminated options and why |
|---|---|---|
| <100 nodes, single cluster | **Architecture B** (Podman Quadlets + Ansible) | Nomad adds scheduler complexity without benefit at this scale |
| >100 nodes, or multi-cluster on shared hosts | **Architecture A** (Nomad OSS + Ansible) | Architecture B degrades at scale; Nomad now approved under BSL |
| Multi-cluster, different hardware profiles, shared hosts | **Architecture A + dedicated nodes per hardware profile** | OSS node-pool isolation is ACL only; Nomad Enterprise closes the gap but charges — separate approval required |
| Cassandra-only, no Nomad operational experience | **Architecture B** | Nomad control plane adds overhead without proportional benefit |
| Kubernetes allowed | See [20260526-cassandra-bare-metal-container-required.report.md](20260526-cassandra-bare-metal-container-required.report.md) | K8ssandra is the only fully rack-aware option |
| No container requirement | See [20260526-cassandra-bare-metal-nomad-stack.report.md](20260526-cassandra-bare-metal-nomad-stack.report.md) | Native systemd outperforms containers on bare metal |

---

## What This Does Not Cover

- Cassandra 4.x to 5.x migration (SSTable format upgrade, schema compatibility)
- Multi-DC / multi-region replication topology
- TLS mutual authentication (client-to-node and inter-node encryption)
- Secrets management (Vault / OpenBao integration)
- Capacity planning and ring topology (vnodes vs. single-token, rack/DC layout)
- Cassandra Sidecar (CEP-53) deployment once GA — revisit this report at that time

---

## References

- [instaclustr/cassandra-exporter Prometheus](../resources/cassandra/instaclustr-cassandra-exporter-prometheus.summary.md)
- [MCAC Cassandra 5 Deprecation Status](../resources/cassandra/mcac-cassandra5-deprecation-status.summary.md)
- [Criteo Cassandra Exporter Maintenance Status](../resources/cassandra/criteo-cassandra-exporter-maintenance-status.summary.md)
- [Consul HA Reference Architecture](../resources/consul/consul-ha-reference-architecture.summary.md)
- [etcd vs Consul KV Lock Comparison](../resources/consul/etcd-vs-consul-kv-lock-comparison.summary.md)
- [Atlantis GitOps On-Premise Terraform](../resources/nomad/atlantis-gitops-on-premise-terraform.summary.md)
- [Terraform State Backend On-Premise Options](../resources/nomad/terraform-state-backend-on-premise-options.summary.md)
- Superseded report: [20260526-cassandra-bare-metal-oss-onprem.report.md](20260526-cassandra-bare-metal-oss-onprem.report.md)
- Prior report: [20260526-cassandra-bare-metal-no-k8s-container.report.md](20260526-cassandra-bare-metal-no-k8s-container.report.md)
- Prior report: [20260526-cassandra-bare-metal-container-required.report.md](20260526-cassandra-bare-metal-container-required.report.md)
- Prior report: [20260526-cassandra-bare-metal-nomad-stack.report.md](20260526-cassandra-bare-metal-nomad-stack.report.md)

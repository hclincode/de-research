---
title: Cassandra 5.x Bare-Metal Ubuntu 22.04 — Container-Required, Kubernetes-Prohibited
date: 2026-05-26
status: complete
components: [cassandra, nomad, consul, podman, containerd, ansible, reaper, medusa, axonops, terraform]
constraints:
  - deployment: on-premise bare-metal
  - os: ubuntu-22.04
  - container: required
  - kubernetes: prohibited
prior-topics:
  - cassandra-bare-metal-nomad-stack
  - cassandra-bare-metal-container-required
---

## Overview

The prior topics established that (1) without a container requirement, Ansible + systemd + Reaper is best practice, and (2) when containers are required and Kubernetes is allowed, K8ssandra-operator on k3s or kubeadm is the only viable production choice. This report addresses the hardest constraint combination: **containers are required and Kubernetes is prohibited**. Removing Kubernetes eliminates cass-operator — the only maintained open-source rack-aware rolling restart coordinator — and forces every surviving architecture to fill that gap differently.

**Verdict: Nomad + containerd + AxonOps (or an Ansible gossip-probe fallback) is the primary recommendation when the team already operates Nomad or manages multiple workload types. Podman Quadlets + Ansible + Reaper is the correct choice when "container required" means image-packaging discipline on a team that already runs the Topic 1 toolchain. Neither architecture is equivalent to K8ssandra; both require accepting explicit operational gaps documented below.**

---

## Structural Finding: The Gossip-Coordination Gap

No open-source, non-Kubernetes, rack-aware rolling restart coordinator for Cassandra exists in 2026. `cass-operator` (the K8ssandra component that provides this capability) is K8s-native and disappears with the K8s constraint. Every non-K8s architecture must choose one of:

- **AxonOps** (commercial): in-process JVM agent, rack-aware rolling restart, orchestrator-independent. Rack-aware rolling restarts are a commercial-tier feature. Server failure mid-restart has no documented safe suspend/resume path.
- **Ansible + Consul KV lock + `nodetool status` UN state polling** (open source): a functional approximation. Gap: no compaction-pressure check. A node can report UN while actively compacting; draining it mid-compaction increases the risk of streaming pressure on peers.

This gap is architectural, not configurational. It shapes every recommendation that follows.

---

## Why Kubernetes Alternatives Are Eliminated

### Docker Swarm

Docker Swarm has received no substantive development since 2019. Its rolling update model relies on time-based delays (`update_delay`), not state polling — there is no mechanism to gate advancement on actual gossip convergence. Stateful placement is fragile: Swarm has no native equivalent of Kubernetes StatefulSet rack-to-node topology constraints, and volume placement behavior on container restart is not deterministic across node failures. Eliminated on staleness and capability grounds.

### Docker Engine (standalone)

Running Cassandra containers under Docker Engine with `docker run` or Compose imposes ~54 MB idle RSS overhead for the Docker daemon while providing nothing over Podman's daemonless model. Docker Engine adds a daemon single point of failure between the host kernel and container lifecycle, which is strictly worse than Podman's direct-to-systemd integration via Quadlets. Eliminated as strictly dominated by Podman for this use case.

---

## The Two Viable Architectures

**Architecture A: Nomad + containerd/Podman + AxonOps (or Ansible fallback)**
Choose this when: the team already operates Nomad for other workloads, when multiple Cassandra clusters need co-location on shared hardware with scheduling enforcement, or when budget exists for AxonOps commercial tier.

**Architecture B: Podman Quadlets + Ansible + Reaper + Consul KV**
Choose this when: "containers required" is an image-packaging constraint (not a runtime scheduling requirement), the team already runs the Topic 1 Ansible toolchain, or there is no appetite for a new scheduler. This is effectively the Topic 1 winner with container image packaging layered on top.

---

## Architecture A: Nomad — Full Spec

### Small Clusters (6 Nodes)

| Tool | Role | Notes |
|---|---|---|
| Nomad OSS | Container scheduler + placement | Safe when each cluster's nodes are dedicated hardware |
| containerd via `nomad-driver-containerd` | Container runtime | Roblox community plugin; cgroup v2 required |
| Podman via `nomad-driver-podman` | Alternative runtime | Community plugin; cgroup delegation required for rootless |
| Terraform + Nomad provider | Job spec templating | Plan/diff/state; replaces Nomad-Pack |
| Consul KV + service mesh | Lock coordination, port registry | Gossip-probe fallback uses Consul KV for distributed lock |
| AxonOps (commercial tier) | Rack-aware rolling restarts | In-process JVM agent; no new host requirements |
| Ansible gossip-probe (open-source fallback) | Rolling restart approximation | UN polling; no compaction-pressure check |
| Reaper | Repair scheduling | Deployed as a Nomad job; unchanged from Topic 1 |
| Medusa | Backup/restore | Deployed as a Nomad job; unchanged from Topic 1 |
| node-exporter | Host drift detection + volume mount alerting | Monitors `host_volume` paths for missing mounts |

### Large Clusters (100+ Nodes)

| Tool | Role | Notes |
|---|---|---|
| Nomad OSS or Enterprise | Scheduler | Enterprise required only if clusters with different hardware profiles share nodes (see OSS vs Enterprise decision rule below) |
| containerd | Runtime | Preferred over Podman at scale; simpler cgroup model |
| Terraform + Nomad provider | GitOps delivery | State management across cluster fleet |
| Consul KV | Port registry + distributed locks | Collision prevention for multi-cluster co-location |
| AxonOps | Rack-aware rolling restarts | Commercial tier required; open-source fallback viable but unequal |
| nftables | Network isolation for port-offset clusters | Enforces inter-cluster traffic isolation on shared hosts |

### Critical Nomad Job Spec Requirements

The following stanza fields are safety-critical for Cassandra. Omitting any one of them creates a data availability risk.

```hcl
group "cassandra" {
  # Prevent Nomad from rescheduling a failed Cassandra node.
  # Cassandra manages its own data locality; auto-reschedule to a different
  # node would create a ghost node with stale data.
  reschedule {
    attempts  = 0
    unlimited = false
  }

  # Prevent Nomad from auto-restarting in place.
  # All restarts must go through the drain wrapper in the entrypoint.
  restart {
    attempts = 0
    mode     = "fail"
  }

  task "cassandra" {
    # Allow time for nodetool drain to complete before SIGKILL.
    # 600s minimum; increase to 900s if compaction load is high.
    kill_timeout = "600s"

    # SIGTERM triggers the drain wrapper (see entrypoint note below).
    # Do NOT send SIGKILL directly; CASSANDRA-8741.
    kill_signal = "SIGTERM"

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

**Entrypoint drain wrapper (required):** The container entrypoint must trap SIGTERM, run `nodetool drain` (for restart) or `nodetool decommission` (for permanent removal — CASSANDRA-8741), wait for completion, then exit. This is not automatic in the Cassandra Docker image. A permanent node removal that exits without `nodetool decommission` leaves the token ring in an inconsistent state.

### Nomad OSS vs Enterprise Decision Rule

The trigger is whether two clusters with **different hardware profiles** share the same Nomad data-plane nodes.

| Condition | Decision |
|---|---|
| All nodes in a cluster are dedicated to that cluster (no cross-cluster sharing) | Nomad OSS sufficient regardless of cluster count |
| Multiple clusters share nodes with the **same** hardware profile | Nomad OSS + ACL namespace isolation (access control only, not scheduling enforcement) |
| Multiple clusters share nodes with **different** hardware profiles | Nomad Enterprise required — hard node-pool binding prevents a cluster configured for NVMe from scheduling onto SATA nodes |

Nomad Enterprise cost: ~$15–30k/year. Justified only if Nomad already manages non-Cassandra workloads. If Cassandra is the only workload, prefer dedicated nodes per cluster and stay on OSS.

### Terraform Templating Model

Each Cassandra cluster is a Terraform module instantiation. The Nomad provider manages job spec lifecycle: `plan` produces a diff, `apply` submits the job, state tracks the deployed version. Nomad-Pack is not used — it has no diff/preview/rollback capability. One module per cluster; variables parameterize rack count, port offsets, hardware profile, and AxonOps license key.

### AxonOps vs Open-Source Fallback

| Decision condition | Use |
|---|---|
| Rack-aware rolling restart required + budget available | AxonOps commercial tier |
| Rack-aware rolling restart required + no budget | Ansible + Consul KV lock + `nodetool status` UN polling; document the compaction-pressure gap explicitly in your runbooks |
| Team tolerance for mid-restart server failure is low | AxonOps preferred; the open-source fallback has no documented safe suspend/resume |

---

## Architecture B: Podman Quadlets — Full Spec

| Tool | Role | Notes |
|---|---|---|
| Podman + systemd Quadlets | Container lifecycle | `.container` unit files generate systemd units; daemonless |
| Ansible | Rolling restart + provisioning | `nodetool status` UN polling between nodes |
| Reaper | Repair scheduling | Unchanged from Topic 1 |
| Medusa | Backup/restore | Unchanged from Topic 1 |
| Consul KV | Distributed lock for rolling restarts | Prevents concurrent restarts across racks |
| node-exporter | Drift detection | Same as Topic 1 |

### Critical Implementation Decisions

**ExecStopPost drain:** The systemd unit must include `ExecStopPost=/usr/bin/nodetool drain` and `TimeoutStopSec=600` to ensure drain completes before the container exits. Without this, a systemd `stop` terminates the JVM without draining commit log.

**Drain vs decommission (CASSANDRA-8741):** Use `nodetool drain` for rolling restarts (node returns to the ring). Use `nodetool decommission` for permanent node removal. The systemd unit handles drain; decommission must be a separate manual step in the removal runbook.

**Port offset management:** Not required in Architecture B if each host runs one Cassandra instance. If multi-cluster co-location is needed, manage port offsets through Ansible host variables and Consul KV registry.

### Where Architecture B Holds vs Degrades

| Cluster size | Verdict |
|---|---|
| ≤30 nodes, single cluster | Fully viable; Ansible playbook complexity is manageable |
| 30–100 nodes | Viable; rolling restart playbook becomes the operational bottleneck; no stateful scheduler to recover from partial failures |
| 100+ nodes | Degrades; Ansible as de facto orchestrator at this scale lacks the stateful scheduling of Nomad; consider Architecture A |
| Multiple clusters on shared hosts | Not recommended without additional port/isolation tooling; Architecture A preferred |

---

## Multi-Cluster Co-Location via Port Offsets

Port offset co-location allows multiple Cassandra clusters to share bare-metal hosts by assigning each cluster a distinct port range (native CQL 9042 → cluster-2 at 9142 → cluster-3 at 9242, etc.).

**What it enables:** Higher hardware utilization when clusters have complementary I/O profiles (e.g., one batch-heavy, one OLTP-light). Useful when hardware budget is fixed and Nomad Enterprise is not justified.

**What it risks:** JVM heap competition under concurrent GC pressure; gossip cross-contamination if nftables isolation is misconfigured; compaction from cluster A degrading latency for cluster B on shared NVMe.

**Density ceiling:** Do not exceed 3 Cassandra clusters per host. Beyond this, GC pressure and compaction I/O contention become unpredictable.

**Nomad static port enforcement + Consul KV registry recipe:**

```hcl
# In each cluster's Nomad job spec — static port reservation prevents collision
network {
  port "cql"     { static = 9042 }  # cluster-1
  port "storage" { static = 7000 }
  port "jmx"     { static = 7199 }
}
# cluster-2 uses 9142, 7100, 7299; cluster-3 uses 9242, 7200, 7399
```

Register each cluster's port allocation in Consul KV at `cassandra/ports/<hostname>/<cluster-name>` at deploy time. Ops tooling reads this registry before any manual intervention. nftables rules enforce that gossip traffic from cluster-1 nodes cannot reach cluster-2 nodes, even on the same host.

---

## Known Operational Risks

| Risk | Trigger | Mitigation |
|---|---|---|
| Cassandra ghost node after Nomad reschedule | Nomad reschedules a failed task to a different host | `reschedule { attempts = 0 }` in job spec; monitor Nomad allocation status |
| drain incomplete on SIGKILL timeout | `kill_timeout` too short under compaction load | Set `kill_timeout = "600s"` minimum; alert on compaction queue depth |
| `host_volume` path absent on restart | Node rebooted, mount not re-attached before Nomad schedules | node-exporter mount-presence alert on `host_volume` paths; Nomad job will fail to place if volume is not declared |
| AxonOps mid-restart server failure | AxonOps server dies while rolling restart is in progress | No documented suspend/resume; have Ansible fallback runbook ready to take over from last-known-good node |
| Compaction pressure during Ansible rolling restart | Node reports UN but is actively compacting | Add `nodetool compactionstats | grep -c "pending"` check before draining; abort if >0 pending compactions |
| Port collision on multi-cluster co-location | Static port not declared in Nomad spec | Nomad static port reservation enforced at scheduling; Consul KV registry for ops audit |
| Nomad Enterprise license expiry | License lapses; node-pool enforcement drops to OSS behavior | Alert 60 days before expiry; test OSS fallback scheduling behavior before renewal window |

---

## Decision Matrix

| Profile | Recommendation | Eliminated options and why |
|---|---|---|
| No container requirement | Topic 1: Ansible + systemd + Reaper + Consul KV | Containers add overhead with no lifecycle management benefit for Cassandra |
| Container required, K8s allowed | Topic 2: K8ssandra on k3s or kubeadm | Only fully rack-aware, gossip-gated rolling restart coordinator available |
| Container required, K8s prohibited, Nomad already in house | **Architecture A: Nomad + AxonOps** | K8s prohibited; Swarm stagnant; Docker Engine dominated by Podman |
| Container required, K8s prohibited, small team, Topic 1 toolchain already running | **Architecture B: Podman Quadlets + Ansible** | Nomad adds scheduler complexity without benefit when orchestration requirement is just image packaging |
| Container required, K8s prohibited, 100+ nodes, no Nomad, no AxonOps budget | **Architecture A with Ansible gossip-probe fallback** | Architecture B degrades at scale; Nomad OSS with Ansible fallback is the least-bad option; document compaction-pressure gap |
| Container required, K8s prohibited, multiple clusters on shared nodes, different hardware profiles | **Architecture A: Nomad Enterprise** | OSS node-pool isolation is ACL-only, not scheduling-enforcement; only Enterprise enforces hard hardware affinity |

---

## What This Does Not Cover

- Multi-DC replication across datacenters or WAN links
- Cloud deployments (AWS, GCP, Azure) or hybrid bare-metal/cloud topologies
- Cassandra version upgrade paths (4.x → 5.x, SSTable format migration)
- Cassandra security hardening (TLS, JMX auth, RBAC, audit logging)
- Capacity planning and ring topology (vnodes vs. single-token, rack/DC layout)
- Nomad security hardening (ACL policy design, mTLS between agents)
- Disaster recovery across full Nomad server cluster failure

---

## References

- [Prior report: Cassandra Bare-Metal Nomad Stack (Topic 1)](20260526-cassandra-bare-metal-nomad-stack.report.md)
- [Prior report: Cassandra Bare-Metal Container-Required (Topic 2, K8s allowed)](20260526-cassandra-bare-metal-container-required.report.md)
- [HashiCorp Nomad job specification — reschedule stanza](https://developer.hashicorp.com/nomad/docs/job-specification/reschedule)
- [HashiCorp Nomad job specification — restart stanza](https://developer.hashicorp.com/nomad/docs/job-specification/restart)
- [HashiCorp Nomad node pools (Enterprise)](https://developer.hashicorp.com/nomad/docs/concepts/node-pools)
- [nomad-driver-containerd (Roblox community plugin)](https://github.com/Roblox/nomad-driver-containerd)
- [nomad-driver-podman (community plugin)](https://github.com/hashicorp/nomad-driver-podman)
- [AxonOps — orchestrator-independent Cassandra operations](https://axonops.com/)
- [CASSANDRA-8741 — decommission vs drain distinction](https://issues.apache.org/jira/browse/CASSANDRA-8741)
- [Podman Quadlets documentation](https://docs.podman.io/en/latest/markdown/podman-systemd.unit.5.html)
- [Consul KV — distributed locking patterns](https://developer.hashicorp.com/consul/docs/dynamic-app-config/kv)

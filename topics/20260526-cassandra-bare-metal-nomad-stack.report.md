---
title: Cassandra 5.x Bare-Metal Ubuntu 22.04 — Nomad/Docker Stack Evaluation
date: 2026-05-26
status: complete
components: [cassandra, nomad, consul, docker, ansible, systemd, reaper, medusa]
constraints:
  - deployment: on-premise bare-metal
  - os: ubuntu-22.04
clarification-defaults:
  cassandra-version: "5.x latest stable"
  cluster-scale: "6 nodes (small) to 100+ nodes (large), multiple clusters"
  control-plane: "single Nomad/Consul control plane managing all clusters"
  outcome: "debate then recommend"
---

## Overview

This report evaluates whether Nomad + Nomad-Pack + Consul + Docker is best practice for deploying Cassandra 5.x on bare-metal Ubuntu 22.04, where a single control plane manages multiple heterogeneous clusters ranging from 6 to 100+ nodes.

**Verdict: The proposed stack is not recommended as specified.** Nomad as a Cassandra process scheduler is unsafe regardless of cluster size. Docker is acceptable on bare metal only under strict conditions that most teams will not maintain consistently. Nomad-Pack has no role when the underlying scheduler role is removed. What survives from the proposed stack is Consul KV (seed inventory) and, at large scale only, Nomad OSS repurposed as an operational batch job runner — not as the entity that starts or stops Cassandra processes.

---

## Evidence Quality

| Source | Tier | Notes |
|---|---|---|
| Apache Cassandra official docs (cassandra.apache.org) | official | Production recommendations, JDK support matrix |
| HashiCorp Nomad official docs (developer.hashicorp.com) | official | Stateful workloads, node pools, schedulers, NUMA block |
| Instaclustr blog — Cassandra 5.0 features, UCS | vendor-adjacent | Instaclustr is a managed Cassandra vendor |
| AxonOps — OS tuning, memory management docs | vendor-adjacent | AxonOps sells Cassandra monitoring tooling |
| HashiCorp blog — Nomad fault tolerance | vendor | HashiCorp publishes Nomad |
| GitHub kubernetes/kubernetes issue #90133 — container vs bare-metal Cassandra | press | Community benchmark report |
| arXiv 1812.04362 — containerization impact on database performance | analyst | Peer-reviewed |
| Portworx — Expert's Guide to Running Cassandra in Containers | vendor-adjacent | Portworx sells storage for containers |
| TechPlained — Docker vs bare metal 2026 benchmarks | press | Independent benchmark |
| ASF JIRA CASSANDRA-20681 | official | JDK 17 production-ready for Cassandra 5.0 |

**Gaps and confidence limits:**
- No independent benchmark of Nomad + Cassandra in production at 100+ nodes found. The Nomad-for-Cassandra operational gaps are derived from Nomad architectural documentation and Cassandra operational knowledge, not a published case study.
- Nomad Enterprise pricing and exact feature set for namespace-to-node-pool binding not independently verified by a non-HashiCorp source.
- K8ssandra multi-cluster operational pain points are based on operator community reports, not a controlled study.

---

## Components

### `cassandra`

Cassandra 5.x introduces Trie memtables and BTI SSTables (reduced JVM heap pressure via off-heap native memory), Unified Compaction Strategy with shard-parallel compaction (threads proportional to detected vCPUs), and Storage Attached Indexes (SAI) with vector search. These features amplify container misconfiguration risks: fractional CPU quotas cause UCS to spawn wrong shard counts; container memory limits must account for heap + off-heap together; JVM container support requires JDK 17 with `UseContainerSupport` (default). Cassandra requires stable node identity — each node owns a token range and a host UUID. Any orchestrator that reschedules a Cassandra node to a different host triggers bootstrapping, data streaming, and potential token range decommission.

### `nomad`

Nomad's scheduler is designed for stateless workloads. For Cassandra: (1) rescheduling must be explicitly disabled (`prevent_reschedule_on_lost = true`), negating its primary value; (2) container health checks cannot query gossip state — a "healthy" container may still be bootstrapping; (3) Nomad's kill timeout will abort `nodetool decommission` mid-stream, leaving a partial token migration; (4) namespace-to-node-pool hard binding requires Nomad Enterprise — OSS Nomad has no hard fence for multi-cluster isolation. At large scale (100+ nodes), Nomad OSS is useful as a **batch job runner** for operational tasks (drain, rolling restart, Reaper, Medusa jobs) where its durable job state and retry semantics add value. Cassandra still runs under systemd.

### `consul`

Consul's gossip-based service discovery adds minimal value over Cassandra's own gossip protocol for node-to-node communication. **Consul KV is genuinely useful** for dynamic seed inventory — seeds survive node replacement without config file edits. Consul Connect (service mesh) must not be used for Cassandra internode traffic (gossip, CQL, JMX) — the mTLS sidecar proxy adds latency and complexity with no meaningful security gain over network segmentation. Consul is the backend for Nomad and is retained in the large-cluster stack.

### `docker`

Docker on bare metal for Cassandra requires five conditions to be acceptable: `--network=host`, bind-mounted data directories (bypasses overlay2's 5–15% write amplification), host-level kernel tuning enforced before Docker starts, explicit ulimits in Nomad task config, and NUMA pinning on multi-socket hardware. Under these conditions, measured overhead is <2% throughput and <5% latency for sequential I/O — acceptable for compaction-heavy workloads. For P99 <5ms OLTP workloads, native process is required. Native systemd wins on every objective metric for bare metal and is the correct default.

### `ansible` + `systemd`

Ansible provides provisioning and day-N lifecycle management. Its push model has an enforcement gap: kernel tuning (THP, swappiness, I/O scheduler) drifts after kernel upgrades, and the push model only catches drift on the next playbook run. This gap is closed by packaging kernel tuning as a systemd dropin unit + udev rule (see host layer section). At 100+ nodes, Ansible + AWX adds audit trail, RBAC, and rolling restart state tracking. At 6 nodes, plain Ansible CLI is sufficient.

---

## Findings

### 1. OSS Nomad Cannot Safely Isolate Multiple Cassandra Clusters

Namespace-to-node-pool binding — the feature that enforces "this namespace's jobs only schedule into this node pool" — requires Nomad Enterprise. On OSS Nomad, a misconfigured job spec or Nomad-Pack template can place a Cassandra node in the wrong cluster's hardware tier. At 10+ clusters with different hardware profiles, this is an unacceptable blast radius. Evidence: [HashiCorp docs on namespace/datacenter/node_pool isolation](https://support.hashicorp.com/hc/en-us/articles/42285498597011). Confidence: **HIGH** (official source, directly stated).

### 2. Nomad Rolling Restarts Are Unsafe for Cassandra Without Gossip Awareness

Nomad serializes rolling updates via its `update` stanza but checks container health, not gossip state. A Cassandra node can be "healthy" (JVM running, port responding) while still bootstrapping and not yet in UN (Up/Normal) state. Advancing the rolling restart before gossip convergence creates data availability incidents. This is not fixable in OSS Nomad without external scripting that defeats the purpose of using an orchestrator. Evidence: Cassandra gossip protocol documentation; Nomad update stanza docs. Confidence: **HIGH**.

### 3. Docker Is Not the Default Best Practice on Bare Metal for Cassandra

Across independent benchmarks and Cassandra operational documentation, native systemd processes outperform Docker on bare metal on every metric when Docker's strict conditions are not enforced. The overlay2 filesystem adds 5–15% write amplification; bridge networking adds gossip latency; container memory limits interact poorly with JVM off-heap allocation. Under the five strict conditions, Docker overhead drops to <2%/<5% for sequential I/O. Evidence: [arXiv containerization benchmark](https://arxiv.org/pdf/1812.04362), [TechPlained 2026 benchmarks](https://www.techplained.com/docker-vs-bare-metal), [Cassandra production docs](https://cassandra.apache.org/doc/latest/cassandra/getting-started/production.html). Confidence: **HIGH**.

### 4. Cassandra 5.x Features Amplify Container Misconfiguration Risk

UCS shard parallelism uses thread counts derived from detected vCPUs. With `--cpu-shares` or `--cpuset-cpus` (rather than quota limits), the JVM sees all host CPUs and spawns too many compaction shards, causing I/O saturation silently. Trie memtable off-heap allocation is additive to JVM heap — container memory limits set equal to heap target leave no room for off-heap, OS page cache, or compaction buffers. Evidence: [Instaclustr UCS analysis](https://www.instaclustr.com/blog/apache-cassandra-5-0-improving-performance-with-unified-compaction-strategy/), [MCAC documentation](https://cassandra.apache.org/doc/latest/). Confidence: **MEDIUM** (vendor-adjacent primary source; supported by official UCS docs).

### 5. Repair Scheduling Requires Purpose-Built Tooling at Any Meaningful Scale

Full anti-entropy repair across a cluster with RF=3 takes days without segment-aware scheduling. Without Reaper, repairs overlap, I/O contention cascades, and repair never completes before the next hardware failure. This is not an orchestrator problem — Nomad, Kubernetes, and Ansible all have the same gap here. Reaper (TLP/Spotify) is the community standard. Evidence: Reaper project documentation; Cassandra repair documentation. Confidence: **HIGH**.

### 6. K8ssandra on Bare Metal Is Not Obviously Simpler Than Nomad

K8ssandra solves rack awareness (via `CassandraDatacenter` CRDs) and seed management better than Nomad. However, bare-metal Kubernetes requires etcd quorum management, control plane HA, CNI plugin lifecycle, and kubelet cgroup management — a second operations problem layered on top of Cassandra. K8ssandra's benefit materializes only when the team already operates Kubernetes for other workloads and can amortize the control plane. For a Cassandra-only environment starting from bare Ubuntu 22.04, K8ssandra is overengineering. Confidence: **MEDIUM** (architectural reasoning; no direct benchmark).

---

## Decision Matrix

| Profile | Recommendation | Eliminated options and why |
|---|---|---|
| Small clusters (6 nodes), Cassandra-only environment | Ansible + systemd + Consul KV + Reaper + Medusa | Nomad: 50% infra tax for zero benefit. Docker: native systemd is simpler and safer. K8ssandra: overengineering. |
| Large clusters (100+ nodes), Cassandra-only environment | Ansible + AWX + systemd + Consul KV + Nomad OSS (job runner only) + Reaper + Medusa | Docker: not default; native systemd preferred. Nomad as Cassandra scheduler: rejected. Nomad-Pack: no role. K8ssandra: adds K8s control plane without amortization. |
| Any cluster, team already runs Kubernetes | K8ssandra becomes viable | Amortizes K8s control plane. Evaluate K8ssandra-operator maturity for multi-cluster before committing. |
| Any cluster, Nomad Enterprise approved | Nomad + Docker viable with strict conditions | Node-pool isolation gap is closed. All 5 Docker conditions must be enforced in job spec. Not viable for P99 <5ms OLTP. |
| P99 <5ms OLTP SLA, any cluster size | Native systemd only | Remove Docker from consideration entirely regardless of licensing or conditions. |

---

## Host Layer: Non-Negotiable Configuration

### `cassandra-node-tuning` Package

Package as a `.deb` and distribute via existing apt pipeline. Installs two artifacts:

**`/lib/systemd/system/cassandra-tuning.service`**
```ini
[Unit]
Description=Cassandra host kernel tuning
DefaultDependencies=no
Before=cassandra.service
After=local-fs.target sysinit.target

[Service]
Type=oneshot
RemainAfterExit=yes
ExecStart=/usr/lib/cassandra-node-tuning/apply-tuning.sh

[Install]
WantedBy=multi-user.target
```

**`/usr/lib/cassandra-node-tuning/apply-tuning.sh`**
```bash
#!/bin/bash
set -euo pipefail
echo never > /sys/kernel/mm/transparent_hugepage/enabled
echo never > /sys/kernel/mm/transparent_hugepage/defrag
sysctl -w vm.swappiness=1
sysctl -w vm.max_map_count=1048575
sysctl -w vm.dirty_ratio=10
sysctl -w vm.dirty_background_ratio=5
sysctl -w kernel.numa_balancing=0
sysctl -w net.core.rmem_max=67108864
sysctl -w net.core.wmem_max=67108864
sysctl -w net.ipv4.tcp_rmem="4096 87380 67108864"
sysctl -w net.ipv4.tcp_wmem="4096 65536 67108864"
for dev in /sys/block/sd* /sys/block/nvme*n*; do
    [ -e "${dev}/queue/scheduler" ] || continue
    echo none > "${dev}/queue/scheduler" 2>/dev/null || \
    echo mq-deadline > "${dev}/queue/scheduler" 2>/dev/null || true
done
```

**`/etc/udev/rules.d/60-cassandra-io.rules`**
```
ACTION=="add|change", KERNEL=="nvme[0-9]*n[0-9]*", ATTR{queue/scheduler}="none"
ACTION=="add|change", KERNEL=="sd[a-z]*", ATTR{queue/scheduler}="mq-deadline"
```

Additional host requirements:
- Disable `unattended-upgrades` on all Cassandra nodes. Pin kernel version in `/etc/apt/preferences.d/cassandra-kernel`.
- In systemd `cassandra.service` unit: `LimitNOFILE=1048576`, `OOMScoreAdjust=-500`, `CPUAffinity=` set per NUMA layout.
- On multi-socket hardware: `numactl --cpunodebind=0 --membind=0` in `ExecStart`, or use Nomad `numa` block if Nomad manages the process.
- node-exporter textfile collector exposes THP and swappiness values. Alertmanager rule fires `KernelTuningDrift` on drift. No auto-remediation — humans in the loop.

---

## Minimum Viable Toolset

| Concern | Tool |
|---|---|
| Process supervision | systemd |
| Kernel tuning enforcement | `cassandra-node-tuning` deb (systemd unit + udev rules) |
| Provisioning + day-N | Ansible (+ AWX at 100+ nodes) |
| Seed inventory | Consul KV |
| Repair scheduling | Reaper (TLP/Spotify) |
| Backup | Medusa (TLP) |
| Metrics | Prometheus + MCAC + node-exporter |
| Operational job runner (100+ nodes only) | Nomad OSS |

---

## What This Does Not Cover

- Multi-DC replication across datacenters or WAN links
- Cloud deployments (AWS, GCP, Azure)
- Cassandra version upgrade paths (4.x → 5.x, SSTable format migration)
- Cassandra security hardening (TLS, JMX auth, RBAC, audit logging)
- Capacity planning and ring topology (vnodes vs. single-token, rack/DC layout)
- Nomad Enterprise feature evaluation

---

## References

- [Apache Cassandra Production Recommendations](https://cassandra.apache.org/doc/latest/cassandra/getting-started/production.html)
- [CASSANDRA-20681: JDK 17 production-ready for Cassandra 5.0](https://issues.apache.org/jira/browse/CASSANDRA-20681)
- [HashiCorp Nomad: Stateful Workloads](https://developer.hashicorp.com/nomad/docs/stateful-workloads)
- [HashiCorp Nomad: Namespace/datacenter/node_pool isolation](https://support.hashicorp.com/hc/en-us/articles/42285498597011)
- [HashiCorp Nomad: NUMA block](https://developer.hashicorp.com/nomad/docs/job-specification/numa)
- [Instaclustr: Cassandra 5.0 UCS analysis](https://www.instaclustr.com/blog/apache-cassandra-5-0-improving-performance-with-unified-compaction-strategy/)
- [AxonOps: OS tuning for Cassandra](https://axonops.com/docs/data-platforms/cassandra/operations/performance/os-tuning/)
- [arXiv: Dockerization impacts on database performance](https://arxiv.org/pdf/1812.04362)
- [TLP Reaper: Cassandra repair scheduling](http://cassandra-reaper.io/)
- [TLP Medusa: Cassandra backup](https://github.com/thelastpickle/cassandra-medusa)

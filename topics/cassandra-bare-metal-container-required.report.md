---
title: Cassandra 5.x Bare-Metal Ubuntu 22.04 — Container-Required Orchestration
date: 2026-05-26
status: complete
components: [cassandra, kubernetes, k8ssandra, k3s, cilium, openebs, helm, argocd]
constraints:
  - deployment: on-premise bare-metal
  - os: ubuntu-22.04
  - container: required
prior-topic: cassandra-bare-metal-nomad-stack
---

## Overview

The prior report ([cassandra-bare-metal-nomad-stack](cassandra-bare-metal-nomad-stack.report.md)) concluded that containers are not best practice for Cassandra 5.x on bare metal when the choice is optional. This report addresses the follow-up constraint: **containers are now mandatory**. The question becomes which orchestrator and toolchain deliver the safest, most operationally sound containerized Cassandra deployment across heterogeneous bare-metal clusters.

**Verdict: K8ssandra-operator on Kubernetes (k3s or kubeadm) is the only viable production choice once containers are required.** Nomad is eliminated — it lacks gossip-aware rolling restarts, decommission sequencing, and rack topology enforcement, none of which can be added without building a custom plugin. K8ssandra provides all four load-bearing capabilities natively and is tested at 1000 nodes as of v1.32.0 (April 2026).

---

## Evidence Quality

| Source | Tier | Notes |
|---|---|---|
| K8ssandra-operator GitHub (github.com/k8ssandra/k8ssandra-operator) | official | CRD spec, issue tracker, release notes |
| cass-operator GitHub issue #639 (capacity pre-flight deadlock) | official | Known decommission failure mode |
| Kubernetes PodDisruptionBudget docs (kubernetes.io) | official | PDB semantics, minAvailable |
| k3s docs (docs.k3s.io) | official | Embedded etcd, upgrade controller, footprint |
| kubeadm docs (kubernetes.io) | official | Upgrade steps, etcd static pod management |
| OpenEBS NDM docs (openebs.io) | official | BlockDevice CRs, StorageClass tagging |
| Cilium docs (docs.cilium.io) | official | kube-proxy replacement, hostNetwork interaction |
| MetalLB docs (metallb.universe.tf) | official | L2 vs BGP mode requirements |
| HashiCorp Nomad docs (developer.hashicorp.com) | official | Node pool isolation, Enterprise features |
| Portworx — Expert's Guide to Running Cassandra in Containers | vendor-adjacent | Portworx sells storage for containers; used only for corroboration |
| arXiv 1812.04362 — containerization impact on database performance | analyst | Peer-reviewed; cited for overhead baselines |

**Gaps and confidence limits:**
- No independent benchmark of K8ssandra at 100+ bare-metal nodes from a non-vendor source was found. The 1000-node tested claim is from the K8ssandra project itself (official tier, but not independently reproduced).
- cass-operator decommission capacity deadlock (issue #639) is a known open issue; mitigation guidance is from operator community reports, not a controlled study.
- k3s embedded etcd 3.5→3.6 version-jump hazard at v1.34+ is based on etcd upstream release notes and k3s issue tracker, not an independent post-mortem.

---

## Why Nomad Is Eliminated

- **No gossip awareness.** Nomad's rolling update stanza checks container health (TCP port), not gossip state. A node can be "healthy" to Nomad while still bootstrapping. There is no Cassandra plugin for Nomad; implementing correct sequencing requires external scripting that negates the orchestrator's value.
- **No decommission lifecycle hook.** Nomad's kill timeout will abort `nodetool decommission` mid-stream, leaving a partial token migration and a data availability incident. This is an architectural gap, not a configuration knob.
- **No rack topology enforcement.** Nomad has no equivalent of Kubernetes StatefulSet rack-to-node mapping. Rack affinity requires custom constraint expressions per job — brittle, not enforced at scheduling time, and not reconciled on node replacement.

---

## Why K8ssandra Wins

Four capabilities with no equivalent elsewhere:

1. **Gossip-aware rolling restarts.** cass-operator polls the management-api sidecar REST endpoint for actual gossip state (UN/UJ/DN) before advancing each rack. Not port-check health; actual gossip convergence.
2. **Rack-to-StatefulSet topology enforcement.** Each rack maps to a distinct StatefulSet. Pod scheduling is constrained to the correct rack's node labels at Kubernetes scheduling time — not at job-spec authorship time.
3. **Decommission sequencing.** cass-operator triggers `nodetool decommission` before pod termination and waits for completion. The known failure mode (capacity pre-flight deadlock, issue #639) has a defined mitigation: maintain 30–40% free-space headroom.
4. **First-class Reaper and Medusa integration.** Both are declared as two-line CRD fields in the K8ssandraCluster spec. No standalone deployment, no separate scheduling.

---

## Architecture: Small Clusters (6 Nodes)

| Tool | Role | Notes |
|---|---|---|
| k3s (server mode on 3 nodes) | Kubernetes control plane | Co-located on Cassandra nodes; embedded etcd |
| K8ssandra-operator | Cassandra lifecycle | cass-operator + management-api sidecar |
| containerd | Container runtime | Default in k3s; no Docker Engine on nodes |
| Cilium | CNI + kube-proxy replacement | eBPF; Cassandra pods use hostNetwork |
| local-path provisioner | Storage | Bundled with k3s; sufficient for homogeneous hardware |
| MetalLB (L2 mode) | Load balancer | Flat physical network, no BGP required |

**5 critical implementation decisions:**

1. **etcd isolation.** etcd data directory must reside on a dedicated NVMe partition, separate from Cassandra data. Cassandra compaction pushes fsync above etcd's 10ms p99 requirement on shared devices. Failure mode is etcd leader election instability under compaction load.
2. **Co-location model.** Run k3s server (control plane) on 3 of the 6 nodes alongside Cassandra pods. Do not pull nodes out of the Cassandra pool to create pure control plane nodes — at 6 nodes, that reduces the data plane to 4 nodes and thins RF=3 margin to zero slack.
3. **hostNetwork.** Cassandra pods must run with `hostNetwork: true`. CNI is bypassed for Cassandra traffic; Cilium benefits only control-plane and monitoring traffic. Required to avoid gossip and CQL latency added by CNI overlay.
4. **podAntiAffinity.** Mandatory: `podAntiAffinity` with `topologyKey: kubernetes.io/hostname`. Enforces one Cassandra pod per node. Without this, two pods on the same node under hostNetwork will conflict on the same CQL/gossip/JMX ports.
5. **Storage class.** Use k3s bundled local-path provisioner with bind-mounted host paths. No network storage in the Cassandra data path.

**Resource overhead to budget:** k3s server idle: ~768 MB RAM per node. management-api sidecar: ~256 MB per Cassandra pod. Plan accordingly on hardware with <32 GB RAM.

---

## Architecture: Large Clusters (100+ Nodes)

| Tool | Role | Notes |
|---|---|---|
| kubeadm or k3s (dedicated 3-node control plane) | Kubernetes control plane | Isolated from Cassandra data-plane nodes |
| K8ssandra-operator | Cassandra lifecycle | Same as small cluster |
| containerd | Container runtime | Default; no Docker Engine |
| Cilium | CNI + kube-proxy replacement | Same configuration |
| OpenEBS LocalPV Device + NDM | Storage | Required for heterogeneous hardware |
| MetalLB (L2 or BGP) | Load balancer | BGP required only if multi-L3-segment topology |
| ArgoCD or Flux | GitOps delivery | Sync waves for K8ssandraCluster CRDs; rate-limited auto-sync |

**Differences from small cluster:** At 100+ nodes, hardware heterogeneity (mixed NVMe/SATA generations, differing RAM) requires OpenEBS NDM instead of local-path provisioner. NDM DaemonSet auto-discovers block devices and creates BlockDevice CRs. Ansible tags devices at provisioning time (`nvme-fast`, `sata-mixed`), which map to distinct StorageClasses. The K8ssandraCluster CR targets the correct StorageClass per datacenter. Multiple Cassandra clusters can share a single Kubernetes data-plane cluster — no 1:1 mapping is required.

---

## Multi-Cluster Control Plane Model

**Architecture:** 1 dedicated Kubernetes cluster (3 nodes, embedded etcd HA) runs the K8ssandra-operator and ArgoCD/Flux control plane. N data-plane Kubernetes clusters host the actual Cassandra StatefulSets. Multiple K8ssandraCluster CRDs with different hardware profiles can share a single data-plane cluster.

**Network requirements:**
- Single L2 domain (flat physical network, one rack or one switch): MetalLB L2 mode is sufficient. No BGP, no Istio, no service mesh.
- Multi-L3-segment deployments (multiple racks with distinct subnets, routed fabric): MetalLB BGP mode or Calico BGP required for LoadBalancer IP advertisement across segments.

**Blast radius if control plane goes down:** Running Cassandra pods continue without interruption. Reconciliation and automation stop — no rolling restarts, no scale-out, no CRD-driven decommission. This is an acceptable blast radius for most Cassandra workloads. Human-driven `nodetool` operations remain available throughout.

---

## Kubernetes Distribution: k3s vs kubeadm

The deciding factor is **etcd upgrade safety, not deployment simplicity**. k3s's embedded etcd creates a version-jump hazard at v1.34+ (etcd 3.5 to 3.6), where a failed upgrade cannot be cleanly rolled back without etcd snapshot restore procedures that most Cassandra-focused teams have not practiced. kubeadm externalizes etcd as a separate static pod, making the upgrade step explicit and independently verifiable before proceeding. **Recommendation: use k3s for new deployments where the team has Kubernetes experience and a tested etcd backup/restore runbook; use kubeadm for teams whose primary expertise is Cassandra and who need predictable, inspectable upgrade steps.** If the team has not run an etcd restore drill, use kubeadm.

---

## Host Layer: Non-Negotiables

The host tuning layer from the prior report ([cassandra-bare-metal-nomad-stack](cassandra-bare-metal-nomad-stack.report.md), Host Layer section) applies unchanged: `cassandra-node-tuning` deb with systemd `cassandra-tuning.service` and udev `60-cassandra-io.rules`, disabled `unattended-upgrades`, pinned kernel version, and node-exporter for drift detection.

**Additions specific to Kubernetes deployment:**
- kubelet `--cpu-manager-policy=static` and `--topology-manager-policy=single-numa-node` for Cassandra pods on multi-socket hardware. Cassandra pods must be in the Guaranteed QoS class (requests == limits) for CPU Manager to pin them.
- containerd replaces Docker Engine entirely. Remove Docker daemon if previously installed.
- k3s install: `--flannel-backend=none --disable-network-policy` to allow Cilium to take over CNI.

---

## Storage Provisioning at Scale: OpenEBS LocalPV + NDM

**Recipe for heterogeneous 100+ node clusters:**

1. Deploy NDM DaemonSet on all Cassandra data-plane nodes. NDM auto-creates `BlockDevice` CRs for all discovered block devices.
2. Ansible playbook at provisioning time: label each node's devices with `openebs.io/storage-class: nvme-fast` or `openebs.io/storage-class: sata-mixed` based on physical device type detected.
3. Create two StorageClasses: `cassandra-nvme` and `cassandra-sata`, each referencing the appropriate NDM filter label.
4. In the K8ssandraCluster CRD, set `storageConfig.storageClassName` per datacenter to match hardware profile.
5. No network storage (Ceph, NFS, iSCSI) in the Cassandra data path. LocalPV Device is direct-attached only.

---

## Known Operational Risks

| Risk | Trigger | Mitigation |
|---|---|---|
| Decommission capacity deadlock | cass-operator pre-flight check fails if target node cannot stream data to peers (insufficient free space) | Maintain 30–40% free-space headroom per node; monitor with Prometheus alert on `cassandra_storage_load` |
| etcd upgrade hazard (k3s) | k3s v1.34+ embedded etcd 3.5→3.6 version jump | Run etcd snapshot backup before every k3s upgrade; test restore procedure quarterly |
| hostNetwork port conflict | Two Cassandra pods scheduled to same node | `podAntiAffinity` with `topologyKey: kubernetes.io/hostname` is mandatory; add admission webhook to enforce in CI |
| management-api sidecar memory | ~256 MB per pod; multiplies across dense nodes | Include in per-node memory budget; set container resource requests/limits explicitly |

---

## Decision Matrix: When to Revisit

| Condition | Action |
|---|---|
| P99 <5ms OLTP SLA discovered post-containerization | Re-evaluate: hostNetwork + local NVMe should keep overhead <5%; if not, revisit container requirement with stakeholders |
| Team has no Kubernetes experience and no budget for ramp-up | Use kubeadm with managed upstream (e.g., RKE2) to reduce operational surface |
| Nomad Enterprise already licensed and team is Nomad-first | Narrow carve-out: acceptable for small, non-critical Cassandra cluster only. All gossip-awareness gaps remain; manual scripting required. Not a recommendation. |
| Cassandra cluster grows beyond a single Kubernetes data-plane cluster | Split into multiple data-plane clusters; control plane topology unchanged |
| Hardware becomes homogeneous (full NVMe refresh) | Replace OpenEBS NDM with local-path provisioner to reduce operational complexity |

---

## What This Does Not Cover

- Multi-DC replication across datacenters or WAN links
- Cloud deployments (AWS, GCP, Azure) or hybrid bare-metal/cloud topologies
- Cassandra version upgrade paths (4.x → 5.x, SSTable format migration)
- Cassandra security hardening (TLS, JMX auth, RBAC, audit logging)
- Capacity planning and ring topology (vnodes vs. single-token, rack/DC layout)
- Kubernetes security hardening (RBAC, NetworkPolicy, Pod Security Standards)
- Disaster recovery across Kubernetes control-plane cluster failure

---

## References

- [K8ssandra-operator GitHub](https://github.com/k8ssandra/k8ssandra-operator)
- [cass-operator issue #639: decommission capacity pre-flight deadlock](https://github.com/k8ssandra/cass-operator/issues/639)
- [Kubernetes PodDisruptionBudget](https://kubernetes.io/docs/tasks/run-application/configure-pdb/)
- [k3s documentation](https://docs.k3s.io/)
- [kubeadm upgrade documentation](https://kubernetes.io/docs/tasks/administer-cluster/kubeadm/kubeadm-upgrade/)
- [OpenEBS NDM documentation](https://openebs.io/docs/concepts/ndm)
- [Cilium kube-proxy replacement](https://docs.cilium.io/en/stable/network/kubernetes/kubeproxy-free/)
- [MetalLB L2 and BGP modes](https://metallb.universe.tf/concepts/)
- [Prior report: Cassandra Bare-Metal Nomad Stack](cassandra-bare-metal-nomad-stack.report.md)

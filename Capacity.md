# Capacity — prime.kubernerdes.com

Supply (Harvester nodes) vs. demand (VMs for the downstream Kubernetes clusters). Primary purpose: size the SUSE Observability cluster correctly.

Hardware details live in [Hardware.md](./Hardware.md). CPU figures are hardware **threads** (what Harvester exposes), planned at 1:1 vCPU:thread — the conservative view (see [CPU](#cpu)).

**Measured** values come from the running 6-node cluster (currently the community Harvester, Harvester v1.8.2, `10.10.14.0/24`) on 2026-10-21 with all 9 VMs running.

---

## Supply — Harvester Nodes

| Node | Role | Model | Threads | RAM (GB) | OS disk — /dev/sda (GB) | Data disk — NVMe, Longhorn (GB) |
|:-----|:-----|:------|--------:|---------:|------------------------:|--------------------------------:|
| nuc-01 | control-plane | NUC10i7FNH | 12 | 64 | ~2,000 | ~1,000 |
| nuc-02 | control-plane | NUC10i7FNH | 12 | 64 | ~2,000 | ~1,000 |
| nuc-03 | control-plane | NUC10i7FNH | 12 | 64 | ~2,000 | ~1,000 |
| nuc-04 | worker | NUC13ANHi7 | 16 | 64 | ~1,000 | ~1,000 |
| nuc-05 | worker | NUC13ANHi7 | 16 | 64 | ~1,000 | ~1,000 |
| nuc-06 | worker | NUC13ANHi7 | 16 | 64 | ~1,000 | ~1,000 |
| **Total** | | | **84** | **384** | | **5,898** (Longhorn-reported) |

Longhorn uses only the NVMe data disk (`/var/lib/harvester/defaultdisk`, 983 GB each as reported by Longhorn). The OS disk is not used for VM storage.

### Reserved — Harvester System Overhead (measured)

Non-VM memory = node working set minus `virt-launcher` pods (Harvester, RKE2, Longhorn, monitoring, kernel/cache):

| Node | Non-VM memory (GB) | Non-VM CPU (cores) | k8s allocatable CPU |
|:-----|-------------------:|-------------------:|--------------------:|
| nuc-01 | 19.5 | 0.5 | 11 |
| nuc-02 | 9.9 | 0.3 | 11 |
| nuc-03 | 11.4 | 0.4 | 11 |
| nuc-04 | 5.7 | 0.2 | 15 |
| nuc-05 | 6.2 | 0.2 | 15 |
| nuc-06 | 7.8 | 0.1 | 15 |

Planning reserve (rounded up from measured):

| Node class | CPU (threads) | RAM (GB) | Total |
|:-----------|:-------------:|:--------:|:------|
| Gen10 control-plane (×3) | 2 | 20 | 6 T / 60 GB |
| Gen13 worker (×3) | 2 | 8 | 6 T / 24 GB |
| **Total** | | | **12 T / 84 GB** |

**Allocatable (after reserve): 72 threads / 300 GB**

---

## Demand — Kubernetes Cluster VMs

**Plan** (target for prime):

| Cluster | Distro | VMs | vCPU / VM | RAM (GB) / VM | Disk (GB) / VM | Total vCPU | Total RAM (GB) | Total Disk (GB) |
|:--------|:-------|:----|:---------:|:-------------:|:--------------:|-----------:|---------------:|----------------:|
| Rancher Manager | RKE2 | rancher-01/02/03 | 4 | 8 | 32 | 12 | 24 | 96 |
| Observability | RKE2 | observability-01/02/03 | 8 | 16 | 32 | 24 | 48 | 96 |
| Applications | K3s | apps-01/02/03 | 4 | 16 | 32 | 12 | 48 | 96 |
| **Total** | | **9** | | | | **48** | **120** | **288** |

VM disk: 32 GB root disk (SL-Micro 6.2 default image size).

**Currently deployed** (community cluster, for comparison):

| Cluster | vCPU / VM | RAM / VM | Disk / VM | Placement |
|:--------|:---------:|:--------:|:---------:|:----------|
| Rancher Manager | 4 | 8 GiB | 32 GiB | nuc-01, nuc-02, nuc-03 |
| Observability | 8 | 24 GiB | 100 GiB | nuc-04, nuc-05, nuc-06 |
| Applications | 4 | 12 GiB | 40 GiB | nuc-02, nuc-03, nuc-04 |

### Guest Utilization (measured, `kubectl top nodes` inside each cluster)

From `http://10.10.12.10/kubernerdes.php`, 2026-10-21. The applications cluster isn't on that page yet.

| Cluster | Node | CPU | Memory used | Memory % |
|:--------|:-----|----:|------------:|---------:|
| Rancher (8 GiB VMs) | rancher-01 | 362m | 5,651 Mi | 71% |
| | rancher-02 | 358m | 5,644 Mi | 71% |
| | rancher-03 | 330m | 5,438 Mi | 68% |
| Observability (24 GiB VMs) | observability-…-5spv4 | 545m | 13,591 Mi | 56% |
| | observability-…-wxdwc | 397m | 13,099 Mi | 54% |
| | observability-…-x5v88 | 455m | 11,157 Mi | 46% |

- **Rancher:** about 2.3 GiB free per VM. Rancher pods use 1.0–1.7 GiB and kube-apiserver about 1.5 GiB each. 8 GB is workable, but it's the tightest VM in the lab.
- **Observability:** about 37 GiB used across the cluster, with about 1.4 cores total. Today's install is non-HA (single replicas) and the agent runs only on the Rancher cluster (3 nodes). Expect usage to grow once all 12 default nodes report in. Scheduling is driven by pod **requests**: 27.2 GiB for 20-nonha, not current usage.

### Persistent Volumes (Harvester Longhorn via Harvester CSI)

| Consumer | Plan (GB) | Measured | Replicas | Notes |
|:---------|----------:|:---------|:--------:|:------|
| SUSE Observability PVCs | 375 (20-nonha) | 13 PVCs, 285 GiB provisioned, ~72 GB written | 3 | Depends on the selected `sizing.profile` |
| Harvester monitoring (`cattle-monitoring-system`) | 57 | Prometheus 50 + Alertmanager 5 + Grafana 2 GiB | 3 | Harvester built-in; easy to forget |
| Rancher Manager | — | — | | No PVCs by default |
| Applications (NeuVector, sample apps) | TBD | — | 3 | |

### Storage Demand (Longhorn)

VM root disks are created from the image's StorageClass, which has **1 replica**. PVCs use `harvester-longhorn` (default, **3 replicas**).

| Item | Logical (GB) | Replicas | Raw (GB) |
|:-----|-------------:|:--------:|---------:|
| VM root disks (9 × 32) | 288 | 1 | 288 |
| Observability PVCs (20-nonha) | 375 | 3 | 1,125 |
| Harvester monitoring PVCs | 57 | 3 | 171 |
| **Total (plan)** | **720** | | **1,584** |

| | Raw (GB) |
|:--|--:|
| Longhorn supply (6 × 983) | 5,898 |
| Plan demand | 1,584 (27%) |
| Measured scheduled today | 1,661 (28%) — larger VM disks than plan |
| Measured actually written today | 226 |

Longhorn settings (measured): over-provisioning 100% (scheduled size can't exceed disk size), minimal-available 25% (applies to actual written data), default replicas 3. Storage is comfortable.

---

## Headroom

### Cluster-wide

| | Threads | RAM (GB) |
|:--|--:|--:|
| Raw supply | 84 | 384 |
| − Harvester reserve | 12 | 84 |
| − VM demand (plan) | 48 | 120 |
| **Free** | **24** | **180** |

### Per node (intended placement)

Rancher VMs on the Gen10 control-plane nodes; Observability + Applications VMs on the Gen13 workers, one VM per cluster per node. Enforce with Harvester node affinity / anti-affinity, otherwise the scheduler decides (today, apps VMs landed on nuc-02/03/04).

| Node class | VMs per node | Demand / node | Allocatable / node | Free / node |
|:-----------|:-------------|:-------------:|:------------------:|:-----------:|
| Gen10 (nuc-01..03) | rancher-0x | 4T / 8 GB | 10T / 44 GB | 6T / 36 GB |
| Gen13 (nuc-04..06) | observability-0x + apps-0x | 12T / 32 GB | 14T / 56 GB | 2T / 24 GB |

### CPU

Harvester's CPU overcommit is 1000% (memory 150%), so VMs request only vCPU/10 from the scheduler. Measured whole-node CPU use with all 9 VMs running is 1.4–2.4 cores per node. CPU is tight only in the 1:1 planning view; in practice **memory is the binding constraint**.

### Node failure (N-1)

- Lose a Gen13 node (−14 T / −56 GB): **58 T / 244 GB** left vs. 48 / 120 demand.
- Lose a Gen10 node (−10 T / −44 GB): **62 T / 256 GB** left.

Capacity exists to restart VMs elsewhere, but VM root disks have 1 replica: a VM whose node fails stays down until that node returns. Each cluster survives on its other 2 nodes.

---

## SUSE Observability Sizing

### Counting observed nodes

SUSE counts in **default nodes** (4 vCPU / 16 GB). A larger node counts as a multiple, e.g. 12 vCPU / 48 GB = 3. The number in the profile name is default nodes, not machines. Below, each node counts as max(vCPU ÷ 4, RAM ÷ 16), rounded up.

| Observed | Nodes | Size | Default nodes each | Total |
|:---------|:-----:|:-----|:------------------:|------:|
| Rancher Manager | 3 | 4 vCPU / 8 GB | 1 | 3 |
| Observability (self) | 3 | 8 vCPU / 16 GB (plan) or 24 GiB (deployed) | 2 | 6 |
| Applications | 3 | 4 vCPU / 16 GB (plan) or 12 GiB (deployed) | 1 | 3 |
| **Downstream clusters** | **9** | | | **12** |
| *Optional:* Harvester hosts | 6 | 12–16 T / 64 GB | 4 | +24 → **36** |

- **10-nonha is too small:** 12 > 10.
- **20-nonha fits** if the agent goes on the Rancher, Observability, and Applications clusters only (PLAN.md Phase 6), with 8 default nodes of headroom (e.g. room for 2 more 8/16 VMs).
- **50-nonha** is needed if the agent also goes on Harvester, since each 64 GB host counts as 4.

### Profile requirements

Totals are pod requests across the cluster. SUSE recommends adding 20% for uneven pod distribution, control plane, and agents.

| Profile | Default nodes | CPU req / limit | Memory req / limit | +20% (CPU / Mem) | Storage¹ | Min node size |
|:--------|:-------------:|:---------------:|:------------------:|:----------------:|:--------:|:-------------:|
| 10-nonha | ≤ 10 | 7.2 / 16.2 | 23.4 / 24.0 GiB | 8.6 / 28.1 GiB | 335 GiB | 4 vCPU / 8 GB |
| **20-nonha** | **≤ 20** | **9.5 / 20.5** | **27.2 / 27.9 GiB** | **11.4 / 32.6 GiB** | **375 GiB** | 4 vCPU / 8 GB |
| 50-nonha | ≤ 50 | 14.2 / 29.9 | 32.6 / 32.7 GiB | 17.0 / 39.1 GiB | 475 GiB | 4 vCPU / 8 GB |
| 100-nonha | ≤ 100 | 24.1 / 49.5 | 45.7 / 45.8 GiB | 28.9 / 54.8 GiB | 525 GiB | 4 vCPU / 8 GB |
| 150-ha | ≤ 150 | 49.8 / 106.8 | 130.7 / 135.7 GiB | 59.8 / 156.8 GiB | 2,595 GiB | 8 vCPU / 16 GB, ×3 |

¹ Per the SUSE table, retention is 30 days. The page text also says 14 days for non-HA, so the docs disagree with themselves.

Non-HA profiles are "testing only" per SUSE. That's acceptable for this lab; HA starts at about 131 GiB of memory requests and doesn't fit.

### Fit check (allocatable ≈ VM RAM − ~3 GB for SL-Micro + RKE2 per VM)

| Observability VMs | Allocatable RAM | 20-nonha (32.6 GiB) | 50-nonha (39.1 GiB) |
|:------------------|:---------------:|:-------------------:|:-------------------:|
| 3 × 8 / 16 GB (plan) | ~39 GB | Fits, ~6 GB margin | **No**, at the limit |
| 3 × 8 / 24 GiB (deployed) | ~63 GB | Fits comfortably | Fits |

CPU isn't a concern: 24 vCPU vs. 11.4 (20-nonha) or 17 (50-nonha).

### Recommendation

**20-nonha** (agent on the 3 downstream clusters, not Harvester).

- 3 × 8 vCPU / 16 GB works but is tight. **3 × 8 / 24 GiB** (as deployed today) leaves room for 50-nonha later without resizing VMs, and still counts as 2 default nodes per VM. It costs +24 GB from the 180 GB free.
- Storage grows to **375 GiB** of PVCs (≈ 1,125 GiB raw at 3 replicas), above the 300 GB budget. Longhorn has room for it.

Source: [SUSE Observability requirements](https://documentation.suse.com/cloudnative/suse-observability/latest/en/setup/install-stackstate/requirements.html) (retrieved 2026-10-21)

---

## Open Items

- [ ] Application PVC requirements (NeuVector, sample apps)
- [ ] Confirm the SUSE Observability profile: 20-nonha recommended; 50-nonha if Harvester hosts are observed
- [ ] Raise the Observability PVC budget to 375 GiB (20-nonha)
- [ ] Reconcile plan vs. deployed sizes (O11y 16 vs. 24 GiB RAM, 32 vs. 100 GiB disk; apps 16 vs. 12 GiB RAM, 32 vs. 40 GiB disk)
- [ ] VM root disk replicas: accept 1 replica, or use a 3-replica image StorageClass
- [ ] Decide on VM placement rules (affinity / anti-affinity)
- [ ] Rancher VMs at ~70% memory with 8 GiB: consider 12–16 GiB
- [ ] Add the applications cluster to kubernerdes.php and capture its utilization

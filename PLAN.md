# PLAN — prime.kubernerdes.com

Roadmap for the **Prime MVP** milestone: deploy the SUSE portfolio on the 6-NUC lab using SUSE Prime software (subscription), pulled directly from the Prime registry.

See [README.md](./README.md) for goals and environment overview, and [Hardware.md](./Hardware.md) for the hardware inventory.

---

## Decisions Log

| Decision | Choice | Date | Rationale |
|:---------|:-------|:----:|:----------|
| IP scheme | README is authoritative: rancher `.30`, observability `.40`, apps `.50`/`.51`, harvester VIP `.100`, nodes `.101`–`.106` on `10.10.15.0/24` | 2026-10-19 | Simpler, role-aligned octets; Hardware.md (`.210/.220/.230`) is legacy |
| Image source | Pull directly from the SUSE Prime registry (internet) | 2026-10-19 | Prime subscription; no need for an airgap workflow. Hauler/Harbor deferred |
| Automation | Bash + `env.sh` for admin host and bootstrap; OpenTofu (`harvester` + `rancher2` providers) for VMs and downstream clusters | 2026-10-19 | Readable bootstrap, declarative infra; Ansible only if it clearly adds value |
| PKI | HashiCorp Vault PKI + cert-manager (Vault issuer) in the MVP | 2026-10-19 | Single trusted CA across all clusters from day one |
| IdM | **Open** | — | Candidates: Keycloak, FreeIPA, other |

### Open Items
- [ ] IdM selection (Keycloak / FreeIPA / other)
- [ ] VM node IPs within each role (e.g. rancher-01/02/03 → `.31`–`.33`? observability `.41`–`.43`?; apps nodes must avoid `.51`)
- [ ] Where Vault runs (proposed: VM on `nuc-00` so the CA exists before any cluster)
- [ ] Secret storage for SCC / Prime registry credentials and Vault tokens (never in git)
- [ ] Load balancing for VIPs (kube-vip vs. Harvester LB vs. HAProxy on `nuc-00-03`)

---

## Phase 0 — Repo Hygiene
- [ ] Reconcile Hardware.md with README (IP scheme, 6-node prime layout: 3× Gen10 control-plane, 3× Gen13 workers)
- [ ] README: fix title (`homelab.kubernerdes.com` → `prime.kubernerdes.com`), drop "free, as-in beer", replace Hauler/Harbor Day-1 steps
- [ ] Remove stray file `1` (old README copy)
- [x] Add `.gitignore` (swap files, tfstate, secrets)
- [ ] Define repo layout and fill in README "Repository Structure":
  ```
  Scripts/env.sh            # common vars; sources env.d/${ENVIRONMENT}.sh
  Scripts/env.d/prime.sh    # prime-specific vars (IP_PREFIX, MACs, domain)
  tofu/                     # OpenTofu: harvester + rancher2
  docs/                     # narrative docs (docs.prime.kubernerdes.com)
  ```

**Done when:** README and Hardware.md agree; layout exists.

## Phase 1 — Prerequisites
- [ ] SCC subscription active; Prime registry credentials obtained
- [ ] Credentials stored outside git (e.g. `~/.config/kubernerdes/prime.env`, sourced by `env.sh`)
- [ ] DNS records in `prime.kubernerdes.com` (nodes, VIPs, wildcard `*.apps.prime.kubernerdes.com` → `.51`)
- [ ] DHCP reservations for nuc-01..06 (MACs)
- [ ] Harvester ISO + SL-Micro image downloaded

**Done when:** all names resolve; credentials can `helm registry login` / `docker login` to the Prime registry.

## Phase 2 — Admin Host (`nuc-00`)
- [ ] Confirm homelab infra VMs (DNS, DHCP/TFTP, HAProxy) are healthy
- [ ] Install tooling: `kubectl`, `helm`, `tofu`, `vault`, `jq`, `yq`
- [ ] PXE/HTTP boot artifacts for Harvester (optional)

**Done when:** tooling installed; `Scripts/env.sh` sources cleanly.

## Phase 3 — PKI (Vault)
- [ ] Deploy Vault (VM on `nuc-00`, proposed)
- [ ] Root CA (offline-ish) + intermediate CA for `prime.kubernerdes.com`
- [ ] PKI role(s) and AppRole/Kubernetes auth for cert-manager
- [ ] Distribute CA trust bundle (nodes, workstation, Rancher `privateCA`)

**Done when:** a test cert issues from Vault and validates against the root.

## Phase 4 — Harvester (SUSE Virtualization)
- [ ] Install 3 control-plane nodes (nuc-01..03, Gen10), cluster VIP `.100`
- [ ] Join 3 worker nodes (nuc-04..06, Gen13)
- [ ] Networks: VM network on 1GbE onboard NIC; storage network on 2.5GbE USB NIC
- [ ] Upload SL-Micro image; create VM network(s)
- [ ] Harvester TLS from Vault CA

**Done when:** 6 nodes healthy; a test VM boots on the VM network.

## Phase 5 — Rancher Manager (Rancher Prime)
- [ ] OpenTofu: 3× SL-Micro VMs on Harvester (on control-plane nodes)
- [ ] RKE2 (Prime) cluster, VIP `.30`
- [ ] cert-manager + Vault `ClusterIssuer`
- [ ] Rancher Prime via Helm (`rancher.prime.kubernerdes.com`)
- [ ] Import Harvester into Rancher (Virtualization Management)

**Done when:** Rancher UI reachable with a Vault-issued cert; Harvester shows as managed.

## Phase 6 — Observability (SUSE Observability)
- [ ] Rancher-provisioned RKE2 cluster on Harvester, VIP `.40`
- [ ] SUSE Observability install (license key)
- [ ] Agents on Rancher, observability, and apps clusters

**Done when:** all clusters report into SUSE Observability.

## Phase 7 — Applications (SUSE Security)
- [ ] Rancher-provisioned K3s cluster, VIPs `.50` (API) / `.51` (apps ingress)
- [ ] NeuVector Prime
- [ ] Sample app on `*.apps.prime.kubernerdes.com`

**Done when:** NeuVector UI reachable; sample app served via wildcard with a Vault-issued cert.

## Phase 8 — Day 2
- [ ] Backups: rancher-backup operator, Vault snapshots, Harvester VM backups (NAS/NFS)
- [ ] Upgrade procedures (Harvester, RKE2/K3s, Rancher, apps)
- [ ] IdM integration with Rancher / Harvester / apps
- [ ] Optional: Harbor pull-through cache for Prime registry
- [ ] Narrative docs published to docs.prime.kubernerdes.com

# Hardware Inventory

Physical hardware, bill of materials, and network layout for the **prime** environment and the shared homelab infrastructure it depends on.

See [Capacity.md](./Capacity.md) for VM/cluster resource consumption and headroom.

## Systems

| System | Purpose | Environment | Model | CPU | Threads | RAM (GB) | OS disk — /dev/sda SSD (GB) | Data disk — NVMe (GB) |
|:-------|:--------|:------------|:------|:----|------:|:--------:|---------------:|----------------:|
| gateway | Firewall / router / default gateway | homelab | Sophos XG88 | — | — | — | — | — |
| nuc-00 | Admin Host | homelab | NUC13ANHi3 | i3-1315U | 8 | 32 | — | 512 |
| nas | NAS / NFS storage | homelab | ASUS X99-PRO/USB 3.1 | Xeon E5-2630 v3 | 16 | 94 | 1000 | — |
| nuc-01 | Harvester control-plane 1 | prime | NUC10i7FNH | i7-10710U | 12 | 64 | 1843 | 932 |
| nuc-02 | Harvester control-plane 2 | prime | NUC10i7FNH | i7-10710U | 12 | 64 | 1843 | 932 |
| nuc-03 | Harvester control-plane 3 | prime | NUC10i7FNH | i7-10710U | 12 | 64 | 1843 | 932 |
| nuc-04 | Harvester worker 1 | prime | NUC13ANHi7 | i7-1360P | 16 | 64 | 932 | 932 |
| nuc-05 | Harvester worker 2 | prime | NUC13ANHi7 | i7-1360P | 16 | 64 | 932 | 932 |
| nuc-06 | Harvester worker 3 | prime | NUC13ANHi7 | i7-1360P | 16 | 64 | 932 | 932 |

- Threads = hardware threads (what Harvester exposes as CPU). i3-1315U is 6C/8T, i7-10710U is 6C/12T, i7-1360P is 12C (4P+8E)/16T, Xeon E5-2630 v3 is 8C/16T.
- nas storage: 1 × 1TB SSD (OS), 3 × 4TB HDD (NAS pool, TrueNAS Scale); NFS share TBD.
- Harvester disks: /dev/sda SSD = OS disk; NVMe = data disk (Longhorn).
- Harvester NICs: 1GbE onboard = management/VM network; 2.5GbE USB = storage network.

## Power Consumption (Estimated)

Values are whole-system estimates (PSU losses not included). Gateway not included.

**Admin**

| System | CPU | Idle (W) | Average (W) | Max (W) |
|:-------|:----|:--------:|:-----------:|:-------:|
| nuc-00 | i3-1315U | 6 | 20 | 54 |
| nas | Xeon E5-2630 v3 | 80 | 120 | 200 |
| **Total** | | **86** | **140** | **254** |

**prime**

| System | CPU | Idle (W) | Average (W) | Max (W) |
|:-------|:----|:--------:|:-----------:|:-------:|
| nuc-01 | i7-10710U | 7 | 28 | 65 |
| nuc-02 | i7-10710U | 7 | 28 | 65 |
| nuc-03 | i7-10710U | 7 | 28 | 65 |
| nuc-04 | i7-1360P | 10 | 35 | 64 |
| nuc-05 | i7-1360P | 10 | 35 | 64 |
| nuc-06 | i7-1360P | 10 | 35 | 64 |
| **Total** | | **51** | **189** | **387** |

**Overall Summary**

Rate: $0.14/kWh — monthly cost = Watts × 730 hr × $0.14 / 1000

| Environment | Idle (W) | Idle $/mo | Average (W) | Avg $/mo | Max (W) | Max $/mo |
|:------------|:--------:|:---------:|:-----------:|:--------:|:-------:|:--------:|
| Admin | 86 | $8.79 | 140 | $14.30 | 254 | $25.96 |
| prime | 51 | $5.21 | 189 | $19.32 | 387 | $39.55 |
| **Grand Total** | **137** | **$14.00** | **329** | **$33.62** | **641** | **$65.51** |

## Naming

- Harvester nodes: **nuc-01/02/03** (Gen10, control-plane) and **nuc-04/05/06** (Gen13, workers).
- VMs: **rancher-01/02/03**, **observability-01/02/03**, **apps-01/02/03**.

## IP Assignments

The supernet is `10.10.12.0/22`. Each environment occupies one `/24`.

| Subnet | Environment | Notes |
|:-------|:------------|:------|
| 10.10.12.0/24 | homelab | Shared infrastructure (gateway, DNS, DHCP, admin, NAS) |
| 10.10.15.0/24 | prime | Harvester cluster and downstream clusters |

Infrastructure IPs — homelab (10.10.12.x), shared across all environments:

| IP | Hostname | Purpose |
|:---|:---------|:--------|
| 10.10.12.1 | gateway | Sophos XG88 — default gateway / firewall |
| 10.10.12.8 | nuc-00-01 | DNS primary + DHCP + TFTP (infra VM on nuc-00) |
| 10.10.12.9 | nuc-00-02 | DNS secondary (infra VM on nuc-00) |
| 10.10.12.10 | nuc-00 | Admin host (Apache + KVM) |
| 10.10.12.11 | nas | NAS / NFS storage (TrueNAS Scale) |
| 10.10.12.12 | librenms | Network monitoring (VM, optional) |
| 10.10.12.93 | nuc-00-03 | HAProxy load balancer (infra VM on nuc-00) |
| 10.10.12.193 | nuc-00-03-vip | HAProxy Keepalived VIP |

prime IPs (10.10.15.x) — see README "IP Assignments" (authoritative):

| Last Octet | Hostname | Purpose |
|:----------:|:---------|:--------|
| .30 | rancher | Rancher Manager cluster VIP |
| .40 | observability | Observability cluster VIP |
| .50 | apps | Applications cluster VIP (administrative / K8s API) |
| .51 | apps | Applications cluster VIP (user apps) |
| .100 | harvester | Harvester cluster VIP |
| .101-.106 | nuc-01..06 | Harvester nodes |
| TBD | rancher-0x, observability-0x, apps-0x | VM node IPs (open item in PLAN.md) |

Wildcard DNS: `*.apps.prime.kubernerdes.com` → `10.10.15.51`

## Bill of Materials

| Total | Unit Cost | Qty | Item |
|------:|----------:|:---:|:-----|
| $350 | $350 | 1 | Intel NUC NUC13ANHi3 (admin host, nuc-00) |
| $2,700 | $900 | 3 | Intel NUC NUC13ANHi7 (Harvester workers, nuc-04..06) |
| — | — | 3 | Intel NUC NUC10i7FNH (Harvester control-plane, nuc-01..03; previously owned) |
| — | — | 1 | Sophos XG88 firewall (previously owned) |
| $304 | $76 | 4 | Chicony A17-120P2A 20V 6A 120W PSU (5.5mm–2.5mm) |
| $36 | $12 | 3 | 1GB USB-C network adapter |
| $110 | $110 | 1 | Portable monitor (ViewSonic VA1655) |
| $20 | $10 | 2 | Power strip |
| $7 | $7 | 1 | Mouse |
| $20 | $10 | 20 | 28 AWG Cat6 cables (10-pack) |
| **$3,547** | | | **Estimated total (excludes previously owned items)** |

## Network Switch Layout

16-port unmanaged switch. Port assignments TBD — required connections:

| Host | Ports | Notes |
|:-----|:-----:|:------|
| gateway (uplink) | 1 | Sophos XG88 |
| nuc-00 | 1 | Admin host |
| nas | 1 | |
| nuc-01..06 (onboard 1GbE) | 6 | Management / VM network |
| nuc-01..06 (USB 2.5GbE) | 6 | Storage network |
| **Total** | **15** | 1 port spare |

## MAC Addresses

MAC addresses are set in `Scripts/env.d/prime.sh`.

| Host | MAC | Model |
|:-----|:----|:------|
| nuc-01 | 88:ae:dd:0b:90:70 | Gen10 |
| nuc-02 | 1c:69:7a:ab:23:50 | Gen10 |
| nuc-03 | 88:ae:dd:0b:af:9c | Gen10 |
| nuc-04 | 48:21:0b:65:ce:e5 | Gen13 (verify) |
| nuc-05 | 48:21:0b:65:c2:c7 | Gen13 (verify) |
| nuc-06 | 48:21:0b:5d:7a:e6 | Gen13 (verify) |

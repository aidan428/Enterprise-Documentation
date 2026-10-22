<img src="../assets/spaldotech-logo.png" alt="Spaldotech Operations" width="72">

> Context document. Converted from `Spaldotech_Operations_Homelab_Technical_Implementation_Guide.docx` (v1.0, September 2026); network sections updated to the as-built 10.0.0.0/16 design in v1.1. SPO-HL-SEC-001 §2 is the authoritative network inventory.

# Spaldotech Operations — Homelab Technical Implementation Guide

**Homelab Platform** · Proxmox VE · OPNsense · Segmented Network · DNS · NFS · Plex · CubeCoders AMP

| Field | Value |
| --- | --- |
| Classification | Internal (Confidential – Internal) |
| Internal domain | intraspald.tech |
| External domain | spaldotechoperations.co.uk |
| Version | 1.1 — 22 October 2026 |
| Owner | Spaldotech Operations |

## 1. Introduction & Scope

### 1.1 Purpose

This document is the authoritative technical implementation guide for the Spaldotech Operations homelab platform. It describes the target architecture, the security controls applied at each layer, and the step-by-step build sequence used to stand up the environment on the available hardware. It is intended to be followed during the initial build and used as an ongoing reference for maintenance, troubleshooting, and future expansion.

### 1.2 Scope

The guide covers the full stack from physical hardware through to externally published services:

- Proxmox VE as the sole hypervisor on a single physical host.
- OPNsense as the perimeter firewall, router, VPN endpoint, and IDS/IPS.
- Network segmentation using VLANs, with default-deny inter-VLAN policy.
- Split-horizon DNS serving the internal domain intraspald.tech via Pi-hole and Unbound.
- Externally published web services under spaldotechoperations.co.uk, fronted by a reverse proxy in a dedicated DMZ segment.
- NFS-based shared storage for internal clients.
- Plex media server.
- Game server hosting via CubeCoders AMP (Application Management Panel).
- Backup, monitoring, and an ongoing hardening/maintenance regime.

### 1.3 Design Principles

Five principles govern every design decision in this document:

- **Segmentation first.** No workload is placed on a shared network with a workload of a different trust level. Internet-facing services are always isolated from internal-only services at the network layer, not just by application-level controls.
- **Default-deny.** Firewall policy denies all traffic by default; every permitted flow is an explicit, documented rule with a stated business reason.
- **No exposed management planes.** Proxmox, OPNsense, and AMP administrative interfaces are never reachable directly from the internet. Remote administration is via VPN only.
- **Least privilege for data services.** NFS exports, DNS zone transfers, and internal APIs are scoped to the smallest possible set of source addresses.
- **Assume compromise of the DMZ.** Anything placed in the DMZ (reverse proxy, published web apps) is treated as potentially hostile territory; rules from the DMZ inward are as restrictive as rules from the WAN.

> **Note:** This guide assumes a single physical host with no hardware redundancy beyond disk-level protection you configure yourself. Section 11 (Backup & Disaster Recovery) is not optional reading — treat it as part of the initial build, not a later addition.

## 2. Hardware Platform

### 2.1 Baseline: HPE ProLiant DL380 Gen9

The platform is a single HPE DL380 Gen9 with 64GB RAM and 4TB of usable storage. Before any software is installed, the following firmware and out-of-band management steps should be completed:

- Update the server firmware (System ROM) and iLO firmware to current versions via HPE Support Portal before installing Proxmox, to avoid known virtualization and NIC compatibility issues on Gen9 hardware.
- Confirm the exact NIC configuration fitted (onboard 4x1GbE FlexibleLOM is common on Gen9; some units add a 10GbE or additional 1GbE mezzanine card). This dictates how many physical interfaces are available for the trunk/WAN split in Section 4.
- Confirm whether a Smart Array controller is fitted in HBA/passthrough mode or RAID mode — this determines whether ZFS (needs raw disk access) or hardware RAID + ext4/LVM (needs a RAID-managed volume) is the correct storage approach, see 2.2.
- Enable Intel VT-x and VT-d (or AMD-V/AMD-Vi) in BIOS — VT-d/IOMMU is mandatory for the WAN NIC PCIe passthrough used in Section 5.

#### 2.1.1 iLO hardening

iLO is out-of-band management and is a higher-value target than almost anything else on the box, since it has power control and virtual media access independent of the OS. Treat it as management-plane, not server-plane:

- Place iLO on the Management VLAN only (Section 4.2), never bridged to LAN or WAN.
- Change the default iLO administrator credentials immediately and disable any default/guest accounts.
- Disable iLO's own remote-support/telemetry "phone home" features unless specifically wanted.
- If iLO Advanced licensing is available, enforce a login timeout and enable two-factor authentication.

### 2.2 Storage layout

With 4TB of usable storage and (in the common Gen9 configuration) ECC RAM, ZFS is the recommended Proxmox storage backend: it gives checksummed data integrity, cheap space-efficient snapshots before every risky change, and native replication if a second box is ever added.

- If the Smart Array controller can be flashed/configured to HBA mode: pass disks through raw and let Proxmox manage a ZFS pool directly (mirrored vdevs if 2+ physical disks are available; a single-disk pool if only one disk is present, accepting no redundancy at the disk layer).
- If the controller cannot do HBA mode: build a hardware RAID volume (RAID1 minimum recommended) in the array controller and install Proxmox onto it with LVM-thin, since ZFS on top of a hardware RAID controller is not recommended.
- Reserve capacity: plan for roughly 20% of the pool to remain free at all times (ZFS performance and Proxmox Backup Server both degrade badly on a nearly-full pool).

Suggested logical carve-up of the 4TB (adjust to actual free space after the above):

| Volume / Dataset | Approx. size | Purpose |
| --- | --- | --- |
| local (ISOs, templates) | 50 GB | Installer ISOs, container templates, cloud-init images |
| vm-storage | 2.2 TB | VM/LXC disks — OPNsense, Pihole, NFS, Plex, AMP, web VMs |
| media (bind-mounted to Plex/NFS) | 1.2 TB | Media library and general NFS-shared data |
| pbs-datastore | 400 GB | Proxmox Backup Server datastore (Section 11) |
| free / headroom | ~150 GB | Snapshot churn, thin-provisioning overhead |

> **Note:** Because this is a single host with a single pool, the PBS datastore above provides recovery from VM/config-level failure (accidental deletion, bad update, ransomware) but not from physical disk or chassis failure. If the data matters, an off-box copy (external USB disk rotated offsite, or cloud sync of the PBS datastore) is strongly advised — see Section 11.3.

## 3. Proxmox VE Host Build

### 3.1 Installation

- Install Proxmox VE (current stable release) using the graphical installer, selecting ZFS (or LVM-thin if hardware RAID, per 2.2) as the target filesystem.
- During installation, assign the Proxmox management interface to the future Management VLAN address range (Section 4.3) rather than DHCP, so it survives later network changes cleanly.
- Set a strong root password and, immediately after first boot, create a named administrative user in the pve realm; avoid day-to-day login as root.
- Remove or disable the "no valid subscription" repository nag if desired, but switch the APT sources from the enterprise repo to the no-subscription repo so `apt update` succeeds without a subscription key.
- Apply all available updates (`apt update && apt full-upgrade`) and reboot before creating any VMs.

### 3.2 Network configuration on the host

The host itself should carry minimal network exposure. Configure Linux bridges as follows:

- **vmbr0** — trunk bridge carrying all internal VLANs (20/30/40/50/99, plus 60 once Monitoring is built), connected to the physical NIC that runs to your managed switch trunk port. VM/LXC virtual NICs attach here with a VLAN tag set per-guest (Proxmox supports VLAN-aware bridges natively — enable "VLAN aware" on vmbr0).
- **vmbr1** — dedicated to the WAN physical NIC, with no IP configured on the Proxmox host side at all. This interface is passed straight through to the OPNsense VM (Section 5.1); the host should not have a routable presence on the WAN segment.
- The Proxmox management IP itself lives only as a VLAN-tagged interface on vmbr0 within the Management VLAN — it must not be reachable from the Gaming, IoT or DMZ VLANs (enforced later in OPNsense firewall rules, Section 5.3).

### 3.3 Host hardening

- Restrict the Proxmox web GUI (port 8006) and SSH (port 22) to the Management VLAN via the built-in Proxmox Firewall (Datacenter → Firewall), in addition to the OPNsense inter-VLAN rules — defence in depth.
- Disable SSH password authentication once key-based access is confirmed working; keep root SSH login disabled.
- Enable two-factor authentication (TOTP) on the Proxmox web GUI for all administrative accounts.
- Install and enable `fail2ban` on the host for the SSH and Proxmox GUI login endpoints.
- Enable automatic security updates for the base Debian packages (unattended-upgrades), while leaving major Proxmox version upgrades manual and deliberate.
- Configure NTP against a reliable source — accurate time is required for TOTP 2FA, certificate validation, and useful log correlation.
- Enable the Proxmox host firewall globally with a default-drop input policy, explicitly allowing only Management-VLAN sourced traffic to the GUI/SSH/cluster ports.

### 3.4 Resource allocation plan

Indicative allocation of the 64GB host memory budget across all guests. Sizes are starting points; adjust once real workloads are observed via Proxmox's built-in monitoring.

| Guest | Type | vCPU | RAM | Notes |
| --- | --- | --- | --- | --- |
| OPNsense | VM | 2 | 4 GB | WAN NIC via PCIe passthrough; 2 vCPU is ample for home-scale throughput |
| dns01 / pihole-01 (Pi-hole + Unbound) | LXC | 1 | 1 GB | MGMT VLAN (10.0.99.10) |
| nfs01 | LXC or VM | 2 | 2–4 GB | VM if ZFS-on-guest is wanted, else LXC bind-mount is lighter |
| proxy01 (reverse proxy) | LXC | 1 | 1 GB | DMZ VLAN, the only thing WAN traffic ever reaches |
| plex01 | LXC | 2–4 | 4–8 GB | Add PCIe/iGPU passthrough if hardware transcoding is desired |
| amp01 (CubeCoders AMP controller) | VM | 4–8 | 16–24 GB | Sized to the busiest concurrent game server mix |
| web01 / web02 | VM | 2 each | 4–8 GB each | DMZ VLAN, backend of the reverse proxy |
| pbs01 (Proxmox Backup Server) | VM | 2 | 2 GB | Isolated backup target, see Section 11 |
| Headroom | — | — | ~8 GB | Snapshot memory, ZFS ARC, burst capacity |

## 4. Network Architecture

### 4.1 Topology overview

The design uses a single physical firewall (OPNsense, virtualised) as the sole boundary between the internet and every internal segment, with a VLAN-aware trunk carrying all internal traffic between OPNsense and the Proxmox virtual switch. No VM is ever placed directly on the WAN, and no traffic crosses between VLANs except through OPNsense's routing and firewall engine — Proxmox's virtual bridges do not route between VLANs themselves.

- Physical WAN NIC → passthrough → OPNsense WAN interface → ISP router/modem (bridge mode on the ISP device if possible, so OPNsense holds the public IP directly).
- Physical LAN NIC → trunk port on the managed switch → VLAN-aware vmbr0 → OPNsense LAN (trunk) interface, with sub-interfaces per VLAN configured inside OPNsense.
- All inter-VLAN and VLAN-to-WAN traffic is routed and filtered by OPNsense; the switch and Proxmox bridge only ever move Ethernet frames, they do not make policy decisions.

### 4.2 VLAN design

| VLAN ID | Name | Subnet | Purpose | Internet access |
| --- | --- | --- | --- | --- |
| — | LAN | 10.0.10.0/24 | Default lab LAN (untagged, OPNsense vtnet1) | Outbound |
| 99 | MGMT | 10.0.99.0/24 | OPNsense GUI, Pi-hole/Unbound, iLO, PBS | Outbound updates only, no inbound |
| 20 | STORAGE | 10.0.20.0/24 | NFS, Plex | Outbound only |
| 30 | GAMING | 10.0.30.0/24 | CubeCoders AMP and hosted game server instances | Outbound + specific inbound game ports only |
| 40 | DMZ | 10.0.40.0/24 | Reverse proxy, published web VMs | Inbound 443 and 8443 (proxy only), outbound restricted |
| 50 | IOT | 10.0.50.0/24 | IoT devices | Outbound only; DNS to Pi-hole, nothing else internal |
| 60 | MONITORING (planned) | 10.0.60.0/24 | SIEM (SPO-HL-SIEM-001) | Outbound 443 only |

Personal devices and Wi-Fi clients are not on a lab VLAN: they sit on the home network, 192.168.68.0/22, behind the ISP router, and reach the lab through the static route for 10.0.0.0/16 via OPNsense's WAN address (192.168.68.62). The Proxmox host is also on the home network, at 192.168.68.10. VLAN tags 20, 30 and 50 are inferred from their subnets; confirm them on OPNsense.

### 4.3 IP addressing plan

A consistent addressing convention keeps firewall rules and DNS records readable:

- `.1` — OPNsense interface (gateway) for every VLAN.
- `.2–.9` — reserved for core infrastructure per VLAN (DNS, NFS, proxy, PBS, iLO).
- `.10–.199` — DHCP scope for that VLAN, with static DHCP reservations for anything named in DNS.
- `.200–.254` — reserved for statically-configured hosts outside DHCP.

### 4.4 Physical cabling / NIC assignment

Confirm physical port mapping against the DL380 Gen9's NIC layout before wiring:

- Onboard NIC port 1 → dedicated WAN, passed through to OPNsense (Section 5.1).
- Onboard NIC port 2 → trunk to managed switch, carrying all VLANs to Proxmox vmbr0.
- iLO dedicated port (or shared with NIC1, if no dedicated port is fitted) → Management VLAN access port on the switch, isolated from the trunk.
- Any remaining onboard/mezzanine ports → left disabled at the switch until a specific need arises (a disabled port cannot be a pivot point).

> **Note:** If only two usable NICs are physically available, the WAN link can alternatively be carried as a tagged VLAN on the same physical trunk, with OPNsense's WAN interface bound to that VLAN sub-interface — but a physically separate WAN NIC via passthrough is preferred wherever available, since it keeps a hostile WAN packet stream off any interface the Proxmox host itself can see.

## 5. OPNsense Firewall

### 5.1 Deployment

- Create the OPNsense VM in Proxmox with 2 vCPU, 4GB RAM, and a small (8–16GB) virtual disk for the OS.
- Enable IOMMU on the Proxmox host (`intel_iommu=on` or `amd_iommu=on` in kernel parameters, plus `vfio` modules loaded) and pass the physical WAN NIC through to the OPNsense VM as a PCIe device, rather than bridging it in software.
- Add a second (virtual) NIC on the VLAN-aware vmbr0 trunk as the OPNsense LAN interface; configure VLAN sub-interfaces inside OPNsense (Interfaces → Assignments → VLANs) for each VLAN in 4.2, all riding this one virtual NIC.
- Install OPNsense from ISO, assign WAN to the passthrough NIC and LAN (trunk) to the virtual NIC, then complete VLAN and interface assignment from the console/GUI.
- Configure DHCP per internal VLAN (scope ranges per 4.3), with OPNsense itself as the DNS server handed out to clients (it will forward to Pi-hole, see Section 6).

### 5.2 Interface assignment summary

| Interface | Bound to | Role |
| --- | --- | --- |
| WAN | Physical NIC (PCIe passthrough) | Internet uplink, holds public IP |
| LAN (trunk) | Virtual NIC on vmbr0 | Parent interface for all VLAN sub-interfaces |
| vlanMGMT (VLAN 99) | Sub-interface of LAN | OPNsense GUI, Pi-hole, iLO, PBS |
| vlanStorage (VLAN 20) | Sub-interface of LAN | NFS, Plex |
| vlanGaming (VLAN 30) | Sub-interface of LAN | AMP and game server instances |
| vlanDMZ (VLAN 40) | Sub-interface of LAN | Reverse proxy, published web VMs |
| vlanIoT (VLAN 50) | Sub-interface of LAN | IoT devices |

> **As built:** the WAN (vtnet0) is a virtual NIC taking a DHCP address, 192.168.68.62/22, from the ISP router rather than holding the public IP; the ISP router port-forwards to it. The LAN (vtnet1, 10.0.10.1/24) is the untagged parent of the VLANs.

### 5.3 Firewall rule sets

Default policy on every interface is block-all inbound; the rules below are the explicit exceptions, evaluated top-down. Rules are grouped by source interface.

#### 5.3.1 WAN inbound

| # | Source | Destination | Port | Action | Reason |
| --- | --- | --- | --- | --- | --- |
| 1 | Cloudflare ranges | DMZ proxy (10.0.40.10) | 443/tcp | Allow | Published HTTPS for spaldotechoperations.co.uk |
| 2 | Any | DMZ proxy (10.0.40.10) | 8443/tcp | Allow | Plex via Caddy (SPO-HL-PRX-001) |
| 3 | Any | OPNsense WAN | 51820/udp | Allow | WireGuard remote administration endpoint |
| 4 | Any | Gaming VLAN specific host:port | per game | Allow (as needed) | Only the exact ports of currently-published game servers |
| 5 | Any | Any | Any | Block + log | Default deny, logged for review |

#### 5.3.2 Inter-VLAN policy matrix

Rows are source, columns are destination. Allow = specific ports only per 5.3.3, not a blanket allow.

| Source \ Dest | MGMT | STORAGE | GAMING | DMZ | IOT | HOME | WAN |
| --- | --- | --- | --- | --- | --- | --- | --- |
| MGMT | — | Allow (admin) | Allow (admin) | Allow (admin) | Allow (admin) | Deny | Allow |
| STORAGE | Deny | — | Deny | Deny | Deny | Deny | Allow |
| GAMING | DNS only | Deny | — | Deny | Deny | Deny | Allow |
| DMZ | Deny | Plex only (proxy → plex01:32400) | Deny | — | Deny | Deny | Allow (restricted) |
| IOT | DNS only | Deny | Deny | Deny | — | Deny | Allow |
| HOME | Allow (admin, DNS) | Allow (NFS/Plex) | Allow (game client ports) | Allow (443) | Deny | — | n/a |

HOME is 192.168.68.0/22, arriving on the WAN interface through the ISP router's static route. "DNS only" means TCP/UDP 53 to Pi-hole at 10.0.99.10.

#### 5.3.3 Notable specific rules

- **STORAGE → anywhere:** no rule permits Storage-VLAN hosts to initiate connections into MGMT, GAMING, DMZ or IOT — NFS and Plex only ever respond to inbound requests from permitted sources.
- **HOME → STORAGE:** allow only 2049/tcp (NFS) from specific known client IPs, and Plex's discovery/streaming ports — not an open allow to the whole subnet.
- **DMZ → MGMT/STORAGE/GAMING/IOT:** denied, except the single proxy-to-Plex rule above. If a DMZ host is compromised, this rule is what stops the compromise pivoting inward.
- **GAMING → WAN:** allowed outbound (game servers need to reach master server lists, updates, Steam/matchmaking services, etc.), but GAMING → any other internal VLAN is denied apart from DNS.
- **IOT:** treated as untrusted; DNS to Pi-hole and outbound Internet only.
- **MGMT → everywhere:** MGMT is the administrator's own launch point, so it is permitted to reach admin ports (SSH/HTTPS management UIs) on every other VLAN — but nothing is permitted to reach MGMT except from MGMT itself, approved home admin sources, or the WireGuard tunnel in 5.5. Every VLAN may reach Pi-hole on port 53.

### 5.4 Intrusion detection / prevention (Suricata)

- Enable Suricata in IPS (blocking) mode on the WAN interface, using the ET Open ruleset at minimum, updated automatically on a daily schedule.
- Enable Suricata in IDS (alerting) mode on the DMZ and Gaming interfaces — IPS mode here risks false-positive blocking of legitimate game traffic, but visibility into exactly what's hitting these two exposed segments is valuable.
- Route Suricata alerts to a log destination reviewed regularly (Section 12) rather than left to accumulate unread.

### 5.5 Remote access — WireGuard

All remote administration (Proxmox GUI, OPNsense GUI itself, AMP admin panel, SSH) is reached exclusively through a WireGuard tunnel terminating on OPNsense, never via direct WAN port-forwards.

- Enable the WireGuard plugin on OPNsense, create a server instance bound to a dedicated tunnel subnet (e.g. 10.0.100.0/24, which no VLAN uses), listening on a single UDP port forwarded from WAN (rule #3 in 5.3.1).
- Issue a unique key pair and peer configuration per administrator device — never share one WireGuard identity across devices.
- Add a firewall rule on the WireGuard interface permitting tunnel clients into MGMT (and, per admin need, Storage/Gaming/DMZ admin ports) — this is the only path by which an external device reaches MGMT.
- Where the client device supports it, enable a kill-switch/always-on tunnel policy for administrator laptops so accidental unencrypted admin sessions cannot happen while off the home network.

## 6. DNS Architecture

### 6.1 Split-horizon design

Two distinct DNS name spaces are served by this platform, and they must never be resolvable through each other's path:

- **intraspald.tech** — the internal-only domain. Resolved exclusively by the internal Pi-hole/Unbound resolver for clients on the home network and every lab VLAN. There is no public DNS zone for this domain, and it must not be delegated or published externally.
- **spaldotechoperations.co.uk** — the external domain, published in public DNS at a registrar/DNS provider, pointing only at the reverse proxy's public presence (Section 9).

### 6.2 Pi-hole + Unbound deployment

- Deploy an LXC container (dns01, as built pihole-01 at 10.0.99.10) on the MGMT VLAN running Pi-hole for DNS filtering/blocklists and DHCP-handed-out resolution.
- Install Unbound alongside Pi-hole on the same container, configured as a local recursive resolver querying the DNS root hints directly — this avoids dependency on any third-party upstream resolver for basic functioning, and pairs well with Pi-hole's "Custom 2" upstream slot pointing at `127.0.0.1#5335`.
- Set OPNsense's DHCP options to hand out dns01's address (10.0.99.10) as the DNS server for every internal VLAN, and configure OPNsense's own DNS resolver/forwarder to relay to dns01 as well, so OPNsense-originated lookups (e.g. dynamic DNS, package updates) are also filtered and logged centrally.
- Deploy a second Pi-hole instance (or at minimum Unbound-only secondary) if resilience against a single dns01 outage is wanted; Pi-hole supports Gravity sync between two instances.

### 6.3 intraspald.tech internal zone

Using Pi-hole's Local DNS Records (or a dedicated Unbound local-zone file for more complex records), create A records for every internally-addressed service, for example:

| Hostname | Points to | Service |
| --- | --- | --- |
| proxmox.intraspald.tech | 192.168.68.10 | Proxmox web GUI |
| opnsense.intraspald.tech | 10.0.99.1 | OPNsense web GUI |
| pihole.intraspald.tech | 10.0.99.10 | Pi-hole admin UI |
| nfs.intraspald.tech | 10.0.20.10 (planned) | NFS server |
| plex.intraspald.tech | 10.0.20.11 (planned) | Plex media server |
| amp.intraspald.tech | 10.0.30.10 (planned) | CubeCoders AMP admin panel |
| pbs.intraspald.tech | 10.0.99.20 (planned) | Proxmox Backup Server |

> **Note:** Admin-panel hostnames (proxmox, opnsense, amp, pbs) resolve internally but must only actually be reachable per the firewall matrix in 5.3.2 — DNS resolution and network reachability are separate controls and both are required.

### 6.4 spaldotechoperations.co.uk external DNS

- Manage authoritative DNS for the domain at the registrar or a dedicated DNS provider (e.g. Cloudflare), not on OPNsense — OPNsense has no public-facing DNS role in this design.
- Publish only the records that correspond to genuinely externally-available services (e.g. an A/AAAA or CNAME for the public-facing website, pointing at the home WAN IP or a Cloudflare Tunnel endpoint per Section 9.4).
- If using dynamic WAN IP (no static IP from the ISP), configure a Dynamic DNS updater on OPNsense against the DNS provider's API, or avoid the issue entirely by using a Cloudflare Tunnel, which does not require a stable inbound IP at all.
- Enable DNSSEC at the registrar/DNS provider for the external zone if supported, and enable CAA records restricting which certificate authorities may issue for the domain.

## 7. Storage Services — NFS

### 7.1 Deployment

- Deploy nfs01 on the Storage VLAN as an LXC container (or a small VM if ZFS-on-guest is preferred for its own snapshot/replication features) with its data disk backed by the media dataset described in 2.2.
- Install `nfs-kernel-server` (Debian/Ubuntu base) and configure NFSv4 only — disable NFSv3 support to avoid its weaker RPC-based access control model.
- Use a single NFSv4 pseudo-root export with sub-exports/binds for logical shares (media, backups, general), which keeps client mount syntax simple and centralises export options.

### 7.2 Export design & access control

- Every export line in `/etc/exports` names explicit client IPs or a tightly-scoped subnet — never `*` and never a whole /24 unless every host in that /24 is genuinely meant to have access.
- Use `rw` only for the specific clients that need write access (e.g. the Plex server for library management, an admin workstation); default everything else to `ro`.
- Apply `root_squash` (the NFS default) on every export — never `no_root_squash` — so a root user on a client cannot act as root on the NFS server's filesystem.
- Restrict the NFS ports (2049/tcp, plus rpcbind if NFSv3 compatibility is ever required) at the OPNsense firewall to only the Storage VLAN and approved home clients, per the matrix in 5.3.2 — Gaming, IoT and DMZ have no firewall path to NFS at all, regardless of export file contents.

### 7.3 Client mount hardening

- Mount with `nosuid,nodev` on clients to prevent a share from being used to introduce setuid binaries or device nodes.
- Where a client only needs to read media (e.g. Plex, a smart TV), mount that specific share read-only at the client end as well as server end — defence in depth against a misconfigured export.
- Avoid mounting NFS shares onto the Gaming-VLAN AMP VM; if game data needs to be shared, prefer AMP's own controlled file-management features or a dedicated, narrowly-scoped export rather than a general-purpose share.

## 8. Media — Plex

### 8.1 Deployment

- Deploy plex01 as an LXC container on the Storage VLAN, with the media library mounted from the NFS/media dataset (bind-mount is preferable to an NFS client mount inside the container, for performance and simplicity, since both live on the same host).
- Claim the server to the Plex account during initial setup, then immediately review Plex's own network settings to disable unnecessary remote-access convenience features that aren't required (see 8.3).

### 8.2 Hardware transcoding

- If the DL380 Gen9 has no discrete GPU, hardware transcoding is unavailable in this generation of server (no useful onboard GPU) — plan for software transcoding CPU headroom (allow 2–4 cores per simultaneous transcode) or add a low-power discrete GPU/Quick Sync capable card with PCIe passthrough to the Plex LXC/VM if transcoding load is expected to be significant.

### 8.3 Exposure considerations

- Do not port-forward Plex's default port (32400) directly from WAN. If remote streaming access is wanted, either route it through the reverse proxy in the DMZ (Plex supports being reverse-proxied) or rely on Plex Relay/Plex's own remote-access negotiation, understanding that this shifts some trust to Plex's cloud infrastructure.
- If reverse-proxying Plex, treat it exactly like any other DMZ-published service in Section 9 — the Plex backend itself still lives on the Storage VLAN and only the proxy's narrow allowed path reaches it (this requires a specific, tightly-scoped DMZ→STORAGE rule limited to the Plex port and proxy source IP only, as an exception to the general DMZ→STORAGE deny in 5.3.2).
- Keep Plex sharing ("Managed users"/"Home") restricted to known individuals; disable public library sharing features.

## 9. External Web Services (spaldotechoperations.co.uk)

### 9.1 DMZ design

The DMZ (VLAN 40) is the only segment that ever receives unsolicited inbound traffic from the internet, and it is architected on the assumption that anything in it may eventually be compromised. Nothing of internal value is stored there.

- Only two classes of host live in the DMZ: the reverse proxy (proxy01) and the backend web application VMs it fronts.
- Backend web VMs accept traffic only from proxy01's address, on the application port — never directly from WAN, and never from any other internal VLAN.
- DMZ hosts have no route to MGMT, STORAGE, GAMING or IOT under any circumstance (5.3.2).

### 9.2 Reverse proxy deployment

- Deploy proxy01 as an LXC container in the DMZ running Caddy, Nginx Proxy Manager, or Traefik (Caddy is a good default for its automatic HTTPS handling).
- Configure a virtual host block per published site under spaldotechoperations.co.uk, each proxying to its specific backend web VM's internal DMZ address and port.
- Add security headers at the proxy layer (HSTS, X-Content-Type-Options, X-Frame-Options/CSP as appropriate to each site) so this policy is enforced centrally rather than per-backend.
- Enable access logging on the proxy and forward it to the logging destination in Section 12.

### 9.3 TLS/certificate strategy

- Use Let's Encrypt via the DNS-01 challenge type against the DNS provider's API (rather than HTTP-01) — this avoids needing port 80 open at all for certificate issuance/renewal, and works even before a site is fully live.
- Enforce TLS 1.2 minimum (prefer TLS 1.3 where all expected clients support it) and disable legacy cipher suites at the proxy.
- Automate renewal (Caddy and most modern proxies do this natively) and alert if a certificate is within 14 days of expiry and hasn't renewed.

### 9.4 Cloudflare Tunnel alternative

For a home internet connection, consider running a Cloudflare Tunnel (or equivalent outbound-only tunnel) from proxy01 instead of, or in addition to, a WAN port-forward:

- The tunnel daemon on proxy01 makes an outbound-only connection to Cloudflare's edge, meaning no inbound port needs to be opened on OPNsense at all for web traffic — this removes an entire class of exposure (port scanning, direct exploitation attempts against whatever is listening on 443).
- DNS for spaldotechoperations.co.uk points at the Cloudflare-provided tunnel endpoint rather than the home WAN IP, which also hides the residential IP address from public DNS records.
- This does shift a meaningful amount of trust to Cloudflare as an intermediary — acceptable for a homelab, but worth stating explicitly as a design trade-off rather than leaving it implicit.

## 10. Game Server Hosting — CubeCoders AMP

### 10.1 Deployment

- Deploy amp01 as a VM (not LXC — AMP and many game servers expect full kernel/networking capabilities that are easier to guarantee in a VM) on the Gaming VLAN, sized per the busiest expected concurrent instance mix (Section 3.4).
- Install the AMP controller (ADS — Application Deployment Service) following CubeCoders' official installation instructions for the chosen Linux distribution.
- Bind the AMP web admin panel to the Gaming-VLAN internal address only; it is reached by administrators via the WireGuard tunnel plus the MGMT→GAMING admin-port allow rule (5.3.2), never from WAN directly.

### 10.2 Network exposure

- Each hosted game server's specific client-facing port(s) are the only things ever forwarded from WAN, on a per-instance, per-port basis, targeted at amp01's Gaming-VLAN address (rule #4 in 5.3.1) — add and remove these forwards as instances are created and retired, rather than leaving a wide port range permanently open.
- Where a game supports it, prefer AMP's own per-instance firewall/allow-list features as an additional layer above the OPNsense rule.
- Do not forward AMP's own management port (typically 8080, or as configured) from WAN under any circumstance — only game client ports are ever WAN-facing.

### 10.3 Per-instance isolation

- Run each game server under its own dedicated OS user/AMP instance user, which AMP does by default — avoid running all instances as a single shared account, since a compromised game server binary should not be able to read or modify a sibling instance's files.
- Keep Gaming-VLAN outbound rules limited to what's genuinely required (HTTP/HTTPS for updates and master-server communication, plus specific game-protocol UDP/TCP as needed) rather than a blanket allow-any-outbound, to reduce the value of the VLAN as an exfiltration or further-attack launch point if a game server is ever compromised via a malicious plugin/mod.
- Apply OS and AMP updates to amp01 on a regular cadence — game server software (and mod/plugin ecosystems in particular) is a common vector for compromise, given how frequently third-party, less-audited code is installed by design.

## 11. Backup & Disaster Recovery

### 11.1 Proxmox Backup Server

- Deploy pbs01 as a small VM with its datastore on the pbs-datastore volume described in 2.2 (ideally, if a second physical disk becomes available, moved to genuinely separate spindles from vm-storage so a single disk failure cannot take out both production data and its backups).
- Register pbs01 as a backup target in Proxmox's Datacenter → Storage configuration, and configure scheduled backup jobs per guest (or all guests) rather than relying on manual/ad-hoc backups.
- Enable PBS's built-in encryption for the backup datastore, with the encryption key stored securely outside the host itself (e.g. a password manager) — a backup that is trivially readable by anyone with disk access is a smaller step up from no backup at all than it appears.

### 11.2 Backup schedule

| Guest / group | Frequency | Retention |
| --- | --- | --- |
| OPNsense (config export) | Weekly + before any change | 12 weekly |
| MGMT/STORAGE guests (dns01, nfs01's config, pbs01 itself) | Daily | 7 daily, 4 weekly |
| DMZ guests (proxy01, web VMs) | Daily | 7 daily, 4 weekly |
| amp01 | Daily | 7 daily, 4 weekly |
| plex01 (config/database, not media) | Daily | 7 daily, 4 weekly |
| Media dataset (bulk storage) | Not backed up via PBS — see note | — |

> **Note:** Bulk media is typically large, re-acquirable, and not worth the backup capacity it would consume; back up the Plex database (watch history, metadata) rather than the media files themselves, and treat the media library as best-effort/non-critical data.

### 11.3 Restore testing and off-box copies

- Perform a full test restore of at least one VM from PBS every quarter, into a temporary/throwaway VM ID, to confirm backups are actually usable — an untested backup is a hypothesis, not a control.
- Periodically copy the PBS datastore (or its most critical subset) to an external USB disk stored off-site, or sync it to a cloud object storage target, since this is a single-host build with no second physical location by default — a fire, theft, or major hardware failure otherwise takes out production and backups together.
- Keep an out-of-band (non-digital or separately-stored) copy of the OPNsense configuration, WireGuard keys, and the master credentials list, since these are needed to rebuild access to everything else if the host is lost entirely.

## 12. Monitoring & Logging

### 12.1 What to monitor

- **Proxmox host:** CPU/RAM/disk/ZFS pool health via the built-in dashboard, plus disk SMART status.
- **OPNsense:** WAN throughput, firewall block/allow counts, Suricata alerts, VPN connection events.
- **Pi-hole:** query volume and top blocked/allowed domains, as a general internal-network health indicator.
- **Reverse proxy and web VMs:** access logs, error rates, TLS certificate expiry.
- **AMP:** instance uptime, resource consumption per game server.

### 12.2 Centralising logs

- Ship OPNsense firewall/Suricata logs, proxy access logs, and Proxmox host auth logs to a small centralised log collector (e.g. a lightweight Syslog/Loki+Grafana stack) on the Monitoring VLAN (60, see SPO-HL-SIEM-001), rather than relying on reading each system's local logs independently after the fact.
- Set a log retention period that balances disk usage against the ability to investigate an incident discovered some time after it began (30–90 days is a reasonable starting point for firewall/proxy logs).
- Configure basic alerting (even simple email/webhook notifications) for: repeated failed logins to Proxmox/OPNsense/AMP, Suricata high-severity alerts, certificate expiry warnings, and backup job failures — the goal is to hear about problems from the monitoring stack, not from the service failing.

## 13. Security Baseline Summary

A consolidated checklist drawn from every section above, useful as a build-completion sign-off and as a periodic (quarterly) re-audit.

### Network

- [ ] Default-deny inter-VLAN policy in place and reviewed against the matrix in 5.3.2.
- [ ] WAN inbound limited to: published HTTPS/HTTP to the proxy, WireGuard, and only currently-active game server ports.
- [ ] No management interface (Proxmox, OPNsense, AMP, iLO) reachable from WAN or from DMZ/GAMING/IOT.

### Identity & access

- [ ] 2FA enabled on Proxmox, OPNsense, and AMP administrative accounts.
- [ ] SSH key-only authentication; root SSH login disabled.
- [ ] Unique WireGuard peer identity per administrator device.
- [ ] No shared/default credentials remain on iLO, OPNsense, Proxmox, Pi-hole, or AMP.

### Data

- [ ] NFS exports scoped to explicit client IPs, `root_squash` enabled, NFSv3 disabled.
- [ ] PBS backups encrypted, scheduled, and restore-tested quarterly.
- [ ] At least one off-box copy of critical backups exists.

### Perimeter

- [ ] Suricata IPS active on WAN; IDS active on DMZ and GAMING.
- [ ] TLS 1.2+ enforced on all published web services; certificates auto-renewing via DNS-01.
- [ ] fail2ban (or equivalent) active on internet-facing/administrative login endpoints.

### Operational

- [ ] Firmware (server, iLO) and all guest OS/application updates on a regular patch cadence.
- [ ] Centralised logging in place for firewall, proxy, and authentication events, with alerting on key failure conditions.
- [ ] This document reviewed and updated whenever the architecture changes (see change log, Appendix D).

## 14. Deployment Runbook

Recommended build order. Each phase should be functionally verified before moving to the next — in particular, do not connect the WAN NIC until Phase 2 is complete and default-deny is confirmed active.

### Phase 1 — Host foundation

1. Update DL380 Gen9 firmware and iLO; harden iLO credentials and network placement (2.1.1).
2. Install Proxmox VE, configure ZFS/storage layout (3.1, 2.2).
3. Configure vmbr0 (VLAN-aware trunk) and vmbr1 (WAN passthrough prep); apply host firewall and hardening baseline (3.2, 3.3).

### Phase 2 — Perimeter

1. Deploy OPNsense VM with WAN passthrough; assign VLAN sub-interfaces (5.1, 5.2).
2. Build the full firewall rule set with default-deny before connecting WAN (5.3).
3. Enable Suricata; configure and test WireGuard remote access from an external network (5.4, 5.5).
4. Only now connect the physical WAN uplink.

### Phase 3 — Core internal services

1. Deploy dns01 (Pi-hole + Unbound); point OPNsense DHCP at it; build the intraspald.tech zone (Section 6).
2. Deploy nfs01 with scoped exports (Section 7).
3. Deploy plex01 (Section 8).

### Phase 4 — External services

1. Deploy proxy01 in the DMZ; obtain certificates via DNS-01 (9.2, 9.3).
2. Deploy first backend web VM; verify it is unreachable except via the proxy.
3. Point spaldotechoperations.co.uk DNS at the proxy (directly or via Cloudflare Tunnel) and confirm end-to-end HTTPS (9.1–9.4).

### Phase 5 — Game hosting

1. Deploy amp01 on the Gaming VLAN (10.1).
2. Stand up the first game server instance; open only its specific port(s) on WAN (10.2).

### Phase 6 — Resilience

1. Deploy pbs01; configure and run first backups for every guest (11.1, 11.2).
2. Perform a test restore; establish an off-box backup copy (11.3).
3. Stand up centralised logging and alerting (Section 12).
4. Walk the checklist in Section 13 and sign off the build.

## Appendices

### Appendix A — VLAN & IP Quick Reference

| VLAN | Subnet | Gateway | Key hosts |
| --- | --- | --- | --- |
| LAN (untagged) | 10.0.10.0/24 | 10.0.10.1 | — |
| MGMT (99) | 10.0.99.0/24 | 10.0.99.1 | OPNsense .1, pihole-01 .10, PBS .20 (planned) |
| STORAGE (20) | 10.0.20.0/24 | 10.0.20.1 | nfs01 .10, plex01 .11 (planned) |
| GAMING (30) | 10.0.30.0/24 | 10.0.30.1 | amp01 .10 (planned) |
| DMZ (40) | 10.0.40.0/24 | 10.0.40.1 | core-proxy-01 .10, ops-web-01 .20 |
| IOT (50) | 10.0.50.0/24 | 10.0.50.1 | IoT devices |
| MONITORING (60, planned) | 10.0.60.0/24 | 10.0.60.1 | core-siem-01 .10 |
| Home (not a lab VLAN) | 192.168.68.0/22 | 192.168.68.1 | Proxmox .10, OPNsense WAN .62 |

### Appendix B — Port Reference

| Service | Port(s) | Exposed to |
| --- | --- | --- |
| HTTPS (published web) | 443/tcp | WAN → core-proxy-01 only, from Cloudflare ranges |
| Plex via proxy | 8443/tcp | WAN → core-proxy-01 only |
| WireGuard | 51820/udp | WAN → OPNsense |
| Proxmox GUI | 8006/tcp | Home admin sources, MGMT VLAN + WireGuard only |
| OPNsense GUI | 443/tcp (LAN) | MGMT VLAN + WireGuard only |
| AMP admin panel | As configured (e.g. 8080/tcp) | Gaming VLAN via MGMT/WireGuard only, never WAN |
| NFS | 2049/tcp | Storage VLAN + approved home clients only |
| DNS | 53/tcp+udp | Home network and all lab VLANs → 10.0.99.10 |
| Plex | 32400/tcp | Storage VLAN; from WAN only via core-proxy-01 per 8.3 |
| Game server ports | Per game, variable | WAN → amp01, added/removed per active instance |

### Appendix C — Maintenance Schedule

| Task | Frequency |
| --- | --- |
| Apply OS/application updates across all guests | Monthly, or promptly for critical CVEs |
| Review OPNsense/Suricata logs for anomalies | Weekly |
| Review firewall rule set against actual running services | Quarterly |
| Test restore from Proxmox Backup Server | Quarterly |
| Rotate/review WireGuard peer list (remove unused devices) | Quarterly |
| Re-audit against Section 13 checklist | Quarterly |
| Firmware update pass (server, iLO) | Semi-annually |
| Full credential rotation review | Annually |

### Appendix D — Document Change Log

| Version | Date | Summary | Author |
| --- | --- | --- | --- |
| 1.0 | September 2026 | Initial technical implementation guide | Spaldotech Operations |
| 1.1 | 22 October 2026 | Network moved to the as-built 10.0.0.0/16 design and VLAN roles | Spaldotech Operations |

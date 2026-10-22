# Spaldotech Operations — Enterprise Documentation

Documentation repository for the Spaldotech Operations homelab (owner: Aidan Spalding). British English throughout.

## Read first

The standards every guide must follow live in `context/`:

- `context/SPO-HL-SEC-001-homelab-security-hardening-standard.md` — security baseline and the authoritative network inventory (§2).
- `context/homelab-technical-implementation-guide.md` — target architecture (VLANs, WireGuard, DMZ, AMP, NFS, Plex, PBS), on the as-built addressing.
- `context/naming-convention-standard.md` — naming for hosts, VMs, DNS, cloud, identity. New resources use `<tenant>-<role>-<seq>` (e.g. `core-proxy-01`) and `intraspald.tech` internally.

**Address space: the lab uses 10.0.0.0/16.** As-built networks (from the OPNsense console, 22 Oct 2026):

| Network | VLAN | Subnet | Gateway | Notes |
| --- | --- | --- | --- | --- |
| Home | — | 192.168.68.0/22 | 192.168.68.1 | ISP router; Proxmox 192.168.68.10; OPNsense WAN 192.168.68.62 |
| LAN | untagged | 10.0.10.0/24 | 10.0.10.1 | OPNsense vtnet1 |
| MGMT | 90 | 10.0.99.0/24 | 10.0.99.1 | Pi-hole 10.0.99.10 |
| Storage | 20 | 10.0.20.0/24 | 10.0.20.1 | |
| Gaming | 30 | 10.0.30.0/24 | 10.0.30.1 | AMP |
| DMZ | 40 | 10.0.40.0/24 | 10.0.40.1 | core-proxy-01 .10, ops-web-01 .20 |
| IoT | 50 | 10.0.50.0/24 | 10.0.50.1 | |
| Monitoring | 60 | 10.0.60.0/24 | 10.0.60.1 | Planned (SPO-HL-SIEM-001) |

Tags 20/30/50 are inferred from the subnets. The home network is a /22, not a /24. SPO-HL-SEC-001 §2 is the authoritative inventory; keep it in step with any network change.

## Conventions

- Document IDs: `SPO-HL-<AREA>-<NNN>`; register every document in `README.md`.
- Each document is kept as Markdown in `docs/` and PDF in `pdf/`.
- Commit messages start with the document ID, e.g. `SPO-HL-NET-001: confirm permanent fix`.

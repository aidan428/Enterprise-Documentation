# Spaldotech Operations — Enterprise Documentation

Documentation repository for the Spaldotech Operations homelab (owner: Aidan Spalding). British English throughout.

## Read first

The standards every guide must follow live in `context/`:

- `context/SPO-HL-SEC-001-homelab-security-hardening-standard.md` — security baseline; reflects the **as-built** addressing (home 192.168.68.0/24, lab 10.0.x.0/24, OPNsense WAN 192.168.68.62, Proxmox 192.168.68.10).
- `context/homelab-technical-implementation-guide.md` — original target architecture (10.10.x.0/24 VLAN plan, WireGuard, DMZ, AMP, PBS).
- `context/naming-convention-standard.md` — naming for hosts, VMs, DNS, cloud, identity. New resources use `<tenant>-<role>-<seq>` (e.g. `core-proxy-01`) and `intraspald.tech` internally.

**Address space: the lab uses 10.0.0.0/16 (10.0.x.0/24 per VLAN).** Use it for all new work. The 10.10.x.x addresses in the implementation guide, and `core` = 10.10.0.0/16 in section 3.4 of the naming standard, are superseded; where they disagree, the hardening standard and the guides in `docs/` reflect what is actually running.

## Conventions

- Document IDs: `SPO-HL-<AREA>-<NNN>`; register every document in `README.md`.
- Each document is kept as Markdown in `docs/` and PDF in `pdf/`.
- Commit messages start with the document ID, e.g. `SPO-HL-NET-001: confirm permanent fix`.

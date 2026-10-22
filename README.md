<p align="center"><img src="assets/spaldotech-logo.png" alt="Spaldotech Operations" width="120"></p>

<h1 align="center">Spaldotech Operations — Enterprise Documentation</h1>

Central register of Spaldotech Operations build guides, runbooks and resolution guides. Each document is kept as Markdown (readable on GitHub) and as a PDF (includes the diagrams).

## Build guides

| ID | Document | Version | Status | Files |
| --- | --- | --- | --- | --- |
| SPO-HL-PRX-001 | [Caddy Reverse Proxy (core-proxy-01) Build Guide](docs/SPO-HL-PRX-001-caddy-reverse-proxy-core-proxy-01-build-guide.md) | 1.0 | Draft — implementation baseline | [PDF](pdf/SPO-HL-PRX-001-caddy-reverse-proxy-core-proxy-01-build-guide.pdf) · [Live doc](https://claude.ai/artifact/AEPUVYquWxeoXZH7WQVK3j) |
| SPO-HL-SIEM-001 | [Central SIEM (core-siem-01) Build Guide](docs/SPO-HL-SIEM-001-central-siem-core-siem-01-build-guide.md) | 1.0 | Draft — implementation baseline | [PDF](pdf/SPO-HL-SIEM-001-central-siem-core-siem-01-build-guide.pdf) · [Live doc](https://claude.ai/artifact/GhSfziQV6caA5WJ81MhQHe) |
| SPO-HL-SLD-001 | [Technic Solder (api.spaldotechoperations.co.uk) Build Guide](docs/SPO-HL-SLD-001-technic-solder-build-guide.md) | 1.0 (as built) | Implemented 18 October 2026 | [PDF](pdf/SPO-HL-SLD-001-technic-solder-build-guide.pdf) · [Live doc](https://claude.ai/artifact/Sxfzv2Zx3KWJVC7QufcMMv) |

## Resolution guides

| ID | Document | Version | Status | Files |
| --- | --- | --- | --- | --- |
| SPO-HL-NET-001 | [OPNsense DNS Routing Loop (10.0.99.10) Resolution Guide](docs/SPO-HL-NET-001-opnsense-dns-routing-loop-resolution-guide.md) | 1.0 | Workaround applied 22 October 2026; permanent fix to confirm | [PDF](pdf/SPO-HL-NET-001-opnsense-dns-routing-loop-resolution-guide.pdf) · [Live doc](https://claude.ai/artifact/5hKG5F85RTaAnQ3dJfXiwf) |

## Context documents

The founding standards the guides above are written against. Converted to Markdown from the original Word documents.

| ID | Document | Version | Status |
| --- | --- | --- | --- |
| SPO-HL-SEC-001 | [Homelab Security Hardening Standard](context/SPO-HL-SEC-001-homelab-security-hardening-standard.md) | 1.1 | Implementation baseline |
| — | [Homelab Technical Implementation Guide](context/homelab-technical-implementation-guide.md) | 1.1 | Target architecture |
| — | [Naming Convention Standard](context/naming-convention-standard.md) | 1.1 | Active |

## Document IDs

`SPO-HL-<AREA>-<NNN>`: an area code, then a three-digit sequence number within that area.

| Area | Meaning |
| --- | --- |
| PRX | Reverse proxy and edge |
| SIEM | Security monitoring |
| SLD | Technic Solder |
| NET | Networking and routing |
| SEC | Security standards |

## Repository layout

```
assets/   logo and shared images
context/  founding standards (security, architecture, naming)
docs/     Markdown version of each document
pdf/      PDF version of each document, with diagrams
```

## Updating a document

The live documents are edited in Claude; this repository holds dated snapshots. After a document changes, re-export its Markdown and PDF, update its row above, and commit with the document ID in the message, e.g. `SPO-HL-NET-001: confirm permanent fix`.

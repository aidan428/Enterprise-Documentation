<img src="../assets/spaldotech-logo.png" alt="Spaldotech Operations" width="72">

> Context document. Converted from `Spaldotech_Operations_Naming_Convention_Standard.docx` (v1.0, September 2026).

# Spaldotech Operations — Naming Convention Standard

**Enterprise Platform** · On-Premises · Cloud · SaaS Platforms · Identity

| Field | Value |
| --- | --- |
| Classification | Internal (Confidential – Internal) |
| Version | 1.0 — September 2026 |
| Author | Aidan Spalding |

## 1. Introduction & Purpose

### 1.1 Purpose

This standard defines how every named object provisioned or managed by Spaldotech Operations is to be named, across on-premises infrastructure, cloud platforms, and SaaS/identity services. Consistent naming makes ownership, environment, and purpose recoverable from a name alone — without needing to open a management console — and is a precondition for reliable automation, cost allocation, and access control as the estate grows across multiple tenants.

### 1.2 Scope

Spaldotech Operations is the technical provider for Spaldotech (the parent organisation) and its subsidiaries. This standard therefore covers a multi-tenant estate, not a single company's IT:

- **On-premises infrastructure:** hypervisor hosts, virtual machines/containers, network devices, VLANs, IP addressing, and storage (as built in the Enterprise Technical Implementation Guide, which this standard formalises and extends).
- **Cloud platforms:** compute, networking, storage, and platform resources, written to be provider-agnostic so it applies whichever of AWS, Azure, or Google Cloud is adopted — or several at once.
- **SaaS and identity:** user accounts, mailboxes, groups, service accounts/app registrations, and end-user devices, written to remain valid under either a Microsoft 365/Entra ID or Google Workspace identity model, pending that decision.
- **DNS and domains:** the internal intraspald.tech zone, spaldotechoperations.co.uk, and any domains belonging to Spaldotech or its subsidiaries.

### 1.3 How to use this document

Section 2 defines the shared building blocks (tokens, codes, and grammar) used everywhere else in the document. Sections 3–6 apply that grammar to a specific layer of the estate. Section 7 explains how a new subsidiary is onboarded into the model. Section 9 is a single consolidated cheat-sheet for day-to-day use once the full standard has been read once.

### 1.4 Governance & change control

- This document is owned by Spaldotech Operations and is the single source of truth for naming; local conventions that predate it should be migrated at the next natural change window, not renamed in bulk purely for compliance.
- Additions to the code tables in Section 2 (new tenant, new region, new resource type) are the only routine changes expected; structural changes to the grammar itself should be versioned and communicated before being applied to new resources.
- Exceptions (e.g. a SaaS vendor that imposes its own naming constraints) are permitted but must be documented against the affected resource, not treated as silent deviations.

## 2. Core Principles

### 2.1 General rules

- Lowercase and hyphen-delimited by default (e.g. `core-dns-01`), which is the lowest common denominator that is valid across on-prem hostnames, DNS labels, and the great majority of cloud resource types.
- Where a target system forbids hyphens (notably Azure Storage Accounts), tokens are concatenated without separators rather than switching to underscores — underscores are avoided everywhere, since they are invalid in DNS labels and in several cloud resource namespaces.
- No spaces, no punctuation beyond the hyphen, and no characters outside a–z, 0–9, and "-" in any name that may ever be used as, or embedded in, a DNS label, URL, or cloud resource identifier.
- Names are descriptive of function and ownership, never of transient technical detail (avoid embedding an IP address, a one-off ticket number, or a person's name in a persistent resource name).
- Every name is unique within its own namespace (e.g. within a VLAN, within a cloud subscription/project, within the email domain); global uniqueness is only required where the platform itself demands it (e.g. S3 bucket names, Azure Storage Account names).
- Sequence numbers are always two digits, zero-padded (01, 02 … 99), to keep alphabetical and numerical sort order identical and to leave room to grow past nine instances without renumbering.

### 2.2 Naming grammar (building blocks)

Every name in this standard is built from an ordered set of tokens, separated by hyphens. Not every layer uses every token — section headings below state which subset applies — but the left-to-right order is always: tenant, environment, site/region, role/resource-type, application/purpose, sequence.

```
<tenant>-<environment>-<site_or_region>-<role_or_type>-<application>-<sequence>
```

Worked example, a production application server for subsidiary "Morrisbury's" in the UK South cloud region:

```
morrisburys-prod-uks-vm-orders-01
```

### 2.3 Environment codes

| Code | Environment | Notes |
| --- | --- | --- |
| prod | Production | Live, customer- or business-facing |
| stg | Staging | Pre-production, production-like configuration |
| test | Test / QA | Functional and integration testing |
| dev | Development | Active development work, expected to be volatile |
| dr | Disaster recovery | Standby/failover copy of a production resource |
| sbx | Sandbox | Throwaway experimentation, time-boxed, not backed up |

### 2.4 Tenant registry

Every named object belongs to exactly one tenant code. "core" is reserved for shared platform infrastructure that serves every tenant at once (the hypervisor, firewall, backbone DNS, and backup platform built in the Technical Implementation Guide) and is not itself a business entity. Add a row here whenever Spaldotech Operations takes on a new subsidiary; codes are 3–5 lowercase letters and, once assigned, are never reused for a different entity even after offboarding.

| Code | Tenant / Entity | Type | Notes |
| --- | --- | --- | --- |
| core | Shared platform (no single owner) | Core infrastructure | Hypervisor, firewall, backbone DNS/NTP, backup platform — provisioned once, consumed by all tenants |
| ops | Spaldotech Operations | Internal / MSP | The provider's own corporate IT: staff accounts, internal tooling, admin devices |
| spd | Spaldotech | Parent organisation | The parent brand served by Spaldotech Operations |
| morrisburys | Morrisbury's Retail | Subsidiary | Primary retail provider |
| deus | Deus Investments | Subsidiary | Investment arm |
| … | … | — | Follow the onboarding checklist in Section 7 |

> **Note:** Keep this table as the single authoritative tenant list — it is referenced by every other section in this document and should be the first thing updated when the corporate structure changes.

## 3. On-Premises Naming

This section formalises the naming already used when the platform was built (see the Enterprise Technical Implementation Guide) and extends it to remain correct once workloads belonging to Spaldotech or a subsidiary are added alongside the shared core platform.

### 3.1 Hypervisor hosts

```
pve-<site>-<sequence>
```

The existing DL380 Gen9 is the first host at the primary site and is named `pve-hq-01`. Additional hosts (a second physical server, or a unit at a different site) increment the sequence or change the site code; site codes are short and physical (`hq` for the primary location), not tenant codes — a hypervisor is core infrastructure regardless of which tenants' guests run on it.

### 3.2 Virtual machines and containers

```
<tenant>-<role>-<sequence>
```

The tenant token is `core` for shared-platform guests and the relevant tenant code for anything dedicated to Spaldotech, a subsidiary, or Spaldotech Operations' own internal use. Applying this retroactively to the guests already documented in the Technical Implementation Guide:

| Existing name | Standardised name | Role |
| --- | --- | --- |
| OPNsense VM | core-fw-01 | Perimeter firewall/router (shared) |
| dns01 | core-dns-01 | Pi-hole + Unbound (shared) |
| nfs01 | core-nfs-01 | NFS server (shared) |
| plex01 | core-plex-01 | Media server (shared) |
| proxy01 | core-proxy-01 | Reverse proxy (shared DMZ ingress) |
| pbs01 | core-pbs-01 | Proxmox Backup Server (shared) |
| amp01 | core-amp-01 | CubeCoders AMP controller (shared) |
| web01 / web02 | ops-web-01 / ops-web-02 | Spaldotech Operations' own published web app |

> **Note:** Renaming already-running guests is optional and should be done at a routine maintenance window, not urgently.

### 3.3 Network devices, interfaces, and VLANs

- Physical/logical network devices: `<type>-<site>-<sequence>`, e.g. `fw-hq-01` (matches `core-fw-01` above, used interchangeably in network diagrams vs. inventory systems), `sw-hq-01` for the managed switch, `ap-hq-01` for a future wireless access point.
- VLAN names stay functional and short as already defined (MGMT, SERVERS, GAMES, DMZ, TRUSTED) for the shared core network. A VLAN dedicated entirely to one tenant (rather than a shared segment) is named `<TENANT>-<PURPOSE>`, e.g. `SUB1-DMZ`, and drawn from that tenant's own reserved VLAN ID block (3.4).

### 3.4 IP address allocation

The core platform already occupies 10.10.x.0/24 per VLAN. Each additional tenant that requires network-level isolation (rather than sharing the core VLANs) is allocated its own /16-scoped block, keeping the same last-octet convention used throughout the Technical Implementation Guide (.1 gateway, .2–.9 infrastructure, .10–.199 DHCP, .200–.254 static).

| Tenant | Reserved range | Status |
| --- | --- | --- |
| core | 10.10.0.0/16 | In use — see Technical Implementation Guide Section 4 |
| ops | 10.11.0.0/16 | Reserved |
| spd | 10.20.0.0/16 | Reserved |
| sub1 | 10.30.0.0/16 | Reserved — allocate on onboarding |
| sub2 | 10.40.0.0/16 | Reserved — allocate on onboarding |

### 3.5 Storage

- ZFS pools/datasets: `<purpose>` at the pool root (e.g. `rpool/vm-storage`, `rpool/media`, `rpool/pbs-datastore`) as already built; a dataset dedicated to one tenant's data is `rpool/<tenant>-<purpose>`, e.g. `rpool/sub1-data`.
- NFS export paths mirror the dataset name they serve; export ACLs are scoped per Section 7.2 of the Technical Implementation Guide regardless of naming.
- Backup jobs: `bkp-<tenant>-<role>-<frequency>`, e.g. `bkp-core-dns-daily`, `bkp-sub1-erp-hourly`.

## 4. DNS & Domain Naming

### 4.1 Internal zone — intraspald.tech

- Shared/core services sit at the root of the zone: `dns.intraspald.tech`, `proxmox.intraspald.tech`, `plex.intraspald.tech`, as already established.
- A tenant that needs its own internal namespace (rather than a handful of shared records) gets a delegated subdomain: `<tenant>.intraspald.tech`, e.g. `deus.intraspald.tech`, with service records underneath it: `erp.deus.intraspald.tech`.
- Hostnames within any zone follow `<service-or-role>.<zone>` and use the same role tokens as the VM naming in Section 3.2 wherever the DNS name refers to a specific host, e.g. `core-nfs-01.intraspald.tech` for host inventory purposes even though the friendly CNAME `nfs.intraspald.tech` is what applications actually use.

### 4.2 External domains

- spaldotechoperations.co.uk is Spaldotech Operations' own public domain (its provider-facing brand, status pages, and administration portals) and follows `<service>.spaldotechoperations.co.uk`, as already used for externally published web services in the Technical Implementation Guide.
- Spaldotech and each subsidiary typically bring, or will register, their own public domain (e.g. spaldotech.co.uk). Spaldotech Operations manages DNS for these on the tenant's behalf, but the naming pattern is identical: `<service>.<the tenant's own registered domain>`, e.g. `www.spaldotech.co.uk`, `portal.<subsidiary-domain>`.
- Internal-only administrative hostnames (Proxmox, OPNsense, AMP, PBS, and equivalent cloud consoles) are never published under any external domain, per the security posture set out in the Technical Implementation Guide — they exist only inside intraspald.tech and the relevant tenant's internal subdomain.

### 4.3 TLS certificate naming

- Certificate common names/SANs match the externally published hostname exactly; wildcard certificates, where used, are scoped to a single owning domain (e.g. `*.spaldotechoperations.co.uk`) and never span two tenants' domains on one certificate.

## 5. Cloud Resource Naming

No cloud platform has been officially onboarded yet. Patterns will be enforced no matter the platform selected.

### 5.1 General pattern

```
<tenant>-<environment>-<region>-<resource_type>-<application>-<sequence>
```

Example — a production database for Spaldotech's order system in the UK South region:

```
spd-prod-uks-db-orders-01
```

Shorter forms are acceptable where a token would be redundant within its own namespace (e.g. omitting `<region>` for a global resource type, or omitting `<sequence>` for a resource that will only ever exist once) — but tokens are never reordered, only dropped when genuinely inapplicable.

### 5.2 Resource type codes (representative, not exhaustive)

| Code | Resource | AWS equivalent | Azure equivalent | GCP equivalent |
| --- | --- | --- | --- | --- |
| vm | Virtual machine / instance | EC2 instance | Virtual Machine | Compute Engine instance |
| net | Virtual network | VPC | Virtual Network (VNet) | VPC network |
| snet | Subnet | Subnet | Subnet | Subnetwork |
| sg | Network security group/rule set | Security Group | Network Security Group | Firewall rule set |
| lb | Load balancer | ELB/ALB/NLB | Load Balancer | Cloud Load Balancing |
| db | Managed database | RDS instance | Azure SQL / Database | Cloud SQL instance |
| st | Object/blob storage | S3 bucket | Storage Account | Cloud Storage bucket |
| func | Serverless function | Lambda | Function App | Cloud Function |
| k8s | Managed Kubernetes cluster | EKS cluster | AKS cluster | GKE cluster |
| kv | Secrets/key management | Secrets Manager / KMS | Key Vault | Secret Manager / KMS |
| rg | Logical grouping container | Account / Org Unit | Resource Group | Project |

### 5.3 Region/site codes

Short, provider-independent region codes are used in names; translate to the provider's own region identifier when calling its API or console — the short code is what keeps a name portable if a workload is ever migrated between providers.

| Code | Location | Typical AWS region | Typical Azure region | Typical GCP region |
| --- | --- | --- | --- | --- |
| uks | UK South | eu-west-2 | uksouth | europe-west2 |
| ukw | UK West | eu-west-2 | ukwest | europe-west2 |
| euw | EU West (Ireland) | eu-west-1 | westeurope | europe-west1 |
| use | US East | us-east-1 | eastus | us-east1 |
| glb | Global / non-regional resource | n/a (global service) | global | global |

### 5.4 Handling provider naming constraints

Where the general pattern in 5.1 would violate a specific provider's rules, tokens are concatenated without hyphens rather than abbreviated further or switched to underscores — the token order and content stay the same, only the separator is dropped.

| Resource | Constraint | Adapted example |
| --- | --- | --- |
| Azure Storage Account | Lowercase alphanumeric only, no hyphens, 3–24 characters | spdprodstuks01 |
| AWS S3 bucket | Globally unique, lowercase, hyphens allowed, no underscores, 3–63 characters | spd-prod-uks-st-orders-01 |
| GCP Cloud Storage bucket | Globally unique, lowercase, hyphens allowed | spd-prod-uks-st-orders-01 |
| Any DNS-linked resource (e.g. public load balancer hostname) | Must be a valid DNS label | Standard pattern already DNS-safe (5.1) |

### 5.5 Tagging / labelling standard

Names identify a resource; tags/labels carry the metadata needed for cost allocation and access review, and are applied identically regardless of provider (using each provider's own tag/label mechanism):

| Tag key | Purpose | Example value |
| --- | --- | --- |
| Tenant | Owning tenant code from the Section 2.4 registry | deus |
| Environment | Environment code from Section 2.3 | prod |
| Application | Application/workload this resource belongs to | orders |
| Owner | Individual or team accountable for the resource | ops-infra-team |
| ManagedBy | Always "SpaldotechOperations" for provider-managed resources | SpaldotechOperations |
| CostCentre | Billing/chargeback code, once assigned | tbc per tenant |
| DataClassification | Public / Internal / Confidential / Restricted | Confidential |

## 6. SaaS & Identity Naming

The identity platform (Microsoft 365/Entra ID, Google Workspace, or another provider) has not been decided yet; the conventions below are written to hold under either model. Once a platform is chosen, add any platform-specific technical identifiers (e.g. an Entra ID tenant name) as an appendix row rather than changing the conventions themselves.

### 6.1 User accounts and mailboxes

- **Personal accounts:** `firstname.lastname@<tenant's primary domain>`, e.g. `jane.doe@spaldotech.co.uk`. Where a name collision occurs, append a numeral (`jane.doe2@…`), never a middle initial or department code, to keep the pattern predictable.
- **Shared/functional mailboxes:** a plain descriptive local part with no tenant prefix needed since the domain already scopes it, e.g. `info@spaldotechoperations.co.uk`, `support@spaldotech.co.uk`.
- **Service accounts** (used by applications/scripts, not a person): `svc-<tenant>-<purpose>@<domain>`, e.g. `svc-core-backup@intraspald.tech` for an internal automation identity, kept clearly distinguishable from human accounts in any access review.

### 6.2 Groups and distribution lists

```
grp-<tenant>-<function>-<access_level>
```

Examples: `grp-ops-it-admins` (full administrative access, Spaldotech Operations staff only), `grp-sub1-finance-readonly` (read-only access to a subsidiary's finance systems). Group naming should make the access being granted legible from the name alone, since group names routinely appear unexplained in access-review reports.

### 6.3 SaaS tenants and workspaces

- Where a SaaS product supports multiple workspaces/tenants under one vendor contract (e.g. a ticketing system, a chat platform), the workspace is named after the tenant it serves using the Section 2.4 codes — `spaldotech`, `sub1` — rather than a marketing name that might later change.
- Where Spaldotech Operations operates a single shared SaaS instance on behalf of all tenants (common for cost reasons with smaller subsidiaries), tenant separation is achieved with the `grp-` naming in 6.2 and the resource is otherwise named `core-<product>`.

### 6.4 API keys, app registrations, and service principals

```
<tenant>-<application>-<purpose>-<environment>
```

Example: `morrisburys-orders-api-prod` for the credential an order-management application uses to call an API in production. Every such identifier is registered against an owner and a rotation schedule regardless of how it is named — naming makes an orphaned credential easy to trace back to its owning tenant and application, not a substitute for that record.

### 6.5 End-user devices

```
<tenant>-<device_type>-<asset_tag>
```

| Device type code | Meaning |
| --- | --- |
| lap | Laptop |
| dtp | Desktop |
| mob | Mobile phone |
| tab | Tablet |

Example: `core-lap-00231` for a Spaldotech Operations-issued laptop tracked under asset tag 00231. Asset tags are assigned sequentially from the organisation's existing asset register and are never reused once retired.

## 7. Multi-Tenant / Subsidiary Model

### 7.1 How the tenant code threads through every layer

The tenant code from the Section 2.4 registry is the one piece of information that ties a resource together across every layer of the estate — the same code appears in the VM name, the IP allocation, the DNS subdomain, the cloud tag, and the SaaS group name for anything belonging to that tenant. This is what makes it possible to answer "what does subsidiary X have, everywhere" without a bespoke inventory system: it is a search for one string across every naming layer.

### 7.2 Onboarding a new subsidiary — checklist

- [ ] Assign a tenant code in the Section 2.4 registry (3–5 lowercase letters, not reused from a previous entity).
- [ ] Reserve an IP block per Section 3.4 if the subsidiary requires network-level isolation; otherwise place its workloads in shared core VLANs with per-resource access control instead.
- [ ] Create the intraspald.tech delegated subdomain (Section 4.1) if the subsidiary will have more than a handful of internal service records.
- [ ] Confirm the subsidiary's own external domain(s), if any, and record the mapping in Appendix D.
- [ ] Create the initial SaaS groups (Section 6.2) needed for staff access, following the `grp-<tenant>-…` pattern from day one.
- [ ] Record the CostCentre value once assigned, so cloud tagging (Section 5.5) is correct from the first resource created rather than retrofitted.

### 7.3 Decommissioning

- A retired tenant's code is retained in the registry (marked inactive, never deleted or reassigned) so historical logs, backups, and audit records naming that code remain unambiguous.
- IP blocks and DNS delegations for a decommissioned tenant are held for a minimum of 90 days after the last resource is removed before being marked available for reuse, to avoid stale references resolving unexpectedly during the transition.

## 8. Reserved Words & Prohibited Patterns

- The tenant code `core` is reserved exclusively for shared platform infrastructure and must never be assigned to a business entity, even temporarily.
- The words `admin`, `root`, `test`, `temp`, `default`, and `null` are not permitted as the sole distinguishing token in any persistent resource name — they convey no ownership or purpose and tend to accumulate untracked resources.
- Environment codes (2.3) are not reused as tenant codes and vice versa, to avoid a name being misread as the wrong token type.
- Personally identifiable information (an individual's name, national insurance/social security number, or similar) is never embedded in an infrastructure resource name, even where it is embedded in a personal user account's mailbox address (6.1), since the two are governed by different retention and disclosure rules.

## 9. Quick Reference

A single consolidated view of every pattern in this document, for day-to-day use once the full standard has been read.

| Layer | Pattern | Example |
| --- | --- | --- |
| Hypervisor host | `pve-<site>-<seq>` | pve-hq-01 |
| VM / container | `<tenant>-<role>-<seq>` | core-dns-01 |
| Network device | `<type>-<site>-<seq>` | fw-hq-01 |
| Shared VLAN | `<FUNCTION>` | SERVERS |
| Tenant-dedicated VLAN | `<TENANT>-<PURPOSE>` | SUB1-DMZ |
| ZFS dataset | `rpool/<tenant>-<purpose>` | rpool/deus-data |
| Backup job | `bkp-<tenant>-<role>-<freq>` | bkp-core-dns-daily |
| Internal DNS (shared) | `<service>.intraspald.tech` | plex.intraspald.tech |
| Internal DNS (tenant) | `<service>.<tenant>.intraspald.tech` | erp.morrisburys.intraspald.tech |
| External DNS | `<service>.<registered domain>` | portal.spaldotechoperations.co.uk |
| Cloud resource | `<tenant>-<env>-<region>-<type>-<app>-<seq>` | spd-prod-uks-db-orders-01 |
| Personal mailbox | `firstname.lastname@<domain>` | jane.doe@spaldotech.co.uk |
| Service account | `svc-<tenant>-<purpose>@<domain>` | svc-core-backup@intraspald.tech |
| SaaS/access group | `grp-<tenant>-<function>-<access>` | grp-sub1-finance-readonly |
| App credential | `<tenant>-<app>-<purpose>-<env>` | morrisburys-orders-api-prod |
| End-user device | `<tenant>-<device_type>-<asset_tag>` | core-lap-00231 |

## Appendices

### Appendix A — Environment Codes

| Code | Environment |
| --- | --- |
| prod | Production |
| stg | Staging |
| test | Test / QA |
| dev | Development |
| dr | Disaster recovery |
| sbx | Sandbox |

### Appendix B — Tenant Registry

| Code | Tenant / Entity | Type |
| --- | --- | --- |
| core | Shared platform | Core infrastructure |
| ops | Spaldotech Operations | Internal / MSP |
| spd | Spaldotech | Parent organisation |
| morrisburys | Morrisbury's Retail | Subsidiary |
| deus | Deus Investments | Subsidiary |

### Appendix C — Resource Type Codes

| Code | Resource |
| --- | --- |
| vm | Virtual machine / instance |
| net | Virtual network |
| snet | Subnet |
| sg | Security group / rule set |
| lb | Load balancer |
| db | Managed database |
| st | Object/blob storage |
| func | Serverless function |
| k8s | Managed Kubernetes cluster |
| kv | Secrets/key management |
| rg | Logical grouping container |

### Appendix D — Domain Ownership Map

| Domain | Tenant | Purpose |
| --- | --- | --- |
| intraspald.tech | core | Internal DNS zone, all tenants |
| spaldotechoperations.co.uk | ops | Provider-facing external domain |
| spaldotech.co.uk | spd | Parent organisation public domain |
| morrisburys.co.uk | morrisburys | Subsidiary public domain |

### Appendix E — Document Change Log

| Version | Date | Summary | Author |
| --- | --- | --- | --- |
| 1.0 | September 2026 | Initial naming convention standard | Aidan Spalding |

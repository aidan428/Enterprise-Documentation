<img src="../assets/spaldotech-logo.png" alt="Spaldotech Operations" width="72">

> Context document. Converted from `Spaldotech_Operations_Homelab_Security_Hardening_Standard.docx` (v1.0, 30 September 2026); updated to the as-built network in v1.1.

# Spaldotech Operations — Homelab Security Hardening Standard

**Step-by-Step Implementation Guide**

| Field | Value |
| --- | --- |
| Document ID | SPO-HL-SEC-001 |
| Version | 1.1 |
| Status | Implementation Baseline |
| Date | 22 October 2026 (v1.0: 30 September 2026) |
| Owner | Spaldotech Operations |
| Classification | Internal / Technical |

**Scope:** Proxmox VE, OPNsense and Debian 13/Pi-hole

**Security posture:** Layered defence, least privilege, explicit network allowlists, hardened management planes and controlled recovery.

## Contents

1. Purpose and Security Objectives
2. Current Environment Baseline
3. Hardening Order and Change Control
4. Proxmox VE Hardening
5. OPNsense Hardening
6. Debian 13 / Pi-hole Hardening
7. Cross-Platform Security Baseline
8. Network Security Rules for the Current Architecture
9. Verification and Acceptance Testing
10. Backup and Recovery Standard
11. Ongoing Operations
12. Final Hardening Checklist
- Appendix A — Example Administrative Port Matrix
- Appendix B — Security Notes and References

## 1. Purpose and Security Objectives

This standard defines the security baseline and step-by-step hardening procedure for the Spaldotech Operations virtualised homelab. It applies to the Proxmox hypervisor, the virtual OPNsense perimeter firewall and the Debian 13 server hosting Pi-hole DNS.

The design goal is defence in depth: minimise exposed management services, enforce least privilege, require strong administrator authentication, separate network security zones, maintain reliable logging and updates, and preserve a controlled recovery path.

The environment follows least privilege, default deny between security zones, management-plane isolation, no direct Internet exposure of management interfaces, authenticated administration, configuration backup before material changes, and documented exceptions.

## 2. Current Environment Baseline

**Hosts**

| Component | Address | Security Role | Primary Management Name |
| --- | --- | --- | --- |
| ISP router | 192.168.68.1 | Upstream/home gateway | home-gw |
| Proxmox | 192.168.68.10/22 | Hypervisor / management | pve-01.intraspald.tech |
| OPNsense WAN | 192.168.68.62/22 (DHCP) | Perimeter WAN | core-fw-01.intraspald.tech |
| OPNsense MGMT | 10.0.99.1/24 | Lab management gateway; Web UI | core-fw-01.intraspald.tech |
| Pi-hole | 10.0.99.10/24 | Authoritative internal DNS service | pihole-01.intraspald.tech |

**Networks** (as shown on the OPNsense console)

| Network | Subnet / Gateway | OPNsense interface | VLAN | Security Role | Alias |
| --- | --- | --- | --- | --- | --- |
| Home | 192.168.68.0/22 | WAN (vtnet0) | — | Upstream trusted-but-external network | HOME_NET |
| LAN | 10.0.10.0/24 / 10.0.10.1 | LAN (vtnet1) | untagged | Default lab LAN | LAN_NET |
| Management | 10.0.99.0/24 / 10.0.99.1 | vlanMGMT (vlan01) | 90 | Infrastructure administration, DNS | MGMT_NET |
| Storage | 10.0.20.0/24 / 10.0.20.1 | vlanStorage (vlan04) | 20† | Storage services | STORAGE_NET |
| Gaming | 10.0.30.0/24 / 10.0.30.1 | vlanGaming (vlan05) | 30† | Game servers (AMP) | GAMING_NET |
| DMZ | 10.0.40.0/24 / 10.0.40.1 | vlanDMZ (vlan03) | 40 | Internet-facing services | DMZ_NET |
| IoT | 10.0.50.0/24 / 10.0.50.1 | vlanIoT (vlan02) | 50† | Untrusted IoT devices | IOT_NET |
| Monitoring (planned) | 10.0.60.0/24 / 10.0.60.1 | — | 60 | SIEM (SPO-HL-SIEM-001) | MON_NET |

† *VLAN tag inferred from the subnet; confirm against Interfaces → Devices → VLAN. The `vlan0N` names are OPNsense device names, not tags.*

*The home network is a /22 (192.168.68.0–192.168.71.255). Use 192.168.68.0/22 in every HOME_NET alias and allowlist, not /24.*

The upstream router contains a static route for the lab supernet (where supported): 10.0.0.0/16 via 192.168.68.62. Home-to-lab traffic is routed, not NATed. Lab-to-Internet traffic uses OPNsense outbound NAT. Proxmox management remains directly reachable on 192.168.68.10 and therefore must be treated as a privileged management endpoint.

## 3. Hardening Order and Change Control

Perform the hardening in this order:

1. Back up OPNsense and Proxmox configuration and record current network access.
2. Harden Proxmox management and host SSH.
3. Harden OPNsense administration and firewall policy.
4. Harden Debian 13/Pi-hole OS and DNS service.
5. Apply service-specific network restrictions.
6. Validate DNS, GUI, SSH, routing and Internet access.
7. Record exceptions and retain a known-good rollback path.

*Do not make multiple high-impact network changes simultaneously. For each stage, maintain an active console/IPMI recovery method. Before disabling an existing access mechanism, verify the replacement method from a second session.*

## 4. Proxmox VE Hardening

### 4.1 Create a dedicated administrative identity

```bash
# Run from the Proxmox shell as root
pveum user add admin@pam
pveum passwd admin@pam
pveum group add pve-admins
pveum acl modify / -group pve-admins -role Administrator
pveum user modify admin@pam -groups pve-admins
```

Use a unique administrator account rather than routinely operating as root through the Web UI. Keep root available for recovery, but do not use it as the everyday administrative identity. Assign the minimum role needed for non-administrative accounts; the example Administrator role is intended for the primary homelab administrator only.

### 4.2 Enable MFA

In the Proxmox Web UI, configure TOTP or another supported second factor for the administrative account. Test a new login in a separate browser session before ending the current privileged session. Keep a documented recovery procedure; do not rely on a single MFA device without a recovery path.

### 4.3 Harden SSH

```bash
# Inspect current SSH configuration
sshd -T | egrep 'permitrootlogin|passwordauthentication|pubkeyauthentication'

# Edit the SSH configuration
nano /etc/ssh/sshd_config

# Recommended baseline
PermitRootLogin no
PasswordAuthentication no
KbdInteractiveAuthentication no
PubkeyAuthentication yes
X11Forwarding no
AllowGroups sshadmins

# Validate before restarting
sshd -t
systemctl restart ssh
```

Create an `sshadmins` group and add only administrators who genuinely require shell access. Use SSH keys. Do not disable root SSH access until key-based access for the replacement administrator has been tested from another session.

### 4.4 Restrict Proxmox Web UI

Do not port-forward TCP 8006 from the Internet. Permit TCP 8006 only from the trusted administration sources. In this environment, the preferred policy is home administration to 192.168.68.10:8006, with no WAN exposure. If a dedicated management VLAN is introduced, migrate administrative access to that VLAN.

### 4.5 Proxmox firewall

Enable the Proxmox firewall at Datacenter and Node level only after confirming the required management rules. Use an explicit allowlist for SSH (TCP/22) and the Web UI (TCP/8006) from approved management networks. Avoid broad 'any-to-any' rules.

### 4.6 Updates and repository hygiene

```bash
apt update
apt full-upgrade

# Review enabled repositories
grep -RhvE '^\s*#|^\s*$' /etc/apt/sources.list /etc/apt/sources.list.d/*.list 2>/dev/null

# Reboot when kernel/security updates require it
reboot
```

Use the appropriate Proxmox repository for the installation/licensing model. Do not mix incompatible repositories or blindly copy repository definitions from another Proxmox release. Apply updates during a controlled maintenance window.

### 4.7 Host attack-surface reduction

Remove unused packages and services, disable unused daemons, avoid installing general-purpose application stacks on the hypervisor, and keep workloads inside VMs/containers. Do not use the Proxmox host as a general-purpose server.

### 4.8 VM security baseline

For each VM: use VirtIO devices where appropriate, disable unused virtual hardware, enable the Proxmox guest firewall where appropriate, use the minimum CPU/RAM required, keep QEMU guest agents only where needed, and avoid attaching a VM directly to the home bridge unless there is a documented requirement.

### 4.9 Proxmox validation

- Confirm administrative login works using the named admin account and MFA.
- Confirm root cannot authenticate over SSH.
- Confirm password-based SSH authentication is disabled.
- Confirm TCP/8006 is reachable only from approved management networks.
- Confirm no Proxmox management port is forwarded from the ISP router.
- Confirm all expected VMs start and OPNsense starts before dependent workloads.
- Confirm a configuration backup exists and can be located.

## 5. OPNsense Hardening

### 5.1 Administrator accounts and MFA

Create a named administrator account and enable TOTP/2FA for Web UI access. Use individual accounts where practical and remove unused accounts. Do not use shared administrator credentials.

### 5.2 Web UI security

Keep the Web UI on HTTPS. Install a certificate whose subject/SAN matches `core-fw-01.intraspald.tech` when the internal PKI is available. Enable HTTP Strict Transport Security only after HTTPS is confirmed working and trusted on administrator workstations. OPNsense documents HTTPS as the recommended Web UI protocol and supports TOTP for the GUI.

### 5.3 SSH

Disable SSH unless it is operationally required. If SSH is required, disable root login and password authentication, use keys, and restrict the listening interface/source networks. Prefer the OPNsense console for recovery.

### 5.4 WAN protection

The WAN is intentionally on the private 192.168.68.0/22 network, so OPNsense must allow the required routed home-to-lab traffic. Keep management access tightly scoped. Do not expose the Web UI, SSH or other administrative services to arbitrary WAN sources. Retain bogon filtering where appropriate for this topology.

### 5.5 Firewall policy

Create the aliases listed in §2: `HOME_NET=192.168.68.0/22`, `LAN_NET=10.0.10.0/24`, `MGMT_NET=10.0.99.0/24`, `STORAGE_NET=10.0.20.0/24`, `GAMING_NET=10.0.30.0/24`, `DMZ_NET=10.0.40.0/24`, `IOT_NET=10.0.50.0/24` and, once built, `MON_NET=10.0.60.0/24`. Use explicit service rules instead of broad network-to-network permits.

| Traffic | Policy | Purpose |
| --- | --- | --- |
| HOME_NET → OPNsense GUI | Allow HTTPS only | Administration |
| HOME_NET → Proxmox | Allow TCP 8006 only | Hypervisor administration |
| HOME_NET → Pi-hole | Allow TCP/UDP 53 | DNS |
| HOME_NET → Lab servers | Allow only required services | Controlled access |
| DMZ → HOME_NET | Deny by default | Containment |
| DMZ → MGMT_NET | Deny by default | Containment |
| GAMING → MGMT/STORAGE/HOME | Deny by default | Containment |
| IoT → MGMT/STORAGE/HOME | Deny by default (DNS to Pi-hole only) | Untrusted isolation |
| Lab → Internet | Allow required outbound | Updates/services |
| Internet/WAN → Management | Deny | No exposed management |

### 5.6 DNS security

If Pi-hole is the designated DNS service, distribute Pi-hole as DNS through DHCP and explicitly permit DNS to the Pi-hole address. Consider blocking direct outbound DNS from client VLANs so clients cannot bypass the approved resolver, while allowing the Pi-hole itself to reach its configured upstream DNS providers.

### 5.7 NTP and time

Configure reliable NTP. Accurate time is essential for TLS, TOTP, logs and incident investigation. Use a consistent time source throughout the lab.

### 5.8 Logging

Enable firewall logging for deny rules that matter operationally, retain system and authentication logs, and consider forwarding OPNsense logs to a dedicated logging/SIEM service in a future monitoring VLAN.

### 5.9 Configuration backup

Export an encrypted OPNsense configuration backup before major changes. Store backups outside the firewall VM and protect them as sensitive credentials because configuration files can contain secrets.

### 5.10 OPNsense validation

- Named administrator can log in with MFA.
- Web UI uses HTTPS and the expected internal DNS name.
- SSH is disabled unless explicitly required.
- Home-to-lab traffic follows the intended static route.
- DMZ cannot initiate connections to Management or Home unless explicitly required.
- Gaming and IoT cannot initiate connections to Storage/Management unless explicitly required.
- Pi-hole receives DNS queries from approved networks only.
- Internet-facing management ports are not exposed.

## 6. Debian 13 / Pi-hole Hardening

### 6.1 Confirm hostname, static address and DNS

```bash
hostnamectl
ip -br addr
ip route
resolvectl status

# Set a permanent hostname if required
sudo hostnamectl set-hostname pihole-01
```

Use a static/reserved address for Pi-hole. Ensure the system's hostname and local DNS record agree. Example: `pihole-01.intraspald.tech → 10.0.99.10`.

### 6.2 Update the operating system

```bash
sudo apt update
sudo apt full-upgrade

# Review packages held back or pending
apt list --upgradable
```

Keep Debian 13 current using Debian security repositories. Schedule regular maintenance rather than allowing a DNS infrastructure host to remain unpatched for long periods.

### 6.3 Automatic security updates

```bash
sudo apt install unattended-upgrades apt-listchanges
sudo dpkg-reconfigure unattended-upgrades

# Verify
systemctl status unattended-upgrades
systemctl is-enabled unattended-upgrades
```

Automatic security updates are appropriate for a small homelab DNS appliance, but monitor reboots and package changes because a DNS service is infrastructure. Keep periodic manual maintenance windows as well.

### 6.4 Create a named sudo administrator

```bash
sudo adduser sysadmin
sudo usermod -aG sudo sysadmin

# Test a separate SSH session before disabling root/password access
```

Do not use the Pi-hole `root` account for routine administration. Use a named account with sudo.

### 6.5 Harden SSH

```bash
sudo nano /etc/ssh/sshd_config.d/99-hardening.conf

# Recommended baseline
PermitRootLogin no
PasswordAuthentication no
KbdInteractiveAuthentication no
PubkeyAuthentication yes
X11Forwarding no
AllowGroups sshadmins

# Add the administrator to the group
sudo groupadd -f sshadmins
sudo usermod -aG sshadmins sysadmin

# Validate and reload
sudo sshd -t
sudo systemctl reload ssh
```

Copy and test an SSH public key before disabling password authentication. Keep console access available through Proxmox for recovery.

### 6.6 Host firewall

Use a host firewall as defence in depth, but make the rules match the routed topology. Pi-hole needs DNS from approved client networks and HTTPS from administrators. SSH should be limited to the management network. Do not expose the Pi-hole DNS service to the Internet.

```bash
# Example nftables policy concept — review interfaces/addresses before applying
sudo apt install nftables
sudo systemctl enable --now nftables

# Inspect active rules
sudo nft list ruleset
```

A suitable final ruleset should allow: established/related traffic; loopback; DNS TCP/UDP 53 from HOME_NET and approved lab VLANs; HTTPS TCP 443 from MGMT/HOME administrator sources; and SSH TCP 22 only from MGMT/HOME administration sources. It should deny unsolicited inbound traffic from all other sources. Because the exact Pi-hole address/interface is environment-specific, build and test the rules before enabling a default-drop policy.

### 6.7 Pi-hole service exposure

Pi-hole should be an internal DNS service, not a public recursive resolver. Use Pi-hole's interface/binding controls and the Debian firewall together. Pi-hole documentation recommends limiting DNS requests to local networks to avoid accidental public DNS exposure; because your home network is routed rather than directly attached to the Pi-hole subnet, use the Pi-hole setting that permits the required routed clients and enforce the actual source allowlist at OPNsense/host-firewall level.

### 6.8 Pi-hole Web UI

Use HTTPS for the Pi-hole administration interface. Pi-hole creates a self-signed certificate by default; for a clean internal deployment, issue a certificate for `pihole-01.intraspald.tech` from the lab's trusted internal CA and configure Pi-hole to use it. Do not expose the Pi-hole Web UI through a public port forward.

### 6.9 Pi-hole account security

Use a strong unique Pi-hole administrator password and enable the strongest supported authentication controls available in the installed Pi-hole release. Do not reuse the Proxmox, OPNsense or Debian administrator password.

### 6.10 DNS upstream security

Select a small number of trusted upstream resolvers or an upstream DNS-over-TLS/DNS-over-HTTPS design if required. Document the choice. Avoid silently mixing many providers. Ensure Pi-hole itself can reach only the required upstream DNS services if you want strict egress control.

### 6.11 Disable unnecessary services

```bash
# Review listening sockets
sudo ss -lntup

# Review enabled services
systemctl list-unit-files --type=service --state=enabled
```

Remove or disable services not required for DNS infrastructure. The desired listening surface should be small: DNS, the Pi-hole Web UI, and SSH only where required for administration.

### 6.12 Debian security review

```bash
# Review failed services
systemctl --failed

# Review recent authentication activity
journalctl -u ssh --since today

# Check listening ports
sudo ss -lntup
```

### 6.13 Pi-hole validation

- DNS resolution works from 192.168.68.0/22 and approved lab VLANs.
- DNS queries from an unapproved source are blocked.
- Pi-hole Web UI is reachable only from approved administration sources.
- SSH is key-only and root login is disabled.
- Pi-hole is not reachable from the Internet/WAN.
- Pi-hole remains functional after a Debian reboot.
- Local DNS records resolve correctly through Pi-hole.
- Pi-hole configuration is backed up.

## 7. Cross-Platform Security Baseline

| Control | Proxmox | OPNsense | Debian/Pi-hole |
| --- | --- | --- | --- |
| Named admin account | Required | Required | Required |
| MFA | Required where supported | Required for GUI | Use strong password; add MFA if supported by deployment |
| Root login over SSH | Disabled | Disabled if SSH enabled | Disabled |
| Password SSH | Disabled | Disabled if SSH enabled | Disabled |
| HTTPS admin | Required | Required | Required for Web UI |
| Internet management exposure | Prohibited | Prohibited | Prohibited |
| Automatic/security updates | Regular maintenance | Regular maintenance | Enabled/regular maintenance |
| Host firewall | Proxmox firewall | pf/OPNsense firewall | nftables defence in depth |
| Configuration backup | Required | Required | Required |
| Central logging | Recommended | Recommended | Recommended |
| Least privilege | Required | Required | Required |

This baseline should be treated as the minimum. Higher-risk services such as DMZ reverse proxies, game servers, databases and storage systems should receive additional service-specific hardening.

## 8. Network Security Rules for the Current Architecture

The security model should make OPNsense the policy enforcement point between security zones. The existence of a working static route does not imply that every home device should be able to access every lab service.

| Source | Destination | Default policy |
| --- | --- | --- |
| 192.168.68.0/22 | 10.0.99.1 HTTPS | Allow administration |
| 192.168.68.0/22 | 10.0.99.10 TCP/UDP 53 | Allow DNS |
| 192.168.68.0/22 | 10.0.99.10 HTTPS | Allow Pi-hole administration |
| 192.168.68.0/22 | 10.0.20.0/24 (Storage) | Deny; add service exceptions |
| 192.168.68.0/22 | 10.0.30.0/24 (Gaming) | Deny; add game-port exceptions |
| 192.168.68.0/22 | 10.0.40.0/24 (DMZ) | Deny; add service exceptions |
| 10.0.40.0/24 (DMZ) | 192.168.68.0/22 | Deny |
| 10.0.40.0/24 (DMZ) | 10.0.99.0/24 (MGMT) | Deny |
| 10.0.40.0/24 (DMZ) | 10.0.20.0/24 (Storage) | Deny |
| 10.0.30.0/24 (Gaming) | 10.0.99.0/24, 10.0.20.0/24 | Deny |
| 10.0.50.0/24 (IoT) | 10.0.99.10 TCP/UDP 53 | Allow DNS |
| 10.0.50.0/24 (IoT) | All other lab and home networks | Deny |
| Lab VLANs | 10.0.60.10 TCP 1514–1515 | Allow SIEM agents (once SPO-HL-SIEM-001 is built) |
| Lab VLANs | Internet | Allow required outbound only |

*Because Proxmox itself sits on 192.168.68.10, OPNsense cannot protect the Proxmox management interface from other devices already trusted by the home LAN. Protect Proxmox with its own firewall, MFA and restricted source networks.*

## 9. Verification and Acceptance Testing

| ID | Test | Expected result |
| --- | --- | --- |
| H01 | Login to Proxmox as named admin | MFA succeeds; no root required |
| H02 | SSH to Proxmox with password | Rejected |
| H03 | SSH to Debian with password | Rejected |
| H04 | SSH as root to Debian | Rejected |
| H05 | Access Proxmox 8006 from home | Allowed |
| H06 | Access Proxmox 8006 from unapproved source | Blocked |
| H07 | Access OPNsense GUI over HTTPS | Allowed from admin source |
| H08 | OPNsense GUI from WAN/Internet | Blocked |
| H09 | Home client resolves Pi-hole local name | Returns 10.0.99.10 |
| H10 | Home client queries Pi-hole DNS | Succeeds |
| H11 | Unapproved client queries Pi-hole DNS | Blocked |
| H12 | DMZ initiates connection to Management | Blocked |
| H13 | IoT or Gaming initiates connection to Storage | Blocked |
| H14 | Lab VM reaches Internet | Succeeds through OPNsense NAT |
| H15 | Pi-hole survives reboot | DNS and Web UI return |
| H16 | Proxmox survives reboot | OPNsense starts and lab returns to service |

```bat
:: Windows DNS verification
nslookup core-fw-01.intraspald.tech
nslookup pve-01.intraspald.tech
nslookup pihole-01.intraspald.tech

:: Windows route verification
route print
tracert 10.0.99.10
```

```bash
# Debian listening services
sudo ss -lntup

# Debian firewall
sudo nft list ruleset
```

## 10. Backup and Recovery Standard

At minimum, maintain: Proxmox configuration backups; OPNsense encrypted configuration backups; Pi-hole configuration/export backups; and a secure copy of administrator recovery material. Backups must not be stored only on the system they protect.

Before major firewall/network changes: export OPNsense configuration, record the current rule set, confirm console access, and test the recovery path. Before Proxmox networking changes: ensure HPE iLO/physical console access. Before Debian firewall changes: keep an existing SSH session open and test a second session after applying the rule.

*A backup that has never been restored is not a proven recovery mechanism. Periodically test restoration in a controlled maintenance window.*

## 11. Ongoing Operations

| Frequency | Activity |
| --- | --- |
| Daily/automated | Monitor failed services, security updates and critical logs |
| Weekly | Review updates and security advisories; check backup success |
| Monthly | Review firewall rules, admin accounts, listening ports and DNS configuration |
| Quarterly | Test restore/recovery procedures and review network segmentation |
| After every major change | Record change, backup configuration, run acceptance tests |
| Annually | Review architecture, trust boundaries, credentials, certificates and documented exceptions |

Maintain a change record for firewall exceptions, port forwards, new management accounts, new public-facing services and changes to the DNS trust model.

## 12. Final Hardening Checklist

- [ ] Proxmox has a named administrative account and MFA enabled.
- [ ] Proxmox root SSH login is disabled.
- [ ] Proxmox password SSH authentication is disabled.
- [ ] Proxmox Web UI is not exposed to the Internet.
- [ ] Proxmox firewall rules restrict management sources.
- [ ] OPNsense has named administrators and GUI MFA.
- [ ] OPNsense Web UI uses HTTPS.
- [ ] OPNsense SSH is disabled unless required; if enabled, it is key-only and restricted.
- [ ] OPNsense default-deny segmentation is implemented.
- [ ] Pi-hole Debian 13 is fully updated.
- [ ] Debian unattended security updates are enabled.
- [ ] Pi-hole root SSH is disabled and administrator SSH is key-only.
- [ ] Pi-hole host firewall restricts DNS, HTTPS and SSH to approved sources.
- [ ] Pi-hole is not publicly exposed.
- [ ] Pi-hole uses a trusted internal certificate where practical.
- [ ] Home DNS traffic to Pi-hole is explicitly permitted.
- [ ] DMZ cannot initiate connections into Management/Home by default.
- [ ] Gaming and IoT cannot access Storage/Management by default.
- [ ] Configuration backups exist for all three platforms.
- [ ] Acceptance tests H01–H16 have passed or deviations are documented.

**Security status: IMPLEMENTATION BASELINE** — hardening should be completed before additional Internet-facing services are deployed.

## Appendix A — Example Administrative Port Matrix

| Service | Port | Source | Destination | Action |
| --- | --- | --- | --- | --- |
| Proxmox Web UI | TCP 8006 | HOME_NET/MGMT_NET | 192.168.68.10 | Allow |
| Proxmox SSH | TCP 22 | Admin source only | 192.168.68.10 | Allow if required |
| OPNsense Web UI | TCP 443 | HOME_NET/MGMT_NET | 10.0.99.1 | Allow |
| OPNsense SSH | TCP 22 | Admin source only | 10.0.99.1 | Deny unless required |
| Pi-hole DNS | TCP/UDP 53 | HOME_NET + approved VLANs | 10.0.99.10 | Allow |
| Pi-hole Web UI | TCP 443 | Admin source only | 10.0.99.10 | Allow |
| Pi-hole SSH | TCP 22 | Admin source only | 10.0.99.10 | Allow if required |
| All management | Any | Internet/WAN | Lab management | Deny |

*Do not expose the listed management services through ISP-router port forwarding.*

## Appendix B — Security Notes and References

This standard is deliberately conservative. It favours explicit allowlists and layered controls over obscurity. Changing an SSH port alone is not considered a security control equivalent to disabling password authentication, restricting source networks and using keys.

Vendor documentation consulted for this standard includes official OPNsense documentation covering administration, HTTPS, SSH, 2FA and firewall behaviour, and official Pi-hole documentation covering interface binding and TLS. Debian and Proxmox procedures should be checked against the exact installed release before applying repository or package-management changes.

<img src="../assets/spaldotech-logo.png" alt="Spaldotech Operations" width="72">

> Snapshot of the live document in Claude. Edit the [live doc](https://claude.ai/artifact/GhSfziQV6caA5WJ81MhQHe), then re-export. PDF: [SPO-HL-SIEM-001-central-siem-core-siem-01-build-guide.pdf](../pdf/SPO-HL-SIEM-001-central-siem-core-siem-01-build-guide.pdf)

> **22 October 2026:** network addresses in this Markdown were updated to the as-built design (MGMT 10.0.99.0/24, Pi-hole 10.0.99.10, home 192.168.68.0/22). The PDF and live doc have not been re-exported yet.

# Spaldotech Operations — Central SIEM (core-siem-01) Build Guide

Oct 18, 2026 · @Aidan Spalding

core-siem-01 is an Ubuntu 24.04 VM on a new MONITORING VLAN (60, 10.0.60.10) running Wazuh 4.14 as a single all-in-one node. Every VM and the Proxmox host run a Wazuh agent; OPNsense and Suricata send syslog. The SIEM raises alerts by email and webhook, and its dashboard is reachable only from admin sources.

| Field | Value |
| --- | --- |
| Document ID | SPO-HL-SIEM-001 |
| Version | 1.0 |
| Status | Draft — implementation baseline |
| Date | 18 October 2026 |
| Owner | Spaldotech Operations |
| Classification | Internal / Technical |
| Related standards | SPO-HL-SEC-001 §5.8 and §11, Technical Implementation Guide §12, SPO-HL-PRX-001 (Caddy proxy) |

**Scope:** the core-siem-01 VM, the MONITORING VLAN and its firewall rules, Wazuh server components, agents on all lab hosts, OPNsense and Suricata log forwarding, detection content and alerting. Network flow capture (packet-level IDS beyond Suricata) is out of scope.

## 1. Purpose and design

The SIEM gives one place to see what is happening across every lab host, and raises an alert when something needs action. Today each VM keeps its own logs, so an attack on the proxy would only be noticed by someone reading its journal.

> **Diagram:** log and alert flow · sources, SIEM, destinations. See the [PDF version](../pdf/SPO-HL-SIEM-001-central-siem-core-siem-01-build-guide.pdf) or the [live doc](https://claude.ai/artifact/GhSfziQV6caA5WJ81MhQHe).

Each Linux host runs a Wazuh agent that sends its logs and file-integrity events to core-siem-01. OPNsense, which runs no agent, sends firewall and Suricata events over syslog. The SIEM decodes everything, matches it against rules, and sends the serious alerts out by email and webhook.

The design keeps the principles of the existing standards:

- **Data only flows inward:** agents open the connection to the SIEM. The SIEM cannot open connections into any other segment, so a compromised SIEM is not a route into the lab.
- **Management stays private:** the dashboard has no port forward and no public DNS record. It is reached from admin sources or over WireGuard, like Proxmox and OPNsense.
- **Secrets stay out of the SIEM:** file monitoring reports changes to files that hold credentials, but never stores their contents (§9.1).
- **No automated blocking:** Wazuh's active response is left off. In a single-host lab, an automatic block could cut off management of the very systems it protects.

Wazuh is the platform because one agent covers log collection, file integrity, configuration checks and vulnerability detection, which suits a lab of this size. Its central components need about 8 GB of RAM, the main cost of this design.

## 2. Build parameters

The SIEM follows the Naming Convention Standard (§3.2) and gets its own VLAN, as SPO-HL-SEC-001 §5.8 proposes. Adjust the VM ID and bridge name to match your host.

| Parameter | Value |
| --- | --- |
| VM name / hostname | core-siem-01 |
| FQDN (inventory) | core-siem-01.intraspald.tech |
| Friendly internal name | siem.intraspald.tech (CNAME) |
| Proxmox VM ID | 160 (suggested) |
| OS | Ubuntu Server 24.04 LTS, minimal. Documented exception to the Debian baseline: Wazuh does not list Debian 13 for its central components |
| vCPU / RAM | 4 vCPU / 8 GB, ballooning off |
| Disk | 100 GB (VirtIO SCSI, discard on). Wazuh's floor is 50 GB for up to 25 agents and 90 days of alerts |
| VLAN | 60 (MONITORING), new |
| Subnet / gateway | 10.0.60.0/24 / 10.0.60.1 (OPNsense) |
| Address | 10.0.60.10/24 (static) |
| DNS | Pi-hole 10.0.99.10 |
| Wazuh version | 4.14.x (Wazuh 5.0 is still in beta; do not use it here) |
| Backup job | bkp-core-siem-daily |

### Port matrix

| Service | Port | Source | Notes |
| --- | --- | --- | --- |
| Agent events | TCP 1514 | Every host with an agent | Agent-initiated; the SIEM never connects to agents |
| Agent enrolment | TCP 1515 | Every host with an agent | Password-protected (§6.3) |
| Syslog | UDP 514 | OPNsense only (10.0.60.1) | Firewall and Suricata events |
| Dashboard | TCP 443 | ADMIN\_SRC only | Over WireGuard when remote |
| SSH | TCP 22 | ADMIN\_SRC only | Key-only |
| Indexer API | TCP 9200 | Local only | Blocked at the host firewall |
| Server API | TCP 55000 | Local only | Used by the dashboard on the same host |
| Egress | TCP 443 | SIEM to Internet | Vulnerability feed, package updates, webhook alerts |
| Egress | TCP 587 | SIEM to mail provider | Email alerts via a local relay (§10.1) |
| Egress | TCP/UDP 53, UDP 123 | SIEM to Pi-hole, OPNsense | DNS and time |

## 3. Prerequisites

Complete these before creating the VM. Memory is the gate most likely to fail.

- [ ] At least 9 GB of RAM free on the Proxmox host after all current guests are running (`free -h` on the host, or Datacenter → Summary). If not, shrink another guest first; the Wazuh indexer performs badly when starved.
- [ ] 100 GB free in the VM storage pool, keeping the 20% ZFS headroom from the Technical Implementation Guide §2.2.
- [ ] OPNsense configuration exported and Proxmox configuration backed up (SPO-HL-SEC-001 §3).
- [ ] VLAN 60 allowed on the trunk to the Proxmox bridge, if the switch filters VLANs.
- [ ] Ubuntu Server 24.04 LTS ISO in Proxmox local storage, checksum verified.
- [ ] An alert destination decided: a mailbox for email alerts, a webhook (Slack, Discord or similar) for urgent ones, or both.
- [ ] SMTP credentials for that mailbox, if using email, stored in the password manager.
- [ ] Pi-hole local DNS records added: `core-siem-01.intraspald.tech` → 10.0.60.10, and `siem.intraspald.tech` as a CNAME to it.

## 4. Network: MONITORING VLAN and firewall rules

Agents always connect inward to the SIEM, and the SIEM never needs to open a connection to a monitored host. That lets every VLAN, including the DMZ, reach it on two ports only, while the MONITORING VLfN itself reaches nothing private except DNS and time.

### 4.1 Create VLAN 60 on OPNsense

1. Interfaces → Devices → VLAN: add parent = the LAN trunk interface, tag 60, description `MONITORING`.
2. Interfaces → Assignments: assign the new VLAN device, enable it, name it `MONITORING`.
3. Set IPv4 to static, 10.0.60.1/24. Leave DHCP off; the only host is static.
4. Leave "Block private networks" and "Block bogon networks" unticked on this internal interface.

### 4.2 Aliases

| Alias | Type | Content |
| --- | --- | --- |
| SIEM\_HOST | Host | 10.0.60.10 |
| MON\_NET | Network | 10.0.60.0/24 |
| WAZUH\_AGENT\_PORTS | Port | 1514, 1515 |
| PIHOLE | Host | 10.0.99.10 |
| PVE\_HOST | Host | 192.168.68.10 |

ADMIN\_SRC and RFC1918 already exist from SPO-HL-PRX-001 §10.1.

### 4.3 Rules into the SIEM

Add one pass rule on each interface that has agents behind it. On interfaces that end in an RFC 1918 block, place it above that block.

| Interface | Source | Destination | Port | Purpose |
| --- | --- | --- | --- | --- |
| MGMT | MGMT\_NET | SIEM\_HOST | TCP WAZUH\_AGENT\_PORTS | Pi-hole and other MGMT hosts |
| DMZ | DMZ\_NET | SIEM\_HOST | TCP WAZUH\_AGENT\_PORTS | Proxy and web VM. Insert above rule 6 in SPO-HL-PRX-001 §10.2 |
| Storage | STORAGE\_NET | SIEM\_HOST | TCP WAZUH\_AGENT\_PORTS | Plex, NFS and similar |
| Gaming / IoT / LAN | that interface's net | SIEM\_HOST | TCP WAZUH\_AGENT\_PORTS | Each segment that has VMs |
| WAN (home side, routed) | PVE\_HOST | SIEM\_HOST | TCP WAZUH\_AGENT\_PORTS | The Proxmox host's agent |
| MGMT and WAN (home side) | ADMIN\_SRC | SIEM\_HOST | TCP 443, 22 | Dashboard and SSH |

OPNsense's own syslog to the SIEM leaves from the firewall itself, so it needs no pass rule.

### 4.4 MONITORING interface rules (the SIEM's egress)

| # | Action | Proto | Source | Destination | Port | Purpose |
| --- | --- | --- | --- | --- | --- | --- |
| 1 | Pass | UDP | SIEM\_HOST | This Firewall | 123 | NTP |
| 2 | Pass | TCP/UDP | SIEM\_HOST | PIHOLE | 53 | DNS |
| 3 | Block + log | any | MON\_NET | This Firewall | any | No access to the firewall GUI or SSH |
| 4 | Block + log | any | MON\_NET | RFC1918 | any | The SIEM never reaches into other segments |
| 5 | Pass | TCP | SIEM\_HOST | any | 443 | Vulnerability feed, packages, webhook alerts |
| 6 | Pass | TCP | SIEM\_HOST | any | 587 | Email alerts via the mail provider |
| 7 | Block + log | any | MON\_NET | any | any | Default deny |

Replies to agent connections leave through the firewall's state table, so rule 4 does not interfere with them.

### 4.5 Host firewalls on monitored VMs

VMs with a default-drop egress policy must also allow the agent out. On core-proxy-01, add this line to the `output` chain in `/etc/nftables.conf`, above the `$PRIVATE` log and drop lines, then reload with the dead-man switch from SPO-HL-PRX-001 §11.1:

```nft
    ip daddr 10.0.60.10 tcp dport { 1514, 1515 } accept
```

Do the same on any other VM whose firewall restricts outbound traffic.

## 5. Create the VM and harden Ubuntu

### 5.1 Create the VM

Run from the Proxmox shell. Replace `vmbr0`, `vm-storage` and the ISO filename with the names on your host.

```bash
qm create 160 \
  --name core-siem-01 \
  --ostype l26 --machine q35 \
  --cores 4 --cpu x86-64-v2-AES --memory 8192 --balloon 0 \
  --scsihw virtio-scsi-single \
  --scsi0 vm-storage:100,discard=on,iothread=1,ssd=1 \
  --ide2 local:iso/ubuntu-24.04-live-server-amd64.iso,media=cdrom \
  --boot 'order=scsi0;ide2' \
  --net0 virtio,bridge=vmbr0,tag=60,firewall=1 \
  --agent enabled=1 \
  --onboot 1 --startup order=2,up=60 \
  --tags 'core;monitoring;siem'
```

Startup order 2 brings the SIEM up straight after OPNsense, so it is already listening when the other guests boot and their start-up events are captured.

### 5.2 Proxmox guest firewall

Create `/etc/pve/firewall/160.fw`:

```ini
[OPTIONS]
enable: 1
policy_in: DROP
policy_out: ACCEPT
macfilter: 1

[IPSET agents]
10.0.0.0/16 # all lab VLANs
192.168.68.10 # Proxmox host

[IPSET admin_src]
10.0.99.0/24 # MGMT_NET
192.168.68.50 # admin workstation - replace with yours

[RULES]
IN ACCEPT -source +agents -p tcp -dport 1514:1515 -log nolog # Wazuh agents
IN ACCEPT -source 10.0.60.1 -p udp -dport 514 -log nolog # OPNsense syslog
IN HTTPS(ACCEPT) -source +admin_src -log nolog # dashboard
IN SSH(ACCEPT) -source +admin_src -log nolog
IN Ping(ACCEPT) -source +admin_src -log nolog
```

### 5.3 Install Ubuntu Server 24.04

1. Choose **Ubuntu Server (minimized)**.
2. Network: manual IPv4, 10.0.60.10/24, gateway 10.0.60.1, name server 10.0.99.10, search domain `intraspald.tech`.
3. Storage: use the entire disk with LVM. Expand the root logical volume to the full disk when the installer offers it; it defaults to half.
4. Profile: hostname `core-siem-01`, user `sysadmin`.
5. Tick **Install OpenSSH server**. Import your SSH key if offered. Install no featured snaps.
6. After reboot, detach the ISO (`qm set 160 --ide2 none,media=cdrom`) and snapshot: `qm snapshot 160 base-install`.

### 5.4 Base hardening

```bash
sudo apt update && sudo apt full-upgrade -y
sudo apt install -y qemu-guest-agent unattended-upgrades curl gnupg apt-transport-https
sudo systemctl enable --now qemu-guest-agent

# Time from OPNsense
sudo mkdir -p /etc/systemd/timesyncd.conf.d
printf '[Time]\nNTP=10.0.60.1\n' | sudo tee /etc/systemd/timesyncd.conf.d/opnsense.conf
sudo systemctl restart systemd-timesyncd && timedatectl timesync-status
```

The MONITORING VLAN allows only TCP 443 outbound. If `apt update` fails to reach the Ubuntu mirrors over plain HTTP, change the `URIs:` lines in `/etc/apt/sources.list.d/ubuntu.sources` to `https://`. If a mirror then rejects HTTPS, add a narrow TCP 80 rule for the SIEM instead.

Apply the same SSH drop-in as the proxy (SPO-HL-PRX-001 §5.2): `sshadmins` group, key-only, no root, no forwarding. Test a second session before closing the first.

### 5.5 Host firewall (ufw)

Ubuntu ships ufw, so use it here rather than raw nftables. The Wazuh indexer (9200) and server API (55000) stay closed to the network by the default deny.

```bash
sudo ufw default deny incoming
sudo ufw default allow outgoing
sudo ufw allow from 10.0.0.0/16   to any port 1514,1515 proto tcp
sudo ufw allow from 192.168.68.10 to any port 1514,1515 proto tcp
sudo ufw allow from 10.0.60.1     to any port 514 proto udp
sudo ufw allow from 10.0.99.0/24  to any port 22,443 proto tcp
sudo ufw allow from 192.168.68.50 to any port 22,443 proto tcp   # admin workstation
sudo ufw enable
sudo ufw status numbered
```

## 6. Install Wazuh and secure it

Wazuh's installation assistant puts the indexer, server and dashboard on one host. That is the deployment Wazuh recommends for up to about 100 agents ([Wazuh quickstart](https://documentation.wazuh.com/current/quickstart.html)).

### 6.1 Run the installation assistant

```bash
cd ~
curl -sO https://packages.wazuh.com/4.14/wazuh-install.sh
sudo bash ./wazuh-install.sh -a
```

It takes several minutes and ends with "Installation finished", printing the dashboard address, the user `admin` and its generated password. Then retrieve the passwords for the internal indexer and API users:

```bash
sudo tar -O -xvf wazuh-install-files.tar wazuh-install-files/wazuh-passwords.txt
```

Store all of them in the password manager. `wazuh-install-files.tar` also holds the cluster's certificates and private keys: copy it into the encrypted config archive (§11.2), then remove it from the home directory with `shred -u wazuh-install-files.tar`.

### 6.2 Pin the version

An unplanned upgrade of one component can leave the others incompatible, so Wazuh recommends disabling its repository after installation. Hold the packages too:

```bash
sudo sed -i 's/^deb /#deb /' /etc/apt/sources.list.d/wazuh.list
sudo apt-mark hold wazuh-indexer wazuh-manager wazuh-dashboard
sudo apt update
```

Upgrades are then deliberate, following §11.3.

### 6.3 Require a password for agent enrolment

Without this, anything that can reach TCP 1515 can register itself as an agent and fill the SIEM with noise. Generate a password, store it in the password manager, and configure the manager:

```bash
sudo bash -c 'openssl rand -base64 24 > /var/ossec/etc/authd.pass'
sudo chmod 640 /var/ossec/etc/authd.pass
sudo chown root:wazuh /var/ossec/etc/authd.pass
```

In `/var/ossec/etc/ossec.conf`, inside the existing `<auth>` section, set:

```xml
<use_password>yes</use_password>
```

Then `sudo systemctl restart wazuh-manager` ([Wazuh: password authentication](https://documentation.wazuh.com/current/user-manual/agent-enrollment/security-options/using-password-authentication.html)).

### 6.4 Dashboard access

1. Browse to https://siem.intraspald.tech from an admin source and sign in as `admin`. The browser warns about the self-signed certificate; that is expected for now.
2. Keep `admin` as the break-glass account. Its password lives only in the password manager.
3. When the internal CA from SPO-HL-SEC-001 §5.2 is available, issue a certificate for `siem.intraspald.tech` and replace the dashboard's self-signed one.

### 6.5 Check the services

```bash
systemctl is-active wazuh-indexer wazuh-manager wazuh-dashboard   # three lines of "active"
sudo ss -lntup | grep -E ':(443|514|1514|1515|9200|55000)\b'
sudo ufw status   # 9200 and 55000 must not appear as allowed
```

## 7. Agent rollout

Every Linux host gets the same agent, configured centrally through groups on the manager. The manager monitors itself as agent 000, so core-siem-01 needs no agent.

### 7.1 Create the agent groups

Groups carry the shared configuration in §9. Create them on the manager before any agent enrols:

```bash
for g in linux proxmox dns dmz-proxy dmz-web; do
  sudo /var/ossec/bin/agent_groups -a -g "$g" -q
done
sudo /var/ossec/bin/agent_groups -l
```

| Host | Address | Groups | Notes |
| --- | --- | --- | --- |
| pve-01 (Proxmox host) | 192.168.68.10 | linux, proxmox | Debian-based; enrol last, once the method is proven |
| pihole-01 | 10.0.99.10 | linux, dns |  |
| core-proxy-01 | 10.0.40.10 | linux, dmz-proxy | Open the nftables egress first (§4.5) |
| Web VM | 10.0.40.20 | linux, dmz-web |  |
| Plex, NFS, AMP | as built | linux, plus a role group | Add when each VM is built |

### 7.2 Install the agent on each host

Run on the host being enrolled. The password is typed once and never written to the shell history:

```bash
# Repository
sudo apt-get install -y gnupg apt-transport-https curl
curl -s https://packages.wazuh.com/key/GPG-KEY-WAZUH \
  | sudo gpg --no-default-keyring --keyring gnupg-ring:/usr/share/keyrings/wazuh.gpg --import
sudo chmod 644 /usr/share/keyrings/wazuh.gpg
echo 'deb [signed-by=/usr/share/keyrings/wazuh.gpg] https://packages.wazuh.com/4.x/apt/ stable main' \
  | sudo tee /etc/apt/sources.list.d/wazuh.list
sudo apt-get update

# Install and enrol (set the groups for this host from the table above)
read -rs -p 'Wazuh enrolment password: ' WP; echo
sudo WAZUH_MANAGER='10.0.60.10' \
     WAZUH_REGISTRATION_PASSWORD="$WP" \
     WAZUH_AGENT_GROUP='linux,dmz-proxy' \
     apt-get install -y wazuh-agent
unset WP

sudo systemctl daemon-reload
sudo systemctl enable --now wazuh-agent

# Pin the agent to the manager's version
sudo sed -i 's/^deb /#deb /' /etc/apt/sources.list.d/wazuh.list
sudo apt-mark hold wazuh-agent
sudo apt-get update
```

An agent must never be newer than the manager, which is why the agent is held as well. The installation variables are described in Wazuh's [Linux deployment variables](https://wazuh-documentation-49-master.readthedocs.io/en/latest/user-manual/agent/agent-enrollment/deployment-variables/deployment-variables-linux.html) reference.

### 7.3 Confirm enrolment

On the manager:

```bash
sudo /var/ossec/bin/agent_control -l     # each host listed as Active
```

On the agent, `sudo tail -n 20 /var/ossec/logs/ossec.log` should show "Connected to the server". If it shows enrolment errors, check the password, then the firewall path on TCP 1515.

### 7.4 The Proxmox host

The agent runs on the hypervisor like any Debian host. Two cautions:

- `/etc/pve` is Proxmox's cluster filesystem, not ordinary disk. Monitor it on a schedule (§9.1), never with real-time file integrity monitoring.
- Do not let any active response from Wazuh act on the Proxmox host. Section 9 enables none; leave it that way, so an automated block can never cut off hypervisor management.

## 8. OPNsense and Suricata over syslog

OPNsense runs no agent. It sends selected logs to the SIEM over syslog instead, which covers firewall blocks, Suricata alerts, configuration changes and logins without adding software to the firewall.

### 8.1 Accept syslog on the manager

Add this block inside `<ossec_config>` in `/var/ossec/etc/ossec.conf`, alongside the existing `<remote>` block for agents:

```xml
<remote>
  <connection>syslog</connection>
  <port>514</port>
  <protocol>udp</protocol>
  <allowed-ips>10.0.60.1/32</allowed-ips>
  <local_ip>10.0.60.10</local_ip>
</remote>
```

Then `sudo systemctl restart wazuh-manager`. Only the firewall's MONITORING address may send, matching the ufw rule in §5.5.

### 8.2 Send from OPNsense

1. System → Settings → Logging → Remote: add a destination.
2. Transport UDP(4), hostname 10.0.60.10, port 514. Leave RFC 5424 unticked; Wazuh decodes the classic format more reliably.
3. Applications: select firewall (filterlog), Suricata, audit, and the web GUI and SSH entries. Leave the rest out to keep volume down.
4. Levels: informational and above.
5. Services → Intrusion Detection → Administration: tick **Enable eve syslog output**, so Suricata alerts reach syslog as JSON.

Only firewall rules with logging ticked produce events. The default block rules already log; log pass rules only where an audit trail is wanted, or the SIEM fills with routine traffic.

### 8.3 Check events arrive and decode

```bash
sudo tcpdump -ni any -c 5 udp port 514   # packets from 10.0.60.1
```

Then, in the dashboard under Threat Hunting, filter on the OPNsense agent location (`10.0.60.1`) and confirm firewall blocks show decoded source, destination and port fields. Wazuh's stock ruleset includes decoders for pfSense's filterlog format, which OPNsense shares; if the fields stay undecoded, treat it like the Suricata case below.

If Suricata events arrive with the JSON left undecoded, add this decoder to `/var/ossec/etc/decoders/local_decoder.xml` and restart the manager:

```xml
<decoder name="opnsense-suricata">
  <program_name>suricata</program_name>
  <plugin_decoder>JSON_Decoder</plugin_decoder>
</decoder>
```

With the JSON decoded, Wazuh's built-in Suricata rules raise alerts from the `alert` events.

## 9. Detection content

Each group's configuration lives on the manager in `/var/ossec/etc/shared/<group>/agent.conf` and is pushed to its agents automatically. Custom rules go in `/var/ossec/etc/rules/local_rules.xml`. After any change, check the syntax, then restart the manager:

```bash
sudo /var/ossec/bin/verify-agent-conf
sudo systemctl restart wazuh-manager
```

### 9.1 Group configuration

**linux** (every Linux host). Debian 13 has no `/var/log/auth.log` by default, so SSH and sudo events exist only in the journal. Add the journald source unless the agent's own `ossec.conf` already reads it, or events will arrive twice:

```xml
<agent_config>
  <localfile>
    <log_format>journald</log_format>
    <location>journald</location>
  </localfile>
  <syscheck>
    <directories check_all="yes" realtime="yes" report_changes="yes">/etc/ssh,/etc/sudoers,/etc/sudoers.d</directories>
    <directories check_all="yes" realtime="yes">/etc/nftables.conf,/etc/passwd,/etc/group,/etc/shadow</directories>
  </syscheck>
</agent_config>
```

`report_changes` stores a diff of each change in the SIEM. It is deliberately off for `/etc/shadow`, so password hashes are never copied there.

**dmz-proxy** (core-proxy-01):

```xml
<agent_config>
  <syscheck>
    <directories check_all="yes" realtime="yes" report_changes="yes">/etc/caddy,/usr/local/sbin/update-cf-ips</directories>
    <nodiff>/etc/caddy/caddy.env</nodiff>
    <directories check_all="yes" realtime="yes">/usr/bin/caddy.custom</directories>
  </syscheck>
  <localfile>
    <log_format>json</log_format>
    <location>/var/log/caddy/access.log</location>
  </localfile>
</agent_config>
```

`nodiff` keeps the Cloudflare API token out of the SIEM: a change to `caddy.env` still alerts, but its contents are never stored.

**proxmox** (pve-01). `/etc/pve` is checked on the default 12-hour schedule, not in real time (§7.4):

```xml
<agent_config>
  <syscheck>
    <directories check_all="yes">/etc/pve/firewall,/etc/pve/user.cfg,/etc/pve/storage.cfg</directories>
    <directories check_all="yes" realtime="yes" report_changes="yes">/etc/network/interfaces</directories>
  </syscheck>
</agent_config>
```

**dns** (pihole-01):

```xml
<agent_config>
  <syscheck>
    <directories check_all="yes" realtime="yes" report_changes="yes">/etc/pihole/pihole.toml,/etc/dnsmasq.d</directories>
  </syscheck>
</agent_config>
```

### 9.2 Custom rules

Add to `/var/ossec/etc/rules/local_rules.xml`. Custom rule IDs start at 100000.

```xml
<group name="caddy,web,">
  <rule id="100100" level="0">
    <decoded_as>json</decoded_as>
    <field name="logger">^http.log.access</field>
    <description>Caddy access log entry</description>
  </rule>
  <rule id="100101" level="3">
    <if_sid>100100</if_sid>
    <field name="status">^4\d\d$</field>
    <description>Caddy: client error $(status) on $(request.host)</description>
  </rule>
  <rule id="100102" level="10" frequency="20" timeframe="60">
    <if_matched_sid>100101</if_matched_sid>
    <same_field>request.client_ip</same_field>
    <description>Caddy: 20 or more client errors in 60 s from one visitor (likely scanning)</description>
  </rule>
</group>

<group name="proxmox,authentication_failed,">
  <rule id="100110" level="5">
    <program_name>pvedaemon</program_name>
    <match>authentication failure</match>
    <description>Proxmox: failed web UI login</description>
  </rule>
  <rule id="100111" level="10" frequency="5" timeframe="300">
    <if_matched_sid>100110</if_matched_sid>
    <description>Proxmox: 5 failed web UI logins in 5 minutes</description>
  </rule>
</group>
```

Test each rule against a sample line before relying on it: run `sudo /var/ossec/bin/wazuh-logtest`, paste a real Caddy access-log line or the Proxmox journal line for a failed login, and confirm the expected rule ID fires.

### 9.3 Built-in capabilities to confirm

| Capability | Where to check | Notes |
| --- | --- | --- |
| SSH brute force | Threat Hunting | Stock rules fire on repeated failed SSH logins once journald is read |
| Configuration assessment | Configuration Assessment, per agent | Runs CIS-style checks by default. If a host shows no policy, Wazuh has none for that OS release yet |
| Vulnerability detection | Vulnerability Detection | On by default; needs the SIEM's TCP 443 egress to Wazuh's feed |
| Active response | — | Deliberately left off. Automated blocking on a single-host lab risks locking out management |

## 10. Alerting

Every Wazuh rule has a level from 0 to 15. Three bands decide where an alert goes, so routine noise stays on the dashboard and only things worth acting on interrupt you.

| Level | Destination | Typical events |
| --- | --- | --- |
| 12 and above | Webhook (phone) and email | Critical file changed (§10.3), repeated Proxmox login failures escalated, high-severity Suricata alerts |
| 10–11 | Email | SSH brute force, web scanning (rule 100102), Proxmox login bursts (100111), multiple firewall blocks |
| Below 10 | Dashboard only | Single failed logins, routine file changes, package installs |

### 10.1 Email

Wazuh sends mail to an SMTP server without authentication, so a local Postfix relay handles the login to your mail provider:

```bash
sudo DEBIAN_FRONTEND=noninteractive apt-get install -y postfix libsasl2-modules
sudo postconf -e 'inet_interfaces = loopback-only' \
  'relayhost = [smtp.your-provider.example]:587' \
  'smtp_sasl_auth_enable = yes' \
  'smtp_sasl_security_options = noanonymous' \
  'smtp_sasl_password_maps = hash:/etc/postfix/sasl_passwd' \
  'smtp_tls_security_level = encrypt'
sudo install -m 0600 /dev/null /etc/postfix/sasl_passwd
sudoedit /etc/postfix/sasl_passwd
# [smtp.your-provider.example]:587 siem@spaldotechoperations.co.uk:<app password>
sudo postmap /etc/postfix/sasl_passwd
sudo systemctl restart postfix
```

Use an app password or a dedicated sending account, never your own mailbox password. Some providers disable SMTP authentication by default; check yours allows it for this account.

In `/var/ossec/etc/ossec.conf`, set these values in the existing `<global>` and `<alerts>` blocks:

```xml
<global>
  <email_notification>yes</email_notification>
  <smtp_server>localhost</smtp_server>
  <email_from>siem@spaldotechoperations.co.uk</email_from>
  <email_to>you@spaldotechoperations.co.uk</email_to>
  <email_maxperhour>12</email_maxperhour>
</global>

<alerts>
  <email_alert_level>10</email_alert_level>
</alerts>
```

`email_maxperhour` stops an incident from flooding the inbox; further alerts are batched.

### 10.2 Webhook to your phone

Wazuh's built-in Slack integration also works with Discord, which accepts Slack-format messages when `/slack` is appended to a Discord webhook URL. Add inside `<ossec_config>`:

```xml
<integration>
  <name>slack</name>
  <hook_url>https://discord.com/api/webhooks/XXXX/YYYY/slack</hook_url>
  <level>12</level>
  <alert_format>json</alert_format>
</integration>
```

For Slack itself, use the Slack incoming-webhook URL as it is. The webhook URL is a secret: anyone holding it can post to the channel.

### 10.3 Raise critical file changes to level 12

Wazuh reports file changes at level 7, below both alert bands. This rule lifts changes to the files that matter most into the phone band. Add it to `local_rules.xml`:

```xml
<group name="syscheck,critical_files,">
  <rule id="100120" level="12">
    <if_sid>550,553,554</if_sid>
    <field name="file">^/etc/caddy|^/usr/bin/caddy|^/etc/ssh|^/etc/nftables.conf|^/etc/pve|^/etc/sudoers</field>
    <description>Critical file changed on $(agent.name): $(file)</description>
  </rule>
</group>
```

Rules 550, 553 and 554 are Wazuh's file modified, deleted and added events. Restart the manager after saving, and prove it with test W08 in §12.

## 11. Retention, backup and updates

### 11.1 Retention: 90 days

Alerts are kept for 90 days, the period Wazuh's sizing assumes and the upper end of the Technical Implementation Guide §12.2 range. Two stores need limits.

**Indexer (what the dashboard searches).** In the dashboard, open Indexer management → Index Management → State management policies, create a policy named `wazuh-alerts-90d`, and paste:

```json
{
  "policy": {
    "description": "Delete Wazuh alert indices after 90 days",
    "default_state": "hot",
    "states": [
      {
        "name": "hot",
        "actions": [],
        "transitions": [{ "state_name": "delete", "conditions": { "min_index_age": "90d" } }]
      },
      {
        "name": "delete",
        "actions": [{ "delete": {} }],
        "transitions": []
      }
    ],
    "ism_template": [{ "index_patterns": ["wazuh-alerts-*"], "priority": 100 }]
  }
}
```

The template applies to indices created from now on. Attach the policy to any existing `wazuh-alerts-*` indices from the Indices page.

**Manager log files.** The manager also keeps compressed daily alert files on disk. Clear anything older than 90 days with a daily cron entry:

```bash
echo '30 3 * * * root find /var/ossec/logs/alerts /var/ossec/logs/archives -type f -mtime +90 -delete' \
  | sudo tee /etc/cron.d/wazuh-retention
```

Watch disk use. When the disk passes 95% the indexer makes its indices read-only and new alerts stop being stored, so treat 80% as the point to act.

### 11.2 Backup

- **VM backup:** Proxmox job `bkp-core-siem-daily` to PBS, retaining 7 daily and 4 weekly copies, with PBS encryption on. It captures the indexed alert history as well as the configuration.
- **Config archive:** after every change, take an encrypted copy and store it off the host. It contains the enrolment password, agent keys, certificates and the SMTP credentials:

```bash
sudo tar czf - /var/ossec/etc /etc/wazuh-indexer /etc/wazuh-dashboard \
  /etc/postfix/main.cf /etc/postfix/sasl_passwd /etc/cron.d/wazuh-retention \
  | gpg --symmetric --cipher-algo AES256 -o core-siem-01-config-$(date +%F).tar.gz.gpg
```

Add the `wazuh-install-files.tar` from §6.1 to the first archive.

### 11.3 Updates

Ubuntu's own packages update through unattended-upgrades. Wazuh is held (§6.2) and upgraded deliberately, at most quarterly unless a security advisory says otherwise:

1. Read the release notes for the target version, and snapshot the VM: `qm snapshot 160 pre-wazuh-upgrade`.
2. Re-enable the repository and release the holds.
3. Upgrade in Wazuh's order: indexer, then manager, then dashboard. Confirm each service is active before the next.
4. Upgrade the agents only after the manager. An agent must never run a newer version than the manager.
5. Re-apply the holds, disable the repository, and run the tests in §12.

Wazuh 5.0 is a major release and will need its own plan; do not move to it as a routine upgrade.

## 12. Verification and acceptance tests

Each alerting test proves the whole chain: the event is collected, a rule fires, and the alert reaches the right destination. Record results alongside SPO-HL-SEC-001 tests H01–H16 and SPO-HL-PRX-001 tests P01–P23.

| ID | Test | How | Expected |
| --- | --- | --- | --- |
| W01 | Services running | `systemctl is-active wazuh-indexer wazuh-manager wazuh-dashboard postfix` | All active |
| W02 | Dashboard from admin source | Browse https://siem.intraspald.tech from MGMT or the admin workstation | Login page loads |
| W03 | Dashboard from other sources | Same URL from a non-admin home device | Times out |
| W04 | Internal APIs closed | From MGMT: `nc -zvw3 10.0.60.10 9200`, then port 55000 | Both time out |
| W05 | All agents active | `sudo /var/ossec/bin/agent_control -l` on the manager | Every host from §7.1 listed as Active |
| W06 | Enrolment needs the password | On a throwaway VM: `sudo /var/ossec/bin/agent-auth -m 10.0.60.10` with no password | Enrolment refused |
| W07 | SIEM cannot reach other segments | From core-siem-01: `nc -zvw3 10.0.40.10 22` and `nc -zvw3 192.168.68.10 8006` | Both time out |
| W08 | Critical file change reaches the phone | On core-proxy-01: `sudo touch /etc/caddy/Caddyfile` | Rule 100120 at level 12; webhook and email arrive within a minute |
| W09 | Secrets never stored | `sudo touch /etc/caddy/caddy.env`, then open the event | Alert raised; no file contents or diff in the event |
| W10 | SSH brute force | From an admin source, 8 failed SSH logins to pihole-01 with a wrong user | Brute-force alert at level 10; email arrives |
| W11 | Web scanning | Externally: `for i in $(seq 25); do curl -s -o /dev/null https://www.spaldotechoperations.co.uk/nope$i; done` | Rule 100102 fires; email arrives; visitor IP is the test device |
| W12 | Proxmox login failures | 5 failed logins to the Proxmox web UI within 5 minutes | Rule 100111 fires; email arrives |
| W13 | Firewall blocks decoded | From core-proxy-01: `nc -zvw3 10.0.99.1 443` | Block event with decoded source, destination and port |
| W14 | Suricata alerts decoded | From the OPNsense shell: `curl -s http://testmynids.org/uid/index.html` | Suricata alert visible with decoded signature fields |
| W15 | Vulnerability results | Vulnerability Detection page, a few hours after enrolment | Each agent shows a package inventory and findings |
| W16 | Retention policy attached | Index Management → Indices | `wazuh-alerts-*` indices show the `wazuh-alerts-90d` policy |
| W17 | Survives reboot | Reboot core-siem-01, then repeat W01 and W05 | All services active; agents reconnect within 5 minutes |

## 13. Operations and sign-off

### 13.1 Adding a new VM

1. Choose its groups: `linux` plus a role group. Create the role group and its `agent.conf` first if it is new (§9.1).
2. Confirm the VM's VLAN has the agent rule into SIEM\_HOST (§4.3), and open its host firewall egress if it has one (§4.5).
3. Install and enrol the agent (§7.2).
4. Check it shows Active (W05), then make one test change to a monitored file to prove collection.

### 13.2 Routine operations

| Frequency | Activity |
| --- | --- |
| Daily, automated | Email and webhook alerts; retention cron |
| Weekly | Review the dashboard for level 7–9 events that never reached email; check disk use on core-siem-01 |
| Monthly | Review the vulnerability report and patch findings; check every agent is Active and none is disconnected |
| Quarterly | Run W08, W10 and W11 to prove alerting still works end to end; review custom rules for noise; consider a Wazuh upgrade (§11.3) |
| On every new service | Add its config paths to a group's file monitoring, and a rule if its logs need one |
| After every change | Config archive (§11.2), and the tests touched by the change |

Tune before you mute. When an alert fires too often, raise its threshold or narrow its rule in `local_rules.xml`, rather than lowering the email level for everything.

### 13.3 Sign-off checklist

- [ ] RAM headroom confirmed and VLAN 60 created with the rules in §4.
- [ ] core-siem-01 built per §5, with guest firewall and ufw active.
- [ ] Wazuh 4.14 installed, packages held and repository disabled.
- [ ] Every generated password and `wazuh-install-files.tar` stored off the host, then the tar removed.
- [ ] Enrolment password required.
- [ ] Agents enrolled on every host in §7.1, all Active and held.
- [ ] OPNsense syslog and Suricata eve output arriving and decoded.
- [ ] Group configurations and custom rules from §9 and §10.3 loaded and tested.
- [ ] Email relay and webhook delivering.
- [ ] 90-day retention on indexer and disk.
- [ ] Backup job running and first config archive stored off the host.
- [ ] Tests W01–W17 passed or deviations recorded.
- [ ] SPO-HL-SEC-001 updated: VLAN 60 added to §2 and §8, central logging marked implemented in §7, and the Ubuntu exception recorded.
- [ ] SPO-HL-PRX-001 updated: the agent egress line in §11 and the DMZ agent rule in §10.2.

### Sources

- [Wazuh quickstart: all-in-one install and hardware requirements](https://documentation.wazuh.com/current/quickstart.html)
- [Wazuh: enrol agents with password authentication](https://documentation.wazuh.com/current/user-manual/agent-enrollment/security-options/using-password-authentication.html)
- [Wazuh: installing the agent on Linux](https://documentation.wazuh.com/current/installation-guide/wazuh-agent/wazuh-agent-package-linux.html)
- [Wazuh: Linux deployment variables](https://wazuh-documentation-49-master.readthedocs.io/en/latest/user-manual/agent/agent-enrollment/deployment-variables/deployment-variables-linux.html)

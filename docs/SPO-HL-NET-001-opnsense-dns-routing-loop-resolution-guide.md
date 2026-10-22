<img src="../assets/spaldotech-logo.png" alt="Spaldotech Operations" width="72">

> Snapshot of the live document in Claude. Edit the [live doc](https://claude.ai/artifact/5hKG5F85RTaAnQ3dJfXiwf), then re-export. PDF: [SPO-HL-NET-001-opnsense-dns-routing-loop-resolution-guide.pdf](../pdf/SPO-HL-NET-001-opnsense-dns-routing-loop-resolution-guide.pdf)

# Spaldotech Operations — OPNsense DNS Routing Loop (10.0.99.10) Resolution Guide

Oct 22, 2026 · @Aidan Spalding

A host route for 10.0.99.10 on OPNsense overrode the route to the 10.0.99.0/24 network and sent DNS traffic back to the ISP router, creating a routing loop. Deleting the host route restored DNS. The permanent fix stops OPNsense learning DNS servers from the WAN's DHCP lease, which is what re-creates the route.

| Field | Value |
| --- | --- |
| Document ID | SPO-HL-NET-001 |
| Version | 1.0 |
| Status | Workaround applied 22 October 2026; permanent fix to confirm |
| Owner | Spaldotech Operations |
| Classification | Internal / Technical |
| Related standards | SPO-HL-PRX-001 (Caddy proxy and OPNsense) |

**Scope:** the OPNsense routing table, its General and Routes settings, and the static route for 10.0.0.0 on the ISP router. The DNS server at 10.0.99.10 itself is not changed.

## 1. Symptoms

Only 10.0.99.10 was unreachable; other 10.0.x hosts, such as 10.0.50.51, still worked. That rules out the ISP router's static route for 10.0.0.0.

- DNS lookups against 10.0.99.10 time out from the 192.168.68.0/24 network.
- `tracert 10.0.99.10` alternates between 192.168.68.1 (ISP router) and 192.168.68.62 (OPNsense) until it hits 30 hops:

```
  1     3 ms     3 ms     3 ms  192.168.68.1
  2     7 ms     4 ms     4 ms  192.168.68.62
  3     9 ms     7 ms     6 ms  192.168.68.1
  4    10 ms     8 ms     8 ms  192.168.68.62
  ...
```

A traceroute that bounces between two hops is a routing loop, not a firewall block or a stopped service.

## 2. Root cause

OPNsense held a /32 host route for 10.0.99.10 pointing at 192.168.68.62, its own WAN address. The most specific route always wins, so this one entry overrode the correct route for the whole 10.0.99.0/24 network, for that address only.

| Route on OPNsense | Matches | Next hop | Result |
| --- | --- | --- | --- |
| 10.0.99.10/32 (the fault) | 10.0.99.10 only | 192.168.68.62, OPNsense itself | Wins; goes nowhere |
| 10.0.99.0/24 | The DNS server's network | The correct interface | Ignored for 10.0.99.10 |
| default | Anything unmatched | 192.168.68.1 | Where the packet ended up |

With nowhere valid to go, the packet left via the default route to the ISP router. The ISP router's static route sends all of 10.0.0.0 back to OPNsense, and the two passed it back and forth until the TTL ran out.

> **Diagram:** the routing loop · 10.0.99.10 traffic between the ISP router and OPNsense. See the [PDF version](../pdf/SPO-HL-NET-001-opnsense-dns-routing-loop-resolution-guide.pdf) or the [live doc](https://claude.ai/artifact/5hKG5F85RTaAnQ3dJfXiwf).

### Where the host route comes from

The route is not in `/conf/config.xml`: the only match for 10.0.99.10 is the DNS server entry, which has no gateway set. That points to a route learned at runtime:

1. OPNsense's WAN takes its address by DHCP from the ISP router.
2. The ISP router's DHCP hands out 10.0.99.10 as the DNS server.
3. With **Allow DNS server list to be overridden by DHCP/PPP on WAN** ticked, OPNsense adopts that server and adds a host route to it through the WAN.

This also explains why it started without any change: a WAN DHCP renewal, reboot or gateway event re-adds the route. Confirm it with the checks in §4.1 before relying on it.

## 3. Immediate fix

Deleting the host route restores DNS at once. It is temporary: the route returns at the next WAN renewal, reboot or route reload until §4 is done.

From the OPNsense shell (console option 8, or SSH):

```bash
route -n get 10.0.99.10          # gateway: 192.168.68.62 = the fault is present
route delete -host 10.0.99.10
route -n get 10.0.99.10          # should now show the 10.0.99.x interface, not 192.168.68.x
```

Then re-run `tracert 10.0.99.10` from a client. It should pass 192.168.68.1, then 192.168.68.62, then reach 10.0.99.10.

## 4. Permanent fix

Stop OPNsense adopting DNS servers from the WAN's DHCP lease. The manually set 10.0.99.10 stays in use, without the host route.

### 4.1 Confirm the source

```bash
cat /tmp/*_nameserver*                      # DNS servers learned from DHCP
grep -n "dnsallowoverride" /conf/config.xml  # present = the override is on
```

If the first shows 10.0.99.10 and the second returns a line, the DHCP override is the source. If not, see §7.

### 4.2 Turn off the override

1. Go to **System → Settings → General**.
2. Untick **Allow DNS server list to be overridden by DHCP/PPP on WAN**.
3. Check 10.0.99.10 is still listed under DNS servers, with the gateway set to none.
4. Save.

### 4.3 Clear the route and prove it stays gone

```bash
route delete -host 10.0.99.10     # if it is back
route -n get 10.0.99.10
```

Then reboot OPNsense, so the WAN takes a fresh DHCP lease, and run `route -n get 10.0.99.10` again. It must still show the 10.0.99.x path.

Alternatives, if the override must stay on: give the OPNsense WAN a static address, or set a DHCP reservation on the ISP router that hands OPNsense a different DNS server.

## 5. Prevention

A blackhole route for 10.0.0.0/8 stops any future missing route turning into a loop. The real, more specific routes still win; when one goes missing, OPNsense drops the traffic instead of passing it back to the ISP router.

1. Go to **System → Routes → Configuration** and add a route.
2. Network address: **10.0.0.0/8**. Gateway: **Null4**. Description: `Blackhole unrouted 10.x`.
3. Save and apply.

Afterwards, a traceroute to an unreachable 10.x address stops at 192.168.68.62, which points straight at OPNsense.

Two habits keep the fault from coming back:

- Do not let the ISP router's DHCP hand internal DNS servers to OPNsense's WAN unless the override in §4.2 is off.
- Never use 10.0.99.10, or any internal host, as a gateway **Monitor IP**: monitoring adds the same kind of host route.

## 6. Verification tests

| ID | Test | How | Expected |
| --- | --- | --- | --- |
| N01 | No host route | `route -n get 10.0.99.10` on OPNsense | The 10.0.99.x interface, not 192.168.68.62 |
| N02 | Override off | `grep -n "dnsallowoverride" /conf/config.xml` | No output |
| N03 | Path is clean | `tracert 10.0.99.10` from a client | 192.168.68.1, 192.168.68.62, then 10.0.99.10 |
| N04 | DNS answers | `nslookup spaldotechoperations.co.uk 10.0.99.10` from a client | An answer from 10.0.99.10 |
| N05 | Other 10.x hosts | `ping 10.0.50.51` from a client | Replies |
| N06 | Blackhole in place | `route -n get 10.0.200.1` on OPNsense (any unused 10.x address) | Gateway Null4 / reject |
| N07 | Survives reboot | Reboot OPNsense, then N01, N03 and N04 | All pass |

## 7. Troubleshooting

Work from OPNsense's routing table outwards: `route -n get <address>` shows which route actually wins.

| Symptom | Cause | Fix |
| --- | --- | --- |
| Host route returns after a reboot | The override is still on, or the route has another source | Repeat §4.1; run `grep -n "10.0.99.10" /conf/config.xml` and check for `<monitor>` or `<staticroutes>` matches |
| `/tmp/*_nameserver*` is empty but the route returns | A gateway Monitor IP or a manual route | **System → Gateways → Configuration**: clear the Monitor IP or tick **Disable Host Route**. **System → Routes → Configuration**: remove any 10.0.99.10/32 |
| A gateway uses 192.168.68.62 as its address | Gateway pointed at OPNsense itself | Set it to the real next hop, never OPNsense's own address |
| Traceroute stops at 192.168.68.62 | Blackhole route doing its job: no route to that subnet | Check the subnet's interface under **Interfaces → Overview**, or its route under **System → Routes → Status** |
| DNS works from OPNsense but not clients | Client side or ISP router route | Check the ISP router's 10.0.0.0 static route still points at 192.168.68.62 |
| Clients lose all name resolution during a fault | 10.0.99.10 is their only DNS server | Add a secondary DNS server in the ISP router's DHCP settings |

## 8. Sign-off checklist

- [x] Host route for 10.0.99.10 deleted; DNS restored (§3).
- [ ] Source confirmed as the DHCP override (§4.1).
- [ ] Override turned off and saved (§4.2).
- [ ] Route still absent after a reboot (§4.3, N07).
- [ ] 10.0.0.0/8 → Null4 blackhole route added (§5).
- [ ] No gateway uses an internal host as its Monitor IP.
- [ ] N01–N07 pass.

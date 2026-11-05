# Cisco IOS ACL Cheat Sheet — Homelab

As of Nov 4, 2026

## Core rules

An ACL is read top-down, first match wins, and anything that matches nothing is dropped by an invisible `deny any` at the end.

- **One ACL per interface, per direction, per protocol.** Applying a second one replaces the first.
- **In vs out:** `in` = traffic arriving on that interface; `out` = traffic leaving it. On a router-on-a-stick subinterface, `in` means "from hosts in that VLAN."
- **Placement:** standard ACLs close to the destination (they only see source IP); extended ACLs close to the source (drop early).
- **Wildcard masks are inverted subnet masks:** 0 = must match, 1 = don't care. Wildcard = 255.255.255.255 − subnet mask.
- **Empty ACL applied to an interface = permit all.** An ACL with one entry gets the implicit deny.

| Subnet mask | Wildcard | Example |
| --- | --- | --- |
| /32 255.255.255.255 | 0.0.0.0 | `host 10.10.40.30` |
| /24 255.255.255.0 | 0.0.0.255 | `10.10.40.0 0.0.0.255` |
| /16 255.255.0.0 | 0.0.255.255 | `10.10.0.0 0.0.255.255` |
| /0 | 255.255.255.255 | `any` |

| Numbered range | Type |
| --- | --- |
| 1–99, 1300–1999 | Standard IP |
| 100–199, 2000–2699 | Extended IP |

## Standard ACLs

Standard ACLs match source IP only, so they're best for "who can reach this" rules like locking down SSH.

```
! Numbered
access-list 10 permit host 10.10.99.50
access-list 10 permit 10.10.10.0 0.0.0.255
access-list 10 deny any log

! Named (preferred — easier to edit)
ip access-list standard MGMT-ONLY
 permit 10.10.99.0 0.0.0.255
 permit 192.168.0.0 0.0.0.255
 deny any log
```

Remove a whole numbered ACL with `no access-list 10`; a named one with `no ip access-list standard MGMT-ONLY`.

## Extended ACLs

Extended ACLs match protocol, source, destination and ports — this is what you'll use for most inter-VLAN rules.

```
[permit|deny] <protocol> <src> <src-wildcard> [port-op port] <dst> <dst-wildcard> [port-op port] [established] [log]
```

```
ip access-list extended EXAMPLE
 permit tcp 10.10.50.0 0.0.0.255 host 10.10.60.10 eq 80
 permit tcp 10.10.50.0 0.0.0.255 host 10.10.60.10 eq 443
 permit udp any host 10.10.40.30 eq 53
 permit tcp any host 10.10.40.30 eq 53
 permit icmp any any echo
 permit icmp any any echo-reply
 permit tcp any any established
 deny ip any any log
```

| Keyword | Meaning |
| --- | --- |
| `eq 80` / `neq` / `lt` / `gt` | Port equals / not / less than / greater than |
| `range 6000 6010` | Port range |
| `established` | TCP with ACK or RST set (return traffic only) |
| `log` / `log-input` | Syslog a match (`log-input` adds ingress interface + MAC) |
| `object-group` | Reuse groups of hosts or ports (IOS-XE on the ISR) |

| Service | Match |
| --- | --- |
| SSH | `tcp eq 22` |
| DNS (Pi-hole) | `udp eq 53` + `tcp eq 53` |
| DHCP | `udp eq 67` / `udp eq 68` |
| HTTP / HTTPS | `tcp eq 80` / `tcp eq 443` |
| Proxmox web UI | `tcp eq 8006` |
| Jellyfin | `tcp eq 8096` (8920 HTTPS) |
| Omada controller | `tcp eq 8043`, `tcp eq 8088`, `udp eq 29810`, `tcp range 29811 29816` |
| NTP | `udp eq 123` |
| Syslog / SNMP | `udp eq 514` / `udp eq 161` |

Omada ports vary by controller version — confirm against your install.

## Editing with sequence numbers

Every ACL line gets a sequence number (10, 20, 30…), so you can insert or delete single lines without retyping the list.

```
show ip access-lists EXAMPLE          ! see the numbers

ip access-list extended EXAMPLE
 15 permit tcp any host 10.10.60.10 eq 22   ! insert between 10 and 20
 no 30                                       ! delete line 30
 exit

ip access-list resequence EXAMPLE 10 10      ! renumber: start 10, step 10
```

Numbered ACLs can be edited the same way: `ip access-list extended 110` opens access-list 110 in named-ACL mode.

Add remarks so future-you (and your portfolio) knows why a line exists:

```
ip access-list extended EXAMPLE
 5 remark Allow lab clients to reach nginx test server
```

## Applying ACLs

Your inter-VLAN routing happens on the ISR4331 subinterfaces, so that's where most ACLs go.

```
! ISR4331 — router-on-a-stick subinterface
interface GigabitEthernet0/0/1.50
 ip access-group VLAN50-IN in

! Remove it
interface GigabitEthernet0/0/1.50
 no ip access-group VLAN50-IN in

! Lock down SSH to the router (standard ACL on VTY)
line vty 0 4
 access-class MGMT-ONLY in
 transport input ssh
```

```
! Catalyst 3560X — routed SVI (only if the switch does the routing: ip routing + SVIs)
interface Vlan40
 ip access-group SERVERS-IN in

! 3560X — port ACL on a Layer 2 access port (inbound only)
interface GigabitEthernet0/10
 ip access-group PORT-FILTER in

! 3560X — VLAN ACL (VACL): filters traffic inside a VLAN, even same-subnet
ip access-list extended BLOCK-HOST
 permit ip host 10.10.30.50 any
vlan access-map IOT-MAP 10
 match ip address BLOCK-HOST
 action drop
vlan access-map IOT-MAP 20
 action forward
vlan filter IOT-MAP vlan-list 30
```

In a VACL, `permit` means "match this traffic" and the `action` decides what happens — not the same as a normal ACL.

## Verify and troubleshoot

Hit counters are your best friend: generate test traffic, then check which line's counter went up.

| Command | What it shows |
| --- | --- |
| `show access-lists` | All ACLs with line numbers and match counts |
| `show ip access-lists NAME` | One IP ACL |
| `show ip interface Gi0/0/1.50` | Which ACL is applied in/out on that interface |
| `show running-config \| section access-list` | ACL config only |
| `show running-config interface Gi0/0/1.50` | Interface config incl. access-group |
| `clear access-list counters [NAME]` | Reset hit counts before a test |
| `show logging \| include IPACCESSLOG` | Matches from lines tagged `log` |
| `show vlan access-map` / `show vlan filter` | VACLs on the 3560X |
| `terminal monitor` | Show log messages live in your SSH session |

Test from your LXC containers: `curl http://10.10.60.10`, `nc -zv 10.10.60.10 22`, `ping`, `dig @10.10.40.30 example.com`.

## Lab recipes

Start with the practice VLANs (acl-client 10.10.50.10 → acl-server 10.10.60.10 running nginx), then move the proven patterns to the real VLANs.

**Drill 1 — allow web only, block everything else from VLAN 50 to 60**

```
ip access-list extended VLAN50-IN
 5 remark Web to acl-server only
 10 permit tcp host 10.10.50.10 host 10.10.60.10 eq 80
 20 permit icmp host 10.10.50.10 host 10.10.60.10 echo
 30 deny ip 10.10.50.0 0.0.0.255 10.10.60.0 0.0.0.255 log
 40 permit ip any any
!
interface GigabitEthernet0/0/1.50
 ip access-group VLAN50-IN in
```

Expected: `curl` works, `ping` works, `ssh 10.10.60.10` fails and shows up in the log.

**Drill 2 — same goal, applied outbound toward VLAN 60**

```
interface GigabitEthernet0/0/1.50
 no ip access-group VLAN50-IN in
interface GigabitEthernet0/0/1.60
 ip access-group VLAN50-IN out
```

Compare the counters — same result, but outbound filtering wastes a routing lookup on dropped packets.

**Drill 3 — stateless return traffic with `established`**

```
ip access-list extended VLAN60-IN
 10 permit tcp host 10.10.60.10 eq 80 10.10.50.0 0.0.0.255 established
 20 deny ip 10.10.60.0 0.0.0.255 10.10.50.0 0.0.0.255 log
 30 permit ip any any
```

The server can answer web requests but can't start new connections into VLAN 50.

**Real lab — IoT (VLAN 30): internet + DNS only**

```
ip access-list extended IOT-IN
 5 remark DHCP to the router and DNS to Pi-hole
 10 permit udp any any eq 67
 20 permit udp 10.10.30.0 0.0.0.255 host 10.10.40.30 eq 53
 30 permit tcp 10.10.30.0 0.0.0.255 host 10.10.40.30 eq 53
 40 deny ip 10.10.30.0 0.0.0.255 10.10.0.0 0.0.255.255 log
 50 deny ip 10.10.30.0 0.0.0.255 192.168.0.0 0.0.255.255 log
 60 permit ip 10.10.30.0 0.0.0.255 any
!
interface GigabitEthernet0/0/1.30
 ip access-group IOT-IN in
```

**Real lab — Management (VLAN 99): reachable only from Home VLAN and your PC's home subnet**

```
ip access-list extended TO-MGMT
 10 permit ip 10.10.10.0 0.0.0.255 10.10.99.0 0.0.0.255
 20 permit ip 192.168.0.0 0.0.0.255 10.10.99.0 0.0.0.255
 30 permit tcp 10.10.99.0 0.0.0.255 any established
 40 deny ip any 10.10.99.0 0.0.0.255 log
 50 permit ip any any
!
interface GigabitEthernet0/0/1.99
 ip access-group TO-MGMT out
```

Adjust 192.168.0.0/24 if your home subnet is different. If something other than the ISR hands out DHCP for a VLAN, change line 10 of IOT-IN to point at that server.

## Gotchas and lockout safety

The classic mistake is applying an ACL over SSH that blocks your own session, so set a safety net first.

```
reload in 10          ! router reboots to saved config in 10 min
! ...apply and test the ACL...
reload cancel         ! it worked — cancel the reboot
copy running-config startup-config
```

On IOS-XE you can also use `configure replace` with an archive if you've set one up.

- **Never `write mem` before testing** — the saved config is your rollback.
- **Add your permits before the deny.** Building an ACL line by line on a live interface can cut you off after the first line; build it first, apply it last.
- **The implicit deny drops DHCP, DNS and routing traffic too.** If clients lose addresses after applying an ACL, you forgot UDP 67/68.
- **Inbound ACLs also filter traffic to the router itself** (SSH, ping to the gateway, DHCP).
- **Router-generated traffic isn't filtered by outbound ACLs.** Pings from the ISR will "work" even when hosts are blocked — test from the containers.
- **`established` isn't stateful.** For true return-traffic tracking use reflexive ACLs or Zone-Based Firewall on the ISR, or the OPNsense firewall you're considering.
- **Put the most specific lines first.** A broad `permit` above a narrow `deny` means the deny never hits — check counters at 0.

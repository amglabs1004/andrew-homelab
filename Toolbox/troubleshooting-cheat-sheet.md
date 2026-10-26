# Homelab Troubleshooting Cheat Sheet

When something breaks, start at the bottom of the stack and work up: cable and port, then VLAN, then routing, then the service, then DNS. Each command below says what it shows and when to reach for it.

## Contents

- [Lab quick reference](#lab-quick-reference)
- [Cisco IOS: checking what's going on](#cisco-ios-checking-whats-going-on)
- [Cisco IOS: configuration](#cisco-ios-configuration)
- [Proxmox and LXC containers](#proxmox-and-lxc-containers)
- [Pi-hole, Unbound, and DNS testing](#pi-hole-unbound-and-dns-testing)
- [From your Windows PC](#from-your-windows-pc)
- [Troubleshooting order](#troubleshooting-order)

## Lab quick reference

| Item | Address / port | Notes |
| --- | --- | --- |
| ISR4331 (VLAN 99 mgmt) | 10.10.99.1 | Router and gateway for every lab VLAN; SSH in here |
| ISR4331 uplink (home side) | 192.168.0.191 | DHCP reservation on the TP-Link home router |
| 3560X switch (VLAN 99 mgmt) | 10.10.99.2 | Port G0/47 is the trunk to the ISR |
| ISR to switch link | ISR G0/0/1 to 3560X G0/47 | Not G0/0/2 and not the management port |
| Home router static route | 10.10.0.0/16 via 192.168.0.191 | How the home network reaches the lab |
| Jellyfin (LXC) | 10.10.40.20 | VLAN 40 Servers |
| Pi-hole (LXC) | 10.10.40.30 | VLAN 40 Servers; Unbound on 127.0.0.1 port 5335 |
| Proxmox host | 10.10.40.10 VLAN 40 | OptiPlex, behind the 3560X |

| VLAN | Name | Subnet |
| --- | --- | --- |
| 10 | Home | 10.10.10.0/24 |
| 20 | Gaming / Entertainment | 10.10.20.0/24 |
| 30 | IoT | 10.10.30.0/24 |
| 40 | Servers / Infrastructure | 10.10.40.0/24 |
| 99 | Management | 10.10.99.0/24 |

## Cisco IOS: checking what's going on

These commands only read state, so they're safe to run any time. Run them in privileged EXEC mode (`enable`), and add `| include <word>` to filter long output.

| Command | What it shows | Reach for it when |
| --- | --- | --- |
| `show ip interface brief` | Every interface with its IP and up/down status | First check on the ISR. Anything down or missing an IP? |
| `show interfaces status` | Switch ports with VLAN, speed, duplex, connected or not | A device isn't getting a link or is in the wrong VLAN (3560X) |
| `show interfaces G0/47` | Errors, drops, and counters for one port | Link is up but traffic is flaky. Look for CRC errors and input errors |
| `show vlan brief` | Which VLANs exist and which access ports are in each | A host can't reach its gateway. Check that its port is in the right VLAN |
| `show interfaces trunk` | Trunk ports, allowed VLANs, native VLAN | A VLAN works on one side but not across the ISR link |
| `show ip route` | The routing table | Traffic isn't leaving the lab. Check for the default route and connected subnets |
| `show cdp neighbors` | Directly connected Cisco devices and the ports they use | Confirming which port is cabled to what |
| `show mac address-table` | MACs the switch learned and their ports | Finding which port a device (Pi-hole, Proxmox) is on |
| `show arp` | IP-to-MAC mappings the device knows | The switch sees the device at layer 2, but you can't ping it |
| `show running-config` | The live configuration | Checking what's actually applied. Use `\| section interface` to narrow it |
| `show ip ssh` / `show users` | SSH status and who's logged in | SSH isn't working, or you want to confirm your session |
| `show ntp associations` / `show clock` | NTP sync status and the current time | Logs have wrong timestamps, or PnP is still chatty |
| `show pnp profile` | Cisco Plug-and-Play status | Confirming PnP is off (it caused the time-pnp.cisco.com lookups) |
| `ping <ip> source G0/0/1.40` | Reachability from a specific interface | Testing whether one VLAN can reach a target |
| `traceroute <ip>` | Each hop to a destination | Finding where traffic stops between the lab and home network |

## Cisco IOS: configuration

These commands change the device. Enter config mode with `configure terminal`, leave with `end`, and save with `copy running-config startup-config` (or `write memory`). Unsaved changes are lost on reboot.

**VLANs and access ports (3560X)**

```
vlan 40
 name Servers
interface G0/5
 switchport mode access
 switchport access vlan 40
```

Creates a VLAN and puts a port in it. Use it when you add a new device or move one to a different VLAN.

**Trunk to the ISR (3560X G0/47)**

```
interface G0/47
 switchport trunk encapsulation dot1q
 switchport mode trunk
 switchport trunk allowed vlan 10,20,30,40,99
```

Carries every VLAN over one cable to the router. The 3560X needs the `encapsulation dot1q` line before `mode trunk`. Check this first when a VLAN can't reach its gateway.

**Router-on-a-stick subinterfaces (ISR G0/0/1)**

```
interface G0/0/1
 no shutdown
interface G0/0/1.40
 encapsulation dot1Q 40
 ip address 10.10.40.1 255.255.255.0
```

Gives each VLAN a gateway on the router. Repeat for each VLAN. The parent interface must be `no shutdown`, or every subinterface stays down.

**Switch management IP (3560X)**

```
interface vlan 99
 ip address 10.10.99.2 255.255.255.0
 no shutdown
ip default-gateway 10.10.99.1
```

Lets you SSH to the switch and lets it reach other subnets.

**Default route out to the home network (ISR)**

```
ip route 0.0.0.0 0.0.0.0 <home router IP>
```

Sends anything not in the lab out the uplink. If the lab can't reach the internet, check `show ip route` for this line.

**SSH access**

```
hostname ISR
ip domain-name lab.local
crypto key generate rsa modulus 2048
username admin privilege 15 secret <password>
ip ssh version 2
line vty 0 4
 login local
 transport input ssh
```

Sets up SSH instead of console. The key generation needs a hostname and domain name set first.

**Shutting off Cisco PnP and setting NTP**

```
no pnp profile pnp-zero-touch
ntp server <NTP server IP or pool address>
```

Stops the PnP agent from repeatedly looking up time-pnp.cisco.com, and gives the device a real time source. Syntax varies by IOS-XE version, so run `show pnp profile` first to see the profile name.

**Recovering from mistakes**

| Command | What it does | Use it when |
| --- | --- | --- |
| `default interface G0/5` | Wipes a port back to factory config | A port's config is a mess and you want a clean start |
| `no <command>` | Removes that one line of config | Undoing a single change |
| `reload` | Reboots and loads the saved config | You made unsaved changes that broke access and want to roll back |

## Proxmox and LXC containers

Run these on the Proxmox host shell, either from the web UI (node → Shell) or over SSH. `<id>` is the container number shown in the web UI, such as 100.

| Command | What it does | Reach for it when |
| --- | --- | --- |
| `pct list` | Lists every container with its ID, status, and name | Finding a container's ID, or seeing what's stopped |
| `pct start <id>` / `pct stop <id>` / `pct reboot <id>` | Starts, stops, or restarts a container | A service is hung, or after a config change |
| `pct enter <id>` | Opens a root shell inside the container | Running commands in Pi-hole or Jellyfin without SSH |
| `pct config <id>` | Shows the container's config, including network, VLAN tag, and IP | A container has the wrong IP or isn't on VLAN 40 |
| `pct set <id> -net0 name=eth0,bridge=vmbr0,tag=40,ip=10.10.40.30/24,gw=10.10.40.1` | Changes a container's network settings | Moving a container to a new VLAN or IP |
| `cat /etc/network/interfaces` | Shows the host's bridge and VLAN setup | The host itself lost connectivity, or containers can't use VLAN tags |
| `ip -br addr` | Short list of interfaces and IPs | Quick check on the host or inside a container |
| `ip route` | The default gateway and routes | A container can ping its subnet but nothing beyond it |
| `systemctl restart networking` | Reloads the host's network config | After editing `/etc/network/interfaces` |
| `pveversion` | Proxmox version | Before following a guide or updating |
| `apt update && apt full-upgrade` | Updates the host or a container | Routine maintenance. Run it on the host and inside each container separately |

For VLAN tags to work on containers, the `vmbr0` bridge needs **VLAN aware** turned on (Datacenter → node → Network → vmbr0).

## Pi-hole, Unbound, and DNS testing

Run these inside the Pi-hole container (`pct enter <id>` from Proxmox). Unbound is now Pi-hole's only upstream, so if Unbound stops, DNS stops for every device using Pi-hole.

**Pi-hole**

| Command | What it does | Reach for it when |
| --- | --- | --- |
| `pihole status` | Shows whether Pi-hole is running and blocking | First check when DNS seems broken |
| `pihole -t` | Live feed of every query as it happens | Watching what a device is looking up, or why something got blocked |
| `pihole -q <domain>` | Shows which blocklist (if any) contains a domain | A site is broken and you suspect Pi-hole is blocking it |
| `pihole -g` | Re-downloads and rebuilds the blocklists | After adding a new list, or if the block count looks wrong |
| `pihole reloaddns` | Restarts the DNS service | After a config change, or if queries hang. Older v5 installs use `pihole restartdns` |
| `pihole enable` / `pihole disable 5m` | Turns blocking on, or off for a set time | Quick test of whether Pi-hole is the cause of a problem |
| `pihole -up` | Updates Pi-hole | Routine maintenance |

**Unbound**

| Command | What it does | Reach for it when |
| --- | --- | --- |
| `systemctl status unbound` | Shows whether Unbound is running, plus recent errors | DNS is broken but `pihole status` looks fine |
| `systemctl restart unbound` | Restarts Unbound | Unbound is stopped or stuck |
| `unbound-checkconf` | Checks the config file for errors | After editing Unbound's config, before restarting |
| `journalctl -u unbound -n 50` | Last 50 lines of Unbound's log | Unbound won't start and you need the error message |

**Testing DNS with dig**

| Command | What it tests | Reach for it when |
| --- | --- | --- |
| `dig pi-hole.net @127.0.0.1 -p 5335` | Unbound answers on its own, skipping Pi-hole | Isolating whether the problem is Unbound or Pi-hole |
| `dig fail01.dnssec.works @127.0.0.1 -p 5335` | DNSSEC is rejecting bad signatures. Expect **SERVFAIL** | Confirming Unbound's DNSSEC validation works |
| `dig dnssec.works @127.0.0.1 -p 5335` | DNSSEC passes good signatures. Expect **NOERROR** | Same check, the passing side |
| `dig google.com @10.10.40.30` | The full path through Pi-hole | Testing from another machine whether Pi-hole answers |
| `dig doubleclick.net @10.10.40.30` | Blocking works. Expect **0.0.0.0** | Confirming ads are actually being blocked |

In dig output, check the `status:` field. **NOERROR** means it worked, **SERVFAIL** means the resolver failed or DNSSEC rejected it, and **NXDOMAIN** means the name doesn't exist. `Query time` tells you how slow it was.

If `dig` isn't installed in the container, run `apt install dnsutils`.

## From your Windows PC

Your PC is on the home network (192.168.x.x), so these also test the static route into the lab. Run them in Command Prompt or PowerShell.

| Command | What it does | Reach for it when |
| --- | --- | --- |
| `ipconfig /all` | Your IP, gateway, and DNS servers | Checking which DNS server your PC is actually using |
| `ipconfig /flushdns` | Clears the PC's DNS cache | After a Pi-hole change, or when a site loads stale results |
| `ping 10.10.40.30` | Basic reachability to Pi-hole | Testing whether the home network can reach the lab at all |
| `tracert 10.10.40.30` | Each hop on the way to the lab | Ping fails. See whether it dies at the TP-Link, the ISR, or the switch |
| `nslookup google.com 10.10.40.30` | Asks Pi-hole directly for an answer | Testing Pi-hole from your PC without changing your DNS settings |
| `ssh admin@10.10.99.1` | SSH into the ISR | Getting to the router CLI |
| `route print` | The PC's routing table | Lab traffic seems to be going the wrong way |
| `Test-NetConnection 10.10.40.20 -Port 8096` | Checks whether a specific port is open (PowerShell) | Jellyfin pings but the web page won't load |

## Troubleshooting order

Work from the bottom up and stop at the first step that fails. That's where the problem is.

1. **Physical link.** Is the port up? Run `show ip interface brief` on the ISR and `show interfaces status` on the switch.
2. **VLAN.** Is the device's port in the right VLAN, and does the trunk carry it? Run `show vlan brief` and `show interfaces trunk`.
3. **Gateway.** From the device, can you ping its gateway (for example 10.10.40.1)? If not, check the ISR subinterface.
4. **Routing.** Can you reach other subnets and the internet? Run `show ip route` and `traceroute`. From home, check the TP-Link static route.
5. **Service.** Is the container running and is the service up? Run `pct list`, then `pihole status` or `systemctl status unbound`.
6. **DNS.** Does a name resolve? Use `dig` inside the container, then `nslookup` from your PC.

A quick way to tell DNS problems from network problems: if `ping 8.8.8.8` works but `ping google.com` fails, the network is fine and DNS is the problem.

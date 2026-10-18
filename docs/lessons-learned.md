# Lessons Learned

This section will document problems encountered during my project, how they were diagnosed, and how they were resolved.
The goal is not only to document successful configurations, but also to document troubleshooting and mistakes along the way.

## Index

1. [Router-to-switch link not working (wrong interface + management port)](#1-router-to-switch-link-not-working-wrong-interface--management-port)
2. [Lab containers had no internet access (NAT interfaces all set to outside)](#2-lab-containers-had-no-internet-access-nat-interfaces-all-set-to-outside)

---

## 1. Router-to-switch link not working (wrong interface + management port)

**Problem:** The link between the ISR 4331 and the Catalyst switch wasn't passing traffic.

**Cause:** Two mistakes stacked on top of each other:

* I configured interface G0/0/2 on the router, but the port actually cabled to the switch was G0/0/1.
* The Ethernet cable was plugged into the router's management port (Gi0), which is an out-of-band port in its own management VRF, it isn't meant to route regular data traffic.

**Fix:** Moved the configuration to G0/0/1, plugged the cable into G0/47, instead of the management port.

**Takeaway:** Confirm the physical port before configuring the logical one.
Next time I'll check the port labels and link lights, and use `show ip interface brief` to verify which interface is actually up and what's on the other end before troubleshooting the config.

---

## 2. Lab containers had no internet access (NAT interfaces all set to outside)

**Problem:** Jellyfin (LXC container at 10.10.40.20) couldn't fetch metadata. `ping api.themoviedb.org` returned "Temporary failure in name resolution."

**Diagnosis:** The error looked like a DNS problem, but I tested one layer at a time from the container:

* `ping 10.10.40.1` (VLAN 40 gateway on the ISR): worked
* `ping 192.168.0.1` (home router, one hop past the ISR): worked
* `ping 1.1.1.1` (internet by IP, no DNS involved): failed

Local routing was fine and traffic was dying on its way to the internet, so DNS was only a symptom. On the ISR, `show ip nat statistics` showed G0/0/0, G0/0/1, and G0/0/1.40 all listed under **Outside interfaces**, with no inside interfaces at all, and `show ip nat translations` showed 0 entries while the container was pinging.

**Cause:** I had applied `ip nat outside` to every interface, including the lab-facing ones. NAT only translates traffic that enters an inside interface and leaves an outside interface, so with no inside interface nothing from the lab was ever translated.

**Fix:**
```
interface GigabitEthernet0/0/1
 no ip nat outside
 ip nat inside
interface GigabitEthernet0/0/1.40
 no ip nat outside
 ip nat inside
```
G0/0/0 (uplink to the home router) stays `ip nat outside`. Verified translations appeared in `show ip nat translations`, confirmed the container could reach 1.1.1.1 and resolve api.themoviedb.org, then saved with `wr mem`.

**Takeaway:** An error message points to where the failure showed up, not necessarily where it started. Ping by IP before blaming DNS. For NAT, `show ip nat statistics` is the fastest check because it lists which interfaces are inside and outside. Every new VLAN subinterface (10, 20, 30) will also need `ip nat inside`.

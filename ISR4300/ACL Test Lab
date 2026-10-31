# ACL Test Lab — Setup

## Purpose

The purpose of this lab was to give me a test environment to practice setting up ACLs and see how they work. I built it separately so a mistake wouldn't take down my working lab VLANs or my home network.

## Switch (Catalyst 3560X)

First, I created two separate test VLANs: VLAN 50 (ACL-TEST-CLIENT) and VLAN 60 (ACL-TEST-SERVER).

Next, I added VLANs 50 and 60 to the trunk on G0/47, which connects the switch to the ISR. This lets the router see traffic from the new VLANs.

Then I converted G0/1, the port my Proxmox server uses, from an access port to a trunk port. As an access port it only carried VLAN 40, but the test containers on Proxmox need to be in VLANs 50 and 60, so the port has to carry more than one VLAN.

I kept VLAN 40 as the native VLAN. Native VLAN traffic stays untagged, so the Proxmox host and my existing LXC containers (Pi-hole and Jellyfin) kept working without any changes.

I limited the trunk to VLANs 40, 50, and 60, so the port only carries the VLANs Proxmox actually needs.

I verified the trunk with `show interfaces trunk`, then confirmed Pi-hole and Jellyfin were still reachable by pinging them from my PC.

## Router (ISR4331)

I created router-on-a-stick subinterfaces G0/0/1.50 (10.10.50.1) and G0/0/1.60 (10.10.60.1) to act as the gateways for the new VLANs.

I added `ip nat inside` to both subinterfaces. My existing NAT-LAB ACL already covers 10.10.0.0/16, so the new subnets got internet access without changing the ACL.

## Proxmox

I created two Debian 12 LXC containers:

| ID | Name | VLAN | IP | Role |
| --- | --- | --- | --- | --- |
| 150 | acl-client | 50 | 10.10.50.10 | Runs the tests (curl) |
| 160 | acl-server | 60 | 10.10.60.10 | Target (nginx web server) |

I pointed both containers at Pi-hole (10.10.40.30) for DNS, then installed nginx on acl-server and curl on acl-client.

## What I learned

- A VLAN splits one physical switch into separate logical networks. Devices in different VLANs can't talk to each other without a router.
- A trunk is a port or cable that carries several VLANs at once, using 802.1Q tags to keep them separate.
- The native VLAN is the one VLAN a trunk carries untagged. It keeps untagged devices working when an access port becomes a trunk.
- Router-on-a-stick routes between VLANs over one trunk cable, using one subinterface per VLAN.
- `ip nat inside` marks which side of NAT an interface is on. Translation only happens when traffic goes from inside to outside and matches the NAT ACL.
- Traffic within the same VLAN stays on the switch. Only traffic between VLANs reaches the router, which is where the ACLs live.

## Mistakes I made

- **Wrong template name.** `pveam download` fetches new templates from the internet. `pveam list local` shows the ones I already have, and `pct create` needs that exact name.
- **Pasting a block of commands.** `pct enter` moved me into the container partway through, so the rest of the commands ran in the wrong place. Now I run commands one at a time and watch the prompt: `root@pve01` is the Proxmox host, and `root@acl-server` is inside a container.
- **nginx on the wrong container.** I installed it on acl-client by mistake. It was harmless, and I removed it with `apt purge`.

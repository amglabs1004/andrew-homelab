# andrew-homelab

# Home Lab ##

This repository is for the documentation, design, deployment, configuration, and troubleshooting of my personal home lab.

My goal for this project is to build hands-on experience with enterprise networking, virtualization, security, and system administration.

## Goals ##

* Build and maintain a virtualized environment using Proxmox VE
* Gain hands-on experience with Cisco networking equipment
* Configure and manage a Cisco ISR 4331 router
* Configure and manage a Cisco Catalyst switch
* Deploy and configure a managed wireless network
* Learn WAN routing concepts and router-to-router connectivity
* Practice IPv4 addressing and subnetting
* Practice static and dynamic routing
* Configure and troubleshoot routing protocols
* Design and implement VLANs and network segmentation
* Configure trunking and inter-VLAN routing
* Build and manage virtual machines
* Practice Windows and Linux system administration
* Learn DNS, DHCP, Active Directory, and other network services
* Document troubleshooting and configuration changes
* Configure ACLs
* Practice standard and extended ACL configs
* Implement traffic filtering between network segments
* Develop firewall and router security policies
* Secure management access using SSH
* Test and document security controls and their impact on network traffic
* Develop practical skills that can be applied to an IT/networking career

### Hardware ###

* Dell OptiPlex — Proxmox virtualization host
* Cisco ISR 4331/K9 router
* Cisco Catalyst switch
* Omada wireless access point

### Virtualization ###

* Proxmox VE
* Virtual machines for testing and learning


## Documentation ##

* [Lessons Learned](docs/lessons-learned.md) - problems I ran into, how I diagnosed them, and how I fixed them
* [Network Addressing Scheme](docs/network-addressing.md) - VLANs, subnets, and static addresses
* [Build Log](Build%20Log/) - dated progress entries
* Router configs: [ISR4300](ISR4300/)
* Switch configs: [Cisco-Switch](Cisco-Switch/)
* Service setup notes: [Jellyfin](notes/jellyfin-setup.md), [Pi-hole](notes/pihole-setup.md), [Tailscale remote access](notes/tailscale-remote-acess.md)

### Project Status ###
Area	                                                     

Proxmox VE host (Dell OptiPlex)	                     

Cisco ISR 4331 router	                             

Router ↔ Catalyst 3560X switch link	                    

SSH management access	                                  

Omada wireless (TP-Link EAP610)	                        

Services (Pi-hole, DNS, DHCP, Jellyfin, cloud storage)	        

VLANs / inter-VLAN routing

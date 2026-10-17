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


## Documentation ###

**In progresss**

### Project Status ###

**In Progress**

This is a living project. Documentation will be updated as infrastructure is added, redesigned, and tested.

### Lessons Learned ###
This section will document problems encountered during my project, how they were diagnosed, and how they were resolved.
The goal is not only to document successful configurations, but also to document troubleshooting and mistakes along the way.

1. Router-to-switch link not working (wrong interface + management port)
Problem: The link between the ISR 4331 and the Catalyst switch wasn't passing traffic.

Cause: Two mistakes stacked on top of each other:

I configured interface G0/0/2 on the router, but the port actually cabled to the switch was G0/0/1.

The Ethernet cable was plugged into the router's management port (Gi0), which is an out-of-band port in its own management VRF, it isn't meant to route regular data traffic.

Fix: Moved the configuration to G0/0/1, plugged the cable into G0/47, instead of the management port.
Takeaway: Confirm the physical port before configuring the logical one.
Next time I'll check the port labels and link lights, and use `show ip interface brief` to verify which interface is actually up and what's on the other end before troubleshooting the config.

# Network Addressing Scheme

## VLANs

| VLAN | Name | Subnet | Devices |
|------|------|--------|---------|
| 10 | Home | 10.10.10.0/24 | Phones, tablet, PC and other trusted personal devices |
| 20 | Gaming & Entertainment | 10.10.20.0/24 | Nintendo consoles, PS5, TV and Roku |
| 30 | IoT / Smart Home | 10.10.30.0/24 | Blink Sync Module and future smart-home devices |
| 40 | Servers / Infrastructure | 10.10.40.0/24 | Dell OptiPlex running Proxmox, server VMs and lab services |
| 99 | Network Management | 10.10.99.0/24 | Switch management, router management and wireless access point management |

## Static Addresses

| Device | Address |
|--------|---------|
| ISR 4331 (management) | 10.10.99.1 |
| Catalyst switch (management) | 10.10.99.2 |
| Proxmox VE | 10.10.40.10 |

## Router Links

* G0/0/0 - link to home default gateway, 192.168.0.191. Static route created. DHCP reservation created on home router.
* G0/0/1 - G0/47 Catalyst link to ISR

# Pi-hole Setup

## Overview
Deployed Pi-hole as an LXC container to provide DNS-based ad/tracker blocking, paired with Unbound for independent recursive DNS resolution.

## Install
Used the community-scripts Proxmox VE helper script to provision the LXC:

bash -c "$(wget -qLO - https://github.com/community-scripts/ProxmoxVE/raw/main/ct/pihole.sh)"


## Network Placement Decision
At the start I planned to keep all lab services on an isolated lab network, separate from the home network. Pi-hole is the exception: since its purpose is to filter DNS for household devices, 
it was connected to the home network rather than the isolated lab segment.

This gives Pi-hole (and by extension the Proxmox host) a foot on the home network, which should be the only exception since Pi-hole only functions if it can see real household DNS traffic.

## Unbound
Installed Unbound alongside Pi-hole to act as a full recursive resolver rather than forwarding queries to a third-party DNS provider (e.g. Cloudflare, Google). 
When prompted during setup for DNS-over-TLS forwarding mode, chose not to forward. I opted for full recursive resolution instead, since the goal was avoiding a third-party seeing DNS query patterns entirely, 
not just encrypting them in transit.

Unbound can be installed and left inactive by not switching Pi-hole's upstream DNS setting over to it, it only becomes active once selected in Pi-hole's DNS settings.

##Current Road Block: Xfinity Gateway DNS Override
The Xfinity gateway I have does not give me a way to override DNS network-wide through either the mobile app or the admin panel, this is a known limitation of Xfinity XB-series gateways.

##Planned resolution: Probably just going to buy a different modem/router to get passed this.

##Status
Pi-hole and Unbound are running; network-wide DNS pointing is on hold until the dedicated router is in place.

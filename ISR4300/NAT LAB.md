# NAT Lab

This config set up NAT (Network Address Translation) on the ISR so the lab can reach the internet. Here’s why it was needed and what each part does.

## The problem it solves

Your lab devices use 10.10.x.x addresses. When Jellyfin sent a request to TMDB, the packet reached the internet fine. The reply came back to your TP-Link addressed to 10.10.40.20, though, and the TP-Link only NATs its own 192.168.0.x devices. It didn’t know how to handle that reply, so it got dropped.

NAT fixes this by having the ISR swap the lab address for its own home-network address (192.168.0.191) on the way out. To the TP-Link, all lab traffic looks like it comes from one ordinary device on your home network.

## Line by line

### Part 1: the ACL (which traffic gets translated)

```
ip access-list extended NAT-LAB
 deny   ip 10.10.0.0 0.0.255.255 192.168.0.0 0.0.0.255
 permit ip 10.10.0.0 0.0.255.255 any
```

In a NAT ACL, “permit” doesn’t mean “allow through.” It means “translate this.”

* 0.0.255.255 is a wildcard mask, the inverse of a subnet mask. 0 means “must match” and 255 means “anything.” So 10.10.0.0 0.0.255.255 means any 10.10.x.x address, which covers all of my VLANs.
* deny: traffic from the lab to your home network is not translated. When your PC opens Proxmox, the reply has to come back from 10.10.40.10, the address your PC connected to. If it came back as 192.168.0.191, your PC would discard it as unrelated traffic.
* permit: everything else from the lab (the internet) is translated.
* ACLs are read top-down and stop at the first match, so the deny has to come first.

### Part 2: inside and outside

```
interface g0/0/0
 ip nat outside
interface g0/0/1
 ip nat inside
interface g0/0/1.40
 ip nat inside
```

This tells the ISR which side is which. Outside is G0/0/0, toward your home router and the internet. Inside is the lab interfaces. NAT only happens when traffic crosses from an inside interface to an outside one, or back. Each new VLAN subinterface (.10, .20, .30) needs its own ip nat inside.

### Part 3: the rule that turns it on

```
ip nat inside source list NAT-LAB interface g0/0/0 overload
```

In plain terms: “For traffic from inside whose source address matches NAT-LAB, replace it with G0/0/0’s IP address.”

* interface g0/0/0 uses whatever IP G0/0/0 has, so it keeps working even if DHCP changes that address.
* overload is PAT (Port Address Translation). It lets many lab devices share that one IP. The ISR gives each connection a unique source port and keeps a table so it knows which reply goes to which device. That’s what show ip nat translations displays.

### Part 4: save

```
end
write memory
```

end leaves config mode. write memory copies the running config to startup config so the changes survive a reboot.

## One packet, start to finish

1. Jellyfin (10.10.40.20) sends a request to TMDB and it reaches the ISR on G0/0/1.40 (inside).
2. It matches the permit line, so the ISR rewrites the source to 192.168.0.191:50001 and records the mapping.
3. The TP-Link does its own NAT to your public IP and sends it to the internet.
4. The reply comes back to 192.168.0.191:50001. The ISR looks up port 50001, sees it belongs to 10.10.40.20, and rewrites the destination back.
5. Jellyfin gets its poster.

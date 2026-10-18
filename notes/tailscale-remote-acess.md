# Remote Access via Tailscale

## Goal
Access Jellyfin from outside the home network (e.g. phone on cellular data) without opening ports on the home router or exposing services directly to the internet.

## Why Tailscale
Chose Tailscale because it encrypts traffic by default, and is simple to set up. Felt like a good fit for sharing access with a small number of personal devices/people rather than the general public.

## Install
```bash
curl -fsSL https://tailscale.com/install.sh | sh
tailscale up
```

## Lesson Learned: TUN Device Access in LXC
`tailscale up` failed with: failed to connect to local tailscaled; it doesn't appear to be running

Root cause: LXC containers do not have TUN device access by default, which `tailscaled` requires to create its network interface.

**Fix (from the Proxmox host, not inside the container):**
```bash
pct set <CTID> -features nesting=1,keyctl=1
```
Then added the following lines to `/etc/pve/lxc/<CTID>.conf`: 
lxc.cgroup2.devices.allow: c 10:200 rwm
lxc.mount.entry: /dev/net dev/net none bind,create=dir

Restarted the container (`pct restart <CTID>`, or stop/start if restart hangs), then `tailscale up` succeeded.

## Result
Jellyfin is reachable from any device joined to the same tailnet via its Tailscale IP (`10.x.x.x:8096`), without any router-level port forwarding.

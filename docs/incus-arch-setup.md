# Incus Installation and Setup on Arch Linux

This document records the exact steps used to install and initialize Incus on this Arch Linux host, including resolving idmap issues and launching a Debian 12 container.

## Environment
- OS: Arch Linux (`/etc/os-release` → PRETTY_NAME=Arch Linux)
- Kernel: `6.14.9-arch1-1`
- Arch: `x86_64`
- Incus packages from official `extra` repo (no external repository required)

## Install Incus
1) Install packages
```bash
sudo pacman -Sy --noconfirm incus incus-tools
# Optional extras:
# sudo pacman -S incus-ui distrobuilder
```

2) Enable and start Incus (socket/service)
```bash
# Prefer socket activation when available
sudo systemctl enable --now incus.socket
# If needed (or for persistence), also enable the service
sudo systemctl enable --now incus.service

# Verify
incus version
# Expected: Client/Server 6.15 (as of this setup)
```

## Initialize Incus
We used automatic initialization with a directory storage backend. This created a default `dir` storage pool and a managed bridge `incusbr0` with NAT.
```bash
sudo incus admin init --auto --storage-backend=dir

# Inspect results
sudo incus storage list
sudo incus network list
```
Expected outcome example:
- Storage: pool `default` (driver `dir`)
- Network: `incusbr0` bridge (e.g., `10.49.149.1/24`), managed=YES

## Fix idmap (subuid/subgid) if needed
During first container launch we hit:
```
Error: System doesn't have a functional idmap setup
```
On this system `/etc/subuid` and `/etc/subgid` were empty. We configured subordinate ID ranges for `root` and restarted Incus:
```bash
# Allocate a large subordinate range for root (adjust as needed)
echo 'root:1000000:1000000000' | sudo tee -a /etc/subuid >/dev/null
echo 'root:1000000:1000000000' | sudo tee -a /etc/subgid >/dev/null

# Restart daemon to pick up idmap
sudo systemctl restart incus.service || sudo systemctl restart incusd.service
```
Notes:
- Choose ranges that do not overlap any existing entries.
- Many distros use `root:100000:65536`; larger ranges are fine if unused.

## Launch Debian 12 container
```bash
# First attempt name conflicted; final container name used here is deb12
sudo incus launch images:debian/12 deb12

# Check status and networking
incus list deb12
sudo incus exec deb12 -- cat /etc/debian_version
sudo incus exec deb12 -- sh -c 'ip -4 -o addr show eth0 | awk "{print \$4}"'
```
Example results observed:
- Debian version: `12.11`
- State: RUNNING
- IPv4: `10.49.149.60/24` on `eth0` (via `incusbr0` NAT)

## Optional: non-root usage
Add your user to the Incus admin group for passwordless local access, then re-login:
```bash
sudo usermod -aG incus-admin $USER
# Log out/in or `newgrp incus-admin`
```

## Quick commands
```bash
# List images
incus image list images:

# Launch Ubuntu example
sudo incus launch images:ubuntu/22.04 jammy1

# Shell into a container
sudo incus exec deb12 -- bash

# Stop/Delete
sudo incus stop deb12
sudo incus delete deb12
```

## Troubleshooting tips
- idmap: Ensure `/etc/subuid` and `/etc/subgid` contain valid ranges; restart Incus after changes.
- Name conflicts: `UNIQUE constraint failed: instances.name` → delete stale record: `sudo incus delete --force <name>`.
- Networking: `incusbr0` provides NAT by default. If no IPv4, check local firewall/NAT rules and ensure the bridge is up.
- Services: If `incus.socket` is inactive, enable it; for debugging, `journalctl -u incus.service -f`.

---
This host is currently configured with a `dir` storage pool `default` and a managed bridge `incusbr0`; the container `deb12` is running and reachable on the internal subnet.

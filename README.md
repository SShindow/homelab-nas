# Home NAS with Secure Remote Access (TrueNAS + Tailscale)

A self-hosted Network Attached Storage system built from repurposed desktop hardware, providing secure, cross-platform remote file access from anywhere in the world.

![Architecture Diagram](docs/architecture.svg)

## Overview

I wanted reliable, secure access to personal files across countries and devices without relying on a third-party cloud storage provider. This project repurposes an old desktop PC into a full NAS using **TrueNAS Community Edition**, with **Tailscale** providing a zero-config VPN mesh so the NAS is reachable from any registered device — no port forwarding, no exposed public IP, no dynamic DNS hacks.

**Goals:**
- Centralized, redundant file storage at home
- Secure remote access from Windows, iOS, and Android
- No reliance on commercial cloud storage
- Persistent, low-maintenance setup that survives reboots

## Architecture

| Layer | Technology |
|---|---|
| Operating System | TrueNAS Community Edition |
| Storage Pool | ZFS mirror (2x HDD) |
| OS Drive | Dedicated SSD |
| Remote Access | Tailscale (WireGuard-based mesh VPN) |
| File Sharing Protocol | SMB (Samba) |
| Clients | Windows, iOS, Android |

See [`docs/architecture.svg`](docs/architecture.svg) for the full diagram.

## Hardware

- CPU: Intel Pentium G3240
- RAM: 16GB DDR3
- Motherboard: ASUS H81M-I
- Storage: 2x 2TB Seagate HDD (mirrored ZFS pool), 128GB Kingston SSD (boot/OS)

## Setup Steps

1. **Install TrueNAS Community Edition** on the SSD boot drive.
2. **Create a ZFS pool** using the two HDDs in a mirror configuration for redundancy.
3. **Create a dataset** and a dedicated TrueNAS user account for file access permissions.
4. **Install Tailscale** via the TrueNAS Apps catalog, with Userspace networking enabled.
5. **Register the NAS as a persistent Tailscale node** using a *reusable*, *non-ephemeral* auth key — this ensures the machine doesn't drop off the network after a reboot.
6. **Create an SMB share** and expose it to the local network / Tailscale mesh.
7. **Connect each client device:**
   - **Windows:** Mapped the share as a persistent drive (`Z:`) with "Reconnect at sign-in" enabled.
   - **iPhone:** Connected via the Files app using `smb://<tailscale-ip>/<share-name>`.
   - **Android (Galaxy Tab S7):** Connected via My Files → Network Storage → Add manually, using SMBv2/SMBv3 and the Tailscale IP.
8. **Verified persistence** — confirmed TrueNAS apps auto-restart correctly after a full reboot.

## Key Challenges & Fixes

| Problem | Root Cause | Fix |
|---|---|---|
| Android device intermittently lost VPN connectivity in the background | Android's battery optimization was killing Tailscale's background process | Settings → Apps → Tailscale → Battery → set to **Unrestricted** |
| NAS occasionally disappeared from the Tailscale network after reboot | Auth key was set to ephemeral, causing the node registration to expire | Regenerated the key as **Reusable**, **Ephemeral: off** |

## Results

- All three client devices (Windows, iOS, Android) reliably connect to the NAS over Tailscale from outside the home network.
- Storage is redundant via ZFS mirroring, protecting against single-disk failure.
- Setup survives reboots and power interruptions without manual intervention.

## Lessons Learned

- Mesh VPNs like Tailscale remove almost all the traditional networking pain (port forwarding, dynamic DNS, firewall rules) from remote NAS access.
- Mobile OS battery management is an under-documented source of "random" VPN drops — worth checking first when debugging intermittent connectivity.
- Auth key configuration (ephemeral vs. reusable) matters more than it seems for long-running headless nodes.

## Future Improvements

- [ ] Automated off-site/cloud backup of critical datasets
- [ ] Monitoring/alerting (e.g., Uptime Kuma or TrueNAS alert integrations)
- [ ] Additional self-hosted services (media server, document management) on the same box

---

**Stack:** TrueNAS CE · ZFS · Tailscale (WireGuard) · SMB/Samba

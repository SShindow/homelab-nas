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
- Automated backup of critical cloud files (Google Drive) to local storage

This is a living project — new capabilities are added and documented incrementally as they're built.

## Architecture

| Layer | Technology |
|---|---|
| Operating System | TrueNAS Community Edition |
| Storage Pool | ZFS mirror (2x HDD) |
| OS Drive | Dedicated SSD |
| Remote Access | Tailscale (WireGuard-based mesh VPN) |
| File Sharing Protocol | SMB (Samba) |
| Clients | Windows, iOS, Android |
| Cloud Backup | TrueNAS Cloud Sync Task (rclone) — Google Drive → NAS |

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

## Module: Google Drive → NAS Backup

**Goal:** Keep a local, one-way backup of a critical Google Drive folder so those files stay accessible even if Google Drive is unavailable — without risking accidental deletions propagating from the cloud side.

**Why one-way instead of live 2-way sync:** TrueNAS has no native bidirectional sync option. True 2-way sync would require manually running `rclone bisync` on a cron schedule — more moving parts, and higher risk of conflicts or unexpected deletions. A one-way pull-and-copy was chosen as the safer default; bidirectional sync remains a possible future upgrade if convenience ever outweighs that safety tradeoff.

**Setup steps:**
1. Created a dedicated dataset for the backup target (`GGDrive-backup`), separate from other datasets.
2. Added a Google Drive credential via OAuth under TrueNAS's Cloud Credentials.
3. Created a **Cloud Sync Task** (Data Protection → Cloud Sync Tasks), configured as:
   - **Direction:** Pull (Drive → NAS only)
   - **Transfer Mode:** Copy (adds/updates files; never deletes from the NAS if a file is removed on Drive — this is what makes it safe as a backup, not a mirror)
   - **Remote path:** the target Google Drive folder
   - **Destination:** the `GGDrive-backup` dataset
4. Set a recurring schedule so the task runs automatically rather than requiring manual triggers.
5. Confirmed the export format handling for native Google Docs/Sheets/Slides files (which aren't real files on Drive and need explicit export-format conversion to back up correctly).
6. Ran a **Dry Run** first to verify the file list and paths before committing to a real transfer.
7. Executed the real sync — confirmed successful.
8. Added the `GGDrive-backup` dataset to the existing SMB share, so the backed-up files are reachable from all three client devices over Tailscale, same as the rest of the NAS.

**Status:** ✅ Fully operational — Google Drive files now back up automatically to the NAS on a schedule, viewable from any connected device.

## Key Challenges & Fixes

| Problem | Root Cause | Fix |
|---|---|---|
| Android device intermittently lost VPN connectivity in the background | Android's battery optimization was killing Tailscale's background process | Settings → Apps → Tailscale → Battery → set to **Unrestricted** |
| NAS occasionally disappeared from the Tailscale network after reboot | Auth key was set to ephemeral, causing the node registration to expire | Regenerated the key as **Reusable**, **Ephemeral: off** |
| Needed cloud backup without risking cloud-side deletions wiping local copies | Live 2-way sync isn't natively supported and adds deletion risk | Used one-way **Pull + Copy** Cloud Sync Task instead of bidirectional sync |

## Results

- All three client devices (Windows, iOS, Android) reliably connect to the NAS over Tailscale from outside the home network.
- Storage is redundant via ZFS mirroring, protecting against single-disk failure.
- Setup survives reboots and power interruptions without manual intervention.

## Lessons Learned

- Mesh VPNs like Tailscale remove almost all the traditional networking pain (port forwarding, dynamic DNS, firewall rules) from remote NAS access.
- Mobile OS battery management is an under-documented source of "random" VPN drops — worth checking first when debugging intermittent connectivity.
- Auth key configuration (ephemeral vs. reusable) matters more than it seems for long-running headless nodes.

## Future Improvements

- [x] ~~Automated off-site/cloud backup of critical datasets~~ → done via Cloud Sync Task (see above)
- [ ] True live 2-way sync (`rclone bisync` + cron) — only if convenience outweighs backup-safety tradeoff
- [ ] Monitoring/alerting (e.g., Uptime Kuma or TrueNAS alert integrations)
- [ ] Additional self-hosted services (media server, document management) on the same box

---

**Stack:** TrueNAS CE · ZFS · Tailscale (WireGuard) · SMB/Samba · rclone (Cloud Sync)

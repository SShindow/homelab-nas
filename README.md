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
| Local Point-in-Time Recovery | ZFS periodic snapshots (daily + weekly) |
| Personal VPN / Internet Egress | Tailscale Exit Node (NAS advertises `0.0.0.0/0` + `::/0`) |
| Video Surveillance / NVR | Frigate (RTSP from Tapo C200, recorded to the ZFS pool) |
| Observability | Prometheus + Grafana, with node_exporter as a host binary |

See [`docs/architecture.svg`](docs/architecture.svg) for the full diagram.

## Hardware

- CPU: Intel Pentium G3240
- RAM: 16GB DDR3
- Motherboard: ASUS H81M-D (BIOS v2106)
- Storage: 2x 2TB Seagate HDD (mirrored ZFS pool), 128GB Kingston SSD (boot/OS)
- GPU: GTX 650 (available but not currently installed — power draw vs. hardware transcoding tradeoff still under evaluation)
- Router: ISP-issued GPON ONT (no WAN-side Wake-on-LAN support)
- Camera: TP-Link Tapo C200 (1080p, RTSP), Wi-Fi, recording to the NAS rather than its 32GB SD card

**Deployment note:** the NAS is physically hosted at a remote location (family home) and administered entirely over the network — all setup, testing, and troubleshooting shown here was done remotely, with family assisting on-site only for physical steps (e.g., swapping hardware).

## Setup Steps

1. **Install TrueNAS Community Edition** on the SSD boot drive.
2. **Create a ZFS pool** using the two HDDs in a mirror configuration for redundancy.
3. **Create a dataset** and a dedicated TrueNAS user account for file access permissions.
4. **Install Tailscale** via the TrueNAS Apps catalog, initially with **Userspace networking enabled** — sufficient for the NAS to act as a Tailscale client.
   *(Reversed later: Userspace mode runs its own TCP/IP stack inside the container and cannot forward other devices' traffic, so it had to be disabled — and Host Network mode enabled — when the NAS became an exit node. See [Tailscale Exit Node](docs/tailscale-exit-node.md).)*
5. **Register the NAS as a persistent Tailscale node** using a *reusable*, *non-ephemeral* auth key — this ensures the machine doesn't drop off the network after a reboot.
6. **Create an SMB share** and expose it to the local network / Tailscale mesh.
7. **Connect each client device:**
   - **Windows:** Mapped the share as a persistent drive (`Z:`) with "Reconnect at sign-in" enabled.
   - **iPhone:** Connected via the Files app using `smb://<tailscale-ip>/<share-name>`.
   - **Android (Galaxy Tab S7):** Connected via My Files → Network Storage → Add manually, using SMBv2/SMBv3 and the Tailscale IP.
8. **Verified persistence** — confirmed TrueNAS apps auto-restart correctly after a full reboot.

## Modules

Each module is a self-contained build with its own goals, decisions, problems hit, and verification.

| Module | What it does | Status |
|---|---|---|
| **[Google Drive → NAS Backup](docs/google-drive-backup.md)** | One-way (Pull + Copy) Cloud Sync so cloud-side deletions can't propagate to the local copy. | ✅ Operational |
| **[Power-Loss Resilience](docs/power-loss-resilience.md)** | Root-caused a failing CMOS battery that silently reset the BIOS auto-power-on setting; verified with a real fuse cut. | ✅ Verified |
| **[ZFS Snapshot Strategy](docs/zfs-snapshots.md)** | Daily and weekly periodic snapshots for instant local point-in-time recovery, layered with the off-site cloud copy. | ✅ Operational |
| **[Tailscale Exit Node](docs/tailscale-exit-node.md)** | The NAS as a personal VPN egress. Includes an upstream Tailscale bug, its diagnosis, and an ASN-based verification method. | ✅ Verified |
| **[Camera NVR (Frigate)](docs/camera-nvr-frigate.md)** | Tapo C200 RTSP recorded to the ZFS pool with no vendor cloud, plus end-to-end performance benchmarking. | ✅ Operational |
| **[Observability](docs/monitoring-prometheus-grafana.md)** | Prometheus + Grafana with a host-installed node_exporter, persistent host-path config, reboot-verified. | ✅ Operational |

## Key Challenges & Fixes

| Problem | Root Cause | Fix |
|---|---|---|
| Android device intermittently lost VPN connectivity in the background | Android's battery optimization was killing Tailscale's background process | Settings → Apps → Tailscale → Battery → set to **Unrestricted** |
| NAS occasionally disappeared from the Tailscale network after reboot | Auth key was set to ephemeral, causing the node registration to expire | Regenerated the key as **Reusable**, **Ephemeral: off** |
| Needed cloud backup without risking cloud-side deletions wiping local copies | Live 2-way sync isn't natively supported and adds deletion risk | Used one-way **Pull + Copy** Cloud Sync Task instead of bidirectional sync |
| TrueNAS Apps occasionally failed on cold boot with "Unable to determine default interface" | Known Docker/TrueNAS Apps timing/race condition on startup, not a real misconfiguration | Re-select the Apps pool or restart the Docker service after boot |
| NAS didn't power back on after a full home power cut, even with "Restore AC Power Loss" set | Failing CMOS battery — only exposed by full power cuts, since graceful shutdowns rely on PSU standby power instead | Replaced the CR2032 CMOS battery; verified with a real fuse-cut test |
| Tailscale's "Advertise Exit Node" checkbox had no effect — the node never appeared as an exit node at all | Container was already authenticated (`Auth Once`), so `containerboot` applies settings via `tailscale set`, which silently ignores `--advertise-exit-node` from `TS_EXTRA_ARGS` ([upstream bug](https://github.com/tailscale/tailscale/issues/14496)) | Applied the flag directly to the daemon: `tailscale set --advertise-exit-node`; added a post-upgrade check to the runbook |
| Userspace networking (used in the original Tailscale install) cannot support exit-node routing | In Userspace mode Tailscale runs its own TCP/IP stack inside the container and never touches host routing, so it can't forward other devices' traffic | Disabled Userspace, enabled Host Network, and set the IPv4/IPv6 forwarding sysctls |
| Frigate's ONVIF auto-probe returned "No RTSP URLs found" for the Tapo C200 | Known Tapo/Frigate ONVIF incompatibility — not a config error | Manual selection → brand "Other" → literal RTSP URL. (Tapo's real ONVIF port is 2020, not the wizard's default of 80) |
| Frigate's first-login admin password never appeared in the container logs | Known TrueNAS SCALE + Frigate interaction | Set `auth.reset_admin_password: true`, restart, watch `docker logs -f` live to catch it — then set it back to `false` or it regenerates every restart |
| RTSP authentication rejected the Tapo cloud credentials | RTSP uses a separate local "Camera Account", not the cloud login | Created the Camera Account in the Tapo app under Advanced Settings |
| Disk benchmark reported an impossible 3+ GB/s on two mechanical drives | `dd if=/dev/zero` on a pool with inherited lz4 compression measures the compressor, not the disks | Benchmarked with `/dev/urandom` + `conv=fdatasync` at a size larger than RAM — real figure 80.2 MB/s |
| Prometheus refused connections on every attempt | TrueNAS launches it with `--web.listen-address=0.0.0.0:30104`, not the documented default 9090 | Used the TrueNAS-assigned port everywhere, including the self-scrape target (`localhost:30104`) |
| Prometheus was running and healthy but collecting nothing — "No scrape pools found" | TrueNAS ships the app with an empty scrape config and no bundled exporter | Wrote `prometheus.yml` by hand and installed node_exporter separately on the host |
| Grafana could not reach Prometheus by service name or container IP | The two apps sit on separate Docker bridge networks, so container DNS doesn't resolve between them | Addressed the host's LAN IP (`192.168.1.4:30104`) instead of the container |
| Prometheus config vanished after an app reinstall | Config was on ixVolume, which is not durable across reinstalls and is permission-denied from the host shell | Moved config to a dedicated dataset mounted as a **Host Path**, owned `568:568` |

## Results

- All three client devices (Windows, iOS, Android) reliably connect to the NAS over Tailscale from outside the home network.
- Storage is redundant via ZFS mirroring, protecting against single-disk failure.
- Setup survives reboots and power interruptions without manual intervention.
- The NAS doubles as a personal VPN exit node — client devices can route their full internet connection out through the home line, verified by a change of originating ASN (Viettel → VNPT) with no DNS leak and no measurable throughput penalty.
- The NAS records a security camera continuously to the ZFS pool with no vendor cloud involvement, survives full reboots unattended, and is viewable remotely over Tailscale at ~1.42 Mbps for one stream.
- System metrics are collected and dashboarded end to end — ~2,770 host metrics scraped every 15s into Prometheus and rendered in Grafana, with the exporter surviving a full reboot unattended via a Post Init script.

## Lessons Learned

- Mesh VPNs like Tailscale remove almost all the traditional networking pain (port forwarding, dynamic DNS, firewall rules) from remote NAS access.
- Mobile OS battery management is an under-documented source of "random" VPN drops — worth checking first when debugging intermittent connectivity.
- Auth key configuration (ephemeral vs. reusable) matters more than it seems for long-running headless nodes.
- BIOS "power on after power loss" settings can silently depend on a healthy CMOS battery — this only surfaces during a *full* power cut, since normal shutdowns are masked by the PSU's standby power. Worth testing with a real power cut, not just a reboot, before trusting unattended recovery.
- No single backup mechanism covers every failure mode — off-site cloud copy protects against hardware loss, while local ZFS snapshots protect against accidental deletion/edits with near-zero recovery time. Layering both is more robust than relying on either alone.
- When a GUI reports success and the feature still doesn't work, stop trusting the layer above and query the daemon's own state. `tailscale debug prefs` located a fault that neither the TrueNAS UI nor the Tailscale admin console surfaced at all — and its *other* fields (`NetfilterMode`, `NoSNAT`) ruled out whole categories of cause at the same time.
- A test that cannot fail isn't a test. Checking a public IP while sitting on the same network as the exit node would have "passed" whether or not the feature worked; comparing ASNs across two genuinely different access networks was what made the result mean something.
- Domestic and international throughput on the same line differed by ~2.5×. Benchmark against the path that matches the real use case, not the one that produces the nicer number.
- Configuration applied outside a management UI becomes invisible to that UI's rebuild path. The `tailscale set` workaround works, but it would vanish on a rebuild — which makes it a runbook item, not a one-time fix.
- A benchmark can be confidently wrong. `dd if=/dev/zero` on an lz4-compressed ZFS pool measures compression, not disk throughput — and the giveaway was that the result was *too good* (3+ GB/s from two mechanical drives). Implausibly good numbers deserve the same scrutiny as bad ones.
- Measurement conditions have to hold still for the whole sample. A remote-bandwidth reading was discarded because the laptop silently switched from cellular to Wi-Fi mid-test; the error showed up as a 3× jump between two readings that should have matched.
- Reboot resilience is only proven if it's verified from outside the LAN. Checking from inside the house would have skipped the Tailscale reconnect and the camera's Wi-Fi rejoin — the two links most likely to fail unattended.
- An appliance OS quietly invalidates the standard tutorial. TrueNAS reassigned Prometheus's port, shipped it with an empty scrape config, bundled no exporter, and isolated the app networks from each other — four defaults broken at once. Reading the *running process's own arguments* rather than the upstream documentation was what actually resolved it.
- "Persistent" storage has degrees. ixVolume survives restarts but not reinstalls — which is exactly the case where losing hand-written config hurts most. Configuration and disposable data deserve different storage decisions: config went to a host path, the time-series database stayed on ixVolume on purpose.
- Running something outside the appliance's supported model is a legitimate choice, but it has to be written down. node_exporter as a host binary buys the ZFS collector and real `/proc` visibility, and costs supportability, manual updates, and an unauthenticated metrics endpoint. The trade is worth making — and worth stating.

## Future Improvements

- [x] ~~Automated off-site/cloud backup of critical datasets~~ → done via [Cloud Sync Task](docs/google-drive-backup.md)
- [x] ~~Unattended recovery from a full power outage~~ → done via [CMOS battery fix](docs/power-loss-resilience.md)
- [x] ~~ZFS snapshot strategy for point-in-time recovery~~ → done via [daily/weekly snapshots](docs/zfs-snapshots.md)
- [x] ~~Personal VPN / self-hosted internet egress~~ → done via [Tailscale exit node](docs/tailscale-exit-node.md)
- [x] ~~Local-only camera recording, off the vendor cloud~~ → done via [Frigate NVR](docs/camera-nvr-frigate.md)
- [ ] GTX 650 reinstallation for Jellyfin hardware transcoding — evaluating whether the transcoding benefit is worth the added power draw
- [ ] Jellyfin media server setup
- [ ] True live 2-way sync (`rclone bisync` + cron) — only if convenience outweighs backup-safety tradeoff
- [x] ~~System metrics collection and dashboards~~ → done via [Prometheus + Grafana](docs/monitoring-prometheus-grafana.md)
- [ ] Alert rules on top of Prometheus (disk >85%, sustained CPU >80%) — metrics exist, alerting doesn't yet
- [ ] Uptime Kuma for service-reachability checks — complements Grafana rather than duplicating it (Grafana answers "how is the box doing", Uptime Kuma answers "is the service reachable")
- [ ] ZFS-specific Grafana dashboard using the `node_zfs_*` collector
- [ ] TLS / basic auth in front of node_exporter's `/metrics` endpoint
- [ ] Nextcloud (personal cloud), Vaultwarden (password manager), Pi-hole (network-wide ad blocking)
- [ ] PiKVM for true out-of-band hardware access (board has no IPMI)
- [ ] Self-hosted DNS (Pi-hole / AdGuard Home) as the tailnet resolver, so exit-node queries terminate on owned infrastructure instead of the ISP's
- [ ] Tailscale subnet router to reach home LAN devices remotely (currently out of scope)
- [ ] Tailscale ACLs / auto-approvers so routes re-advertise without manual approval after a rebuild
- [ ] SQM / fq_codel on the router to reduce bufferbloat under load
- [ ] Cross-country verification of the exit node from Europe
- [ ] Coral USB TPU for person/stranger detection and alerting (currently CPU detection only)
- [ ] Hard camera isolation — VLAN + egress block, so the camera cannot reach TP-Link at all
- [ ] Set up the second Tapo C200 in Germany
- [ ] Resolve the Frigate `/dev/shm` size warning on TrueNAS SCALE 25.10
- [ ] Cross-country verification of Frigate remote viewing from Europe

---

**Stack:** TrueNAS CE · ZFS · Tailscale (WireGuard mesh + exit node) · SMB/Samba · rclone (Cloud Sync) · Frigate + go2rtc (NVR) · Prometheus + Grafana + node_exporter

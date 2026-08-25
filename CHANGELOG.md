# Changelog

All notable changes to this project are documented here, in the order they were built. This project is developed and documented incrementally — each entry reflects a real, completed milestone rather than a planned roadmap.

## [Unreleased]
- Evaluating GTX 650 reinstallation for Jellyfin hardware transcoding
- Jellyfin media server setup

## 2026 — Jellyfin media streaming
- Deployed Jellyfin with the library on a dedicated `tank/media` dataset, deliberately outside the snapshot scope, and config on a durable host path
- Moved a 35 GB library out of the private dataset with `zfs rename` — a metadata-only operation — and documented its two side effects (broken Cloud Sync task, dataset leaving the SMB share)
- Migrated 208 episodes across four video and three subtitle naming conventions using a dry-run-by-default script, with four preview passes before anything moved
- Recovered two accidental `rm -rf` deletions from ZFS snapshots; the second required the weekly snapshot because the daily had already aged out
- Mounted the media library read-only, choosing container isolation over NFO metadata sidecars
- Measured transcoding: only browser + XviD transcodes; native client direct-plays everything. Grafana showed 95.9% CPU and 3.1% I/O during a transcode — purely compute-bound
- Evaluated and rejected the spare GTX 650 on four independent grounds (Kepler driver EOL, gen-1 NVENC being H.264-only, insufficient compute capability for Frigate, and the problem case disappearing with the native client)

## 2026 — Tailscale subnet router
- Advertised the home LAN (`192.168.1.0/24`) alongside the exit node, in a single `tailscale set` call since the flag replaces rather than appends
- Approved in the admin console and verified from mobile data by loading the router's admin UI; persists across reboot

## 2026 — Client lineup change
- Replaced the Windows laptop with a MacBook Air M2; the Mac now mounts the SMB share over Tailscale via Finder
- Documentation updated to reflect macOS/iOS/Android as the current client set
- A Windows gaming PC is planned and will be added as a fourth client

## 2026 — Observability with Prometheus + Grafana
- Deployed Prometheus and Grafana on TrueNAS, with node_exporter v1.9.0 installed as a host binary rather than a container so the ZFS collector and host /proc, /sys visibility are retained
- Worked out that TrueNAS assigns Prometheus a non-default port (30104), ships it with an empty scrape config, and bundles no exporter — none of the standard setup guides apply unmodified
- Resolved Grafana being unable to reach Prometheus: the two apps sit on separate Docker bridges, so the host LAN IP is the working address rather than any container name or IP
- Moved Prometheus config off ixVolume onto a dedicated host-path dataset after a reinstall wiped it; kept the TSDB on ixVolume deliberately, since time-series data is disposable and config is not
- Pinned the NAS IP by DHCP reservation so the hardcoded scrape target cannot drift
- Auto-start via a Post Init script; verified across a full reboot with both scrape targets returning UP unattended
- Imported the Node Exporter Full dashboard (ID 1860); ~2,770 host metrics scraped at 15s intervals, 30d retention
- Documented the supportability trade-off: node_exporter is outside TrueNAS's supported model, updates are manual, and /metrics is unauthenticated on the LAN and tailnet
- Deferred: alert rules, a ZFS-specific dashboard, and TLS/auth in front of the metrics endpoint

## 2026 — Camera NVR with Frigate
- Deployed Frigate (0.17.2) on TrueNAS to record a Tapo C200's RTSP stream to the ZFS pool, retiring the camera's SD card as primary storage and keeping footage off the vendor cloud
- Chose Frigate over plain ffmpeg/go2rtc so future person detection is a config change rather than a stack migration
- Dual-stream setup through go2rtc: 1080p for recording, 640×360 for detection/audio, with connection reduction enabled
- Worked around a Tapo/Frigate ONVIF incompatibility ("No RTSP URLs found") using a literal RTSP URL; documented the Tapo's real ONVIF port (2020) and the separate local Camera Account requirement
- Worked around the TrueNAS + Frigate first-login password bug via `auth.reset_admin_password`
- Pinned the camera's IP with a DHCP reservation so a lease change can't silently break the RTSP source
- Verified full reboot resilience end to end, checked from outside the LAN over Tailscale
- Benchmarked the stack: 80.2 MB/s baseline ZFS write, 319.19 MiB/hour per camera, 13-15% CPU, ~1.42 Mbps for remote live view
- Corrected an invalid `dd if=/dev/zero` disk benchmark that was measuring lz4 compression rather than disk throughput, and discarded a remote-bandwidth sample contaminated by a mid-test network switch
- Deferred: person/stranger detection pending a Coral USB TPU; hard VLAN isolation of the camera

## 2026 — Tailscale Exit Node (Personal VPN)
- Configured the NAS as a Tailscale exit node, so client devices can route their full internet connection out through the home line
- Enabled `net.ipv4.ip_forward` and `net.ipv6.conf.all.forwarding` as TrueNAS sysctl tunables
- Disabled Userspace networking and enabled Host Network — Userspace mode cannot forward other devices' traffic
- Diagnosed the "Advertise Exit Node" checkbox having no effect: with `Auth Once` and an already-authenticated node, `containerboot` applies settings via `tailscale set`, which ignores `--advertise-exit-node` from `TS_EXTRA_ARGS` (tailscale#14496, truenas/apps#3486)
- Worked around it by applying the flag directly to the daemon; documented as a post-upgrade runbook check since it lives outside TrueNAS config
- Designed an ASN-based verification method after recognising that the obvious public-IP test would have produced a false positive when run from the same network as the NAS
- Confirmed no DNS leak, a direct (non-relayed) IPv6 peer-to-peer path, and no measurable throughput penalty from the tunnel
- Benchmarked the line: ~155–170 Mbps upstream international, ~439 Mbps domestic — the ~2.5x gap matters for predicting cross-country performance

## 2026 — ZFS Snapshot Strategy
- Added daily (7-day retention) and weekly (4-week retention) periodic ZFS snapshots on the primary dataset
- Established a layered backup model: off-site Cloud Sync for durability + local snapshots for instant point-in-time recovery

## 2026 — Power-Loss Resilience & Unattended Recovery
- Diagnosed BIOS "Restore AC Power Loss" setting silently resetting after full power cuts
- Root-caused to a failing CMOS battery, masked during graceful shutdowns by PSU standby power
- Replaced CMOS battery and verified full unattended recovery through a real fuse-cut test
- Ruled out Wake-on-LAN as a solution (router itself loses power in a fuse cut)

## 2026 — Google Drive → NAS Backup
- Configured a one-way (Pull + Copy) Cloud Sync Task via TrueNAS's built-in rclone integration
- Deliberately rejected bidirectional sync (`rclone bisync`) due to deletion/conflict risk and lack of native GUI support
- Verified with a dry run before enabling the live schedule

## 2026 — Tailscale Remote Access
- Installed Tailscale as a TrueNAS App in Userspace networking mode
- Connected Windows (mapped drive), iOS (Files app), and Android (My Files) clients via SMB over the Tailscale mesh
- Fixed intermittent Android disconnects caused by background battery optimization
- Fixed NAS dropping off the Tailscale network after reboot by switching to a reusable, non-ephemeral auth key

## 2026 — Initial NAS Build
- Repurposed a desktop PC (Intel Pentium G3240, 16GB DDR3, ASUS H81M-D) into a TrueNAS Community Edition NAS
- Created a ZFS mirror pool across two 2TB HDDs, with a dedicated SSD for the OS
- Set up initial dataset structure and user accounts

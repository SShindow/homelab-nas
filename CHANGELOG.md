# Changelog

All notable changes to this project are documented here, in the order they were built. This project is developed and documented incrementally — each entry reflects a real, completed milestone rather than a planned roadmap.

## [Unreleased]
- Evaluating GTX 650 reinstallation for Jellyfin hardware transcoding
- CPU upgrade to an Intel Core i7-4790 (non-K) planned for the next on-site visit; not yet installed
- Disk replacement planned for the ZFS pool's recurring-checksum-error drive (see the [ZFS pool disk fault](docs/troubleshooting.md#7-storage-diagnosing-a-faulted-disk-without-touching-hardware) entry below)
- Lidarr paused mid-build — resume points tracked in [Media Automation](docs/arr-stack.md#extension-lidarr-music--paused-curated-artist-scope)

## 2026 — Friend access: Wizarr + Cleanuparr
- Added once the plan changed from "one test account" to eventually sharing Jellyfin with up to ~10 friends across Vietnam, Germany, and the US
- **Wizarr** turns Jellyfin account creation into a self-service invite link (expiry + library scoping) instead of manually creating each friend's account and relaying a password over chat; it only automates the Jellyfin step, so importing each friend into Seerr for requests stays manual
- **Cleanuparr** auto-cleans stalled/failed/malicious downloads out of the download client and tells Sonarr/Radarr/Lidarr to re-search — added ahead of more people generating download traffic than one person's own usage patterns
- Diagnosed the arr-stack Custom App getting stuck "Deploying" after adding both services: `docker ps -a` found Wizarr stuck in `Created` while Cleanuparr had started fine, ruling out the initial `ghcr.io`-registry-flakiness hypothesis; the real cause was a **port collision** with an unrelated `homepage` dashboard container already bound to Wizarr's assigned port — reassigned the port and it started cleanly
- Found Wizarr needed the NAS's own **Tailscale IP**, not its LAN IP, to reach Jellyfin — the opposite of Seerr's working setup for the identical kind of problem; documented as a "don't assume a fix generalizes" case in the [Troubleshooting Playbook](docs/troubleshooting.md#container-to-container-networking)
- Fixed a trailing slash in the Jellyfin server URL that broke Wizarr's final "Test & Add" step with an HTTP 404, even though the library scan on the same URL had already succeeded
- Verified end-to-end: created a test invite, created a Jellyfin account through it, logged in, and played a film successfully
- Fixed Cleanuparr's directory-remapping fields never working — its container had no `/data` mount at all; added the same `tank/media:/data` mount every other app in the stack uses
- Tdarr was evaluated and explicitly declined at this stage — no HEVC-capable hardware anywhere in this NAS, so it wouldn't currently change what's playable

## 2026 — ZFS pool disk fault (incident)
- A CRITICAL alert fired for the `tank` mirror: one disk (`sda`) FAULTED, diagnosed entirely remotely with no physical access to the machine
- `sudo smartctl -a /dev/sda` failed outright with `INQUIRY failed` rather than returning SMART data; `dmesg -T | grep sda` showed `hostbyte=DID_BAD_TARGET`, a signature of the SATA controller failing to address the drive at all — a different failure class from a media read error — with errors scattered across random sectors rather than clustered, leaning toward a connection/bus issue over classic media wear
- Treated a full reboot as a legitimate low-risk remote diagnostic (it re-initializes the SATA controller and re-enumerates every device on the bus) rather than just a fix-of-last-resort; the pool stayed protected on the mirror's other disk throughout regardless of outcome
- The disk came back online after reboot and resilvered clean (0 errors) — but a follow-up `zpool scrub` (which verifies every block in the pool, not just the ones a resilver touched) found the same disk's checksum-error count climb from 4 to 19 afterward, changing the read from "one-off blip" to "recurring pattern, plan a real replacement"
- `errors: No known data errors` held true throughout — a healthy mirror repairs checksum mismatches automatically from the other disk — but the trend, not either single event, was the real signal
- Corrected the hardware record along the way: at least `sda` is a WD Green WDC_WD20EZRX-00D8PB0 (5400 RPM), not a Seagate as originally logged
- Ran `zpool clear` and planned a physical drive replacement for the next on-site visit; full remote diagnostic method captured in the [Troubleshooting Playbook](docs/troubleshooting.md#7-storage-diagnosing-a-faulted-disk-without-touching-hardware)

## 2026 — Media Automation: the arr-stack (Prowlarr, a download client, Sonarr, Radarr, Bazarr, Seerr) + Lidarr

> **Note:** this module is a personal, educational exercise in self-hosted service orchestration and container networking. The download-client implementation and any source/indexer configuration are intentionally omitted below — see the disclaimer at the top of [docs/arr-stack.md](docs/arr-stack.md).

- Built a request-to-playback pipeline: Seerr (requests) → Prowlarr (source aggregation) → Sonarr/Radarr/Lidarr (matching/management) → a download client → Bazarr (subtitles) → Jellyfin (playback), deployed as a single TrueNAS SCALE Custom App (`arr-stack`) on a shared Docker bridge network (`arrs-network`)
- Hit the classic `localhost`-vs-container-name bug on every single app-to-app pairing in the stack (Prowlarr↔Sonarr/Radarr/Lidarr, Sonarr/Radarr↔download client) — fixed each by swapping in the target's container name
- Fixed Seerr crash-looping on first boot (`EACCES` on `/app/config/logs/`) by manually `chown`-ing its config directory — the only non-linuxserver image in the stack, so it doesn't get the automatic config-dir chown the other five images get for free
- Worked through several rounds of download-client configuration fixes (WebUI/auth settings, save-path layout, per-category routing) before the pipeline moved files reliably end to end — specifics intentionally omitted, consistent with the module's disclaimer
- Found Radarr's default `HD-1080p` quality profile silently included an essentially uncompressed Remux-1080p tier, defeating the point of choosing 1080p over 4K for storage reasons — unchecked it; Sonarr's equivalent profile was already clean
- Deliberately chose 1080p over 4K stack-wide: no genuine 4K masters for most of the library, no hardware in this NAS that can decode HEVC, and meaningful storage headroom savings
- Hit VNPT (the Vietnamese ISP)'s DNS interference a second and third time, on one of Prowlarr's sources and on Seerr's entire TMDB-backed discover/search — same fix as before, a per-container `dns: [1.1.1.1, 1.0.0.1]` override
- Adopted deliberate source redundancy (several sources spread across movies/TV/anime categories) after enough individual public sources proved unreliable for reasons unrelated to this NAS; one source remains permanently unavailable due to an access restriction and isn't worth working around given coverage is already met
- **Extension: Lidarr.** Added for music, then reframed mid-build once it became clear Lidarr manages music at artist/album granularity, not the individual-track granularity Spotify offers — narrowed scope to curated lossless archiving of a small set of favorite artists rather than a broad auto-managed library, and set its quality profile to `Lossless` accordingly. **Paused before the end-to-end test was confirmed complete**, once it became clear the current desktop speakers can't resolve a lossless-vs-lossy difference in practice — the underlying reasoning still holds, just no audible payoff on the current playback chain yet
- **Real-world throughput finding, discovered from the first real movie streamed through the pipeline rather than during setup:** a generic speedtest from the NAS shows a healthy 161 Mbps, but the actual Vietnam↔Germany path sustains only ~3.78 Mbps with real packet loss — see the Jellyfin entry below for the full diagnosis. TV-episode bitrates stream live from Germany fine; movie-tier bitrates need a capped stream or a local download first

## 2026 — Jellyfin media streaming
- Deployed Jellyfin with the library on a dedicated `tank/media` dataset, deliberately outside the snapshot scope, and config on a durable host path
- Moved a 35 GB library out of the private dataset with `zfs rename` — a metadata-only operation — and documented its two side effects (broken Cloud Sync task, dataset leaving the SMB share)
- Migrated 208 episodes across four video and three subtitle naming conventions using a dry-run-by-default script, with four preview passes before anything moved
- Recovered two accidental `rm -rf` deletions from ZFS snapshots; the second required the weekly snapshot because the daily had already aged out
- Mounted the media library read-only, choosing container isolation over NFO metadata sidecars
- Measured transcoding: only browser + XviD transcodes; native client direct-plays everything. Grafana showed 95.9% CPU and 3.1% I/O during a transcode — purely compute-bound
- Evaluated and rejected the spare GTX 650 on four independent grounds (Kepler driver EOL, gen-1 NVENC being H.264-only, insufficient compute capability for Frigate, and the problem case disappearing with the native client)
- A Movies library was added later, once the arr-stack (above) started delivering files automatically
- **2026-08-28 — Germany-distance playback test, completed.** Direct play confirmed bitrate-bound as predicted: ~1.98 Mbps for an H.264 episode over Tailscale from Germany, Jellyfin container at 1.73% CPU. The one browser-transcode case failed to start over the same distance — ffmpeg launched but the browser gave up after ~50s each time, with the container at 158.36% CPU; inconclusive at the time whether that's a distance effect or the NAS simply carrying more background load (Frigate, arr-stack) than when the original transcode benchmark was taken
- **2026-08-28 — Real-world remote playback throughput, root-caused.** A stream failing on a heavier release (Blu-ray-tier bitrate, Atmos audio) led to an `iperf3` test run directly over the Tailscale link, ruling out NAS CPU (74.7% idle), a DERP-relayed connection (confirmed genuinely peer-to-peer via `tailscale status`), and raw bandwidth (a generic speedtest showed a healthy 161.83 Mbps) in that order. The real cause: sustained throughput of only ~3.78 Mbps over the actual Vietnam↔Germany route, with heavy retransmissions and a collapsing TCP congestion window — real international packet loss, not a bandwidth ceiling, and not fixable from the NAS or Jellyfin side. A seedbox-relay alternative was researched and declined; the practical workaround is a manually bitrate-capped stream, or downloading fully before playback, for anything above TV-episode bitrates. Full detail in [Media Streaming](docs/media-jellyfin.md#real-world-remote-playback-throughput)

## 2026 — Tailscale subnet router
- Advertised the home LAN (`192.168.1.0/24`) alongside the exit node, in a single `tailscale set` call since the flag replaces rather than appends
- Approved in the admin console and verified from mobile data by loading the router's admin UI; persists across reboot
- **2026-08-28 — Cross-country verification from Germany, completed.** `https://192.168.1.1` loaded cleanly over Tailscale from a real Germany connection with no manual route selection needed

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
- **2026-08-28 — Cross-country remote-view test from Germany, completed.** Real Vietnam↔Germany figure: ~0.74 Mbps (~92 KB/s down / ~4.4 KB/s up) over a ~90s sample — noticeably lower than the Vietnam-cellular baseline (~1.42 Mbps), most likely scene-dependent H.264 bitrate rather than a network effect; either way, trivial next to the home connection's throughput
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
- **2026-08-28 — Cross-country verification from Germany, completed.** Confirmed the exit node with a direct public-IP/ASN comparison (finally meaningful once the two endpoints are genuinely in different countries): Deutsche Telekom/Darmstadt with the exit node off, VNPT/Ho Chi Minh City with it on
- Measured the real cost across that distance: 91.09→65.86 Mbps down (≈28%) and 10ms→243ms idle latency (+230ms) — consistent with actual Frankfurt↔Ho Chi Minh City geography, and a far larger penalty than the same-country mobile-vs-exit-node comparison ever showed

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

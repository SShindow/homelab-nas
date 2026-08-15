# Changelog

All notable changes to this project are documented here, in the order they were built. This project is developed and documented incrementally — each entry reflects a real, completed milestone rather than a planned roadmap.

## [Unreleased]
- Evaluating GTX 650 reinstallation for Jellyfin hardware transcoding
- Jellyfin media server setup

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

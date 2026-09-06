# Media Automation — the arr-stack, Lidarr, and Friend Access (Wizarr + Cleanuparr)

*Part of the [homelab-nas](../README.md) project.*

---

**Goal:** turn "find a file → download it → put it where Jellyfin can see it" into a pipeline: request in one place, everything else happens on its own. Seerr (requests) → Prowlarr (indexer aggregation) → Sonarr/Radarr/Lidarr (TV/movies/music matching and management) → qBittorrent (download) → Bazarr (subtitles) → [Jellyfin](media-jellyfin.md) (playback, already shipped). Later extended with Wizarr (self-service account invites) and Cleanuparr (stalled-download cleanup) once friend access moved from "one test account" to "up to ~10 people across three countries."

**Note on continuity:** the initial deploy work for this module was done in a separate chat session, then brought back into this project's tracking with screenshots and live container output as the source of truth — worth stating since an earlier compose draft (env-var-driven, with a separate reverse-proxy network) floating around in this project's notes was **not** what actually got built. Everything below reflects what is actually running.

## Architecture

Nine containers, deployed as a single TrueNAS SCALE Custom App (`arr-stack`, via Apps → Discover Apps → Custom App → Install via YAML) on one Docker bridge network, `arrs-network`. A user-defined bridge gives working container-name DNS (e.g. `http://qbittorrent:8080` resolves between containers) with no separate reverse-proxy network needed. No reverse proxy is used — every service gets a direct TrueNAS-style high port, matching the convention already established by Prometheus/Grafana. Nothing is exposed through the router; everything is LAN/Tailscale-only.

| Service | Role | Port | Container port | Image |
|---|---|---|---|---|
| Prowlarr | Indexer aggregation | 30089 | 9696 | `lscr.io/linuxserver/prowlarr` |
| qBittorrent | Download client | 30080 (+ 6881 TCP/UDP for P2P) | 8080 | `lscr.io/linuxserver/qbittorrent` |
| Sonarr | TV/anime management | 30090 | 8989 | `lscr.io/linuxserver/sonarr` |
| Radarr | Movie management | 30091 | 7878 | `lscr.io/linuxserver/radarr` |
| Bazarr | Subtitle management | 30092 | 6767 | `lscr.io/linuxserver/bazarr` |
| Seerr (Jellyseerr) | Request front-end | 30093 | 5055 | `ghcr.io/seerr-team/seerr` |
| Lidarr | Music management | 30094 | 8686 | `lscr.io/linuxserver/lidarr` |
| Wizarr | Jellyfin invite links | 30097 | 5690 | `ghcr.io/wizarrrr/wizarr` |
| Cleanuparr | Stalled-download cleanup | 30096 | 11011 | `ghcr.io/cleanuparr/cleanuparr` |

All six original services (through Seerr) run `PUID=568`/`PGID=568`/`UMASK=002` — the same "apps" user convention as the rest of this project — and kept the exact `container_name:` set in the YAML rather than getting an `ix-`-prefixed rename the way Frigate/Jellyfin/Prometheus/Grafana did, since this Custom App doesn't override container naming. Config for every app lives on a host-path dataset, `tank/appdata/arr/<app>:/config`, following the same durable-storage pattern already proven for Prometheus (see [Observability](monitoring-prometheus-grafana.md)) rather than the ixVolume default that doesn't survive a reinstall.

### Storage layout

```
/mnt/tank/media
├── downloads
│   ├── torrents         # completed downloads land here — NOT downloads/ itself
│   └── incomplete       # in-progress pieces, kept separate
├── movies
├── tv
│   └── How I Met Your Mother   (pre-existing, Seasons 01–09, predates this stack)
├── music                # added for Lidarr
└── extras               (pre-existing)
```

Every container that touches media (qBittorrent, Sonarr, Radarr, Bazarr, Lidarr) mounts the **whole** `tank/media` dataset as a single `/data`, rather than separate volumes for downloads vs. library. This is the standard arr-stack pattern for a reason: with downloads and the library on the same filesystem, Sonarr/Radarr/Lidarr can hardlink or atomic-move a completed download into place instead of copying it — faster, and no duplicate-space window mid-import.

## Deployment problems and fixes

**Seerr crash-looped on first boot — `EACCES: permission denied, mkdir '/app/config/logs/'`.** The other five images (all linuxserver-based) auto-chown their bind-mounted `/config` to `PUID:PGID` via an s6-init step on container start. Seerr's image doesn't do that — its compose sets `user: "568:568"` directly with no init/chown step — so its config directory stayed owned by whatever created it (`root:root`, from Docker auto-creating the bind-mount path on first run). Fix: `sudo chown -R 568:568 /mnt/tank/appdata/arr/seerr` then restart. **General lesson: any non-linuxserver image added to this stack needs its config directory ownership checked manually** — the auto-chown behaviour isn't universal just because it's been reliable so far.

**qBittorrent's WebUI returned a bare `Unauthorized` — not a login problem.** Recent qBittorrent versions added Host Header Validation, which rejects any request outright unless it arrives via `localhost` or a whitelisted domain — and this NAS is reached via its Tailscale IP, not `localhost`. Fix: stop the container, edit `qBittorrent.conf` directly (`.../qbittorrent/qBittorrent/qBittorrent.conf` on the host), add `WebUI\HostHeaderValidation=false` under `[Preferences]`, restart. No `WEBUI_*` environment variable exposes this setting — it has to be a direct config-file edit.

**qBittorrent path typos, three times over.** The default save path (`/downloads/`) doesn't exist in this container — only `/data` is mounted. The first "corrected" path, `/data/downloads/`, was also wrong: the real structure is `downloads/torrents` (completed) + `downloads/incomplete` (in-progress), with `downloads/` itself just being the parent. Final correct values: Default Save Path `/data/downloads/torrents`, incomplete path `/data/downloads/incomplete`. A third typo (`dedownloads`) turned up in the global incomplete-path field and propagated into category dialogs before being caught.

**Category paths silently did nothing until one specific setting was found.** `Default Torrent Management Mode` is deliberately left on `Manual` (not `Automatic`) — Automatic mode lets qBittorrent auto-relocate files whenever a category or save path changes, which risks silently breaking the hardlinks Sonarr/Radarr depend on. But Manual mode **ignores per-category save paths entirely unless `Use Category paths in Manual Mode` is checked** — an easy setting to miss, and without it the `movies`/`tv`/`music` category split is purely decorative.

**The classic `localhost` bug, hit on every single app-to-app pairing in this stack.** Prowlarr↔Sonarr, Prowlarr↔Radarr, Prowlarr↔Lidarr, Sonarr/Radarr↔qBittorrent — every one of these defaulted its "Server" field to `http://localhost:<port>`, which from a given container's own perspective resolves to itself, not the other container. Every single case needed the same fix: swap `localhost` for the target's container name (`http://prowlarr:9696`, `http://qbittorrent:8080`, etc.) on `arrs-network`. See [troubleshooting.md](troubleshooting.md) for why this is worth checking first, reflexively, any time two containers in this stack won't talk to each other.

**qBittorrent's WebUI username is `admin`, not the TrueNAS account.** Only the password was ever customized on first login — the username was left at its default. Pointing Radarr/Sonarr at the TrueNAS SSH username caused an opaque "Unable to connect" test failure that didn't distinguish a network problem from an auth problem; a raw `docker exec radarr curl -v http://qbittorrent:8080` from inside the container was what isolated it as an auth failure, not a network one.

**Radarr silently grabbed a 20–40GB `Remux-1080p` release under a profile meant to keep things space-reasonable.** The default `HD-1080p` quality profile includes Remux-1080p as an allowed (and top-ranked) tier within "1080p" — a straight, essentially uncompressed Blu-ray rip, which defeats the entire point of choosing 1080p over 4K for storage reasons. Fix: unchecked Remux-1080p in the profile, leaving Bluray/WEB/HDTV-1080p enabled. Sonarr's equivalent profile was checked separately and turned out already clean (no Remux-1080p tier present).

**VNPT's ISP-level DNS interference turned out to be a systemic risk, not a one-off site problem** — hit on Prowlarr (Nyaa.si SSL handshake failures) and again on Seerr (every TMDB-backed discover/search call 500ing) before being recognized as the same underlying pattern. See [troubleshooting.md](troubleshooting.md) for the full diagnosis and the now-standard fix (a per-container `dns: [1.1.1.1, 1.0.0.1]` override).

## Indexers (Prowlarr)

Distinguishing two Prowlarr concepts up front: **Indexers** are the actual torrent search sites (YTS, TPB, Nyaa.si, ...); **Apps** are the consumers Prowlarr pushes working indexers into (Sonarr/Radarr/Lidarr) via sync.

Redundancy was adopted deliberately after enough individual sites turned out to be unreliable for reasons with nothing to do with this NAS — public torrent indexers are volunteer-run, with no uptime guarantee, subject to legal takedowns and mirror rotation. Rather than chasing each site's uptime individually, the stack now runs **8 indexers spread across categories** so Prowlarr silently routes around whichever one is down on a given day:

- **Movies:** YTS, Knaben, LimeTorrents, The Pirate Bay
- **TV:** Knaben, LimeTorrents, The Pirate Bay, showRSS
- **Anime:** AnimeTosho, Nyaa.si, SubsPlease

**1337x remains unavailable** — all 5 mirror domains fail Prowlarr's connection test with `blocked by CloudFlare Protection`. The fix (deploying FlareSolverr as an indexer proxy) is known but not pursued, since coverage is already met without it.

Sonarr and Radarr are connected to Prowlarr as Apps with Full Sync. One non-bug worth noting: a category-only indexer like YTS (movies-only) correctly shows up in Radarr but not Sonarr — that's Prowlarr's category-aware sync working as intended, not a missing connection.

## Quality decision: 1080p, not 4K

Chosen deliberately for the whole stack rather than defaulting to "biggest available": storage headroom (~1.7 TiB usable pool, with 208 TV episodes alone running ~150GB+ at 1080p), no genuine 4K master exists for most of the library's actual content, and no hardware in this NAS can decode HEVC even if it existed (the spare GTX 650 is Kepler-generation NVENC/NVDEC, H.264-only; the Pentium G3240's Haswell Quick Sync predates HEVC support entirely — see [Media Streaming](media-jellyfin.md) for the full GPU evaluation). Radarr/Sonarr/Lidarr are all set to `HD-1080p` profiles (with the Remux tier removed, above) rather than a mixed 720p/1080p profile, so anything already below 1080p shows as upgrade-eligible instead of being silently accepted as good enough.

## Real-world throughput finding (important, not obvious from a generic speedtest)

Discovered via the first real movie downloaded and streamed through this pipeline, not during initial setup — worth calling out here since it changes how this whole stack should actually be used day to day. Full detail, including the `iperf3` diagnostic that found it, lives in [Media Streaming](media-jellyfin.md#real-world-remote-playback-throughput). Short version: a generic speedtest from the NAS showed a healthy 161 Mbps, but the real Vietnam↔Germany path sustains only ~3.78 Mbps with real packet loss — a structural international-routing limitation, not a config problem. TV-episode bitrates (~2 Mbps) stream live from Germany without issue; Blu-ray-tier movie bitrates do not, and need either a manually bitrate-capped stream or a full local download before playing. **Practical consequence for this stack:** it's used for occasional downloads of rare films, watched either quality-capped live or downloaded-then-played-locally — not as a daily-driver live-streaming service.

## Extension: Lidarr (music) — paused, curated-artist scope

Lidarr was added as a 7th service on the same pattern as everything above (`lidarr` container, `lscr.io/linuxserver/lidarr`, port 30094, `/mnt/tank/media:/data`, `/mnt/tank/appdata/arr/lidarr:/config`). The build itself was routine — the interesting part is a real scope change mid-build.

**Original goal:** a broad Spotify-replacement, auto-managing a large music library.

**Revised goal, same day:** Lidarr manages music at **artist/album granularity, not individual-track or playlist granularity** — it downloads whole albums/EPs/singles, it cannot cherry-pick a single song out of an album the way Spotify can. Once that was clear, the goal narrowed to **curated lossless archiving of a small set of specific favorite artists** (e.g. Yoasobi, Ado) whose music isn't available in lossless quality on the local Spotify catalogue — not a full automated sweep.

That reframing changed the quality-profile decision too: `Lossless` (a strict FLAC/ALAC allow-list with no fallback) is a real risk for a broad auto-managed library, since it can leave an album permanently ungrabbed if no lossless release exists anywhere. For a small, manually-curated set of artists, that risk stops applying — any gap is immediately visible and decidable case by case — so the root folder's Quality Profile was set to `Lossless` deliberately.

**Indexer coverage needed no new indexer.** Nyaa.si — already in the stack for anime — has a dedicated "Audio - Lossless" category used heavily by exactly the J-pop/anime-adjacent community this scope targets, and it was already set to sync all categories from its anime setup. Two dedicated music trackers (RuTracker, TorrentGalaxy) were considered and declined.

**Status: paused before the end-to-end test was confirmed complete**, for a genuinely practical reason: the desktop speakers in use (a budget Logitech Z523 2.1 set) aren't capable of resolving a lossless-vs-high-bitrate-lossy difference in practice, which undercuts the immediate payoff of the `Lossless` profile — the underlying reasoning (no lossless option locally) still holds, there's just no audible benefit on the current playback chain. Resume points, in order: (1) decide `Standard` vs. `Lossless` given real playback hardware — switching later is non-destructive, Lidarr just treats existing files as below-cutoff and searches for an upgrade; (2) confirm/retry the first Yoasobi download (Nyaa.si was down mid-test, fell back to The Pirate Bay, never confirmed complete); (3) add a Jellyfin Music library; (4) revisit Finamp for offline mobile listening.

**Note for anyone extending this further:** Seerr/Jellyseerr has **no Lidarr integration at all** (Radarr/Sonarr only) — there's no "request an album" front-end, artists have to be searched and added directly in Lidarr's own UI. The closest friend-facing alternative in this ecosystem is the Discord bot **Requestrr**, not pursued here.

## Extension: friend access — Wizarr + Cleanuparr

Added once the plan changed from "one test account" to eventually sharing Jellyfin with up to ~10 close friends across Vietnam, Germany, and the US. Two tools, evaluated and added together:

- **Wizarr** — turns account creation into a self-service invite link (with expiry/library scoping), instead of manually creating each friend's Jellyfin account and relaying a password over chat. It only automates the Jellyfin account step — it does not auto-provision a linked Seerr account (its own docs describe it as just "guiding" users toward Overseerr/Jellyseerr) — so importing each friend into Seerr for requests stays a manual step either way.
- **Cleanuparr** — auto-cleans stalled/failed/malicious downloads out of qBittorrent and tells Sonarr/Radarr/Lidarr to re-search, useful once more people generate download traffic than one person's own usage patterns.

**Tdarr was evaluated and explicitly declined** at this stage — it would only reformat existing files to a unified codec, and with no HEVC-capable hardware anywhere in this NAS (see the Quality decision above), it wouldn't currently change what's playable or fix any real problem; revisit only if hardware changes.

### Deploy incident: the whole Custom App got stuck "Deploying"

Adding both services and redeploying left the arr-stack Custom App showing "Deploying" indefinitely. `docker ps` confirmed all 7 pre-existing containers were untouched and healthy — the redeploy of a Custom App to add new services does not disturb unrelated containers. `docker ps -a --filter name=wizarr` showed Cleanuparr had actually started fine, but **Wizarr was stuck in `Created`, never started**.

The initial hypothesis — a stuck pull from `ghcr.io`, a registry nothing else in this stack had used before (everything else came from the already-cached `lscr.io`) — turned out to be **wrong**: Cleanuparr also pulls from `ghcr.io` and started immediately, ruling that out. The real cause: **a port collision.** Wizarr's assigned port, `30095`, was already bound by an unrelated `homepage` dashboard container running on this NAS. `docker start wizarr` surfaced the exact error (`Bind for 0.0.0.0:30095 failed: port is already allocated`), confirming it. Fix: reassigned Wizarr to `30097` in the YAML and redeployed — it started cleanly. **Lesson: the Applications page's status badge is not reliable ground truth** — `docker ps -a`, `docker logs`, and `docker top` are.

### Wizarr → Jellyfin connection: two non-obvious findings

**Needed the NAS's own Tailscale IP, not its LAN IP, to reach Jellyfin — the opposite of Seerr's working setup.** Seerr reaches Jellyfin at the LAN IP (`192.168.1.4:30014`), since Jellyfin runs on its own separate Docker network. Trying the same pattern for Wizarr (`192.168.1.4:30013`) silently found zero libraries when scanning; switching to the NAS's Tailscale IP (`100.x.x.x:30013`) immediately found all three libraries. Root cause not fully pinned down (Wizarr's container networking apparently resolves the two paths differently than Seerr's), but the working answer is confirmed empirically — worth checking first if this stack ever adds another Jellyfin-integrated tool that behaves like Seerr rather than like Wizarr.

**A trailing slash in the server URL broke the final "Test & Add" step with an HTTP 404 — even though the library scan on the same URL had already succeeded.** Wizarr's connection-verification call likely does simple string concatenation onto the base URL, so a trailing `/` produces a double-slash request path. Removing the trailing slash fixed it immediately.

**End-to-end test: confirmed working.** Created a test invitation (1-day expiry, Movies + TV libraries), opened the link in a private browser window, created a Jellyfin account through it, logged in, and played a film successfully.

### Cleanuparr: the missing volume mount

Cleanuparr's container originally only had `/config` mounted — no access to the actual downloaded files at all, unlike every other app in this stack, which meant its "Download Directory Source/Target" path-remapping fields could never actually work no matter what was typed into them. Fix: added the same `tank/media:/data` mount every other app already uses. With that in place, Cleanuparr sees files at the exact same `/data/downloads/...` path qBittorrent reports, so both directory-remapping fields are correctly left blank — no translation needed. Connected to qBittorrent and all three *arr apps via the standard container-name pattern (`http://qbittorrent:8080`, `http://sonarr:8989`, `http://radarr:7878`, `http://lidarr:8686`).

## Still open

- 1337x indexer remains Cloudflare-blocked; FlareSolverr not deployed (not urgent, coverage already met by the other 7 indexers).
- Lidarr paused mid-build — see the four resume points above.
- HIMYM's Bazarr subtitle count is stuck at 182/208 (S06E21–24 English, ~20 episodes of S09 Vietnamese) — a manual search was triggered but the count never moved; accepted as-is rather than pursued further.
- HIMYM's old-codec (XviD) episode audit/selective-redownload was considered and explicitly deprioritized.
- No reverse proxy — deferred by choice.
- qBittorrent's global Torrent Management Mode was found set to `Automatic` during a later health check, contradicting the originally-documented `Manual` decision (either changed in an untracked session, or the original change never actually applied). Toggled to Automatic for the one affected torrent as a same-day fix; has not visibly broken anything so far, but is only one data point against the original hardlink-safety concern — worth re-checking if Radarr/Sonarr imports ever start behaving oddly.

**Status:** ✅ Operational end-to-end — Seerr → Prowlarr → Sonarr/Radarr → qBittorrent → Bazarr → Jellyfin confirmed working with a real request, download, and remote playback; Wizarr and Cleanuparr both deployed, connected, and verified working for friend access. Lidarr deployed and configured but paused before its end-to-end test was confirmed.

# Observability — Prometheus + Grafana

*Part of the [homelab-nas](../README.md) project.*

---

**Goal:** real system metrics and dashboards for the NAS — both a practical diagnostic tool (is the Pentium actually coping with Frigate?) and a documented module in its own right.

**What makes this non-trivial on TrueNAS:** almost nothing works the way the standard Prometheus tutorials assume. TrueNAS assigns its own ports, ships Prometheus with no scrape configuration at all, bundles no exporter, and exposes no native Prometheus metrics of its own — Graphite is the only officially supported export path. Five separate assumptions from the usual guides had to be unlearned before a single metric was collected.

## Five things that aren't what the docs assume

**1. Prometheus does not listen on 9090.** TrueNAS launches it with `--web.listen-address=0.0.0.0:30104`. Every "connection refused" during setup traced back to assuming the default port — including the self-scrape target, which must be `localhost:30104`, not `localhost:9090`.

**2. The shipped scrape config is empty.** A fresh install reports **"No scrape pools found"** under Target health. Prometheus collects nothing at all — not even itself — until a config is written. It's running, healthy, and completely idle.

**3. There is no bundled node-exporter.** The TrueNAS Prometheus app is Prometheus alone. Port 9100 refuses connections because nothing is listening on it. And TrueNAS exposes no Prometheus-format metrics natively, so without an exporter there is genuinely nothing to scrape.

**4. Container-to-container DNS does not resolve.** Grafana (`172.16.3.2`) and Prometheus (`172.16.2.2`) sit on *separate* Docker bridge networks. All of these failed:

```
http://prometheus:9090
prometheus.prometheus.svc.cluster.local
http://172.16.2.2:30104          # direct container IP
```

**What works is the host's LAN address:** `http://192.168.1.4:30104`. The same rule applies to the node-exporter scrape target — address the host, not the container.

**5. ixVolume storage isn't persistent enough for config.** App config lived at `/mnt/.ix-apps/app_mounts/prometheus/config/`, which is permission-denied from the host shell (writable only via `docker exec`) — and it was **wiped on app reinstall**. TrueNAS's own documentation recommends host paths over ixVolume for anything that must persist.

## Architecture

**node_exporter v1.9.0, running on the host — deliberately not containerised.** A containerised exporter loses the ZFS collector and host-level `/proc` and `/sys` visibility, which is most of the reason to run it on a NAS at all. Installed to the pool rather than to the OS dataset so it survives TrueNAS upgrades:

```
/mnt/tank/node_exporter-1.9.0.linux-amd64/
```

It exposes roughly **2,770 `node_*` metrics** on `:9100`.

**Auto-start** via System → Advanced Settings → Init/Shutdown Scripts (Post Init):

```bash
nohup /mnt/tank/node_exporter-1.9.0.linux-amd64/node_exporter > /dev/null 2>&1 &
```

**Prometheus config on a host path, not ixVolume.** A dedicated dataset `tank/prometheus-config` — a sibling of `swimming-pool`, placed deliberately *outside* the `sshindow-private` snapshot scope so monitoring config doesn't ride along in personal-data snapshots. Owned `568:568` (the apps user), mounted via Storage Configuration → Prometheus Config Storage → **Host Path**.

Retention `30d`. The TSDB itself was left on ixVolume — time-series data here is disposable, and losing it costs history, not configuration.

```yaml
# prometheus.yml — 15s scrape interval
scrape_configs:
  - job_name: prometheus
    static_configs:
      - targets: ['localhost:30104']
  - job_name: node-exporter
    static_configs:
      - targets: ['192.168.1.4:9100']
```

**Grafana** — datasource `http://192.168.1.4:30104`, dashboard **Node Exporter Full (ID 1860)** imported and populating.

**DHCP reservation** on the router pins the NAS's `enp3s0` interface to `192.168.1.4`, so the hardcoded scrape target can't drift after a lease change. Same reasoning as the camera's reservation in the [NVR module](camera-nvr-frigate.md) — a hardcoded address is only safe if something guarantees it.

Both UIs are reachable remotely over Tailscale (Prometheus `:30104`, Grafana `:30037`) with no port forwarding.

## Verification

**Full NAS reboot, passed unattended.** The Post Init script fired, node_exporter came back on its own, and both scrape targets returned **UP** with no manual intervention. That's the check that matters for a host binary launched by a startup script — the mechanism most likely to silently not fire.

**First real readings**, taken shortly after boot with Frigate recording: CPU ~43.8% busy, load 25.5%, RAM 26.8% of 16 GiB, root filesystem 0.1%. No swap configured, which is normal for this setup.

## Known limitation — stated plainly

**node_exporter is not supported by TrueNAS.** It's a host binary placed outside the appliance's management model, which means:

- Updates are manual
- A SCALE upgrade may break it
- The default config exposes `/metrics` **unauthenticated** to anyone who can reach `:9100`

Mitigations in place: the binary lives on the pool so it survives upgrades, the Post Init script re-launches it after reboot, and exposure is limited to the LAN plus the tailnet with no port forwarding. **TLS and basic auth are not configured** — on an untrusted network this would need fixing before deployment.

## Runbook / gotchas

```bash
# Reload config without restarting (and without losing scrape continuity)
sudo docker kill --signal=SIGHUP ix-prometheus-prometheus-1
```

- **`sudo cat > file` silently fails.** The redirect is performed by the *unprivileged* shell before `sudo` ever runs. Use `sudo tee file > /dev/null << 'EOF'` instead.
- **Container name `ix-prometheus-prometheus-1` is stable; the container ID is not** — it changes on every restart, so never cache the ID in a script.
- **Grafana admin login recovery:** the initial password wasn't accepted, `grafana-cli` is absent from the image, and the API reset failed. A plain `docker stop` + `docker start` made `admin:admin` work, after which the password was changed in the UI.
- The `sshindow` user needs **Permit Sudo** (Accounts → Users) for any Docker CLI work.
- **Changing retention restarts the app and drops collected history.** The 30-day window effectively starts from the moment of that change — set retention before you start caring about the data.

## Still open

- Let data accumulate for a few days before screenshotting for the repo — a week captures day/night cycles and Frigate's daily rhythm, which a fresh install can't show.
- A ZFS-specific dashboard. Currently only a small fraction of the 2,770 available metrics is in use; the `node_zfs_*` collector is there and unexploited.
- Alert rules (disk >85%, CPU sustained >80%). None configured yet — this is currently observability without alerting.

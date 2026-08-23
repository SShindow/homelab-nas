# Power-Loss Resilience

*Part of the [homelab-nas](../README.md) project.*

---

**Goal:** Since the NAS is managed entirely remotely, it needs to survive a full home power interruption (fuse trips, outage) and come back online automatically — without a smart plug, UPS, or WoL trigger, and without needing anyone on-site to intervene.

**Problem discovered:** After a hard power cut (fuse off, not a graceful shutdown), the motherboard's BIOS setting `Restore AC Power Loss → Power On` was reverting to a default that left the machine powered off after power returned. A graceful shutdown never showed this issue, because the PSU's standby power (+5VSB) keeps BIOS settings alive — it's *only* a full power cut that exposes reliance on the onboard CMOS battery.

**Root cause:** A depleted/failing CMOS (CR2032) battery. When standby power is also removed (full power cut), BIOS configuration falls back to the CMOS battery to retain state — a weak battery meant the "power on after power loss" setting silently reset every time.

**Fix & verification:**
1. Replaced the CR2032 CMOS battery on the motherboard (on-site, physical task).
2. Re-entered BIOS and set **Restore AC Power Loss → Power On**.
3. Ran a full real-world test: cut power at the home fuse box → restored it → confirmed the router powered back on independently → confirmed the NAS auto-booted without manual input → confirmed Tailscale auto-reconnected and the NAS was reachable remotely again, fully unattended.

**Why not WoL / a smart plug:** True Wake-on-LAN was ruled out — the fuse cut also kills power to the router itself, so there's no network path to send a WoL packet over in the first place. A smart plug or UPS could add scheduled/remote power-cycling, but wasn't necessary once the actual root cause (CMOS battery) was fixed — the home's own fuse restoring power is now sufficient to bring the whole stack back unattended.

**Status:** ✅ Verified end-to-end — fuse off → fuse on → router boots → NAS auto-boots → Tailscale auto-reconnects, with zero manual steps.

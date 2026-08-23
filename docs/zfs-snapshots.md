# ZFS Snapshot Strategy

*Part of the [homelab-nas](../README.md) project.*

---

**Goal:** Add fast, local point-in-time recovery on top of the existing off-site Google Drive backup — so accidental deletions or changes can be undone instantly without depending on a cloud restore.

**Design:** Two periodic snapshot tasks on `tank/swimming-pool/sshindow-private`:

| Task | Schedule | Retention | Naming schema |
|---|---|---|---|
| Daily | 02:00 every day | 7 days | `daily-%Y%m%d-%H%M` |
| Weekly | 03:00 every Sunday | 4 weeks | `weekly-%Y%m%d` |

**Rationale:**
- Daily snapshots give fine-grained recovery from accidental deletion or edits within the last week.
- Weekly snapshots extend that safety net to roughly a month of history, at lower storage overhead.
- This creates a **layered backup strategy**: Google Drive Cloud Sync (COPY) provides off-site durability against total local hardware loss, while ZFS snapshots provide instant, local, low-latency recovery for day-to-day mistakes — each covering a failure mode the other doesn't.

**Status:** ✅ Tasks enabled and scheduled. First daily and weekly snapshots run automatically without manual intervention.

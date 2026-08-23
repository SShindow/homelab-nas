# Google Drive → NAS Backup

*Part of the [homelab-nas](../README.md) project.*

---

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

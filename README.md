# ThanksTim!

macOS restore TUI for when Migration Assistant hurt you.

This tool restores selected macOS/system settings and user data from a mounted Time Machine backup (the `macOS - Data` folder). It is designed for “fresh install, then restore in a known-good order” and keeps detailed, per-session logs (no mixing between sessions).

## Safety Model (Why this exists)
- **No silent rollbacks**: the app doesn’t “undo” work if something fails.
- **Retry-safe by design**: you can rerun phases repeatedly after failures (no `rsync --delete*`, no `--remove-source-files`, no `--inplace`, no automatic cleanup).
- **Inventory first**: phases support inventory/dry-run, restore, and logs, with explicit success/failure UI.
- **Python control plane, rsync/sudo data plane**: the app orchestrates; `rsync` performs the file operations.
- **Per-session state**: everything is stored under a user-writable sessions directory (not inside the app bundle/repo).
- **Not an incremental tool**: this is intended for bulk restoration of a full system state after a fresh install. You *can* use it for piecemeal or incremental restores, but that’s outside the intended flow and is at your own peril.

## Before You Start (Important)
**Step 0: Disable automatic Time Machine backups.**
Consider installing homebrew first and update BASH and RSYNC. It'll work as-is but it wont hurt and will speed things up.

System Settings → Time Machine: set to **Manual** / disable automatic backups.  
Some restore phases can restore prior system state and may re-enable automatic backups from the selected restore point.

## How To Find the “Sacred Path” (`macOS - Data`)
1. Connect to your Time Machine disk (local or network).
2. In Finder, right-click the machine’s `.sparsebundle` → **Open With** → **DiskImageMounter**.
   - If encrypted (it should be), enter the password and **DO NOT** store it in Keychain.
   - Wait for the disk image to mount and appear in Finder.
3. In Finder, open the latest (or desired) backup and navigate into **`macOS - Data`**.
4. Open Terminal and drag the **`macOS - Data`** folder into the Terminal window.
5. Copy the **entire** path that appears in Terminal.
6. Paste it into ThanksTim as the **Time Machine Backup Path** and verify.

The app also includes the same instructions in the prerequisites screen (`Path Help`).

## Install & Run (Single Binary) — Recommended
### Prereqs (Target Mac)
- macOS is installed and you can boot into it normally (not Recovery).
- An admin user exists on the target Mac.
  - Best results: create the user with the **same short username** as the source user in the backup.
  - Different shortname is supported, but you’ll get an explicit FAFO warning and you may run into UID/GID/ownership weirdness.
- The Time Machine backup is mounted (local disk or network sparsebundle) and you have the full `macOS - Data` path.

### Zero-Knowledge Quick Start (Download → Run)
This is a Terminal app (not a double-click GUI). These steps assume a default macOS setup.

1. Download the latest **ThanksTim** binary from the GitHub Releases page.
   - It will usually download to **Downloads** as a `.zip`.
2. Open Finder → **Downloads** and double-click the `.zip` to extract `ThanksTim`.
   - The file has **no extension**. That is normal.
3. (Optional) Copy `ThanksTim` to an SD card / USB stick / external drive.
   - You can run it from there or copy it to the target Mac’s Desktop/Downloads.
4. Open **Terminal**:
   - Press **Command + Space**, type `Terminal`, press **Return**.
5. In Terminal, go to the folder that contains `ThanksTim`:
   - If it’s in Downloads:
     - `cd ~/Downloads`
   - If it’s on the Desktop:
     - `cd ~/Desktop`
   - Tip: you can drag the `ThanksTim` file from Finder into Terminal to paste its full path.
6. Make it executable and run it:
   - `chmod +x ./ThanksTim`
   - `./ThanksTim`
7. If macOS blocks it (Gatekeeper/quarantine):
   - `xattr -dr com.apple.quarantine ./ThanksTim`
   - Then run `./ThanksTim` again.

### Run From Any Drive
You can run the binary from:
- Local disk (Desktop/Downloads/etc.)
- USB stick / SD card / external drive

Session state is still stored in your user state directory by default (see **Sessions & State**).

### Steps
1. Copy `ThanksTim` to the target Mac (any folder / any drive).
2. In Terminal:
   - `chmod +x ./ThanksTim`
   - `./ThanksTim`
   - Optional: `mv ThanksTim thankstim && ./thankstim`
3. If macOS blocks it (Gatekeeper/quarantine):
   - `xattr -dr com.apple.quarantine ./ThanksTim`

### Don’t Run From Recovery Terminal
Running from Recovery is unsupported:
- Your “home” becomes `/var/root` (session state won’t land where you expect and may not persist).
- The tool targets real paths like `/Users`, `/Library`, `/Applications`; in Recovery that does not automatically point at the installed OS volume.

## Sessions & State
- Default state directory (macOS): `~/Library/Application Support/ThanksTim/sessions`
- Override state base directory with `THANKSTIM_STATE_DIR` (sessions stored under `<base>/sessions`)
- Each phase writes into its own directory under the session:
  - `phases/<phase>/runs/*.log` (complete logs, including long file lists)
  - Companion artifacts (when applicable): `*_errors.log`, `*_overwrites.log`, `*_results.log` (also listed in `View Logs`)
  - `phases/<phase>/state.json` (latest run summary)
  - `phases/<phase>/history.jsonl` (append-only run history)

## Prefetch Inventory (Optional, Recommended)
After **PHASE 0 → Verify** succeeds, ThanksTim can prompt for your sudo password once and start a **lightweight background prefetch** (inventory-only):

- Populates the Dashboard **Overview** pane with counts/sizes (apps, brew, personal folders, postgres clusters).
- Writes prefetch logs under the relevant phase directories (look for `*_prefetch*.log` under `phases/*/runs/`).
- Prefetch provides estimates for the Overview; phase-level inventories/dry-runs remain the source of truth.

Notes:
- The sudo password is stored **in memory only** (never written to disk). If you restart the app, you’ll be prompted again.
- You can skip the prefetch prompt and still run inventories per phase as usual.

## Prerequisites Screen Behavior
- **Source path and source user must be valid** (no FAFO for invalid source).
- **Target can differ from source** and/or **destination can be non-existent**:
  - The app shows a warning and requires an explicit **FAFO** acknowledgment to proceed.
  - Restore phases create destination directories as needed (the original script behavior).

## Summary / Report
The Dashboard includes a **SUMMARY / REPORT** entry which shows:
- What was detected in the backup (prefetch/inventory summaries)
- What you selected per phase
- What restored OK/FAIL (based on latest per-phase restore progress)

## Notes / Warnings
- PostgreSQL: inventory is available; restore is labeled **DANGER** and is implemented for **Homebrew clusters** only (Postgres.app / /Library/PostgreSQL are inventory-only).
- After Phase 1 (system-level settings), a reboot is required/recommended.
- Dev environment restore is “all-or-nothing”: it rsyncs the source user home (`Users/<source>/.`) into the target home (`/Users/<target>/`) and may overwrite existing dotfiles/config (and includes secrets like `.ssh`/`.gnupg`).
- Homebrew restore supports Intel (`usr/local/Cellar`) and Apple Silicon (`opt/homebrew/Cellar`) inventories; inventory uses `sudo` for backup access but installs run as your user (no `sudo`). Homebrew must already be installed.
- Applications restore is usually fine on a clean system after Phase 1, but in edge cases you may hit Gatekeeper/quarantine prompts (fix: `sudo xattr -dr com.apple.quarantine /Applications/App.app`).
- The tool is designed for “fresh install → restore in one session”. It’s safe to retry immediately after failures, but if you start using the machine in between runs, rerunning restores can overwrite those new changes with the backup’s state.

## Advanced: UID/GID Mismatch (dscl) — DANGER
If you create a temporary user first, your “real” user can end up as UID 502 instead of 501. Since Time Machine preserves ownership by numeric UID, this can cause permission weirdness after restore.

ThanksTim does **not** automate UID changes. If you decide to use `dscl`, read the in-app **ADVANCED: UID/GID FIX (DSCL)** screen and proceed only if you know how to recover from a bad UID change (keep an unaffected admin user).

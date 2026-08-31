# Server OS Snapshot & Disaster Recovery Runbook

## 📌 Overview
This runbook outlines the standard operating procedure (SOP) for taking a full bare-metal OS snapshot of a Linux VPS using `tar`, securely transferring it to an off-site cloud storage (Google Drive/AWS S3) via `rclone`, and executing disaster recovery (full and partial) to restore normal operations.

---

## 🛠️ Phase 1: Taking the OS Snapshot

We use the native `tar` utility to create an archive of the entire system root (`/`), excluding dynamic, temporary, and unnecessary application files (like `.cache` and `node_modules`).

### The Backup Command
Execute the following command as the `root` user:

```bash
sudo tar -cvpzf /lightsail-os-backup.tar.gz \
  --exclude=/lightsail-os-backup.tar.gz \
  --exclude=/root/.cache \
  --exclude=/home/*/.cache \
  --exclude=/proc \
  --exclude=/tmp \
  --exclude=/sys \
  --exclude=/mnt \
  --exclude=/media \
  --exclude=/dev \
  --exclude=/run \
  --exclude='node_modules' \
  /
```

### 💡 Flag & Logic Breakdown
*   `-c`: **Create** a new archive.
*   `-v`: **Verbose** output (lists files as they are processed).
*   `-p`: **Preserve permissions** (Crucial for system stability).
*   `-z`: **Gzip compression** (reduces final file size).
*   `-f`: **Filename** (Must be the last flag before the file name).
*   `--exclude`: Prevents infinite loops (excluding the backup file itself) and skips pseudo-filesystems (`/proc`, `/sys`), temp directories, and heavy dependency folders like `node_modules`.
*   `/` (at the end): Dictates that the archival process begins from the absolute root directory.

---

## ☁️ Phase 2: Offsite Transfer

A backup stored on the same server is a single point of failure. Transfer the archive offsite immediately using `rclone`.

```bash
# Copy the backup to Google Drive (or any S3/Object storage)
rclone copy /lightsail-os-backup.tar.gz gdrive:backups/

# Verify the transfer is successful before deleting the local copy (optional)
# rm /lightsail-os-backup.tar.gz
```

*(Note: VPS providers like DigitalOcean/AWS offer 1-click snapshots from their web panels. While highly recommended for full infrastructure backups, manual `tar` snapshots are essential for cross-cloud migrations or bare-metal servers without control panels.)*

---

## 🚨 Phase 3: Disaster Recovery & Restoration

### Scenario A: Partial Restoration (Recommended for Specific Data Loss)
If only specific directories (e.g., `/var/www` or `/etc/nginx`) are accidentally deleted or corrupted, do NOT restore the entire OS, as this will overwrite running system processes and cause "file busy" errors.

**1. Locate the exact path inside the archive:**
*Note: `tar` automatically strips the leading `/` during creation for safety.*
```bash
tar -tzf /lightsail-os-backup.tar.gz | grep "var/www"
tar -tzf /lightsail-os-backup.tar.gz | grep "etc/nginx"
```

**2. Extract ONLY the required directories to the root:**
```bash
sudo tar -xvpzf /lightsail-os-backup.tar.gz -C / var/www etc/nginx --numeric-owner
```

### Scenario B: Full System Restoration
If the server is severely compromised, download the backup and extract it starting from the root directory.

```bash
# Download from offsite storage
rclone copy gdrive:backups/lightsail-os-backup.tar.gz /

# Execute full restore
sudo tar -xvpzf /lightsail-os-backup.tar.gz -C / --numeric-owner
```
*(Warning: You will see "Exiting with failure status due to previous errors" at the end. This is normal behavior when `tar` attempts to overwrite active kernel/running system processes in `/sbin` or `/proc`.)*

### Scenario C: Safe Extraction (Testing/Inspection)
To extract the backup contents into a local working directory without overwriting any actual system files, simply omit the `-C /` flag.
```bash
mkdir ~/backup-inspect && cd ~/backup-inspect
sudo tar -xvpzf /lightsail-os-backup.tar.gz
```
To extract while strictly preventing the overwriting of any existing files, use the `-k` (`--keep-old-files`) flag:
```bash
sudo tar -xkvpzf /lightsail-os-backup.tar.gz -C /
```

---

## ⚠️ Crucial Concept: `--numeric-owner`
Linux manages permissions via `UID` (User ID) and `GID` (Group ID), not usernames. 
*   When migrating backups between different servers, a username (e.g., `admin`) might have `UID 1000` on Server A, but `UID 1001` on Server B. 
*   Using `--numeric-owner` forces `tar` to assign permissions based on the original numeric IDs rather than trying to match string usernames. 
*   **Best Practice:** Ensure the destination server has identical usernames mapped to the exact same UIDs as the original server to prevent configuration conflicts in services like Nginx or Systemd.

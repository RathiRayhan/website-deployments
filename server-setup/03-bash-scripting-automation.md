# Topic 3: Bash Scripting, Cron Jobs & Backup Automation

## Overview
This runbook covers the third and final topic of Phase 1. It contains standard operating procedures (SOPs) and automated scripts for Linux server provisioning, task scheduling via Cron, and disaster recovery/backup automation.

---

## Part 1: Bash Scripting & Server Provisioning

### 1.1 Overview
This script automates standard server provisioning tasks: verifying/creating specific application directories and bulk-creating system users silently without interactive prompts.

**Prerequisites:**
- **OS:** Ubuntu/Debian (Linux)
- **Permissions:** `sudo` access required
- **Shell:** Bash (`/usr/bin/bash`)

### 1.2 The Script (`setup.sh`)
```bash
#!/usr/bin/bash

# Exit immediately if a command exits with a non-zero status
set -e 

DIR_PATH="/opt/myapp-data2"

# 1. Directory Provisioning
if [ -d "$DIR_PATH" ]; then
        echo "[INFO] Folder '$DIR_PATH' already exists."
else
        echo "[INFO] Folder does not exist. Creating..."
        sudo mkdir -p "$DIR_PATH"
        echo "[SUCCESS] Folder '$DIR_PATH' created successfully."
fi

# 2. User Provisioning
for i in {1..3}; do
        # -m flag creates the user's home directory
        sudo useradd -m "another-test$i"
        echo "[SUCCESS] User 'another-test$i' created."
done
```

### 1.3 Troubleshooting & Key Learnings
- **The `set -e` Fail-Safe:** Highly recommended for production scripts. It prevents the script from executing subsequent commands if a prior command (e.g., `mkdir` failing due to a lack of `sudo`) fails.
- **Loop Syntax (`{ }` vs `[ ]`):** In Bash, ranges in `for` loops must use curly braces (e.g., `{1..3}`). Using square brackets `[1..3]` is treated as literal characters and will result in invalid syntax or malformed usernames.
- **`useradd` vs `adduser`:**
    - `adduser`: A high-level, interactive script. Unsuitable for silent automation as it stops execution to prompt for passwords and user details.
    - `useradd`: A low-level binary. Perfect for non-interactive scripts. Always append the `-m` flag to ensure the user's home directory is created.

---

## Part 2: Cron Job Automation & Production Logging

### 2.1 Cron Fundamentals & Syntax
Cron is a time-based job scheduler in Linux used to automate repetitive tasks (e.g., health checks, backups, log rotations).

The crontab syntax consists of 5 time-and-date fields followed by the command:
```bash
* * * * * command_to_execute
┬ ┬ ┬ ┬ ┬
│ │ │ │ │
│ │ │ │ └─ Day of week (0 - 7) (0 or 7 is Sunday)
│ │ │ └─── Month (1 - 12)
│ │ └───── Day of month (1 - 31)
│ └─────── Hour (0 - 23)
└───────── Minute (0 - 59)
```

* **Wildcards & Operators:**
  * `*` : Every/All.
  * `*/n` : Step values (e.g., `*/2` in the first field means every 2 minutes).
  * `@reboot` : Runs once at system startup.

### 2.2 Production Best Practices for Cron
Cron executes in a highly restricted environment with a minimal `$PATH`. To prevent "Command not found" errors, strictly adhere to the following industry standards:

* **Script Location:** Never store production scripts in the user's home directory (`~/`). 
  * System-wide scripts: `/opt/scripts/` or `/usr/local/bin/`
* **Absolute Paths:** Always use the full absolute path for both the script and the internal commands within the script. Find command paths using `which <command>`.
* **Environment Variables:** Explicitly define the `PATH` at the top of the crontab file.
* **Disable Default Mail:** Prevent cron from filling up the local mail spool by disabling email alerts. Add `MAILTO=""` at the top of the crontab.

### 2.3 Logging & Error Handling (`2>&1`)
Since cron runs in the background without a terminal, standard output (stdout) and standard error (stderr) must be explicitly captured.

```bash
>> /var/log/health.log 2>&1
```
**Breakdown:**
* `>> /var/log/health.log` : Appends standard output (success logs) to the file.
* `2>&1` : Redirects Standard Error (Channel 2) to the same location as Standard Output (Channel 1).

### 2.4 Implementation: Standard Setup
**Setup:** `sudo crontab -e`

```bash
MAILTO=""
PATH=/usr/local/sbin:/usr/local/bin:/sbin:/bin:/usr/sbin:/usr/bin

# Run system health check every 2 minutes and capture all logs
*/2 * * * * /opt/scripts/health_check.sh >> /var/log/health_check.log 2>&1
```

### 2.5 Disaster Recovery & Troubleshooting Simulation
**Scenario:** A cronjob is scheduled but the expected outcome isn't happening.

1. **Check if Cron fired the job:**
   ```bash
   sudo journalctl -u cron --since "1 hour ago"
   ```
2. **Check the custom log file for script errors:**
   ```bash
   cat /var/log/health_check.log
   ```
   *Simulation Result:* Found `Permission denied` error.
3. **Root Cause Fix:** The script lacked execution permissions. Fixed via:
   ```bash
   sudo chmod +x /opt/scripts/health_check.sh
   ```

---

## Part 3: Backup Automation & Retention Policy

### 3.1 Overview
This section outlines a robust, automated backup solution designed to compress critical server data, securely transfer it to a remote vault, and automatically manage disk space via a strict retention policy. 

**Core Tools Used:**
*   `tar`: For efficient data compression.
*   `rsync`: For secure, delta-transfer synchronization.
*   `find`: For automated log rotation and backup retention.

### 3.2 The Backup Script (`server_backup.sh`)
This script handles the compression of the target directory and the immediate transfer to the backup vault. It utilizes the `--remove-source-files` flag to ensure the source server's storage is not consumed by staging files.

**File Location:** `/opt/scripts/server_backup.sh`  
**Permissions:** `sudo chmod +x /opt/scripts/server_backup.sh`

```bash
#!/bin/bash
set -e

# ==========================================
# Configuration Variables
# ==========================================
SOURCE_DIR="/home/harry/important_data"
BACKUP_TEMP_DIR="/tmp"
REMOTE_VAULT="/tmp/backup_vault/" # Note: Use user@remote_ip:/path/ for remote servers

# Generate a unique timestamped filename
BACKUP_FILENAME="backup_$(date +%Y-%m-%d_%H-%M-%S).tar.gz"

# ==========================================
# Execution
# ==========================================
# Step 1: Compress the target directory into the staging area
tar -czvf ${BACKUP_TEMP_DIR}/${BACKUP_FILENAME} ${SOURCE_DIR}

# Step 2: Transfer to the vault and instantly remove the local staging copy
rsync -avzh --remove-source-files ${BACKUP_TEMP_DIR}/backup_*.tar.gz ${REMOTE_VAULT}
```

### 3.3 Automation via Cron
Scheduled via the root user's crontab to ensure necessary permissions for reading protected system directories.

**Setup:** `sudo crontab -e`

```bash
MAILTO=""
PATH=/usr/local/sbin:/usr/local/bin:/sbin:/bin:/usr/sbin:/usr/bin

# Schedule execution: Every day at 00:00 (Midnight)
0 0 * * * /opt/scripts/server_backup.sh >> /var/log/backup.log 2>&1
```

### 3.4 Data Retention Policy (Cleanup)
To prevent the destination server's storage from overflowing, a retention policy is enforced.

**Best Practice:** In a multi-server architecture, this command should be scheduled as a separate cron job on the **Destination/Backup Server**.

```bash
# Delete .tar.gz files in the vault older than 7 days
find /tmp/backup_vault/ -type f -name "*.tar.gz" -mtime +7 -delete
```

### 3.5 Backup Troubleshooting
If the backup fails to generate or transfer, inspect the log file:

```bash
cat /var/log/backup.log
```
*   **Common Issue (Permission Denied):** If `tar` throws permission errors, ensure the script is executing with sufficient privileges (root) or that the source directory allows read access. Fix using `chmod` and `chown` accordingly.

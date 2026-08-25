# MySQL/MariaDB Migration & Disaster Recovery Runbook

## 📌 Overview
This runbook outlines the standard operating procedure (SOP) for safely backing up a production MySQL/MariaDB database, securely transferring it across servers using a local machine as a bastion/middleman, and accurately restoring it on the destination server. It also covers direct migrations to Managed Cloud Databases (AWS RDS, PlanetScale, Aiven).

---

## 🛠️ Phase 1: Local Database Backup (Export)

We use the `mysqldump` utility to generate a logical backup (`.sql` file) of the target database.

### 1. Identify the Target Database
Log into the database shell to verify the database name:
```bash
sudo mysql
MariaDB [(none)]> SHOW DATABASES;
MariaDB [(none)]> \q
```

### 2. Execute the Backup Command
To back up a database (e.g., `wordpress_db`), execute the following command:
```bash
sudo mysqldump -u root wordpress_db > wordpress_db_backup.sql
```
**Command Breakdown:**
*   `sudo`: Runs the command with system root privileges.
*   `mysqldump`: The MySQL utility used to export database structures and data.
*   `-u root`: Specifies the MySQL user (`root`).
*   `wordpress_db`: The name of the target database to export.
*   `>`: Standard shell redirection, pushing the output to a file.
*   `wordpress_db_backup.sql`: The output plain-text file.

**💡 Authentication Context & Variations:**
*   **Unix Socket Authentication (Default on modern Linux):** By using `sudo`, we authenticate as the system `root`, allowing us to bypass the MySQL password prompt. 
*   **Password/Standard Authentication:** If Unix socket authentication is disabled, or you are using a non-root user, you MUST include the `-p` flag to prompt for a password:
    ```bash
    mysqldump -u root -p wordpress_db > wordpress_db_backup.sql
    ```

Verify the backup file was created successfully:
```bash
ls -lh wordpress_db_backup.sql
```

---

## ☁️ Phase 2: Secure Transfer (The Middleman Approach)

Transferring files directly between two remote servers (Server A to Server B) requires placing your private SSH key on Server A, which is a **massive security risk**. Instead, we use our local machine as a secure middleman.

### 1. Download to Local Machine
Pull the backup file from the source server to your local machine:
```bash
scp -P 5259 user@<SOURCE_IP>:/home/user/wordpress_db_backup.sql .
```
**Command Breakdown:**
*   `scp`: Secure Copy Protocol, transfers files securely over SSH.
*   `-P 5259`: Specifies a custom SSH port (default is 22).
*   `user@<SOURCE_IP>:/path/...`: The remote source user, IP, and exact file path.
*   `.`: Represents the current directory on your local machine (destination).

### 2. Upload to Destination Server
Push the file from your local machine to the destination server using your private key:
```bash
scp -i ~/.ssh/your_private_key wordpress_db_backup.sql admin@<DESTINATION_IP>:/home/admin/
```

---

## 🚨 Phase 3: Local Restoration (Import)

### ⚠️ CRITICAL WARNING: `mysqldump` vs `mysql`
A very common pitfall is attempting to restore a database using the `mysqldump` command. 
*   **`mysqldump`** is ONLY for exporting/creating backups. If you use it with a `<` (input) redirection, it will ignore the file, attempt to dump an empty database, and print the output to your terminal screen.
*   **`mysql`** is the standard client used for importing/restoring data.

### 1. Create an Empty Database
Unlike some NoSQL databases, MySQL requires the destination database to exist before importing data into it.
```bash
sudo mysql
MariaDB [(none)]> CREATE DATABASE wordpress_db;
MariaDB [(none)]> \q
```

### 2. Execute the Restore Command
Use the standard `mysql` client to import the `.sql` file into the newly created database:
```bash
sudo mysql -u root wordpress_db < wordpress_db_backup.sql
```
**Command Breakdown:**
*   `mysql`: The MySQL client utility used here for importing.
*   `-u root`: The MySQL user.
*   `wordpress_db`: The destination database created in the previous step.
*   `<`: Shell input redirection, feeding the `.sql` file into the database.

*(Note: A successful restoration will return no output in the terminal.)*

### 3. Verify Restoration
```bash
sudo mysql
MariaDB [(none)]> USE wordpress_db;
MariaDB [(none)]> SHOW TABLES;
```

---

## 🌐 Phase 4: Managed Cloud Migration (AWS RDS, PlanetScale, Aiven)

When working with managed cloud databases, you do not have SSH access to the underlying server. Instead, the provider gives you a remote Connection Host/IP, Username, and Password. You control the migration entirely from your local machine.

### Scenario A: Push Local Backup to Managed Cloud
To import your local backup into a remote managed MySQL instance:
```bash
mysql -h [CLOUD_HOST_URL] -u [CLOUD_USER] -p [CLOUD_DB_NAME] < wordpress_db_backup.sql
```
**Command Breakdown:**
*   `mysql`: The import utility (running locally).
*   `-h [CLOUD_HOST_URL]`: The `--host` flag tells the client to connect to the remote cloud server instead of the local machine.
*   `-u [CLOUD_USER]`: The username provided by the cloud platform.
*   `-p`: Prompts for the cloud database password securely.
*   `[CLOUD_DB_NAME]`: The target database on the cloud server.
*   `< wordpress_db_backup.sql`: The local backup file being pushed.

### Scenario B: Pull Backup from Managed Cloud to Local
To take a backup of a live cloud database and download it directly to your local machine:
```bash
mysqldump -h [CLOUD_HOST_URL] -u [CLOUD_USER] -p [CLOUD_DB_NAME] > cloud_backup.sql
```
**Command Breakdown:**
*   `mysqldump`: The export utility (running locally).
*   `-h [CLOUD_HOST_URL]`: Connects to the remote cloud database.
*   `> cloud_backup.sql`: Streams the downloaded data directly into a local `.sql` file.

---

## 🎩 Operational Tip: Recovering Lost Credentials

In unmanaged or legacy environments, database credentials may be undocumented or lost by the stakeholders. If you only have root SSH access, you can securely extract the active database credentials directly from the application's configuration files without needing to reset passwords and cause downtime:

*   **WordPress:** `cat /var/www/html/wp-config.php | grep DB_`
*   **Node.js/Laravel/Python:** `cat /path/to/app/.env`

These files contain the active database name, username, and password in plain text, allowing you to perform necessary administrative tasks seamlessly.

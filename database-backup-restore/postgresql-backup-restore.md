# PostgreSQL & Supabase (Managed Cloud) Migration Runbook

## 📌 Overview
This Standard Operating Procedure (SOP) details the production-level processes for backing up, verifying, and restoring PostgreSQL databases. It covers local environments (VPS to VPS) as well as managed cloud platforms like Supabase.

---

## 🛠️ Phase 1: Local Backup Methods

### Method A: Plain Text Backup (Basic)
Useful for human-readable dumps, similar to standard MySQL backups.
```bash
sudo -u postgres pg_dump pern_stack > normal_backup.sql
```
**Command Breakdown:**
*   `sudo -u postgres`: Executes the command as the Linux system user `postgres` (bypassing password prompts).
*   `pg_dump`: The PostgreSQL utility for exporting a database.
*   `pern_stack`: The name of the target database being backed up.
*   `>`: Standard shell redirection, pushing the output into a file.
*   `normal_backup.sql`: The output plain-text file.

### Method B: Custom Format Backup (Production Standard)
**Why Custom Format?** 
Unlike plain text, the custom format (`--format=c`) is compressed by default, saving disk space. More importantly, it allows for selective restoration (restoring specific tables) and parallel processing, making it significantly more powerful for large production databases.

**Prerequisite: Fixing Permission Errors**
Running `pg_dump` with a custom file output in a user's home directory often throws a `Permission denied` error because the `postgres` system user lacks write access there.
```bash
# 1. Create a dedicated, secure backup directory
sudo mkdir -p /var/backups/postgres
# 2. Change ownership to the postgres user
sudo chown postgres:postgres /var/backups/postgres
# 3. Restrict permissions (Only owner can read/write)
sudo chmod 700 /var/backups/postgres
```

**Execute Custom Backup:**
```bash
sudo -u postgres pg_dump --format=c pern_stack --file=/var/backups/postgres/pern_stack_production_backup.dump
```
**Command Breakdown:**
*   `sudo -u postgres`: Run as the `postgres` superuser.
*   `pg_dump`: The export utility.
*   `--format=c`: Specifies the custom archive format (compressed, binary-like).
*   `pern_stack`: The target database name.
*   `--file=/path/to/...`: Defines the exact output file path instead of relying on shell redirection (`>`).

---

## 🔍 Phase 2: Backup Verification
Always verify the integrity of a custom dump before attempting a restore.

```bash
sudo -u postgres pg_restore --list /var/backups/postgres/pern_stack_production_backup.dump
```
**Command Breakdown:**
*   `pg_restore`: The PostgreSQL utility for restoring custom/binary format backups.
*   `--list`: Scans the backup file and prints the Table of Contents (TOC) to the terminal without restoring any data.
*   `/var/backups/...`: The target backup file to scan.

---

## 🚨 Phase 3: Local Disaster Recovery (Restoration)

### 1. Pre-requisite: Database Creation
Unlike MongoDB, PostgreSQL requires an empty database to exist before restoring data into it.
```bash
# Enter the shell and create the DB if it doesn't exist
sudo -u postgres psql
postgres=# CREATE DATABASE pern_stack;
postgres=# \q
```

### 2. Execute Restoration
```bash
sudo -u postgres pg_restore -d pern_stack -1 /var/backups/postgres/pern_stack_production_backup.dump
```
**Command Breakdown:**
*   `pg_restore`: The restoration utility for custom formats.
*   `-d pern_stack`: Specifies the destination database name (`pern_stack`).
*   `-1`: Executes the entire restore as a "Single Transaction". If any error occurs, the whole process rolls back.
*   `/var/backups/...`: The input backup file path.

*(Note: After restoration, grant necessary connection privileges to specific application users if required).*

---

## ☁️ Phase 4: Managed Cloud Migration (Supabase)

Supabase databases contain internal schemas (`auth`, `storage`, `graphql`, etc.) for their backend services. To avoid conflicts, we strictly isolate the `public` schema (where the actual application data lives).

### Scenario A: Backup Supabase ➡️ Restore Locally
**1. Dump from Supabase:**
```bash
pg_dump --format=c --no-owner --no-privileges -n public "postgresql://postgres:[PASSWORD]@[SUPABASE_HOST]:5432/postgres" --file=/var/backups/postgres/supabase_backup.dump
```
**Command Breakdown:**
*   `pg_dump`: The export utility (run locally, no `sudo` needed for remote connections).
*   `--format=c`: Custom compressed format.
*   `--no-owner`: Strips ownership commands so the local database doesn't try to assign tables to Supabase-specific cloud users.
*   `--no-privileges`: Prevents exporting access privileges/grants.
*   `-n public`: Targets ONLY the `public` schema (ignores internal schemas).
*   `"postgresql://..."`: The remote Connection URI/String containing credentials and host.
*   `--file=...`: The local output path.

**2. Restore Locally (Handling Existing Schemas):**
When restoring to a newly created local database, PostgreSQL creates an empty `public` schema by default. To prevent `schema "public" already exists` errors, use the cleanup flags:
```bash
sudo -u postgres pg_restore -d supabase_local -1 --clean --if-exists /var/backups/postgres/supabase_backup.dump
```
**Command Breakdown:**
*   `-d supabase_local`: The local target database.
*   `-1`: Single transaction mode.
*   `--clean`: Drops (deletes) existing database objects (like the default `public` schema) before restoring them from the dump.
*   `--if-exists`: Prevents the `--clean` command from throwing errors if it tries to drop something that doesn't exist yet.

### Scenario B: Backup Local ➡️ Restore to Supabase
When pushing a local backup to a fresh Supabase project, **DO NOT** use `--clean`. Dropping schemas in Supabase will break their internal API and PostgREST configurations.

```bash
pg_restore -d "postgresql://postgres:[PASSWORD]@[SUPABASE_HOST]:5432/postgres" -n public --no-owner --no-privileges -1 /var/backups/postgres/pern_stack_production_backup.dump
```
**Command Breakdown:**
*   `pg_restore`: The restoration utility.
*   `-d "postgresql://..."`: Connects directly to the Supabase target database via URI.
*   `-n public`: Ensures only the `public` schema from the backup is restored.
*   `--no-owner` & `--no-privileges`: Prevents pushing local VPS user permissions to the Supabase cloud environment.
*   `-1`: Single transaction mode for safety.

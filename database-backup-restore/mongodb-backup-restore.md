# MongoDB Disaster Recovery & Cloud Migration Runbook

## 📌 Overview
This standard operating procedure (SOP) details the exact steps for verifying, backing up, and restoring MongoDB databases using the `mongodump` and `mongorestore` utilities. It covers both local server environments and managed cloud migrations (MongoDB Atlas).

---

## 🛠️ Phase 1: Database Exploration & Verification

Before taking a backup, it is a standard practice to verify the target database and its collections.

### 1. Access the MongoDB Shell
Connect to the local MongoDB instance:
```bash
mongosh
```
*(Note: Use `mongo` for older versions).*

### 2. Verify Databases and Collections
Locate the target database (e.g., `IMS`) and verify its internal structure:
```javascript
test> show dbs;
test> use IMS;
IMS> show collections;
```
To ensure the collections contain the expected data, query a specific collection (e.g., `products`):
```javascript
IMS> db.products.find().pretty();
IMS> exit;
```

---

## 📦 Phase 2: Production Backup (Archive Mode)

By default, `mongodump` exports databases as a nested directory structure containing `.bson` and `.json` files. While useful for local debugging, transferring directories across servers is inefficient and prone to permission errors. 

**Production Standard:** We use the `--archive` flag to stream the entire backup into a single, easily transportable file.

```bash
mongodump --db=IMS --archive=IMS_backup.archive
```
**Command Breakdown:**
*   `mongodump`: The utility for creating a binary export of the database contents.
*   `--db=IMS`: Specifies the exact database to back up (`IMS`).
*   `--archive=...`: Instructs the utility to output a single archive file instead of a directory structure.

*(Verify the creation of the file using `ls -lh IMS_backup.archive`)*

---

## 🚨 Phase 3: Disaster Recovery (Restoration)

### 1. Simulating a Disaster (Optional/Lab only)
To test the backup, we can drop the existing database:
```bash
mongosh
test> use IMS;
IMS> db.dropDatabase();
IMS> show dbs;  # Verify IMS is gone
IMS> exit;
```

### 2. Restoring from Archive
Unlike relational databases, MongoDB does not require you to create an empty database prior to restoration. It will automatically recreate the database and collections based on the archive file.

```bash
mongorestore --archive=IMS_backup.archive
```
**Command Breakdown:**
*   `mongorestore`: The utility for loading data from a binary database dump.
*   `--archive=...`: Tells the utility to read the input from the specified archive file.

*(Once completed, log back into `mongosh` and use `show dbs` to verify the restoration.)*

---

## ☁️ Phase 4: Cloud Migration (MongoDB Atlas)

To migrate data between a local server and a managed cloud database like MongoDB Atlas, we utilize the exact same tools but append the cluster's Connection String (`--uri`).

### ⚠️ Prerequisite: IP Whitelisting
Before running any remote commands against MongoDB Atlas, you **MUST** whitelist your local machine's or VPS's IP address in the Atlas Dashboard (under *Network Access*). Otherwise, the connection will time out.

### Scenario A: Push Local Backup to MongoDB Atlas
To restore a local backup archive directly into a remote Atlas cluster:
```bash
mongorestore --uri="mongodb+srv://<username>:<password>@cluster0.abcde.mongodb.net/" --archive=IMS_backup.archive
```
**Command Breakdown:**
*   `mongorestore`: The restoration utility (runs locally).
*   `--uri="..."`: The MongoDB Atlas connection string. This tells the tool to connect to the remote cloud cluster using the provided credentials and routing information, rather than looking for a local database.
*   `--archive=...`: The path to the local archive file you want to upload and restore.

### Scenario B: Pull Backup from MongoDB Atlas to Local
To take a backup of a live Atlas database and download it to your local machine:
```bash
mongodump --uri="mongodb+srv://<username>:<password>@cluster0.abcde.mongodb.net/" --archive=atlas_backup.archive
```
**Command Breakdown:**
*   `mongodump`: The export utility (runs locally).
*   `--uri="..."`: The remote Atlas connection string pointing to the cloud cluster you want to back up.
*   `--archive=...`: Instructs the tool to stream the downloaded remote data directly into a single archive file on your local machine.

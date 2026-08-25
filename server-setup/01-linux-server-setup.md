# Phase 1 - Topic 1: Linux Server Setup, SSH Hardening & UFW

## 📌 Objective
Establish a secure baseline for a Linux production server (Ubuntu/Debian). This runbook covers user management, key-based authentication, SSH service hardening, bypassing cloud-provider default configurations, and network security using UFW.

---

## 1. User Management & Permissions
**Concept:** Operating as `root` is a severe security risk. Always create a standard user with `sudo` privileges to enforce the Principle of Least Privilege.

*   **Initial Login:** `ssh root@[server_ip]`
*   **Create Standard User:** `adduser [username]`
*   **Grant Sudo Access:** `usermod -aG sudo [username]`
*   **Manual Sudoers Edit (Advanced):**
    *   Command: `EDITOR=nano sudo visudo`
    *   Syntax: `[Username] ALL=(ALL:ALL) ALL`
*   **Switch User:** `su - [username]`
*   **File Permissions (Basics):** `chown` (change owner), `chmod` (change permissions).

---

## 2. Secure Shell (SSH) & Key-Based Authentication
**Concept:** Password logins are highly vulnerable to brute-force attacks. Key-pair authentication is the industry standard.

*   **Generate SSH Key Pair (On Local Machine):**
    `ssh-keygen -t ed25519` *(Creates a fast, highly secure key)*
*   **Deploy Public Key to Server:**
    Copy the public key content to `~/.ssh/authorized_keys` on the remote server.
*   **Strict Permission Requirements (CRITICAL):**
    If permissions are too broad, the SSH daemon will silently reject the key and fall back to a password prompt.
    *   `chmod 700 ~/.ssh` *(Directory: Owner read/write/execute only)*
    *   `chmod 600 ~/.ssh/authorized_keys` *(File: Owner read/write only)*
*   **Service Logging (Troubleshooting):** 
    `sudo journalctl -u ssh.service` *(Note: CentOS/Arch uses `sshd.service`)*

---

## 3. SSH Hardening & Cloud-Init Bypass (The "First Match Wins" Rule)
**Concept:** Prevent bot attacks by changing the default port, disabling root login, and strictly enforcing key-only access.
**The Problem:** Cloud providers (AWS, DigitalOcean, etc.) often use `cloud-init` scripts that forcefully inject `PasswordAuthentication yes` into `/etc/ssh/sshd_config.d/*.conf`, overriding your main configuration file.

*   **The Solution (Bypassing cloud-init):**
    SSH processes configuration files in alphabetical/numerical order, and the **First Match Wins**. We create a high-priority file to lock in our security settings before the cloud scripts are read.

    1.  **Create Priority Config:** `sudo nano /etc/ssh/sshd_config.d/00-security.conf`
    2.  **Add Hardening Rules:**
        ```text
        Port 5259
        PermitRootLogin no
        PasswordAuthentication no
        KbdInteractiveAuthentication no
        ```
        *(Note: `KbdInteractiveAuthentication no` prevents interactive password prompts even if standard password auth is disabled).*
    3.  **Verify Syntax:** `sudo sshd -t` *(No output = Syntax is perfect)*
    4.  **Restart SSH Service (Socket-aware):**
        ```bash
        sudo systemctl daemon-reload
        sudo systemctl restart ssh.socket
        sudo systemctl restart ssh
        ```
    5.  **Test Hardening:** `ssh -o PubKeyAuthentication=no -p 5259 [username]@[server_ip]` *(This MUST fail if hardening is successful)*

---

## 4. Uncomplicated Firewall (UFW)
**Concept:** Deny all incoming traffic by default. Only open ports strictly required for specific services.

*   **Installation & Status:**
    `sudo apt install ufw`
    `sudo ufw status verbose`
*   **Set Default Policies:**
    `sudo ufw default deny incoming`
    `sudo ufw default allow outgoing`
*   **Open Required Ports (CRITICAL SEQUENCE):**
    *⚠️ WARNING: Always allow your custom SSH port BEFORE enabling the firewall to prevent locking yourself out of the server.*
    *   `sudo ufw allow 5259/tcp` *(Custom SSH)*
    *   `sudo ufw allow 80/tcp` *(HTTP)*
    *   `sudo ufw allow 443/tcp` *(HTTPS)*
    *(Note: Appending `/tcp` prevents UFW from unnecessarily opening UDP ports).*
*   **Enable Firewall:** `sudo ufw enable`
*   **Deny vs. Reject:**
    *   `REJECT`: Blocks connection but informs the sender politely.
    *   `DENY` (`DROP`): Silently drops the packet. (Preferred in production to waste attackers' time).
*   **ICMP (Ping) Masking (Stealth Mode):**
    To prevent external scanners from verifying the server is online via ping:
    *   `sudo nano /etc/ufw/before.rules`
    *   Find the `ok icmp codes` section and change the `echo-request` rule from `ACCEPT` to `DROP`.

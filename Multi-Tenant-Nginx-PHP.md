# Production-Grade Multi-Tenant Web Server Architecture (Nginx & PHP-FPM)

## 📌 Overview
This runbook details the deployment of a secure, multi-tenant web hosting environment using Nginx and PHP-FPM 8.3 on an Ubuntu server. 

### ⚠️ Basic/Learning vs. Client-Ready (Production) Standard
*   **Basic Setup:** All websites on the server run under a single default user (`www-data`). If a single PHP site is compromised, the attacker gains read/write access to all other sites on the server.
*   **Production Setup (Implemented Here):** Each website (tenant) gets a strictly isolated Linux user and a dedicated PHP-FPM pool. Nginx (`www-data`) operates solely as a reverse proxy, dropping requests into the specific tenant's Unix socket. This ensures complete horizontal isolation between clients.

---

## 🚀 Step-by-Step Implementation

### Step 1: Create the Tenant User
Create an isolated user for the specific website to ensure strict privilege separation.
```bash
sudo useradd -m -s /bin/bash user1
```

### Step 2: Set Up Directory Structure & Permissions
Create the web root and assign ownership to the tenant user, NOT the web server. Nginx will still be able to read static files due to the `755` directory permissions.
```bash
sudo mkdir -p /var/www/first-site/public
# Create a test PHP file
echo "<?php echo 'Site is live!'; ?>" | sudo tee /var/www/first-site/public/index.php
# Grant ownership strictly to the tenant
sudo chown -R user1:user1 /var/www/first-site/
```

### Step 3: Configure Dedicated PHP-FPM Pool
Create a dedicated PHP-FPM pool configuration so the tenant's PHP code executes under their own user privileges.
```bash
sudo nano /etc/php/8.3/fpm/pool.d/first-site.conf
```
Note: We could have also use the original `www.conf` file and `sudo cp /etc/php/8.3/fpm/pool.d/www.conf /etc/php/8.3/fpm/pool.d/first-site.conf`

**Configuration (`first-site.conf`):**
```ini
[first-site]
; PHP code executes as the tenant user
user = user1
group = user1

; Listen on a unique socket for this tenant
listen = /run/php/first-site.sock

; Nginx (www-data) must have permission to knock on this door (socket)
listen.owner = www-data
listen.group = www-data

; Process Management
pm = dynamic
pm.max_children = 5
pm.start_servers = 2
pm.min_spare_servers = 1
pm.max_spare_servers = 3
```

### Step 4: Configure Nginx Virtual Host (Server Block)
Point Nginx to the specific tenant's document root and Unix socket.
```bash
sudo nano /etc/nginx/sites-available/first-site.com.conf
```
**Key Configuration Snippet:**
```nginx
server {
    listen 80;
    server_name first-site.com;
    root /var/www/first-site/public;
    index index.php index.html;

    location / {
        try_files $uri $uri/ /index.php?$query_string;
    }

    location ~ \.php$ {
        include snippets/fastcgi-php.conf;
        # Crucial: Point to the tenant's isolated socket
        fastcgi_pass unix:/run/php/first-site.sock;
    }
}
```
Enable the site and verify syntax:
```bash
sudo ln -sf /etc/nginx/sites-available/first-site.com.conf /etc/nginx/sites-enabled/
sudo nginx -t
```

### Step 5: Apply Changes
Restart the services to generate the new socket and reload the web server.
```bash
sudo systemctl restart php8.3-fpm
sudo systemctl reload nginx
```

---

## 🛠️ Common Errors & Troubleshooting Log

During deployment, several classic server errors might occur. Here is the diagnostic playbook:

### 1. `500 Internal Server Error`
*   **Log Output:** `rewrite or internal redirection cycle while internally redirecting to "/index.html"`
*   **Root Cause:** Nginx is instructed to serve `index.html` as a fallback, but the file or the configured `root` directory does not exist. Nginx falls into an infinite loop trying to find it.
*   **Fix:** Verify the `root` path in the Nginx config perfectly matches the actual directory path using `ls -la`. 

### 2. `403 Forbidden`
*   **Log Output:** `directory index of "/var/www/.../" is forbidden`
*   **Root Cause:** The `index` directive specifies a file (e.g., `index.php`), but that file is missing from the directory. Nginx attempts to list the directory contents (autoindex), which is blocked by default security policies.
*   **Fix:** Ensure `index.php` or `index.html` exists inside the `public` root directory.

### 3. `502 Bad Gateway`
*   **Log Output:** `connect() to unix:/var/run/php/...sock failed (2: No such file or directory)`
*   **Root Cause:** Nginx is attempting to forward the PHP request to a Unix socket that does not exist. This happens if the PHP-FPM pool is unconfigured, failed to start (e.g., due to a non-existent OS user), or wasn't restarted.
*   **Fix:** 
    1. Verify the tenant user exists (`cat /etc/passwd | grep user1`).
    2. Check if the socket was generated (`ls -la /run/php/`).
    3. Restart PHP-FPM (`sudo systemctl restart php8.3-fpm`).

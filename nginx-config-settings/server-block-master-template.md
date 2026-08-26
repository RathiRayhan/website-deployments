# Nginx Master Configuration Templates (Production Ready)

This document contains standard, industry-ready Nginx server block templates for various tech stacks. 

**Important:** When a client uses Cloudflare (Proxy turned ON / Orange Cloud), you MUST configure the global Cloudflare Real IP list first (Section 1). Only then use the site-specific templates.

---

## 1. Global Cloudflare Setup (Prerequisite for CF proxied sites)
**File Location:** `/etc/nginx/conf.d/cloudflare.conf`
*(Do not put this inside individual site configurations. Set it once globally.)*

```nginx
# IPv4 Ranges
set_real_ip_from 173.245.48.0/20;
set_real_ip_from 103.21.244.0/22;
set_real_ip_from 103.22.200.0/22;
set_real_ip_from 103.31.4.0/22;
set_real_ip_from 141.101.64.0/18;
set_real_ip_from 108.162.192.0/18;
set_real_ip_from 190.93.240.0/20;
set_real_ip_from 188.114.96.0/20;
set_real_ip_from 197.234.240.0/22;
set_real_ip_from 198.41.128.0/17;
set_real_ip_from 162.158.0.0/15;
set_real_ip_from 104.16.0.0/13;
set_real_ip_from 104.24.0.0/14;
set_real_ip_from 172.64.0.0/13;
set_real_ip_from 131.0.72.0/22;

# IPv6 Ranges
set_real_ip_from 2400:cb00::/32;
set_real_ip_from 2606:4700::/32;
set_real_ip_from 2803:f800::/32;
set_real_ip_from 2405:b500::/32;
set_real_ip_from 2405:8100::/32;
set_real_ip_from 2a06:98c0::/29;
set_real_ip_from 2c0f:f248::/32;

# Define the header used by Cloudflare
real_ip_header CF-Connecting-IP;
```

---

## 2. Modern App (Node.js backend / Next.js) — WITH Cloudflare
Use this when the application relies on WebSockets (Live Reload, Chat) and is proxied through Cloudflare.

**File Location:** `/etc/nginx/sites-available/yourdomain`

```nginx
server {
    listen 80;
    server_name yourdomain.com www.yourdomain.com;

    # Dedicated logs for Fail2ban targeting (warn level is standard for production)
    access_log /var/log/nginx/yourdomain_access.log;
    error_log /var/log/nginx/yourdomain_error.log warn;

    location / {
        proxy_pass http://127.0.0.1:3000; # Change backend port if needed

        # WebSockets Support
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "upgrade";

        # Identity & Cloudflare Headers
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header CF-Connecting-IP $http_cf_connecting_ip;
    }

    # Security: Deny access to all hidden files (.env, .git, etc.)
    # Logs are kept ON intentionally for Fail2ban to detect and block malicious bots
    location ~ /\. {
        deny all;
    }
}
```

---

## 3. Modern App (Node.js backend / Next.js) — NO Cloudflare (Direct)
Use this for standard VPS deployments without Cloudflare proxy.

```nginx
server {
    listen 80;
    server_name yourdomain.com www.yourdomain.com;

    access_log /var/log/nginx/yourdomain_access.log;
    error_log /var/log/nginx/yourdomain_error.log warn;

    location / {
        proxy_pass http://127.0.0.1:3000;

        # WebSockets Support
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "upgrade";

        # Identity Headers (No CF header needed here)
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    }

    # Security: Deny access to all hidden files (.env, .git, etc.)
    location ~ /\. {
        deny all;
    }
}
```

---

## 4. Static SPA (React / Vue / Angular)
Use this when hosting a compiled static build (e.g., `npm run build`). Single Page Applications (SPAs) require routing all requests back to `index.html`.

```nginx
server {
    listen 80;
    server_name yourdomain.com www.yourdomain.com;

    # Point to the build folder
    root /var/www/yourdomain/html;
    index index.html;

    access_log /var/log/nginx/yourdomain_access.log;
    error_log /var/log/nginx/yourdomain_error.log warn;

    location / {
        # Fallback routing for React/SPA router to handle client-side routing
        try_files $uri $uri/ /index.html;
    }

    # Optional: Cache static assets (images, css, js) for performance
    location ~* \.(js|css|png|jpg|jpeg|gif|ico|svg)$ {
        expires max;
        log_not_found off;
    }

    # Security: Deny access to all hidden files (.env, .git, etc.)
    location ~ /\. {
        deny all;
    }
}
```

---

## 5. PHP / WordPress Site
Use this for standard PHP applications. Requires `php-fpm` to be installed on the VPS.

```nginx
server {
    listen 80;
    server_name yourdomain.com www.yourdomain.com;

    root /var/www/yourdomain/public;
    index index.php index.html index.htm;

    access_log /var/log/nginx/yourdomain_access.log;
    error_log /var/log/nginx/yourdomain_error.log warn;

    location / {
        # Fallback routing for WordPress/PHP frameworks
        try_files $uri $uri/ /index.php?$query_string;
    }

    # Pass PHP scripts to FastCGI server
    location ~ \.php$ {
        include snippets/fastcgi-php.conf;
        # Warning: Make sure the PHP version matches the server's installed version
        fastcgi_pass unix:/var/run/php/php8.1-fpm.sock; 
    }

    # Security: Deny access to all hidden files (.env, .git, .ht, etc.)
    location ~ /\. {
        deny all;
    }
}
```

---

## 6. Full-Stack App (Frontend + Backend on Same Domain + CF Proxied)
Use this when hosting both a static frontend (React/Vue) and an API backend (Node.js) on the exact same domain to avoid CORS issues and save resources. 
- `/` serves the static React build.
- `/api/` proxies requests to the Node.js backend.

```nginx
server {
    listen 80;
    server_name yourdomain.com www.yourdomain.com;

    access_log /var/log/nginx/yourdomain_access.log;
    error_log /var/log/nginx/yourdomain_error.log warn;

    # 1. FRONTEND: Serve Static React/SPA Build
    location / {
        root /var/www/yourdomain/frontend/build;
        index index.html;
        # Fallback for client-side routing
        try_files $uri $uri/ /index.html;
    }

    # 2. BACKEND: Proxy all API requests to Node.js
    location /api/ {
        # Note: Depending on your Node app, you might need to strip the '/api' prefix.
        # For the trailing slash look at the routes folder of the project.
        # But generally, just passing it to the backend port works if the backend expects it.
        proxy_pass http://127.0.0.1:3000;

        # WebSockets Support (if the backend uses Socket.io etc.)
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "upgrade";

        # Identity & Cloudflare Headers
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header CF-Connecting-IP $http_cf_connecting_ip;
    }

    # Optional but good to have: Cache static assets for the frontend
    location ~* \.(js|css|png|jpg|jpeg|gif|ico|svg)$ {
        root /var/www/yourdomain/frontend/build;
        expires max;
        log_not_found off;
    }

    # Security: Deny access to all hidden files (.env, .git, etc.)
    location ~ /\. {
        deny all;
    }
}
```

---

## 7. [BONUS] Auto-Update Cloudflare IPs (Pro-Level Automation)
Cloudflare rarely changes their IP ranges, so a static list is usually fine. However, for premium clients, you can set up a bash script and a cron job to automatically fetch and update these IPs weekly. This guarantees the server will never block real users if Cloudflare updates their network.

### Step 1: Create the Bash Script
Create a script that fetches the official IPs, formats them for Nginx, tests the config, and safely reloads the server.

**Command:**
```bash
sudo nano /usr/local/bin/update-cloudflare-ips.sh
```

**Paste this script:**
```bash
#!/bin/bash
# Script to automatically update Cloudflare IPs for Nginx
# Save this in /usr/local/bin/update-cloudflare-ips.sh

CONF_FILE="/etc/nginx/conf.d/cloudflare.conf"
TEMP_FILE="/tmp/cloudflare_temp.conf"

# Start fresh
echo "# Auto-generated by cron - Cloudflare IPs" > $TEMP_FILE
echo "" >> $TEMP_FILE

# Fetch IPv4
echo "# IPv4 Ranges" >> $TEMP_FILE
curl -s [https://www.cloudflare.com/ips-v4](https://www.cloudflare.com/ips-v4) | sed -e 's/^/set_real_ip_from /' -e 's/$/;/' >> $TEMP_FILE
echo "" >> $TEMP_FILE

# Fetch IPv6
echo "# IPv6 Ranges" >> $TEMP_FILE
curl -s [https://www.cloudflare.com/ips-v6](https://www.cloudflare.com/ips-v6) | sed -e 's/^/set_real_ip_from /' -e 's/$/;/' >> $TEMP_FILE
echo "" >> $TEMP_FILE

# Add the essential header directive
echo "# Define the header used by Cloudflare" >> $TEMP_FILE
echo "real_ip_header CF-Connecting-IP;" >> $TEMP_FILE

# Replace old file with the new one
mv $TEMP_FILE $CONF_FILE

# Safely test and reload Nginx
nginx -t && systemctl reload nginx
```

### Step 2: Make the Script Executable
Give the script permission to run:
```bash
sudo chmod +x /usr/local/bin/update-cloudflare-ips.sh
```

### Step 3: Schedule the Cron Job
Set up a cron job to run this script automatically every Monday at 2:00 AM.

**Command:**
```bash
sudo crontab -e
```

**Add this line at the bottom:**
```bash
0 2 * * 1 /usr/local/bin/update-cloudflare-ips.sh > /dev/null 2>&1
```
*(The `> /dev/null 2>&1` part ensures it runs silently in the background without spamming system mail).*

**Test using:** 
```bash
sudo /usr/local/bin/update-cloudflare-ips.sh
```

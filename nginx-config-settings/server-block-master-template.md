# Nginx Master Configuration Templates (Production Ready)

This runbook contains standard, industry-ready Nginx server block templates for various tech stacks. 

**Important Security Rule:** The Default Catch-All block (Section 1) should be active on every server. For Cloudflare-proxied sites, you MUST configure the Global Cloudflare setup (Section 2) before adding site-specific templates.

---

## 1. Default Catch-All Block (The Security Bouncer)
**File Location:** `/etc/nginx/sites-available/default` (Symlinked to `sites-enabled`)

This is your server's primary defense against IP scanners and automated bots. If a request hits your server via direct IP or an unknown/malicious domain pointing to your IP, this block instantly drops the connection without returning any HTML or standard error page. 

**CRITICAL PRIVACY RULE:** Never use your real domain's SSL certificate in this default block, as it will leak your real domain name during direct IP scans. Always use a dummy certificate.

### Step A: Generate a Dummy Certificate (15-Year Validity)
Run this command once on the server to create a fake 15-year certificate to match Cloudflare's maximum origin certificate lifespan. (Hit Enter if it asks for any details).
```bash
sudo openssl req -x509 -nodes -days 5475 -newkey rsa:2048 -keyout /etc/ssl/private/dummy.key -out /etc/ssl/certs/dummy.crt -subj "/CN=fake-domain.com"
```

### Step B: The Bouncer Configuration
Replace the contents of the default file with this configuration:

```nginx
server {
    # Listen on both IPv4 and IPv6 as the default server
    listen 80 default_server;
    listen [::]:80 default_server;
    
    # Listen on port 443 (HTTPS) to catch encrypted direct IP scanners
    listen 443 ssl default_server;
    listen [::]:443 ssl default_server;
    
    # Dummy SSL Paths (Prevents domain name leakage during SSL Handshake)
    ssl_certificate /etc/ssl/certs/dummy.crt;
    ssl_certificate_key /etc/ssl/private/dummy.key;

    # The underscore (_) catches any unmapped domain or direct IP
    server_name _; 

    # 444 is a special Nginx code that immediately drops the connection (No Response)
    return 444; 
}
```

---

## 2. Global Cloudflare Setup (The Real IP Module)
**File Location:** `/etc/nginx/conf.d/cloudflare.conf`
*(Set globally, do NOT put this inside individual site blocks).*

To make Nginx natively replace Cloudflare's proxy IPs with the actual visitor's IP in your access logs and security tools (like Fail2ban), you must use these two essential directives:

*   `set_real_ip_from <IP_Range>;`
    *   **What it does:** Tells Nginx to *trust* these specific IP ranges (Cloudflare's network). 
    *   **Why you need it:** Nginx shouldn't trust just anyone who sends a fake IP header. By defining Cloudflare's IPs here, Nginx knows: *"If a request comes from these specific servers, I am allowed to trust the hidden header they send."*

*   `real_ip_header CF-Connecting-IP;`
    *   **What it does:** Instructs Nginx to look inside this exact header (`CF-Connecting-IP`) to extract the true visitor's IP address.
    *   **Why you need it:** Once Nginx verifies the request came from a trusted Cloudflare IP, it permanently overwrites Nginx's internal `$remote_addr` variable. Because of this single line, your logs and Fail2ban will automatically see the real user.

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

## 3. Modern App (Node.js backend / Next.js) — WITH Cloudflare
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
        proxy_pass http://127.0.0.1:3000; 

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

> **📝 PRO-TIP: SSL & IPv6 Configuration (Cloudflare Origin vs. Let's Encrypt)**
> 
> **⚠️ CRITICAL IPv6 CHECK:** Before applying ANY IPv6 configurations (for port 80 or 443), you must verify if the server has a **Public IPv6 Address**. 
> Run `ip -6 addr` and specifically look for the keyword **`scope global`**. 
> 
> - **✅ If you see `scope global`** (e.g., `inet6 2606:... scope global`), your server can receive IPv6 traffic from the internet. You MUST use the `listen [::]` lines.
> - **❌ If the output is completely empty:** IPv6 is completely disabled at the kernel level. Using `listen [::]` **WILL CRASH** Nginx on reload.
> - **⚠️ If you only see `scope host` (`::1`) or `scope link` (`fe80:...`):** Nginx won't crash, but the server has no public IPv6 routing. Do not use `listen [::]` as it serves no purpose and just clutters the config.
>
> **Scenario A: Using Cloudflare Origin Certificates (Full Strict Mode)**
> If your domain is proxied through Cloudflare and you are using their Origin CA, you must manually update the server block to support HTTPS:
> 1. Replace `listen 80;` with:
>    `listen 443 ssl;`
>    `listen [::]:443 ssl;` *(Only add this second line if `scope global` was found!)*
> 2. Inject the certificate paths inside the server block:
>    `ssl_certificate /etc/ssl/cloudflare/cert.pem;`
>    `ssl_certificate_key /etc/ssl/cloudflare/key.pem;`
> 
> **Scenario B: Using Let's Encrypt / Certbot (Direct VPS / No Proxy)**
> If you are NOT using Cloudflare, configure your base server block for port 80 first:
> 1. Make sure your base block has:
>    `listen 80;`
>    `listen [::]:80;` *(Only add this second line if `scope global` was found!)*
> 2. Then simply run:
>    `sudo certbot --nginx -d yourdomain.com`
> Certbot will automatically duplicate the block, configure the 443 SSL paths (including IPv6 if it was enabled in step 1), and set up the HTTP to HTTPS redirect for you. Do not manually type the SSL paths.

---

## 4. Modern App (Node.js backend / Next.js) — NO Cloudflare (Direct)
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

## 5. Static SPA (React / Vue / Angular)
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

## 6. PHP / WordPress Site (Production Hardened)
Use this for standard PHP applications and WordPress. Requires `php-fpm` to be installed on the VPS. 
This block includes advanced security and caching rules specifically designed for WordPress.

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
    
    # Security: Prevent PHP execution in uploads directory (Stops malware/shells)
    location ~* /wp-content/uploads/.*\.php$ {
        deny all;
    }

    # Security: Block xmlrpc.php (Prevents DDoS and brute force attacks)
    location = /xmlrpc.php {
        deny all;
        access_log off;
        log_not_found off;
    }
    
    # Performance: Cache static assets aggressively
    location ~* \.(js|css|png|jpg|jpeg|gif|ico|svg|woff|woff2|ttf|eot)$ {
        expires max;
        log_not_found off;
    }
}

> Note: Delete the wp-content and xmlrpc location blocks if deploying a custom PHP/Laravel app that is NOT WordPress.

---

## 7. Full-Stack App (Frontend + Backend on Same Domain + CF Proxied)
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
        # But generally, just passing it to the backend port works if the backend expects it.
        proxy_pass http://127.0.0.1:3000;

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

## 8. [BONUS] Auto-Update Cloudflare IPs (Pro-Level Automation)
Cloudflare rarely changes their IP ranges, so a static list is usually fine. However, for premium clients, you can set up a bash script and a cron job to automatically fetch and update these IPs weekly. This guarantees the server will never block real users if Cloudflare updates their network.

### Step 1: Create the Bash Script
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
DATE=$(date +%Y-%m-%d_%H-%M-%S)

# Start fresh
echo "# Auto-generated by cron - Cloudflare IPs" > $TEMP_FILE

# Fetch IPv4
echo "" >> $TEMP_FILE
echo "# IPv4 Ranges" >> $TEMP_FILE
curl -s [https://www.cloudflare.com/ips-v4](https://www.cloudflare.com/ips-v4) | sed -e 's/^/set_real_ip_from /' -e 's/$/;/' >> $TEMP_FILE

# Fetch IPv6
echo "" >> $TEMP_FILE
echo "# IPv6 Ranges" >> $TEMP_FILE
curl -s [https://www.cloudflare.com/ips-v6](https://www.cloudflare.com/ips-v6) | sed -e 's/^/set_real_ip_from /' -e 's/$/;/' >> $TEMP_FILE

# Add the essential header directive
echo "" >> $TEMP_FILE
echo "# Define the header used by Cloudflare" >> $TEMP_FILE
echo "real_ip_header CF-Connecting-IP;" >> $TEMP_FILE
echo "" >> $TEMP_FILE
echo "# Updated on $DATE" >> $TEMP_FILE

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

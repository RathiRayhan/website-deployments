# Topic 2: Nginx Web Server, Reverse Proxy & SSL Configuration

## Overview
This runbook outlines the industry-standard procedures for configuring Nginx as a highly efficient web server and reverse proxy. It covers deploying static frontends, routing traffic to backend applications (Node.js, Python, etc.) with proper header preservation, securing domains with Let's Encrypt SSL (HTTPS), and automating certificate renewals.

---

## 1. Hosting a Static Website (Frontend)
**Use Case:** Deploying static sites (HTML/CSS/JS) or compiled frontend frameworks (React, Vue, Angular).

### Step 1: Transfer & Setup Web Root
Transfer build files to the server and set up the web root directory.
```bash
sudo mv /path/to/transferred/files /var/www/my-portfolio
sudo chown -R www-data:www-data /var/www/my-portfolio
sudo chmod -R 755 /var/www/my-portfolio
```
*Security Note:* Ensure no `.env` files containing sensitive credentials are placed inside `/var/www/`.

### Step 2: Nginx Static Site Configuration
Create the configuration file: `sudo nano /etc/nginx/sites-available/my-portfolio`
```nginx
server {
    listen 80;
    server_name <your_domain.com> www.<your_domain.com>;

    location / {
        root /var/www/my-portfolio;
        index index.html index.htm;
        try_files $uri $uri/ /index.html; # Crucial for React/Vue Router
    }
}
```
Enable and reload:
```bash
sudo ln -s /etc/nginx/sites-available/my-portfolio /etc/nginx/sites-enabled/
sudo nginx -t && sudo systemctl reload nginx
```

---

## 2. Nginx Reverse Proxy (Production Standard)
**Use Case:** Routing web traffic to backend applications (e.g., running on port 8080) while preserving the client's original IP address and supporting modern real-time features like WebSockets.

### Step 1: Production Reverse Proxy Configuration
Create the configuration file: `sudo nano /etc/nginx/sites-available/backend-app`
```nginx
server {
    listen 80;
    server_name <your_domain.com> www.<your_domain.com>;

    location / {
        proxy_pass [http://127.0.0.1:8080](http://127.0.0.1:8080);
        
        # Production Standard Proxy Headers
        # Ensures the backend application receives the client's actual IP, not the Nginx server's IP.
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;

        # WebSocket Support
        # Prevents 502 Bad Gateway errors for real-time apps (e.g., Socket.io, Chat apps).
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "upgrade";
    }
}
```
Enable and test the configuration:
```bash
sudo ln -s /etc/nginx/sites-available/backend-app /etc/nginx/sites-enabled/
sudo nginx -t && sudo systemctl reload nginx
```

---

## 3. SSL/TLS Implementation & Automation (Let's Encrypt)
**Use Case:** Securing the domain with industry-standard HTTPS encryption and forcing HTTP-to-HTTPS redirection. 

### Step 1: Pre-flight Checks
* Ensure the domain's **A Record** points to the VPS IP in the DNS manager.
* Open necessary ports via UFW Firewall:
```bash
sudo ufw allow 'Nginx Full' # Opens both 80 (HTTP) and 443 (HTTPS)
```

### Step 2: Install Certbot (Official Venv Method)
```bash
sudo python3 -m venv /opt/certbot/
sudo /opt/certbot/bin/pip install --upgrade pip
sudo /opt/certbot/bin/pip install certbot certbot-nginx
sudo ln -s /opt/certbot/bin/certbot /usr/local/bin/certbot
```

### Step 3: Provision SSL Certificates
Execute the automated Nginx plugin. This will solve the ACME challenge, generate certificates, and automatically inject `listen 443 ssl` and `return 301` (HTTPS redirect) into the Nginx block.
```bash
sudo certbot --nginx -d <your_domain.com> -d www.<your_domain.com>
```

### Step 4: Zero-Touch Auto-Renewal
Certificates expire every 90 days. Implement a cronjob to fully automate the renewal process.
```bash
echo "0 0,12 * * * root /opt/certbot/bin/python -c 'import random; import time; time.sleep(random.random() * 3600)' && sudo certbot renew -q" | sudo tee -a /etc/crontab > /dev/null
```
Verify the automation dry-run:
```bash
sudo certbot renew --dry-run
```

---

## 4. Troubleshooting Reference
* **Port 80 Conflict (`Address already in use`):** Remove conflicting symlinks from `/etc/nginx/sites-enabled/` and reload. Never delete the original files from `sites-available`.
* **403 Forbidden Error:** Usually a permissions issue. Verify that the Nginx worker (`www-data`) has read access to the web root (`sudo chown -R www-data:www-data /var/www/path`).
* **502 Bad Gateway:** The backend application (e.g., Node.js process) is either down or listening on the wrong port. Verify backend status before debugging Nginx.
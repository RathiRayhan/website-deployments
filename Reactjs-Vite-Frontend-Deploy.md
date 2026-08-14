# React/Vite SPA Deployment Runbook (Production Standard)

**Objective:** Deploy a React 18 Single Page Application (built with Vite) on an Ubuntu production server using Nginx, secured with Let's Encrypt SSL.

## Prerequisites
Before starting, ensure you have:
* An Ubuntu VM/VPS with root or `sudo` privileges.
* A registered domain name (e.g., `reactjs-vite.rathirayhan.dpdns.org`) pointing to the server's IP address.

---

## Step 1: Environment Setup (Node.js LTS)
For a production-grade environment, always use the **Long Term Support (LTS)** version of Node.js (even-numbered releases like 20, 22, etc.).

1. Install Node.js via NodeSource (Example for Node.js 20 LTS):
```bash
sudo apt-get install -y curl
curl -fsSL https://deb.nodesource.com/setup_20.x | sudo -E bash -
sudo apt-get install -y nodejs
node -v
```
2. Verify installation:
```bash
node -v  # Should output v20.x.x
npm -v
```

---

## Step 2: Application Build Process
1. Clone the repository and navigate into it:
```bash
git clone https://github.com/joaopaulomoraes/reactjs-vite-tailwindcss-boilerplate react-app
cd react-app
```
2. Install dependencies and generate the production build:
```bash
npm install
npm run build
```
*Note: Vite outputs the compiled static assets (HTML, JS, CSS) into a directory named `dist`.*

---

## Step 3: Web Server Setup & Permissions
1. Create the web root directory and copy the build files:
```bash
sudo mkdir -p /var/www/reactjs-vite
sudo cp -r dist/* /var/www/reactjs-vite/
```
2. Set secure ownership and permissions for the Nginx user (`www-data`):
```bash
sudo chown -R www-data:www-data /var/www/reactjs-vite/
sudo chmod -R 755 /var/www/reactjs-vite/
```

---

## Step 4: Nginx Reverse Proxy Configuration
1. Create a new server block configuration file:
```bash
sudo nano /etc/nginx/sites-available/reactjs-vite
```
2. Paste the following production-ready configuration:
```nginx
server {
    listen 80;
    listen [::]:80;
    server_name reactjs-vite.rathirayhan.dpdns.org;

    # Production log file paths for quick troubleshooting
    access_log /var/log/nginx/reactjs-vite_access.log;
    error_log /var/log/nginx/reactjs-vite_error.log;

    # Absolute path to the compiled static assets
    root /var/www/reactjs-vite;
    index index.html;

    # Security header to prevent attackers from framing the website
    add_header X-Frame-Options "SAMEORIGIN";

    location / {
        # Crucial: Forces Nginx to fall back to index.html if the user 
        # refreshes a client-side route (prevents the dreaded 404 error)
        try_files $uri $uri/ /index.html;
    }

    # Custom 404 landing block
    error_page 404 /index.html;
}
```
3. Enable the site, verify syntax, and reload Nginx:
```bash
sudo ln -s /etc/nginx/sites-available/reactjs-vite /etc/nginx/sites-enabled/
sudo nginx -t
sudo systemctl reload nginx
```
*Note: Use `reload` instead of `restart` to avoid dropping active connections.*

---

## Step 5: SSL & Auto-Renewal (Certbot)
1. Issue the SSL certificate:
```bash
sudo certbot --nginx -d reactjs-vite.rathirayhan.dpdns.org
```
2. Verify the systemd timer for automatic renewals (Dry Run):
```bash
sudo certbot renew --dry-run
```
*If the dry run is successful, your certificates will automatically renew before expiration.*

---

## 💡 Quality Assurance: Basic vs. Production Standard
| Metric | Basic Setup | Production Standard (This Runbook) |
| :--- | :--- | :--- |
| **Node.js Version** | Random/Latest version | Strict enforcement of LTS for stability. |
| **SPA Routing** | Returns 404 on page refresh | Uses `try_files` fallback to seamlessly handle deep links. |
| **Logging** | Pollutes default Nginx logs | Dedicated `access` and `error` logs for faster isolation. |
| **Uptime Management** | Uses `systemctl restart nginx` | Uses `systemctl reload nginx` for zero-downtime updates. |

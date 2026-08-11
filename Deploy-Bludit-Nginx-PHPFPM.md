# Production Deployment Runbook: Flat-File CMS (Bludit) with Nginx & PHP-FPM

## Objective
This runbook details the end-to-end deployment of Bludit (a flat-file CMS) on an Ubuntu-based Linux server. It focuses on setting up a highly performant Nginx reverse proxy, correctly routing FastCGI processes to PHP-FPM, and enforcing production-grade security and permissions.

---

## Architecture & Production Standard Guarantee
* **Basic/Amateur Setup:** Using default `php-fpm.sock` symlinks (which break during PHP upgrades), keeping default `777` permissions, and ignoring HTTPS.
* **This Production Setup:** 
  * Explicitly uses the absolute PHP version socket (e.g., `php8.3-fpm.sock`) to prevent upgrade conflicts and ensure zero downtime.
  * Enforces strict `www-data` ownership and `755/644` directory permissions to prevent unauthorized execution and source code leaks.
  * Implements Let's Encrypt SSL/TLS with automatic HTTP-to-HTTPS redirection (301 Permanent Redirect).

---

## Prerequisites
* An Ubuntu Linux environment (VM or VPS).
* Root or `sudo` privileges.
* A valid domain or subdomain pointed to the server's public IP address via DNS A-records.

## Step 1: Install Required Dependencies
Update the system package index and install Nginx, PHP-FPM, and the specific PHP extensions required by the application.

```bash
sudo apt update
sudo apt install nginx php-fpm php-mbstring php-gd php-xml php-json -y
```

## Step 2: Determine the Exact FastCGI Socket
Identify the active PHP version and locate the absolute socket file to avoid symlink-related vulnerabilities.

```bash
php -v
ls -l /var/run/php/
```
> **Critical Note:** Avoid using the `php-fpm.sock` symlink. Note down the absolute file name (e.g., `php8.3-fpm.sock`). This ensures the site remains stable even if the system's default PHP version is upgraded globally.

## Step 3: Setup Web Directory & Application Source
Create the project directory, clone the repository, and apply strict production permissions.

```bash
# Create directory and navigate into it
sudo mkdir -p /var/www/myphpapp
cd /var/www/myphpapp

# Clone the official Bludit repository directly into the current directory
sudo git clone https://github.com/bludit/bludit.git .

# Enforce production security permissions for Nginx user
sudo chown -R www-data:www-data /var/www/myphpapp
sudo chmod -R 755 /var/www/myphpapp
```

## Step 4: Configure Nginx Server Block
Create a new Nginx configuration file for the application to isolate it from the default server blocks.

```bash
sudo nano /etc/nginx/sites-available/myapp
```

Insert the following configuration. Replace `YOUR_DOMAIN_OR_IP` with your actual domain. Ensure the `fastcgi_pass` path matches the socket found in Step 2.

```nginx
server {
    listen 80;
    server_name YOUR_DOMAIN_OR_IP;
    root /var/www/bludit;
    
    index index.php index.html;

    # Isolated Application Logs
    access_log /var/log/nginx/bludit-access.log;
    error_log  /var/log/nginx/bludit-error.log;

    # Clean URL Routing for WordPress
    location / {
        try_files $uri $uri/ /index.php?$query_string;
    }

    # PHP-FPM Processing
    location ~ \.php$ {
        include snippets/fastcgi-php.conf;
        # Ensure the PHP version matches your installed version (e.g., 8.1, 8.3)
        fastcgi_pass unix:/var/run/php/php8.x-fpm.sock;
    }

    # Security: Block access to hidden configuration files
    location ~ /\.ht {
        deny all;
    }
}
```

## Step 5: Enable Site and Test Configuration
Link the configuration file to the enabled sites directory, validate syntax, and apply changes.

```bash
# Remove the default Nginx configuration to prevent routing conflicts
sudo rm /etc/nginx/sites-enabled/default

# Create symlink for the new site
sudo ln -s /etc/nginx/sites-available/myapp /etc/nginx/sites-enabled/

# Test Nginx configuration for syntax errors
sudo nginx -t

# Restart Nginx to apply changes
sudo systemctl restart nginx
```

## Step 6: Secure with SSL/TLS (Let's Encrypt)
Secure the site using Certbot. This will automatically fetch the certificate, modify the Nginx configuration to listen on port 443, and enforce HTTP to HTTPS redirection.

```bash
sudo certbot --nginx
```
*Follow the on-screen prompts to select your domain and agree to the terms.*

---

## Step 7: Disaster Recovery & Troubleshooting Simulation
As a DevOps best practice, verify the setup by simulating a backend failure.

**Simulation: 502 Bad Gateway**
1. Deliberately stop the PHP-FPM service:
   ```bash
   sudo systemctl stop php8.3-fpm
   ```
2. Visit the website in your browser. You will see a `502 Bad Gateway` error.
3. Diagnose the issue by checking the Nginx error logs:
   ```bash
   sudo tail -n 20 /var/log/nginx/error.log
   ```
   *(Expected output: "connect() to unix:/var/run/php/php8.3-fpm.sock failed").*
4. Resolve the issue and restore the site by restarting the service:
   ```bash
   sudo systemctl start php8.3-fpm
   ```

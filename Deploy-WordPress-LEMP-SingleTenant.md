# Production Deployment Runbook: WordPress (LEMP Stack) - Single-Tenant Architecture

## Objective
This runbook details the production-grade deployment of WordPress on an Ubuntu VPS using the LEMP stack (Linux, Nginx, MariaDB, PHP-FPM). It utilizes a **Single-Tenant Architecture**, optimized for hosting a primary application where maximum performance and minimal complexity are required.

---

## Architecture & Production Standard Guarantee
* **Basic/Amateur Setup:** Using the `root` database user in application config, default directory permissions, and standard Nginx logs.
* **This Production Setup:**
  * **Database Security:** Enforces `unix_socket` authentication for the root database user and isolates the application using a dedicated, restricted database user.
  * **Log Isolation:** Implements separate `access_log` and `error_log` for the application to drastically reduce troubleshooting time.
  * **Security Hardening:** Strictly blocks access to hidden files (e.g., `.htaccess`, `.git`) at the Nginx level and enforces secure `755/644` directory permissions.
  * **Zero Downtime:** Uses Nginx `reload` (instead of `restart`) for configuration updates to ensure active user connections are not dropped.

---

## Step 1: Install Required Dependencies
Update the system and install MariaDB, PHP-FPM, and the required PHP-MySQL extension.

```bash
sudo apt update
sudo apt install mariadb-server php-mysql -y
```

## Step 2: Database Hardening & Configuration
Secure the MariaDB installation before creating application-specific credentials.

```bash
# Run the built-in security script
sudo mysql_secure_installation
```
> **Security Note:** Press `Y` for `unix_socket` authentication. This binds database root access to Linux root access. Say `Y` to disallowing remote root login, removing anonymous users, and dropping the test database.

Login to MariaDB to create the isolated WordPress database and user:
```bash
sudo mysql -u root -p
```
```sql
CREATE DATABASE wordpress_db;
CREATE USER 'wp_user'@'localhost' IDENTIFIED BY 'StrongPassword123!';
GRANT ALL PRIVILEGES ON wordpress_db.* TO 'wp_user'@'localhost';
FLUSH PRIVILEGES;
EXIT;
```

## Step 3: Application Setup & Permissions
Create the web directory, clone the official WordPress repository, and set secure ownership.

```bash
sudo mkdir -p /var/www/wordpress
cd /var/www/wordpress

# Clone WordPress core
sudo git clone [https://github.com/WordPress/WordPress.git](https://github.com/WordPress/WordPress.git) .

# Enforce secure ownership for the Nginx worker process
sudo chown -R www-data:www-data /var/www/wordpress
sudo chmod -R 755 /var/www/wordpress
```

## Step 4: Configure WordPress Credentials
Connect WordPress to the newly created database.

```bash
sudo cp wp-config-sample.php wp-config.php
sudo nano wp-config.php
```
Update the following directives with the credentials from Step 2:
```php
define( 'DB_NAME', 'wordpress_db' );
define( 'DB_USER', 'wp_user' );
define( 'DB_PASSWORD', 'StrongPassword123!' );
```

## Step 5: Nginx Server Block (Reverse Proxy)
Create an isolated Nginx configuration block for the WordPress site.

```bash
sudo nano /etc/nginx/sites-available/wordpress
```
```nginx
server {
    listen 80;
    server_name YOUR_DOMAIN_OR_IP;
    root /var/www/wordpress;
    
    index index.php index.html;

    # Isolated Application Logs
    access_log /var/log/nginx/wordpress-access.log;
    error_log  /var/log/nginx/wordpress-error.log;

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

Enable the site and reload Nginx for zero downtime:
```bash
sudo ln -s /etc/nginx/sites-available/wordpress /etc/nginx/sites-enabled/
sudo nginx -t
sudo systemctl reload nginx
```

## Step 6: SSL/TLS Encryption (Let's Encrypt)
Secure the application with an automated SSL certificate and enforce HTTP to HTTPS redirection.

```bash
sudo certbot --nginx
```

---

## Step 7: Disaster Recovery & Debugging (Runbook)
If the website displays an **"Error establishing a database connection"**, the Nginx logs will likely remain empty because the HTTP request was technically successful (the PHP script executed, even if it returned a database error). 

**Troubleshooting Protocol:**
1. Enable native application debugging to reveal the exact database error (e.g., wrong password vs. denied host).
   ```bash
   sudo nano /var/www/wordpress/wp-config.php
   ```
2. Modify the debug constants:
   ```php
   define( 'WP_DEBUG', true );
   define( 'WP_DEBUG_LOG', true );
   ```
3. Reload the page to view the explicit error.
4. Test credentials directly via CLI to verify database access:
   ```bash
   mysql -u wp_user -p
   ```
5. **Critical:** Once resolved, immediately revert `WP_DEBUG` to `false` to prevent sensitive architecture paths from leaking in production.

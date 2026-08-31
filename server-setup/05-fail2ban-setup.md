# Production Runbook: fail2ban Security & Brute-Force Protection

## 📌 Overview & Architecture
`fail2ban` is a robust intrusion prevention software framework that protects computer servers from brute-force attacks. 
The architecture consists of two primary components:
*   **fail2ban-server:** Runs as a background daemon, monitors specified log files for malicious patterns (regex), and dynamically updates firewall rules (e.g., UFW/iptables) to ban offending IPs.
*   **fail2ban-client:** The command-line interface used to communicate with the server, check jail statuses, and manually manage IP bans.

---

## 1. Installation & Service Management

Install the necessary packages:
```bash
sudo apt update
sudo apt install -y fail2ban
```

By default, the `fail2ban` background service is disabled upon installation to prevent accidental lockouts or conflicts on unconfigured servers. Enable and start it:
```bash
sudo systemctl enable --now fail2ban
```

---

## 2. Configuration Principles

**The Golden Rule:** Never modify the default `/etc/fail2ban/jail.conf` file. It will be overwritten during system or package updates. 

All customizations must reside in a `.local` override file. While it is possible to copy the entire file (`sudo cp jail.conf jail.local`), the best practice for a clean and maintainable configuration is to create a new `jail.local` file containing only the `[DEFAULT]` section and the specific jails you intend to enable.

### Baseline Configuration (`/etc/fail2ban/jail.local`)

Create the configuration file:
```bash
sudo nano /etc/fail2ban/jail.local
```

Add the following production-safe baseline:

```ini
[DEFAULT]
bantime  = 1h
findtime = 10m
maxretry = 5
ignoreip = 127.0.0.1/8 ::1 YOUR_PUBLIC_IP/32

[sshd]
enabled = true
port    = 22

[nginx-botsearch]
enabled  = true
port     = http,https
logpath  = /var/log/nginx/*access.log
maxretry = 2

[recidive]
enabled  = true
bantime  = 1w
findtime = 1d
maxretry = 5
```

### 📝 Architectural Notes on Jails:
*   **`ignoreip`:** Whitelists specific IPs from being banned. `/32` denotes a single, exact IPv4 address. *Note: During setup, it is recommended to add your static public IP to prevent accidental lockouts. For a strict production environment, remove external IPs for a cleaner configuration.*
*   **`[sshd]`:** Always enabled by default to prevent brute-force password attacks on the server, regardless of whether password authentication is disabled. Ensure the `port` matches your custom SSH port if modified.
*   **`[nginx-botsearch]`:** Blocks automated bot requests scanning for sensitive credentials (e.g., `/.env`, `/.git`). Requires an active Nginx server.
*   **`[recidive]`:** A cross-jail monitor that identifies repeat offenders and issues a long-term ban (1 week).

---

## 3. Custom WordPress Jail Implementation

Standard SSH hardening does not protect application-layer login surfaces like `/wp-login.php`. A dedicated custom jail is required for WordPress sites.

**Step 1: Define the Filter**
Create a regex filter to identify failed login attempts:
```bash
sudo nano /etc/fail2ban/filter.d/wordpress.conf
```
```ini
[Definition]
failregex = ^<HOST> .* "POST .*wp-login\.php.* HTTP/[0-9.]+" (200|302)
            ^<HOST> .* "POST .*xmlrpc\.php.* HTTP/[0-9.]+" (200|302|403)
```

**Step 2: Validate the Regex**
Always validate the filter against live logs before enabling the jail:
```bash
sudo fail2ban-regex /var/log/nginx/access.log /etc/fail2ban/filter.d/wordpress.conf
```

**Step 3: Enable the Jail**
Append the WordPress configuration to `/etc/fail2ban/jail.local`:
```ini
[wordpress]
enabled  = true
port     = http,https
filter   = wordpress
logpath  = /var/log/nginx/*access.log
maxretry = 5
```

Apply the changes:
```bash
sudo systemctl restart fail2ban
```

---

## 4. Optional / Advanced Jails

Use these with caution depending on specific application requirements.

### A. Nginx Bad Request
Bans IPs generating excessive malformed requests (400 errors). It is pattern-based (not 404-based, avoiding bans on innocent users encountering broken links).
```ini
[nginx-bad-request]
enabled = true
logpath = /var/log/nginx/*error.log
```

### B. Nginx Limit Request (Flood Control)
*Warning: This is not universal DDoS protection. A mis-tuned limit will result in false positives, banning real users and search engine crawlers (e.g., Googlebot).*

**Prerequisite (Nginx Configuration):**
The jail requires Nginx rate limiting to be active.
```nginx
# In /etc/nginx/nginx.conf (http block):
limit_req_zone $binary_remote_addr zone=api_limit:10m rate=5r/s;
limit_req_status 429;

# In the relevant server block (API endpoints):
location /api/ {
    limit_req zone=api_limit burst=10 nodelay;
    proxy_pass [http://127.0.0.1:4000/](http://127.0.0.1:4000/);
}
```

**Jail Configuration:**
```ini
[nginx-limit-req]
enabled  = true
logpath  = /var/log/nginx/*error.log
maxretry = 10
```

---

## 5. Operations & CLI Commands

Use the `fail2ban-client` to monitor and manage the service:

```bash
# View all active jails and ban counts
sudo fail2ban-client status                 

# View detailed status and currently banned IPs for a specific jail
sudo fail2ban-client status wordpress       

# Unban a specific IP
sudo fail2ban-client set wordpress unbanip 1.2.3.4   

# Monitor live ban/unban activity
sudo tail -f /var/log/fail2ban.log          
```
*Note on unbanning:* The unban command returns a boolean integer: `1` (True) if the IP was successfully found and removed from the ban list, and `0` (False) if the IP was not in the ban list.

---

## 6. Edge Cases: Reverse Proxies (Cloudflare)

If the server sits behind a DNS proxy like Cloudflare, specific architectural limitations apply:

1.  **Real IP Requirement:** You MUST configure Nginx to catch the client's real IP (using `set_real_ip_from`). Failing to do so will result in fail2ban reading Cloudflare edge IPs from the logs and banning the proxy itself.
2.  **Firewall Bypass Reality:** The `[wordpress]` jail bans access to the `wp-login.php` path at the local firewall level. However, a local firewall (UFW/iptables) only blocks *direct* IP connections. If Cloudflare proxy is enabled, the attacker's traffic arrives via Cloudflare's trusted IPs. The firewall cannot drop this traffic natively. 
3.  **Advanced Mitigation:** While the current setup effectively logs the true attacker and prevents direct origin bypasses, fully restricting a proxied attacker requires advanced setups, such as fail2ban Cloudflare API integration (banning at the edge) or maintaining a dynamic Nginx `deny` list.

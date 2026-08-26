# Nginx Reverse Proxy Headers: The Ultimate Cheat Sheet

When setting up a reverse proxy (like connecting Nginx to a Node.js or Next.js app), we use several specific configurations and headers. Here is the breakdown of what each "ingredient" actually does in simple English.

## 1. Global Cloudflare Setup (The Prerequisites)
**Location:** `/etc/nginx/conf.d/cloudflare.conf` (This is a global config, NOT inside your site's server block).
Before passing the real IP to your backend, Nginx itself must know how to extract it.

*   `set_real_ip_from <Cloudflare_IP_Range>;`
    *   **What it does:** Tells Nginx, "Only trust IP replacement instructions if the traffic is physically coming from these official Cloudflare IP addresses."
*   `real_ip_header CF-Connecting-IP;`
    *   **What it does:** Instructs Nginx to look inside the `CF-Connecting-IP` header (which Cloudflare attached) to find the real visitor's IP, and automatically replace the Nginx `$remote_addr` variable with it.

---

## 2. The Connection Upgraders (For WebSockets)
**Location:** Inside your site's `server` block (`/etc/nginx/sites-available/your-site`).
Modern apps (Next.js, React, Node.js) often use WebSockets for real-time features (like live chat or hot-reloading). WebSockets need a continuous open connection, unlike standard HTTP which closes the connection after every single request.

*   `proxy_http_version 1.1;`
    *   **What it does:** Forces Nginx to use HTTP/1.1 instead of the default HTTP/1.0.
    *   **Why you need it:** HTTP/1.1 supports "keep-alive" connections (keeping the pipe open). WebSockets cannot work without this.
*   `proxy_set_header Upgrade $http_upgrade;`
    *   **What it does:** Passes the browser's request to "Upgrade" from normal HTTP to WebSockets straight to your backend.
*   `proxy_set_header Connection "upgrade";`
    *   **What it does:** Confirms the switch. It tells the backend, "Yes, we are officially keeping this connection open as a WebSocket."

---

## 3. The Identity Headers (Who is visiting?)
**Location:** Inside your site's `server` block.
When traffic goes through Cloudflare and then Nginx, the backend (Node.js) loses the visitor's real context. These headers pass that lost information back.

*   `proxy_set_header Host $host;`
    *   **What it does:** Forwards the original domain name the user typed in their browser.
    *   **Why you need it:** If your single VPS hosts multiple websites, the backend needs to know which domain the visitor actually asked for.
*   `proxy_set_header X-Real-IP $remote_addr;`
    *   **What it does:** Sends the true IP of the visitor to the backend. (Because of Step 1, `$remote_addr` already holds the real visitor IP, not the Cloudflare IP).
    *   **Why you need it:** So your Node.js app can see the actual user's IP for logging, rate limiting, or security.
*   `proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;`
    *   **What it does:** Keeps a "guest list" of every proxy the request passed through. It takes the existing list and appends the current IP to it, separated by commas.
    *   **Why you need it:** This is the industry standard. Most web frameworks (like Express.js) automatically look for this specific header to figure out where the request came from.

---

## 4. The Cloudflare Special (Optional but safe)
**Location:** Inside your site's `server` block.

*   `proxy_set_header CF-Connecting-IP $http_cf_connecting_ip;`
    *   **What it does:** Passes Cloudflare's raw, untouched header directly to your backend.
    *   **Why you need it:** Just in case the backend developer specifically wrote their code to check for `req.headers['cf-connecting-ip']`. It acts as a safe backup.

---

## 5. The Nginx Real IP Module (Cloudflare Global Config)
**Location:** `/etc/nginx/conf.d/cloudflare.conf` (Set globally, do NOT put this inside individual site blocks).

To make Nginx natively replace Cloudflare's proxy IPs with the actual visitor's IP in your access logs and security tools (like Fail2ban), you must use these two essential directives:

*   `set_real_ip_from <IP_Range>;`
    *   **What it does:** Tells Nginx to *trust* these specific IP ranges (Cloudflare's network). 
    *   **Why you need it:** Nginx shouldn't trust just anyone who sends a fake IP header. By defining Cloudflare's IPs here, Nginx knows: *"If a request comes from these specific servers, I am allowed to trust the hidden header they send."*

*   `real_ip_header CF-Connecting-IP;`
    *   **What it does:** Instructs Nginx to look inside this exact header (`CF-Connecting-IP`) to extract the true visitor's IP address.
    *   **Why you need it:** Once Nginx verifies the request came from a trusted Cloudflare IP (via `set_real_ip_from`), it takes the IP found in this header and permanently overwrites Nginx's internal `$remote_addr` variable. Because of this single line, your `access.log` and Fail2ban will automatically see the real user, not Cloudflare.
    
---

## Quick Summary: When to use what?

| Situation | Which Headers to Use? |
| :--- | :--- |
| **Basic Static Site / Old PHP (No Real-Time Features)** | Step 1 (if CF is used) + `Host`, `X-Real-IP`, and `X-Forwarded-For`. |
| **Modern Web App (Node.js, Next.js, React)** | ALL of the headers in Step 2 and Step 3. You absolutely need the HTTP/1.1 and Upgrade headers for WebSockets to work. |
| **Behind Cloudflare** | Step 1 (Global) + ALL headers in Step 2, 3, and 4. |

**Pro-Tip for Production:** For any Next.js or Node.js gig on Upwork/Fiverr, configure Step 1 globally, then copy-paste the entire block of 6-7 proxy headers as a standard template for the site. It causes zero harm to basic static sites and guarantees that modern apps will work flawlessly without breaking their WebSockets.

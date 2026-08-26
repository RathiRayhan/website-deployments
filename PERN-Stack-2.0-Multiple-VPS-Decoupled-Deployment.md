# PERN Stack Distributed Deployment Guide (Frontend & Backend on Separate VPS)

This runbook documents the deployment of a monolithic PERN (PostgreSQL, Express, React, Node.js) stack application into a distributed architecture using two separate Virtual Private Servers (VPS).

## 🏗 Architecture Overview
*   **VPS 1 (Backend/API):** Hosts PostgreSQL, Node.js/Express Server, PM2, and Nginx (Reverse Proxy).
*   **VPS 2 (Frontend):** Hosts the optimized React build served as static files via Nginx.
*   **Traffic Flow:** Cloudflare Proxy -> Nginx -> Client Browser -> Direct API call to Backend Domain.

---

## 🛠 PART 1: Backend VPS Setup (API & Database)

### 1. Repository Setup
Clone the repository and isolate the backend code.
```bash
sudo git clone https://github.com/Sanjeev-Thiyagarajan/PERN-STACK-DEPLOYMENT /var/www/pern2/
cd /var/www/pern2
# Remove client folder to maintain isolation
sudo rm -rf client/
sudo chown -R admin:admin /var/www/pern2/
sudo chmod -R 755 /var/www/pern2/
```

### 2. Database Initialization
Access the PostgreSQL shell and create the database and user.
```sql
sudo -u postgres psql

CREATE DATABASE pern2db;
CREATE USER pern2user WITH PASSWORD 'StrongPassword123!';
GRANT ALL PRIVILEGES ON DATABASE pern2db TO pern2user;
\q
```
Grant schema permissions:
```bash
sudo -u postgres psql -d pern2db -c "GRANT CREATE ON SCHEMA public TO pern2user;"
```

**Crucial Schema Fix (Legacy Code Issue):**
The original `db.sql` lacked the table creation command. We must inject the schema definition before execution. Add the following to the top of `server/db/db.sql`:
```sql
CREATE TABLE restaurants (
    id BIGSERIAL NOT NULL PRIMARY KEY,
    name VARCHAR(50) NOT NULL,
    location VARCHAR(50) NOT NULL,
    price_range INT NOT NULL check(price_range >= 1 and price_range <= 5)
);
```
Execute the schema file:
```bash
PGPASSWORD='StrongPassword123!' psql -h localhost -U pern2user -d pern2db -f db/db.sql
```

### 3. Backend Environment Configuration
Create a `.env` file in the `server` directory:
```env
PGUSER=pern2user
PGHOST=localhost
PGPASSWORD=StrongPassword123!
PGDATABASE=pern2db
PGPORT=5432
PORT=4003
FRONTEND_URL=https://pern-stack.tmux.qzz.io
```

### 4. Process Management with PM2
Create an `ecosystem.config.js` file in the `server` directory for robust process management:
```javascript
module.exports = {
  apps: [
    {
      name: "pern-app-2",
      script: "server.js",
      instances: "1",
      exec_mode: "fork",
      autorestart: true,
      watch: false,
      max_memory_restart: "200M",
      env: {
        NODE_ENV: "production"
      }
    }
  ]
};
```
Start the backend service:
```bash
pm2 start ecosystem.config.js --env production
pm2 save
```

### 5. Nginx Reverse Proxy & Cloudflare Real IP Config
Create a global configuration to ensure Nginx logs the real visitor IP instead of Cloudflare's proxy IP.
**File:** `/etc/nginx/conf.d/cloudflare.conf`
*(Insert standard Cloudflare IPv4/IPv6 ranges here, followed by:)*
```nginx
real_ip_header CF-Connecting-IP;
```

Create the Backend Server Block:
**File:** `/etc/nginx/sites-available/pern2`
```nginx
server {
    server_name api.rathirayhan.dpdns.org;

    access_log /var/log/nginx/pern2_access.log;
    error_log /var/log/nginx/pern2_error.log warn;

    location / {
        proxy_pass http://127.0.0.1:4003;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "upgrade";
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header CF-Connecting-IP $http_cf_connecting_ip;
    }

    location ~ /\. {
        deny all;
    }
}
```
Enable, test, and apply SSL via Certbot.

---

## 🎨 PART 2: Frontend VPS Setup (React)

### 1. Repository Setup
```bash
sudo git clone https://github.com/Sanjeev-Thiyagarajan/PERN-STACK-DEPLOYMENT /var/www/pern2-frontend/
cd /var/www/pern2-frontend
# Remove server folder to maintain isolation
sudo rm -rf server/
sudo chown -R rayhan:rayhan /var/www/pern2-frontend/
sudo chmod -R 755 /var/www/pern2-frontend/
```

### 2. Resolving Frontend API Routing (The "Same Domain" Dilemma)
**The Problem:** The legacy code assumed the frontend and backend shared the same domain. The `client/src/apis/RestaurantFinder.js` file utilized relative paths:
```javascript
const baseURL = process.env.NODE_ENV === "production" ? "/api/v1/restaurants" : "http://localhost:3001/api/v1/restaurants";
```

**The Solution:** Modify the file to accept a dynamic Environment Variable:
```javascript
const baseURL = process.env.REACT_APP_BACKEND_API || "http://localhost:3001/api/v1/restaurants";
```

> 💡 **Why `REACT_APP_` prefix?** 
> Create React App (CRA) uses Webpack to bundle the application. For security reasons, it strictly ignores any custom environment variable unless it starts with `REACT_APP_`. This prevents sensitive system variables from accidentally leaking into the client-side browser code during the build process.

Create a `.env` file in the `client` root directory:
```env
REACT_APP_BACKEND_API=https://api.rathirayhan.dpdns.org/api/v1/restaurants
```

> 🛠 **Troubleshooting the 404 Error (Option 1 vs Option 2):**
> Initially, setting the variable to just `[https://api.rathirayhan.dpdns.org/](https://api.rathirayhan.dpdns.org/)` resulted in a 404 Not Found error. This happens because the backend API is specifically listening on the `/api/v1/restaurants` route. 
> *   **Option 1 (Developer Approach):** Keep the `.env` root URL, but modify the backend `server.js` routing logic from `app.get("/api/v1/restaurants")` to `app.get("/")`.
> *   **Option 2 (DevOps Approach - Recommended):** Do not alter the backend application logic. Instead, append the exact URI path (`/api/v1/restaurants`) directly to the `.env` file on the frontend. We utilized Option 2 to maintain architectural integrity.

### 3. Build & Compile (Legacy OpenSSL Fix)
Due to the repository's age, standard `npm run build` throws an `ERR_OSSL_EVP_UNSUPPORTED` error in modern Node.js environments (v17+).
**Fix:** Force Node.js to use the legacy OpenSSL provider:
```bash
npm install
NODE_OPTIONS=--openssl-legacy-provider npm run build
```

### 4. Nginx Static Server Block
**File:** `/etc/nginx/sites-available/pern2-frontend`
```nginx
server {
    server_name pern-stack.tmux.qzz.io;

    root /var/www/pern2-frontend/client/build;
    index index.html;

    access_log /var/log/nginx/pern2-frontend_access.log;
    error_log /var/log/nginx/pern2-frontend_error.log warn;

    # React Router fallback
    location / {
        try_files $uri $uri/ /index.html;
    }

    # Cache static assets
    location ~* \.(js|css|png|jpg|jpeg|gif|ico|svg)$ {
        expires max;
        log_not_found off;
    }

    location ~ /\. {
        deny all;
    }
}
```
Enable, test, restart Nginx, and install SSL.

---

## 🧠 PART 3: Architectural Realizations & Notes

During this distributed deployment, several critical DevOps concepts were validated:

1.  **Client-Side API Execution (SPA Architecture):** 
    Initially, there was confusion about whether the Frontend Nginx server needed a `proxy_pass` to the backend API. Realization: React is a Single Page Application (SPA). Once the static `.js` files are downloaded to the user's browser, **the browser itself** makes the HTTP requests directly to the backend API (`api.rathirayhan.dpdns.org`). The frontend Nginx server merely serves the static assets, eliminating the need for proxy headers on the frontend block.
2.  **Cloudflare IP Resolution:** 
    Even with Cloudflare's Proxy (Orange Cloud) enabled, implementing the global `cloudflare.conf` in Nginx successfully parses the `CF-Connecting-IP` header. This ensures application logs reflect the actual visitor's IP address rather than Cloudflare's edge node IPs, which is vital for analytics and Fail2ban security.
3.  **Legacy Code Adaptation:** 
    Deploying older codebases requires a strong understanding of environmental shifts (e.g., OpenSSL standards changing in modern Node). Altering the environment flags (`NODE_OPTIONS`) is a standard DevOps practice to avoid rewriting legacy application code.

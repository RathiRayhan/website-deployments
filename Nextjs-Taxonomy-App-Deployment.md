# Production Deployment Runbook: Next.js Full-Stack Application
**Project:** Taxonomy (via GitHub)
**Architecture:** Next.js (SSR/API), Prisma ORM, MySQL, PM2, Nginx Reverse Proxy.

## 1. System Preparation & Repository Setup

First, create a dedicated directory for the application, clone the repository, and set the appropriate ownership and permissions.

```bash
sudo mkdir -p /var/www/taxonomy
sudo chown -R $USER:$USER /var/www/taxonomy
git clone https://github.com/shadcn-ui/taxonomy /var/www/taxonomy/
cd /var/www/taxonomy
sudo chmod -R 755 /var/www/taxonomy
```

## 2. Database Provisioning (MySQL)

Create a dedicated database and an isolated user for security.

```bash
sudo mysql -u root
```
```sql
CREATE DATABASE taxonomy;
CREATE USER 'taxonomy_user'@'localhost' IDENTIFIED BY 'StrongPassword_123';
GRANT ALL PRIVILEGES ON taxonomy.* TO 'taxonomy_user'@'localhost';
FLUSH PRIVILEGES;
EXIT;
```

## 3. Environment Configuration (`.env`)

Copy the example environment file and update the critical variables.

> **Developer Note (Strict Validation):** Modern Next.js apps often use `.mjs` validation files (like `env.mjs` or Zod) that enforce strict validation during the build process. If third-party API keys (e.g., GitHub, Stripe) are missing, the build will crash. For initial deployment or testing, fill these empty variables with dummy string values to bypass the validation. **Crucial:** The `.env` file MUST reside in the project root (or inside `.next/standalone` if using standalone builds).

```ini
NEXT_PUBLIC_APP_URL=https://taxonomy.rathirayhan.dpdns.org

# This is the redirect url. Same as NEXT_PUBLIC_APP_URL.
NEXTAUTH_URL=https://taxonomy.rathirayhan.dpdns.org 

# Generate a secure JWT token using: openssl rand -base64 32
NEXTAUTH_SECRET=your_generated_secret_here

DATABASE_URL="mysql://taxonomy_user:StrongPassword_123@localhost:3306/taxonomy"
```

## 4. Dependency Management & Resource Optimization

Since the project includes a `pnpm-lock.yaml`, `pnpm` must be used instead of `npm`. Install it globally first: `sudo npm install -g pnpm`.

> **Troubleshooting Note (OOM & Low RAM):** Running `pnpm install` on a 1GB RAM server may get stuck or result in Out Of Memory (OOM) errors. 
> *Solution:* Create a 2GB swap file so the OS has extra memory to page into. Then, to cap the V8 heap during memory-heavy Node steps (install/build), set a max heap size:
> ```bash
> NODE_OPTIONS="--max-old-space-size=2560" pnpm install
> ```
> Note: `--max-old-space-size` limits the V8 heap size during the Node process; the real fix for a 1GB server is the swap file itself.

## 5. Database Migration (Prisma Workflow)

Prisma translates developer code into the relevant database schema. The strict production workflow is as follows:

1. `pnpm install` (Install dependencies)
2. `npx prisma generate` (Generates the Prisma Client)
3. `npx prisma migrate deploy` (Applies migrations to the production DB)
4. `pnpm run build` (Builds the Next.js app)

> **Architectural Note (Prisma Best Practices):** 
> * Always use `migrate deploy` in production. **Never** use `db push` on a live database as it may override the existing schema and cause data loss.
> * *Database Engine Switch:* To change the database (e.g., MySQL to PostgreSQL), modify the `provider` in `prisma/schema.prisma`. You must also delete the existing `prisma/migrations` folder to generate new PostgreSQL-compatible migrations. However, deviating from the developer's chosen database is generally not recommended unless strictly required.

## 6. Application Build

Compile the application for production.

```bash
pnpm run build
```
> **Resource Scaling Note:** If the build gets stuck on "linting and checking validity of types", it is likely a CPU/RAM bottleneck. In our deployment, we moved to a 3GB RAM server to successfully complete the build. Upon success, Next.js generates a hidden `.next` directory (similar to `dist` or `build` folders in decoupled architectures).

## 7. Process Management (PM2)

Use PM2 to run the Next.js application in the background and ensure it restarts on system crashes. 

Initialize PM2 (`pm2 init`) and replace the contents of `ecosystem.config.js` with the Next.js standard configuration:

```javascript
module.exports = {
  apps: [
    {
      name: "taxonomy-app",
      script: "pnpm",
      args: "start",
      instances: 1,
      autorestart: true,
      watch: false,
      max_memory_restart: "1G",
      env: {
        NODE_ENV: "production",
        PORT: 3000
      }
    }
  ]
};
```
Start the application and save the state:
```bash
pm2 start ecosystem.config.js --env production
pm2 save
```
*(Verify health using `pm2 status` and `pm2 logs`).*

## 8. Reverse Proxy Configuration (Nginx)

Next.js handles routing differently than a traditional decoupled React app. 

> **Architectural Comparison (Decoupled vs. Next.js):**
> * **Decoupled React:** The Nginx `location /` block uses the `root` directive to serve static compiled files (e.g., `/var/www/app/dist`). API requests are routed separately using a `location /api/` block.
> * **Next.js:** Because it is a full-stack framework with SSR and built-in API routes, Nginx acts strictly as a reverse proxy. We pass **all** requests (`/`) to the internal port (e.g., 3000) using `proxy_pass`. Ensure no other Node.js apps are running on this port.

Create the Nginx configuration:
```bash
sudo nano /etc/nginx/sites-available/taxonomy
```

```nginx
server {
    listen 80;
    listen [::]:80;
    server_name taxonomy.rathirayhan.dpdns.org;

    access_log /var/log/nginx/taxonomy_access.log;
    error_log /var/log/nginx/taxonomy_error.log;

    location / {
        proxy_pass http://127.0.0.1:3000;

        # WebSocket & Persistent Connection Headers
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection 'upgrade';
        proxy_set_header Host $host;

        # Client IP Forwarding
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
        
        # Bypass Next.js cache for live/upgraded connections
        proxy_cache_bypass $http_upgrade;
    }
}
```

> **Header Explanations:**
> * `Upgrade` & `Connection`: These headers are mandatory for WebSocket (e.g., Socket.io) support or live Server-Sent Events. We use them globally for Next.js to ensure its internal dynamic routing and real-time features work flawlessly.
> * `proxy_cache_bypass`: Informs the Next.js server to ignore cached media and serve the latest live data directly from the server whenever an upgraded request (WebSocket/SSE) arrives.

Enable the configuration and reload Nginx:
```bash
sudo ln -s /etc/nginx/sites-available/taxonomy /etc/nginx/sites-enabled/
sudo nginx -t
sudo systemctl reload nginx
```

## 9. SSL/TLS Certificate (Certbot)

Secure the application with an HTTPS padlock:
```bash
sudo certbot --nginx -d taxonomy.rathirayhan.dpdns.org
```

---

## Appendix: Advanced Optimization (Standalone Build)

For containerized microservices (Docker), Next.js supports a "Standalone" build. 
Adding the following to `next.config.js`:
```javascript
module.exports = {
  output: 'standalone',
}
```
This reduces the deployment size dramatically by creating a `.next/standalone` folder containing only the strictly necessary node modules. 
*Deployment requirement:* To serve a standalone build properly, the `.next/static` and `public` folders must be manually copied into the `.next/standalone` directory. 
*Note:* While highly recommended for Docker deployments to ensure smaller container images, it is generally avoided for manual VPS/PM2 deployments where the standard `.next` folder suffices.

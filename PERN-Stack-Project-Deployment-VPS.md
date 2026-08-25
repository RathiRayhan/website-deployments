# PERN Stack Production Deployment Runbook

**Project Structure:** Root-Level Backend with React Frontend  
**Repository:** [FaztWeb/pern-stack](https://github.com/FaztWeb/pern-stack)  
**Environment:** Ubuntu VPS, Nginx, PostgreSQL, PM2, Node.js

---

## 1. Initial Server Setup & Repository Cloning

Clone the repository into the web directory and set the correct permissions.

```bash
sudo git clone https://github.com/FaztWeb/pern-stack /var/www/pern-stack/
cd /var/www/pern-stack/
```

**DevOps Insight: Directory Permissions & The `www-data` Trap**
> Never give full ownership (`chown`) of the project directory to Nginx's default user (`www-data`). If you do, the active user (e.g., `admin`) will lose write permissions and face `EACCES: permission denied` errors when trying to run build commands like `npm run build`. 
> 
> **The Fix:** Assign ownership to the active working user (`admin`) and set the permissions to `755`. This ensures the owner can read/write/execute freely, while `www-data` (Nginx) has sufficient permissions to *read* the compiled frontend assets without owning them.

```bash
sudo chown -R admin:admin /var/www/pern-stack/
sudo chmod -R 755 /var/www/pern-stack/
```

---

## 2. Database Provisioning (The Ideal Workflow)

**Best Practice Warning:** Always inspect the repository for database initialization files (like `database/init.sql`) *before* starting the app. If you skip this, the backend will throw `500 Internal Server Error` because the expected schema/tables won't exist.

Install PostgreSQL:
```bash
sudo apt install -y postgresql postgresql-contrib
```

### Step 2.1: Create Database and User
Log into the PostgreSQL shell:
```bash
sudo -u postgres psql
```
Execute the following:
```sql
CREATE DATABASE pern_stack;
CREATE USER pern_user WITH PASSWORD 'StrongPassword123!';
GRANT ALL PRIVILEGES ON DATABASE pern_stack TO pern_user;
\q
```

### Step 2.2: Grant Schema Permissions (For PostgreSQL 15+)
Modern PostgreSQL versions restrict the default `public` schema. Grant schema creation rights to your application user. 
*(Note: `-c` stands for `--command`, allowing us to execute a single SQL command directly from the bash terminal).*

```bash
sudo -u postgres psql -d pern_stack -c "GRANT CREATE ON SCHEMA public TO pern_user;"
```

### Step 2.3: Execute the Schema Initialization File
Inject the `init.sql` file into the database as the application user.

```bash
PGPASSWORD='StrongPassword123!' psql -h localhost -U pern_user -d pern_stack -f database/init.sql
```
**Command Breakdown:**
* `PGPASSWORD='...'`: An environment variable that automatically passes the password. (Do NOT use the `-W` flag with this, as `-W` forces an interactive prompt which breaks automation).
* `-h localhost`: Connects via TCP/IP to trigger password authentication.
* `-U pern_user`: Specifies the PostgreSQL user.
* `-d pern_stack`: Connects to this specific database.
* `-f database/init.sql`: Reads and executes SQL commands directly from the specified file.

---

## 3. Backend Deployment (Node.js & PM2)

Navigate to the project root and install backend dependencies.

```bash
npm install
npm install dotenv
```

### Step 3.1: Environment & PM2 Ecosystem Configuration

**DevOps Insight: PM2 and `dotenv` Injection**
> Upon code inspection, the developer included `.env` features in `src/config.js` but failed to import the `dotenv` package (e.g., `require('dotenv').config()`) in the entry file. Without touching the source code, we fix this at the infrastructure level by requiring `dotenv` directly inside the PM2 ecosystem file.
> 
> Also, because `package.json` specifies `"type": "module"`, the ecosystem file MUST be named `.cjs` (CommonJS), otherwise PM2 will throw a `ReferenceError`.

Create the PM2 configuration file:
```bash
nano ecosystem.config.cjs
```

Add the following configuration:
```javascript
require('dotenv').config(); // Automatically loads local .env variables

module.exports = {
  apps: [
    {
      name: "pern-backend",
      script: "./src/index.js",
      instances: "1", 
      exec_mode: "fork", 
      autorestart: true,
      watch: false,
      max_memory_restart: "200M",
      env: {
        ...process.env, // Injects the loaded .env variables into PM2
        NODE_ENV: "production"
      }
    }
  ]
};
```

Create your `.env` file in the root directory:
```bash
nano .env
```
```env
DB_USER=pern_user
DB_PASSWORD=StrongPassword123!
DB_HOST=localhost
DB_PORT=5432
DB_DATABASE=pern_stack
PORT=4000
```

### Step 3.2: Start the Backend Service
```bash
pm2 start ecosystem.config.cjs --env production
pm2 save
```

---

## 4. Frontend Compilation & API Routing Fix

Navigate to the frontend directory and install dependencies:
```bash
cd frontend
npm install
```

### Step 4.1: Fixing Hardcoded API Endpoints (The Relative Path Method)
The frontend code explicitly targets `http://localhost:4000`, which fails in production since the React app runs on the client's browser, not the server. We must patch this to use a relative path (`/api`) so Nginx can proxy it correctly.

Find all hardcoded instances:
```bash
grep -r "http://localhost:4000" src/
```

Patch them globally using `sed`:
```bash
sed -i 's|http://localhost:4000|/api|g' src/components/*.jsx
```
**Command Breakdown:**
* `-i`: In-place editing (saves changes directly to the file).
* `s`: Substitute command.
* `|`: Delimiter. We use `|` instead of `/` to avoid conflict with the slashes in the URL.
* `/api`: The replacement string.
* `g`: Global replacement across the line.

### Step 4.2: Build the Frontend
```bash
npm run build
```

---

## 5. Web Server Configuration (Nginx & SSL)

Create the Nginx server block:
```bash
sudo nano /etc/nginx/sites-available/pern-stack
```

```nginx
server {
    listen 80;
    listen [::]:80;
    server_name pern-stack.yourdomain.com;

    access_log /var/log/nginx/pern-stack_access.log;
    error_log /var/log/nginx/pern-stack_error.log;

    # 1. Serve Frontend React Static Files
    location / {
        root /var/www/pern-stack/frontend/dist;
        index index.html;
        try_files $uri $uri/ /index.html; # Fallback for React Router
    }

    # 2. Proxy Backend Node.js API
    location /api/ {
        proxy_pass http://127.0.0.1:4000/;
        proxy_http_version 1.1;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}
```

**DevOps Insight: The Nginx Trailing Slash**
> Notice the trailing slash in `proxy_pass http://127.0.0.1:4000/;`. By inspecting the backend routes, we see the API does not expect an `/api/` prefix. The trailing slash instructs Nginx to automatically strip the `/api/` from the request URL before forwarding it to the backend.

### Step 5.1: Enable the Site and SSL
```bash
sudo ln -s /etc/nginx/sites-available/pern-stack /etc/nginx/sites-enabled/
sudo nginx -t
sudo systemctl restart nginx
```

Provision an SSL certificate via Certbot:
```bash
sudo certbot --nginx -d pern-stack.yourdomain.com
```

**Deployment Complete.** The application is now live, secure, and production-ready.

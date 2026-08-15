# Production Deployment Runbook: MERN Stack on a Single Ubuntu VPS

## 📌 Architecture Overview
This runbook details the production-grade deployment of a MERN (MongoDB, Express, React, Node.js) stack on a single Ubuntu Virtual Private Server (VPS). 

**Traffic Flow:**
- **Nginx (Ports 80/443):** Acts as the primary web server and reverse proxy.
- **Frontend:** React static files compiled and served directly by Nginx from `/var/www/`.
- **Backend:** Node.js API managed by PM2, running securely on localhost (Port 5000).
- **Database:** MongoDB running locally (Port 27017).
- Nginx intercepts all traffic. Requests to `/` are served static React files. Requests starting with `/api/` are forwarded to the internal Node.js backend.

---

## ⚙️ Phase 1: Database Setup (MongoDB)

1. **Install MongoDB Community Edition:** Follow the official Ubuntu installation guide for the latest version.
2. **Start and Enable Service:** Ensure the database starts automatically on reboot.
   ```bash
   sudo systemctl start mongod
   sudo systemctl enable mongod
   sudo systemctl status mongod
   ```
*Note: MongoDB listens on `127.0.0.1:27017` by default, ensuring it is not exposed to the public internet.*

---

## 🚀 Phase 2: Backend Deployment (Node.js + Express)

**Important Security Note:** Never run application processes or PM2 as the `root` user. Always use a dedicated non-root standard user (e.g., your login user or a dedicated `deploy` user).

1. **Clone the Repository:**
   ```bash
   sudo mkdir -p /var/www/backend-nodejs
   sudo git clone https://github.com/bezkoder/node-express-mongodb.git /var/www/backend-nodejs
   ```

2. **Set Ownership:** Transfer ownership from root to the user that will execute the PM2 process.
   ```bash
   sudo chown -R $USER:$USER /var/www/backend-nodejs
   cd /var/www/backend-nodejs
   ```

3. **Install Dependencies:**
   ```bash
   npm install
   npm install dotenv  # Ensure dotenv is installed for environment variable management
   ```

4. **Environment Variables (`.env`):**
   Create a `.env` file in the backend root directory. Set permissions to `600` and ensure it is added to `.gitignore`.
   ```env
   PORT=5000
   MONGO_URI=mongodb://127.0.0.1:27017/node_express_app
   FRONTEND_URL=https://react-crud-web-api.yourdomain.com
   ```

5. **Process Management with PM2:**
   Install PM2 globally and initialize the ecosystem configuration file for production control.
   ```bash
   sudo npm install -g pm2
   pm2 init simple
   ```
   Edit the generated `ecosystem.config.js`:
   ```javascript
   module.exports = {
     apps : [{
       name: 'backend-nodejs',
       script: 'server.js',
       watch: false,
       instances: '1', // Set to 'max' and use exec_mode: 'cluster' for multi-core scaling
       max_memory_restart: '500M', // Prevents memory leak crashes
       env: {
         NODE_ENV: 'production'
       }
     }],
   };
   ```

6. **Start and Persist Backend:**
   ```bash
   pm2 start ecosystem.config.js --env production
   pm2 save
   pm2 startup systemd
   ```
   *(Run the exact command outputted by `pm2 startup` to generate the systemd service, e.g., `pm2-username`)*.

---

## 🎨 Phase 3: Frontend Deployment (React)

1. **Clone and Configure:**
   ```bash
   git clone https://github.com/bezkoder/react-crud-web-api.git ~/react-crud-web-api
   cd ~/react-crud-web-api
   ```
   **CRITICAL STEP:** Edit `src/http-common.js` (or your respective config file) and change the `baseURL` to target the Nginx reverse proxy endpoint, NOT localhost.
   ```javascript
   // Change from "http://localhost:5000/api" to:
   baseURL: "/api" 
   ```

2. **Build the Application:**
   ```bash
   npm install
   npm run build
   ```

3. **Deploy Static Assets:**
   Move the compiled `build` folder to the web directory and assign proper Nginx ownership (`www-data`).
   ```bash
   sudo mkdir -p /var/www/react-crud-web-api
   sudo cp -r build/* /var/www/react-crud-web-api/
   sudo chown -R www-data:www-data /var/www/react-crud-web-api
   sudo chmod -R 755 /var/www/react-crud-web-api
   ```

---

## 🚦 Phase 4: Nginx Reverse Proxy Configuration

1. **Create the Server Block:**
   ```bash
   sudo nano /etc/nginx/sites-available/react-crud-web-api
   ```
   Insert the following configuration:
   ```nginx
   server {
       listen 80;
       listen [::]:80;
       server_name react-crud-web-api.yourdomain.com;

       access_log /var/log/nginx/react-crud-web-api_access.log;
       error_log /var/log/nginx/react-crud-web-api_error.log;

       # 1. Serve Frontend React Static Files
       location / {
           root /var/www/react-crud-web-api;
           index index.html;
           try_files $uri $uri/ /index.html; # Fallback for React Router
       }

       # 2. Proxy Backend Node.js API
       location /api/ {
           proxy_pass http://127.0.0.1:5000; # No trailing slash here!
           proxy_http_version 1.1;
           proxy_set_header Host $host;
           proxy_set_header X-Real-IP $remote_addr;
           proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
           proxy_set_header X-Forwarded-Proto $scheme;
       }
   }
   ```
   *Architectural Note:* Using `location /api/` explicitly routes API calls. Ensure `proxy_pass` does **not** have a trailing slash (`http://127.0.0.1:5000`) so the full URI is passed to the Express backend.

2. **Enable and Test:**
   ```bash
   sudo ln -s /etc/nginx/sites-available/react-crud-web-api /etc/nginx/sites-enabled/
   sudo nginx -t
   sudo systemctl reload nginx
   ```

---

## 🔒 Phase 5: Security & SSL (Certbot)

1. **Configure UFW (Firewall):**
   ```bash
   sudo ufw allow OpenSSH
   sudo ufw allow 'Nginx Full'
   sudo ufw enable
   ```
   *(Port 5000 remains blocked from the outside, enforcing traffic through Nginx).*

2. **Install SSL Certificate:**
   ```bash
   sudo certbot --nginx -d react-crud-web-api.yourdomain.com
   ```

---

## 🛠️ Common Troubleshooting Scenarios

*   **404 Not Found (Post-Deployment):** Occurs if the browser forces `https://` before Certbot is configured. Fix: Complete the Certbot SSL installation (Phase 5).
*   **Database Connection Failed (CORS or Network Error):** If the frontend fails to fetch data (visible as a failed XHR request in DevTools), the React `baseURL` was likely left as `http://localhost:5000`. Browsers cannot route client-side requests to the user's local machine port. Fix: Rebuild the React app with `baseURL: "/api"`.
*   **API Routing Issues:** If the backend receives corrupted endpoint paths, check the Nginx `proxy_pass` directive. A trailing slash (`http://127.0.0.1:5000/`) strips the matching `/api/` prefix, whereas omitting the slash preserves the exact requested URI.

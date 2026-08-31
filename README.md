# 🚀 Server Deployments & Hosting Portfolio

Welcome to my DevOps & Server Administration portfolio repository!  

This repository serves as a centralized documentation hub for various web applications, CMS platforms, and custom projects I have deployed. It demonstrates my ability to take a project from a raw bare-metal server/VPS to a secure, high-performance, production-ready state.

## 🌟 Production Standards Maintained
For every deployment listed here, I adhere to strict industry best practices:
- **Security First:** SSH hardening, strict firewall configurations (UFW), and rigid file/folder permissions (e.g., `755/644`).
- **Web Servers:** High-performance reverse proxy setups and load balancing using **Nginx**.
- **Encryption:** Automated SSL/TLS certificates via Let's Encrypt (Certbot) with strict HTTP-to-HTTPS (301) redirection.
- **Data Integrity:** Routine automated backups and disaster recovery SOPs for relational and NoSQL databases.
- **Process Management:** Absolute socket paths for FastCGI/PHP-FPM and robust PM2/Systemd configurations for Node.js apps.

---

## 📂 Runbooks & Documentation

I have categorized my deployment guides logically. Click on the links to view the step-by-step production runbooks:

### 🛡️ 1. Server Administration & Security
Core infrastructure setup, automation, and system hardening.
| Topic | Documentation |
|-------|---------------|
| Initial Linux Server Setup & Hardening | [View Runbook](./server-setup/01-linux-server-setup.md) |
| Nginx Reverse Proxy & SSL Configuration | [View Runbook](./server-setup/02-nginx-ssl.md) |
| Bash Scripting & Automation (Cron) | [View Runbook](./server-setup/03-bash-scripting-automation.md) |
| Server Disaster Recovery SOP | [View Runbook](./server-setup/04-server-disaster-recovery-SOP.md) |
| Fail2ban Setup | [View Runbook](./server-setup/05-fail2ban-setup.md) |

### ⚙️ 2. Nginx Configuration & Deep Dives
Advanced routing, templates, and reverse proxy settings.
| Topic | Documentation |
|-------|---------------|
| Nginx API Routing & Path Manipulation (Regex) | [View Runbook](./nginx-config-settings/api-block-notes.md) |
| Reverse Proxy Headers Explained | [View Runbook](./nginx-config-settings/reverse-proxy-headers-explained.md) |
| Server Block Master Template | [View Runbook](./nginx-config-settings/server-block-master-template.md) |

### 💾 3. Database Migration & Disaster Recovery
Production standards for backing up, transferring, and restoring databases.
| Database | Documentation |
|----------|---------------|
| **PostgreSQL** (Local & Supabase) | [View Runbook](./database-backup-restore/postgresql-backup-restore.md) |
| **MySQL / MariaDB** | [View Runbook](./database-backup-restore/mysql-mariadb-backup-restore.md) |
| **MongoDB** (Local & Atlas) | [View Runbook](./database-backup-restore/mongodb-backup-restore.md) |

### 🌐 4. Full-Stack Application Deployments
End-to-end deployment guides for modern web stacks.
| Stack / Project | Technologies Used | Documentation |
|-----------------|-------------------|---------------|
| **PERN 2.0 (Multiple VPS)** | PostgreSQL, Express, React, Node.js (Decoupled) | [View Runbook](./PERN-Stack-2.0-Multiple-VPS-Decoupled-Deployment.md) |
| **PERN Stack (Single VPS)** | PostgreSQL, Express, React, Node.js | [View Runbook](./PERN-Stack-Project-Deployment-VPS.md) |
| **MERN E-commerce** | MongoDB, Express, React, Node.js | [View Runbook](./MERN-Ecommerce-Deployment.md) |
| **MERN Single VPS** | Nginx, PM2, MongoDB | [View Runbook](./MERN-Single-VPS-Deployment.md) |
| **Next.js Taxonomy App** | Next.js, Nginx, PM2 | [View Runbook](./Nextjs-Taxonomy-App-Deployment.md) |
| **React.js / Vite Frontend**| Vite, React, Nginx (Static) | [View Runbook](./Reactjs-Vite-Frontend-Deploy.md) |

### 📝 5. CMS & PHP Deployments
Optimized hosting setups for PHP-based applications.
| Project | Technologies Used | Documentation |
|---------|-------------------|---------------|
| **WordPress (LEMP)** | Nginx, PHP-FPM, MariaDB | [View Runbook](./WordPress-LEMP-Deployment-SingleTenant.md) |
| **Multi-Tenant PHP Setup** | Nginx, Isolated PHP-FPM pools | [View Runbook](./Multi-Tenant-Nginx-PHP.md) |
| **Flat-File CMS (Bludit)**| Nginx, PHP 8.x, PHP-FPM | [View Runbook](./Bludit-Deploy-Nginx-PHPFPM.md) |

---

## 🛠 Core Tech Stack & Tools
- **OS:** Ubuntu / Debian (Servers), Arch Linux (Local Development)
- **Web Servers:** Nginx
- **Databases:** PostgreSQL, MySQL/MariaDB, MongoDB, Supabase
- **Environments:** Node.js, PHP-FPM
- **Security & Networking:** SSL/TLS, UFW, DNS Management (A/CNAME Records), SSH Keys

---
*💡 Open to freelance opportunities on Upwork and Fiverr. Need a secure, highly optimized server setup or seamless database migration? Let's talk!*

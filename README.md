# 🚀 Server Deployments & Hosting Portfolio

Welcome to my DevOps & Server Administration portfolio repository! 

This repository serves as a centralized documentation hub for various web applications, CMS platforms, and custom projects I have deployed. It demonstrates my ability to take a project from a raw bare-metal server/VPS to a secure, production-ready state.

## 🌟 Production Standards Maintained
For every deployment listed here, I adhere to strict industry best practices:
- **Security First:** SSH hardening, strict firewall configurations (UFW), and rigid file/folder permissions (e.g., `755/644`).
- **Web Servers:** High-performance reverse proxy setups and load balancing using **Nginx**.
- **Encryption:** Automated SSL/TLS certificates via Let's Encrypt (Certbot) with strict HTTP-to-HTTPS (301) redirection.
- **Process Management:** Using absolute socket paths for FastCGI/PHP-FPM to guarantee zero downtime and avoid symlink-related vulnerabilities during upgrades.

## 📂 Deployment Runbooks
Here is the list of my documented deployments. Click on the links to view the step-by-step production runbooks:

| Project | Stack / Technology | Documentation |
|---------|--------------------|---------------|
| **Flat-File CMS (Bludit)** | Nginx, PHP 8.x, PHP-FPM, SSL | [View Runbook](./Deploy-Bludit-Nginx-PHPFPM.md) |
| *(More real-world deployments coming soon...)* | | |

## 🛠️ Core Tech Stack & Tools
- **OS:** Ubuntu / Debian (Servers), Arch Linux (Local Environment)
- **Web Servers:** Nginx
- **Environments:** PHP-FPM, Node.js
- **Security & Networking:** SSL/TLS, UFW, DNS Management (A/CNAME Records)

---
*Open to freelance opportunities on Upwork and Fiverr. Need a secure and blazing-fast server setup? Let's talk!*# website-deployments

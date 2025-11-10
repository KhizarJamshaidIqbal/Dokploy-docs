# Dokploy-docs For ME

========================================
DOKPLOY APPLICATION DEPLOYMENT GUIDE
Nginx Reverse Proxy + SSL Configuration
========================================

📋 PREREQUISITES CHECK
----------------------
1. VPS par Dokploy installed aur running ho
2. Nginx installed ho: nginx -v
3. Domain/Subdomain ka DNS A record configured ho (VPS IP point kare)
4. SSL Certificate ready ho (cPanel ya Let's Encrypt se)

========================================
PART 1: DOKPLOY APPLICATION DEPLOYMENT
========================================

Step 1: Application Deploy karna
---------------------------------
1. Dokploy Dashboard: http://YOUR_VPS_IP:3000
2. "Create Application" click karein
3. Application type select karein (Node.js, Docker, etc.)
4. Git repository ya Docker image configure karein
5. Environment variables add karein (agar zaroorat ho)
6. Deploy button click karein

Step 2: Application Port Note karna
------------------------------------
Command: docker ps | grep YOUR_APP_NAME

Output example:
5c279e0133c4   7b628bc21a08   ...   0.0.0.0:3100->5000/tcp   app-name

⚠️ IMPORTANT: External port note karein (example mein: 3100)
Ye port Nginx configuration mein use hoga

========================================
PART 2: NGINX CONFIGURATION
========================================

Step 3: Check Nginx Status
---------------------------
Command: sudo systemctl status nginx

Agar running nahi hai to:
sudo systemctl start nginx
sudo systemctl enable nginx

Step 4: Port Check (Conflict Detection)
----------------------------------------
Command: sudo netstat -tlnp | grep ':80\|:443'

Ya: sudo ss -tlnp | grep ':80\|:443'

⚠️ Agar port 80/443 busy hai to sahi hai - nginx use kar raha hai

Step 5: Nginx Configuration File Banana
----------------------------------------
Command: sudo nano /etc/nginx/sites-available/YOUR_SUBDOMAIN

Configuration Template (WITHOUT SSL - First):
----------------------------------------------
server {
    listen 80;
    server_name YOUR_SUBDOMAIN.YOUR_DOMAIN.com;
    
    location / {
        proxy_pass http://localhost:YOUR_APP_PORT;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection 'upgrade';
        proxy_set_header Host $host;
        proxy_cache_bypass $http_upgrade;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}

⚠️ REPLACE karna hai:
- YOUR_SUBDOMAIN.YOUR_DOMAIN.com → Actual subdomain (e.g., api.epsoldev.com)
- YOUR_APP_PORT → Docker ps se mila external port (e.g., 3100)

Save karna: Ctrl+O, Enter, Ctrl+X

Step 6: Configuration Enable karna
-----------------------------------
Command: sudo ln -s /etc/nginx/sites-available/YOUR_SUBDOMAIN /etc/nginx/sites-enabled/

Verify: ls -la /etc/nginx/sites-enabled/ | grep YOUR_SUBDOMAIN

Step 7: Configuration Test
---------------------------
Command: sudo nginx -t

Output should be: "syntax is ok" and "test is successful"

Step 8: Nginx Reload
---------------------
Command: sudo systemctl reload nginx

Step 9: HTTP Test (Without SSL)
--------------------------------
Command: curl -I http://YOUR_SUBDOMAIN.YOUR_DOMAIN.com

Ya browser mein: http://YOUR_SUBDOMAIN.YOUR_DOMAIN.com

✅ Agar data aa raha hai to HTTP working hai

========================================
PART 3: SSL CERTIFICATE INSTALLATION
========================================

Option A: cPanel se Certificate Copy karna (RECOMMENDED)
---------------------------------------------------------

Step 10a: SSL Files Create karna
---------------------------------

1. Certificate (CRT) file:
   Command: sudo nano /etc/ssl/certs/YOUR_SUBDOMAIN.YOUR_DOMAIN.com.crt
   
   cPanel → SSL/TLS → Certificate (CRT) section copy karein
   Paste karein (-----BEGIN CERTIFICATE----- se -----END CERTIFICATE----- tak)
   Save: Ctrl+O, Enter, Ctrl+X

2. Private Key (KEY) file:
   Command: sudo nano /etc/ssl/private/YOUR_SUBDOMAIN.YOUR_DOMAIN.com.key
   
   cPanel → SSL/TLS → Private Key (KEY) section copy karein
   Paste karein (-----BEGIN RSA PRIVATE KEY----- se -----END RSA PRIVATE KEY----- tak)
   Save: Ctrl+O, Enter, Ctrl+X

3. CA Bundle (CABUNDLE) file:
   Command: sudo nano /etc/ssl/certs/YOUR_SUBDOMAIN.YOUR_DOMAIN.com.cabundle
   
   cPanel → SSL/TLS → Certificate Authority Bundle section copy karein
   Paste karein (multiple certificates ho sakti hain)
   Save: Ctrl+O, Enter, Ctrl+X

Step 11: Nginx Configuration Update (WITH SSL)
-----------------------------------------------
Command: sudo nano /etc/nginx/sites-available/YOUR_SUBDOMAIN

Puri file replace karein is configuration se:
----------------------------------------------
server {
    listen 80;
    server_name YOUR_SUBDOMAIN.YOUR_DOMAIN.com;
    
    # Redirect to HTTPS
    return 301 https://$host$request_uri;
}

server {
    listen 443 ssl;
    server_name YOUR_SUBDOMAIN.YOUR_DOMAIN.com;
    
    # SSL Certificates
    ssl_certificate /etc/ssl/certs/YOUR_SUBDOMAIN.YOUR_DOMAIN.com.crt;
    ssl_certificate_key /etc/ssl/private/YOUR_SUBDOMAIN.YOUR_DOMAIN.com.key;
    ssl_trusted_certificate /etc/ssl/certs/YOUR_SUBDOMAIN.YOUR_DOMAIN.com.cabundle;
    
    location / {
        proxy_pass http://localhost:YOUR_APP_PORT;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection 'upgrade';
        proxy_set_header Host $host;
        proxy_cache_bypass $http_upgrade;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}

⚠️ REPLACE karna hai:
- YOUR_SUBDOMAIN.YOUR_DOMAIN.com (3 jagah)
- YOUR_APP_PORT

Save: Ctrl+O, Enter, Ctrl+X

Option B: Let's Encrypt (Automatic Certificate)
------------------------------------------------

Step 10b: Certbot se Certificate Generate karna
------------------------------------------------
Command: sudo certbot --nginx -d YOUR_SUBDOMAIN.YOUR_DOMAIN.com

Follow prompts:
- Email enter karein
- Terms agree karein
- Automatic redirect to HTTPS: Yes

Certbot automatically nginx configuration update kar dega

========================================
PART 4: FINAL TESTING
========================================

Step 12: Configuration Test
----------------------------
Command: sudo nginx -t

Output: "syntax is ok" and "test is successful"

Step 13: Nginx Reload
----------------------
Command: sudo systemctl reload nginx

Step 14: HTTPS Test
--------------------
Browser mein: https://YOUR_SUBDOMAIN.YOUR_DOMAIN.com

✅ Green padlock aur application data dikhna chahiye

Command line test:
curl -I https://YOUR_SUBDOMAIN.YOUR_DOMAIN.com

(⚠️ curl mein SSL error aa sakta hai but browser mein work karna zaruri hai)

========================================
TROUBLESHOOTING - COMMON ISSUES
========================================

Issue 1: Port already in use
-----------------------------
Error: nginx: [emerg] bind() to 0.0.0.0:80 failed

Solution:
sudo netstat -tlnp | grep ':80'
# Check kaun process use kar raha hai
# Agar koi aur service hai to stop karein
sudo systemctl stop SERVICE_NAME

Issue 2: Wrong website showing on subdomain
--------------------------------------------
Solution:
- Check nginx sites-enabled: ls -la /etc/nginx/sites-enabled/
- Check server_name in configuration
- Disable default site agar conflict hai:
  sudo rm /etc/nginx/sites-enabled/default
  sudo systemctl reload nginx

Issue 3: SSL certificate errors
--------------------------------
Solution:
- Verify certificate files exist:
  ls -la /etc/ssl/certs/YOUR_SUBDOMAIN*
  ls -la /etc/ssl/private/YOUR_SUBDOMAIN*
- Check file permissions:
  sudo chmod 644 /etc/ssl/certs/YOUR_SUBDOMAIN*
  sudo chmod 600 /etc/ssl/private/YOUR_SUBDOMAIN*

Issue 4: Application not responding
------------------------------------
Solution:
- Check Docker container running hai:
  docker ps | grep YOUR_APP
- Check container logs:
  docker logs CONTAINER_ID
- Restart container:
  docker restart CONTAINER_ID

Issue 5: DNS not resolving
---------------------------
Solution:
- DNS propagation check: nslookup YOUR_SUBDOMAIN.YOUR_DOMAIN.com
- DNS mein A record check karein
- TTL ke liye wait karein (usually 5-15 minutes)

========================================
USEFUL COMMANDS REFERENCE
========================================

Nginx Commands:
---------------
sudo systemctl status nginx      # Status check
sudo systemctl start nginx       # Start
sudo systemctl stop nginx        # Stop
sudo systemctl restart nginx     # Restart
sudo systemctl reload nginx      # Reload (without downtime)
sudo nginx -t                    # Test configuration

Docker Commands:
----------------
docker ps                        # Running containers
docker ps -a                     # All containers
docker logs CONTAINER_ID         # View logs
docker restart CONTAINER_ID      # Restart container
docker exec -it CONTAINER_ID sh  # Enter container

File Locations:
---------------
Nginx configs:    /etc/nginx/sites-available/
Nginx enabled:    /etc/nginx/sites-enabled/
SSL certs:        /etc/ssl/certs/
SSL keys:         /etc/ssl/private/
Nginx main conf:  /etc/nginx/nginx.conf

========================================
QUICK DEPLOYMENT CHECKLIST
========================================

□ Application Dokploy par deploy kiya
□ Docker ps se external port note kiya
□ DNS A record configured hai
□ Nginx configuration file banai
□ Configuration enable kiya (symlink)
□ Nginx test kiya (nginx -t)
□ Nginx reload kiya
□ HTTP test kiya (browser/curl)
□ SSL certificate files install kiye (3 files)
□ Nginx configuration SSL ke sath update kiya
□ Nginx test aur reload kiya
□ HTTPS test kiya (browser)
□ Application properly respond kar raha hai

========================================
TEMPLATE FILES FOR QUICK COPY-PASTE
========================================

File: /etc/nginx/sites-available/SUBDOMAIN
-------------------------------------------

# HTTP ONLY (For initial testing):
server {
    listen 80;
    server_name SUBDOMAIN.DOMAIN.com;
    location / {
        proxy_pass http://localhost:PORT;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection 'upgrade';
        proxy_set_header Host $host;
        proxy_cache_bypass $http_upgrade;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}

# HTTPS (Production):
server {
    listen 80;
    server_name SUBDOMAIN.DOMAIN.com;
    return 301 https://$host$request_uri;
}
server {
    listen 443 ssl;
    server_name SUBDOMAIN.DOMAIN.com;
    ssl_certificate /etc/ssl/certs/SUBDOMAIN.DOMAIN.com.crt;
    ssl_certificate_key /etc/ssl/private/SUBDOMAIN.DOMAIN.com.key;
    ssl_trusted_certificate /etc/ssl/certs/SUBDOMAIN.DOMAIN.com.cabundle;
    location / {
        proxy_pass http://localhost:PORT;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection 'upgrade';
        proxy_set_header Host $host;
        proxy_cache_bypass $http_upgrade;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}

========================================
NOTES
========================================

1. Har naye application ke liye unique subdomain use karein
2. Port conflicts avoid karne ke liye Dokploy mein automatic port assignment use karein
3. SSL certificates renew karna zaruri hai (90 days - Let's Encrypt)
4. Regular backups lein: nginx configs aur SSL certificates ki
5. Application logs regularly check karein for issues
6. Production environment mein test karne se pehle development/staging environment use karein

========================================
END OF GUIDE
========================================

Created: November 10, 2025
Last Updated: November 10, 2025
Author: Khizar's Deployment Documentation

# VPS Deployment Guide (Realtime Streaming Website)

Yeh guide 9-step VPS plan ko directly is project ke liye implement karta hai.

## 1) Use-case decide karo

Is project ka primary use-case:
- Video player app host karna
- Proxy endpoint (`/proxy`) serve karna for blocked video sources

## 2) Provider choose karo

Common options:
- DigitalOcean
- AWS Lightsail
- Hetzner
- Linode
- Contabo

Selection checklist:
- Data center location users ke paas ho
- Stable bandwidth + monthly transfer limits
- Managed backups/support requirement

## 3) Server specs select karo

Recommended starting spec:
- 2 vCPU
- 2 GB RAM
- 40+ GB SSD
- Ubuntu 22.04 LTS

## 4) Secure initial setup

### Server update
```bash
sudo apt update && sudo apt upgrade -y
```

### Non-root sudo user (optional but recommended)
```bash
sudo adduser deploy
sudo usermod -aG sudo deploy
```

### SSH hardening
1. SSH key login enable karo
2. `/etc/ssh/sshd_config` me:
   - `PasswordAuthentication no`
   - `PermitRootLogin no`
3. SSH restart:
```bash
sudo systemctl restart ssh
```

### Firewall enable
```bash
sudo ufw allow OpenSSH
sudo ufw allow 80
sudo ufw allow 443
sudo ufw enable
```

## 5) Required software stack install karo

### Node.js LTS + Nginx
```bash
curl -fsSL https://deb.nodesource.com/setup_lts.x | sudo -E bash -
sudo apt install -y nodejs nginx
```

### PM2 (process manager)
```bash
sudo npm install -g pm2
```

## 6) Domain aur SSL configure karo

1. Domain A record ko VPS public IP par point karo.
2. Nginx site config banao:
   ```bash
   sudo nano /etc/nginx/sites-available/streamflow
   ```
3. Basic reverse-proxy config:
   ```nginx
   server {
       listen 80;
       server_name your-domain.com www.your-domain.com;

       location / {
           proxy_pass http://127.0.0.1:4000;
           proxy_http_version 1.1;
           proxy_set_header Host $host;
           proxy_set_header X-Real-IP $remote_addr;
           proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
           proxy_set_header X-Forwarded-Proto $scheme;
       }
   }
   ```
4. Enable + reload:
   ```bash
   sudo ln -s /etc/nginx/sites-available/streamflow /etc/nginx/sites-enabled/
   sudo nginx -t
   sudo systemctl reload nginx
   ```
5. SSL:
   ```bash
   sudo apt install -y certbot python3-certbot-nginx
   sudo certbot --nginx -d your-domain.com -d www.your-domain.com
   ```

## 7) Deployment setup karo

### App deploy
```bash
git clone https://github.com/vajid-ali-ansari/Realtime-Streaming-website.git
cd Realtime-Streaming-website
```

### App run with PM2
```bash
pm2 start server.js --name streamflow
pm2 save
pm2 startup
```

### Environment variables (if needed)
`.env` add karo aur `server.js` me consume karo agar future configs (port, allowed origins, rate limits) manage karne hain.

## 8) Monitoring + backup enable karo

- PM2 monitoring: `pm2 monit`
- Logs: `pm2 logs streamflow`
- Uptime monitoring tools: UptimeRobot / Better Stack
- VPS snapshot ya automated provider backups weekly enable karo

## 9) Performance + security hardening

- `fail2ban` install and enable
- Nginx request rate limiting apply karo
- Automatic security updates:
  ```bash
  sudo apt install -y unattended-upgrades
  sudo dpkg-reconfigure --priority=low unattended-upgrades
  ```
- Monthly patch cycle maintain karo (`apt update && apt upgrade`)

---

## Quick Validation Checklist

- Domain HTTPS pe open ho raha hai
- `http://your-domain.com/proxy?url=...` work kar raha hai
- PM2 process restart ke baad bhi auto-start ho raha hai
- Firewall sirf required ports expose kar raha hai

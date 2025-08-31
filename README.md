# 🌐 PeerTube Complete Server Setup & Configuration Guide

This document combines two aspects of PeerTube setup:
1. **Repository usage** → Dynamically generating secure `production.yaml` using `peertube-server-setup`.
2. **Server setup** → Installing PeerTube from scratch on Ubuntu, configuring PostgreSQL, Redis, Nginx, and systemd.

---

## 📋 Requirements
- Ubuntu 22.04 or 24.04 (fresh server recommended)
- Root or sudo access
- A domain name (e.g., `videos.example.com`)
- Node.js 20 + Corepack/Yarn
- PostgreSQL 14+
- Redis
- ffmpeg 4.1+
- Nginx + Certbot (for HTTPS)
- systemd (to run PeerTube service)

---

# Part 1: 🧩 Repository — Dynamic `production.yaml` Generator

This repo (`peertube-server-setup`) safely stores **PeerTube** server configuration templates and generates the real `production.yaml` file dynamically — without exposing secrets to Git.

## 🚀 Why this repo?
- Securely document server setup.
- Keep reusable templates.
- Dynamically generate `production.yaml` with your chosen parameters (domain, HTTPS, DB, SMTP, languages, resolutions…), while automatically backing up the old one.

## 📦 Repository Contents
```
.
├─ .gitignore                 # Excludes sensitive files from Git
├─ README.md                  # This file
├─ config/
│  └─ default.yaml            # Safe default config template
└─ generate_production_yaml.py# Script to dynamically generate production.yaml
```

### Ignored (sensitive) files
- `config/production.yaml` (DB secrets, API keys)
- `storage/`, `logs/`, `versions/`
- SSL certificates: `*.crt`, `*.key`, `*.pem`
- Database dumps: `*.sql`, `*.dump`
- Temporary dev envs: `venv/`, `.venv/`, `node_modules/`

---

## ⚡️ Quickstart with the repo
1) Clone repo:
```bash
cd /var/www
sudo git clone https://github.com/<you>/peertube-server-setup.git
cd peertube-server-setup
```

2) Make script executable:
```bash
sudo chmod +x generate_production_yaml.py
```

3) Generate `production.yaml` (example):
```bash
sudo ./generate_production_yaml.py   --domain 127.0.0.1   --https false --web-port 9000   --db-host 127.0.0.1 --db-port 5432 --db-user peertube --db-pass 'peertube'   --instance-name "PeerTube" --instance-desc "Local instance (no domain, no SMTP)"   --languages en,de,ar   --resolutions 720p,1080p   --out /var/www/peertube/config/production.yaml
```

4) Restart PeerTube service:
```bash
sudo systemctl restart peertube
sudo systemctl status peertube --no-pager -n 20
```

### Key script options
See `--help` for full list.

| Option | Type/Value | Description |
|--------|------------|-------------|
| `--domain` | string | Public hostname or IP |
| `--https` | true/false | Behind HTTPS or not |
| `--db-user`/`--db-pass` | string | PostgreSQL credentials |
| `--smtp-*` | string/number | SMTP config |
| `--languages` | CSV list | UI languages |
| `--resolutions` | CSV list | Enabled video resolutions |
| `--out` | path | Output for `production.yaml` |

---

# Part 2: 🚀 Full PeerTube Server Setup

This section covers installing PeerTube from scratch.

## 🛠 Step 1: Update your system
```bash
sudo apt update && sudo apt upgrade -y
```

## 🛠 Step 2: Install dependencies
```bash
sudo apt install -y curl wget gnupg lsb-release unzip git vim
curl -fsSL https://deb.nodesource.com/setup_20.x | sudo -E bash -
sudo apt install -y nodejs
sudo corepack enable
sudo apt install -y postgresql postgresql-contrib redis-server ffmpeg
sudo apt install -y nginx certbot python3-certbot-nginx
```

## 🛠 Step 3: Create PeerTube user
```bash
sudo adduser --disabled-password --gecos "" peertube
```

## 🛠 Step 4: Setup database
```bash
sudo -u postgres createuser -P peertube
sudo -u postgres createdb -O peertube peertube
```

## 🛠 Step 5: Download PeerTube
```bash
cd /var/www
sudo -u peertube git clone -b production https://github.com/Chocobozzz/PeerTube.git peertube
cd peertube
sudo -u peertube yarn install --production --pure-lockfile
```

## 🛠 Step 6: Generate production.yaml
Use `peertube-server-setup` repo (Part 1).

## 🛠 Step 7: Configure Nginx
File: `/etc/nginx/sites-available/peertube`
```nginx
server {
  server_name videos.example.com;
  listen 80;
  listen [::]:80;

  location / {
    proxy_pass http://127.0.0.1:9000;
    proxy_http_version 1.1;
    proxy_set_header Host $host;
    proxy_set_header X-Real-IP $remote_addr;
    proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    proxy_set_header X-Forwarded-Proto $scheme;
    proxy_connect_timeout 600s;
    proxy_send_timeout    600s;
    proxy_read_timeout    600s;
    send_timeout          600s;
  }
}
```
Enable site & reload:
```bash
sudo ln -s /etc/nginx/sites-available/peertube /etc/nginx/sites-enabled/
sudo nginx -t && sudo systemctl reload nginx
```
Enable HTTPS:
```bash
sudo certbot --nginx -d videos.example.com
```

## 🛠 Step 8: systemd service
File: `/etc/systemd/system/peertube.service`
```ini
[Unit]
Description=PeerTube
After=postgresql.service redis-server.service

[Service]
User=peertube
WorkingDirectory=/var/www/peertube
Environment=NODE_ENV=production
ExecStart=/usr/bin/node dist/server
Restart=always

[Install]
WantedBy=multi-user.target
```
Enable & start:
```bash
sudo systemctl daemon-reload
sudo systemctl enable peertube
sudo systemctl start peertube
```

---

# ✅ Final Check
- Visit `https://videos.example.com`
- Login with admin account (first startup)
- Upload a test video 🎬

---

# 🔧 Maintenance
- Restart: `sudo systemctl restart peertube`
- Logs: `journalctl -u peertube -n 50 --no-pager`
- Update PeerTube:
```bash
cd /var/www/peertube
sudo -u peertube git pull
sudo -u peertube yarn install --production --pure-lockfile
```

---

# 🛡 Security Notes
- Never commit `production.yaml` to Git.
- Set file perms: `chmod 600 config/production.yaml`, owner `peertube:peertube`.
- Expose only Nginx to the internet.

---

# 🧯 Troubleshooting
- **DB error 28P01:** check DB credentials, test with `psql`.
- **504 Gateway Timeout:** ensure PeerTube is running, adjust Nginx proxy timeouts.
- **Videos not playing:** check hostname/HTTPS match, ffmpeg config.

---

# 📄 License
MIT (or your choice)

---

# ⚡ One-liner: Generate config + restart service

For convenience, you can generate a new `production.yaml` **and restart PeerTube in one command**:

```bash
sudo ./generate_production_yaml.py   --domain videos.example.com   --https true --web-port 9000   --db-host 127.0.0.1 --db-port 5432 --db-user peertube --db-pass 'CHANGE_ME'   --instance-name "MyTube" --instance-desc "Public PeerTube instance"   --languages en,de,ar --resolutions 720p,1080p   --out /var/www/peertube/config/production.yaml && sudo systemctl restart peertube && sudo systemctl status peertube --no-pager -n 20
```

This way, each time you adjust parameters, your instance will be updated and restarted automatically.

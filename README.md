# 🧩 PeerTube Server Setup — Dynamic `production.yaml` Generator

> A repository to safely store **PeerTube** server configurations and dynamically generate the real `production.yaml` file for any server/channel — without exposing secrets to Git.

## 🚀 Why this repo?
- Document server setup in a secure way.
- Keep reusable configuration templates.
- Dynamically generate the final `production.yaml` based on your options (domain, HTTPS, DB, SMTP, languages, resolutions…), with automatic backup of the previous file.

---

## 📦 Repository Contents
```
.
├─ .gitignore                 # Excludes sensitive files from Git
├─ README.md                  # This file
├─ config/
│  └─ default.yaml            # Safe default config template
└─ generate_production_yaml.py# Script to dynamically generate production.yaml
```

### What is *not* tracked by Git (thanks to `.gitignore`)
- `config/production.yaml` (secrets: DB password, API keys…)
- `storage/`, `logs/`, `versions/`, `current/`
- SSL certificates: `*.crt`, `*.key`, `*.pem`
- Database dumps: `*.sql`, `*.dump`
- Temporary dev environments: `venv/`, `.venv/`, `node_modules/`, …

> The idea: you can publish/share this repo safely; secrets remain on the server only.

---

## 🧰 Requirements (on server)
- **Ubuntu 22.04/24.04** (or other modern Linux)
- **Node.js 20** + **Corepack/Yarn**
- **PostgreSQL 14+**
- **Redis**
- **ffmpeg 4.1+**
- **Nginx** (reverse proxy and/or HTTPS)
- **systemd** to manage `peertube` service

> First-time setup: follow the official PeerTube guide, then use this repo to easily generate/update `production.yaml`.

---

## ⚡️ Quickstart
1) **Clone the repo** on your server:
```bash
cd /var/www
sudo git clone https://github.com/<you>/peertube-server-setup.git
cd peertube-server-setup
```

2) **Make script executable**:
```bash
sudo chmod +x generate_production_yaml.py
```

3) **Generate production.yaml** (local example without HTTPS):
```bash
sudo ./generate_production_yaml.py   --domain 127.0.0.1   --https false --web-port 9000   --db-host 127.0.0.1 --db-port 5432 --db-user peertube --db-pass 'peertube'   --instance-name "PeerTube" --instance-desc "Local instance (no domain, no SMTP)"   --languages en,de,ar   --resolutions 720p,1080p   --out /var/www/peertube/config/production.yaml
```

> The script automatically **creates a backup** if `production.yaml` exists, writes the new one, and can set **permissions** (chmod 600) and **ownership** (`peertube:peertube`).

4) **Restart service**:
```bash
sudo systemctl restart peertube
sudo systemctl status peertube --no-pager -n 20
```

---

## ⚙️ Key script options
> Use `--help` for the full list.

| Option | Type/Value | Description |
|---|---|---|
| `--domain` | string | Public hostname (e.g. `videos.example.com`) or IP |
| `--https` | `true/false` | Whether behind HTTPS on Nginx |
| `--web-port` | number | Listening port (default 9000) |
| `--db-host`/`--db-port` | string/number | PostgreSQL host and port |
| `--db-user`/`--db-pass` | string | DB username and password |
| `--db-name` | string | Custom DB name (optional) |
| `--db-ssl` | `true/false` | Enable SSL for DB |
| `--smtp-*` | string/number | SMTP host/port/user/pass |
| `--from-address` | string | Default sender email |
| `--instance-name`/`--instance-desc` | string | Instance name and description |
| `--languages` | CSV list | Languages (e.g. `en,de,ar`) |
| `--resolutions` | CSV list | Enabled resolutions (e.g. `720p,1080p,2160p`) |
| `--hls-enabled` | `true/false` | Enable HLS (recommended) |
| `--web-videos-enabled` | `true/false` | Enable Web Videos (uses more space) |
| `--transcoding-keep-original` | `true/false` | Keep original file after transcoding |
| `--video-quota*` | string | User upload quota (`-1` = unlimited, `50GB` etc) |
| `--out` | path | Output path for `production.yaml` |

---

## 🧪 Example Scenarios
### 1) Local server, no HTTPS
See quickstart example.

### 2) With domain + HTTPS
```bash
sudo ./generate_production_yaml.py   --domain videos.example.com   --https true --web-port 9000   --db-host 127.0.0.1 --db-port 5432 --db-user peertube --db-pass 'CHANGE_ME'   --instance-name "MyTube" --instance-desc "Public instance"   --languages en,de,ar --resolutions 720p,1080p,2160p   --out /var/www/peertube/config/production.yaml
```

### 3) With SMTP enabled
Add for email:
```bash
  --smtp-host smtp.mailprovider.com --smtp-port 587 \
  --smtp-user 'noreply@yourdomain.com' --smtp-pass 'APP_PASSWORD' \
  --smtp-tls false --smtp-starttls-disable false \
  --from-address "PeerTube <noreply@yourdomain.com>"
```

---

## 🌐 Nginx Setup
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

Enable & reload:
```bash
sudo ln -s /etc/nginx/sites-available/peertube /etc/nginx/sites-enabled/
sudo nginx -t && sudo systemctl reload nginx
```

---

## 🛡️ Security tips
- Never commit `production.yaml` (ignored by Git).
- Restrict file perms: `chmod 600`, owner `peertube:peertube`.
- Keep `secrets.peertube` stable once generated.
- Expose only Nginx port to public.

---

## 🧯 Troubleshooting
- **DB error 28P01 (auth failure):** check DB user/pass, try `psql -h 127.0.0.1 -U peertube -d peertube`.
- **504 Gateway Timeout:** ensure PeerTube is running (`ss -lntp | grep ':9000'`), adjust Nginx timeouts, check logs `journalctl -u peertube`.
- **Videos not playing:** ensure `webserver.hostname` matches domain, check HTTPS value, check ffmpeg/transcoding.

---

## 🤝 Contributing
- Open **Issues** for bugs/ideas.
- Submit **Pull Requests** for improvements.

## 📄 License
Add a license file (MIT recommended).

---

## 💬 FAQ
- **Can I use the same script for multiple servers/channels?** Yes, pass different args.
- **Where to put `location /` block?** Inside Nginx site config `/etc/nginx/sites-available/`.
- **How to make script executable?** `sudo chmod +x generate_production_yaml.py` and run with `sudo ./generate_production_yaml.py ...`.

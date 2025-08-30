# PeerTube Server Setup

This repository contains **safe configuration files** and setup notes for my PeerTube server.  
It is meant for **backup and documentation purposes** only.

## 📂 Included
- `.gitignore` → ensures sensitive files are not tracked
- `config/default.yaml` → default PeerTube configuration template (safe to share)

## ❌ Excluded (via .gitignore)
- `config/production.yaml` → contains secrets (DB password, API keys)
- `storage/` → uploaded videos, thumbnails, transcoded files
- `logs/` → runtime logs
- `versions/` and `current/` → release symlinks and folders
- SSL certificates (`*.crt`, `*.key`, `*.pem`)
- Database dumps (`*.sql`, `*.dump`)

## 🚀 Purpose
- Keep track of safe configuration files
- Document server setup
- Provide reproducibility (without exposing private data)

## 🔐 Note
This repo **does not include any sensitive data**.  
Secrets are stored only on the server and are excluded from version control.

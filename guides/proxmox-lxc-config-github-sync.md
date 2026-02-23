# 🚀 Proxmox LXC → GitHub Config Auto-Sync

Automatically back up a Proxmox LXC service configuration folder to GitHub using:

- git  
- rsync  
- cron  

---

## ⚠️ Disclaimer

This setup reflects my personal homelab configuration and workflow.  
All scripts, paths, structure, and automation examples are based on what works in my environment.

Your setup may differ depending on:

- Distribution and Proxmox version
- LXC configuration
- Service paths
- User permissions
- Network and firewall rules

You are responsible for understanding and validating the commands before running them in your own environment.

While this guide aims to be safe and production-ready, I am not responsible for:

- Data loss  
- Misconfigured services  
- Accidental deletions caused by incorrect path changes  
- Security issues caused by exposing secrets  

Always test changes in a controlled manner and ensure you have backups before applying automation involving `rsync --delete` or git history rewrites.

> ⚠️ **Note**
>---
> This guide uses my Homepage config folder as an example:
>
> - Live config: `/opt/homepage/config`
> - Repo mirror: `files/homepage/`
>
> `homepage` is just the example service name.  
> Replace:
>
> - `homepage`
> - `/opt/homepage/config`
> - `files/homepage`
>
> with your own service name and config path.

---

# 📚 Table of Contents

- [Prerequisites](#prerequisites)
- [Overview](#overview)
- [1. Install Required Packages](#1-install-required-packages)
- [2. Create a GitHub Repo](#2-create-a-github-repo)
- [3. Setup SSH Access to GitHub](#3-setup-ssh-access-to-github)
- [4. Clone Your Repository](#4-clone-your-repository)
- [5. Add .gitignore](#5-add-gitignore)
- [6. First Config Sync](#6-first-config-sync)
- [7. Commit and Push](#7-commit-and-push)
- [8. Create Auto-Sync Script](#8-create-auto-sync-script)
- [9. Schedule Daily Sync](#9-schedule-daily-sync)
- [Manual Sync](#manual-sync)
- [Reuse Template](#reuse-template)
- [Security Notes](#security-notes)
- [Troubleshooting](#troubleshooting)

---

# Prerequisites

- You are running commands inside the LXC as root (or via sudo).
- Your service config lives in a folder (example: `/opt/homepage/config`).
- You decided where your Git repo will live (example: `/opt/homelab`).

---

# Overview

### Live config location (example)

```
/opt/homepage/config
```

### GitHub repo structure (example)

```
homelab/
├── guides/
└── files/
    └── homepage/
```

### Behavior

- Live config stays untouched  
- GitHub mirrors the config folder  
- Sync runs once per day  
- Secrets are excluded  

---

# 1. Install Required Packages

Inside the LXC:

```bash
apt update
apt install -y git rsync
```

---

# 2. Create a GitHub Repo

From the GitHub UI:

1. Click **New repository**
2. Choose a name (example: `homelab`)
3. Choose Public or Private
4. Do NOT commit secrets to a public repo
5. Click **Create repository**
6. Copy the SSH URL

Example:

```
git@github.com:YOUR_USERNAME/homelab.git
```

---

# 3. Setup SSH Access to GitHub

## Generate SSH key (inside the LXC)

```bash
ssh-keygen -t ed25519 -C "homelab-sync"
```

Press Enter for defaults.

## Show public key

```bash
cat ~/.ssh/id_ed25519.pub
```

## Add key to GitHub (Deploy Key)

Go to:

```
Repo → Settings → Deploy keys → Add deploy key
```

- Paste the key  
- Enable **Write access**

## Test connection

```bash
ssh -T git@github.com
```

You should see a successful authentication message.

---

# 4. Clone Your Repository

```bash
cd /opt
git clone git@github.com:YOUR_USERNAME/homelab.git
mkdir -p /opt/homelab/files/homepage
```

---

# 5. Add .gitignore

Create `.gitignore` in repo root:

```bash
nano /opt/homelab/.gitignore
```

Add:

```
# secrets
secrets.yaml
secrets.yml
secrets.env
.env

# logs
logs/
*.log
```

Save and exit.

---

# 6. First Config Sync

```bash
rsync -a --delete \
  --exclude=".git" \
  --exclude="secrets.yaml" --exclude="secrets.yml" --exclude="secrets.env" --exclude=".env" \
  --exclude="logs/" --exclude="*.log" \
  /opt/homepage/config/ /opt/homelab/files/homepage/
```

---

# 7. Commit and Push

```bash
cd /opt/homelab
git add -A
git commit -m "Add Homepage config"
git branch -M main
git push -u origin main
```

Verify secrets are NOT included:

```bash
git ls-files | grep -E "(secrets\.ya?ml|secrets\.env|\.env)$" || true
```

---

# 8. Create Auto-Sync Script

Create the script:

```bash
nano /usr/local/bin/homelab-homepage-sync.sh
```

Paste:

```bash
#!/bin/bash
set -e

# ===== Configuration =====
SRC="/opt/homepage/config"
DST="/opt/homelab/files/homepage"
REPO="/opt/homelab"

# ===== Safety Guard =====
if [[ "$DST" == "/opt/homelab" || "$DST" == "/opt/homelab/" || "$DST" == "/opt/homelab/files" || "$DST" == "/opt/homelab/files/" ]]; then
  echo "Refusing to run: DST path is too broad: $DST" >&2
  exit 1
fi

# ===== Sync Config Files =====
rsync -a --delete \
  --exclude=".git" \
  --exclude="secrets.yaml" --exclude="secrets.yml" --exclude="secrets.env" --exclude=".env" \
  --exclude="logs/" --exclude="*.log" \
  "$SRC/" "$DST/"

# ===== Commit & Push If Changed =====
cd "$REPO"

if [[ -n "$(git status --porcelain)" ]]; then
  git add -A
  git commit -m "Auto-sync Homepage config: $(date -Iseconds)"
  git push
fi
```

Make it executable:

```bash
chmod +x /usr/local/bin/homelab-homepage-sync.sh
```

Test manually:

```bash
/usr/local/bin/homelab-homepage-sync.sh
```

---

# 9. Schedule Daily Sync

Open cron:

```bash
crontab -e
```

Add:

```cron
0 9 * * * /usr/local/bin/homelab-homepage-sync.sh >/dev/null 2>&1
```

Verify:

```bash
crontab -l
```

---

# Manual Sync

```bash
/usr/local/bin/homelab-homepage-sync.sh
```

---

# Reuse Template

For another service, change:

```bash
SRC="/path/to/config"
DST="/opt/homelab/files/service-name"
REPO="/opt/homelab"
```

Also create destination folder once:

```bash
mkdir -p /opt/homelab/files/service-name
```

Everything else stays the same.

---

# Security Notes

- Never commit passwords, tokens, or API keys to a public repo.
- Always exclude secrets and logs.
- If secrets were pushed accidentally: rotate them and clean git history.

---

# Troubleshooting

### SSH Permission Denied
Your deploy key is missing or does not have write access.

### Embedded Git Repository Warning
You accidentally synced a `.git` folder into `files/<service>/`.  
Ensure `.git` is excluded in rsync.

### Cron Runs But No Commit Happens
The script only commits if changes are detected. That is expected behavior.

# CI/CD & AWS EC2 Deployment Guide

This document explains how to set up the GitHub Actions CI/CD pipeline and deploy **StayEaseInn** to an AWS EC2 instance.

---

## Architecture Overview

```
GitHub Push → CI Workflow (syntax check + composer) → CD Workflow (SSH → EC2)
```

```
.github/
  workflows/
    ci.yml          ← Runs on every push/PR: PHP syntax + Composer check
    deploy.yml      ← Runs on push to main: CI gate → SSH deploy to EC2
scripts/
  setup-ec2.sh     ← One-time EC2 server setup script
menus/
  .env.example      ← Safe template (committed to Git)
  .env              ← Real secrets (NOT in Git, written by deploy pipeline)
```

---

## Step 1: Launch Your AWS EC2 Instance

### Create the Instance

1. Go to **AWS Console → EC2 → Launch Instance**
2. Choose settings:
   - **AMI:** Ubuntu Server 22.04 LTS (HVM), SSD Volume Type
   - **Instance Type:** `t2.micro` (free tier) or `t3.small` recommended
   - **Storage:** 20 GB gp3 (minimum)
   - **Key Pair:** Create or select an existing key pair (`.pem` file)

3. **Security Group (Inbound Rules):**

| Type  | Protocol | Port | Source            |
|-------|----------|------|-------------------|
| SSH   | TCP      | 22   | Your IP (or 0.0.0.0/0) |
| HTTP  | TCP      | 80   | 0.0.0.0/0         |
| HTTPS | TCP      | 443  | 0.0.0.0/0         |

> 💡 For tighter security, restrict port 22 to your own IP. GitHub Actions also needs port 22 access — either allow 0.0.0.0/0 or add GitHub's IP ranges.

4. Launch the instance and note your **Public IPv4 address** (e.g. `54.123.x.x`).

---

## Step 2: Run the One-Time EC2 Setup Script

SSH into your EC2 instance using your `.pem` key:

```bash
# From your local machine:
chmod 400 your-key.pem
scp scripts/setup-ec2.sh ubuntu@YOUR_EC2_IP:/home/ubuntu/
ssh -i your-key.pem ubuntu@YOUR_EC2_IP
```

On the EC2 instance:

```bash
# Edit the script to set your domain (or leave blank for IP-only):
nano /home/ubuntu/setup-ec2.sh

# Run it as root:
sudo bash /home/ubuntu/setup-ec2.sh
```

> ⚠️ **Save the DB password** printed at the end!

---

## Step 3: Create the GitHub Actions Deploy Key

On your local machine (or on the EC2 instance), generate a dedicated SSH key pair:

```bash
ssh-keygen -t ed25519 -C "github-actions-deploy" -f ~/.ssh/github_deploy_key -N ""
```

This creates:
- `~/.ssh/github_deploy_key` → **Private key** (goes into GitHub Secrets)
- `~/.ssh/github_deploy_key.pub` → **Public key** (goes on the EC2 instance)

### Add the Public Key to EC2

SSH into the EC2 instance:

```bash
# On EC2, as the deploy user:
cat >> /home/deploy/.ssh/authorized_keys << 'EOF'
PASTE_YOUR_PUBLIC_KEY_CONTENTS_HERE
EOF
```

### Test the SSH Connection

```bash
ssh -i ~/.ssh/github_deploy_key deploy@YOUR_EC2_IP
```

---

## Step 4: Add GitHub Secrets

Go to your GitHub repo → **Settings → Secrets and variables → Actions → New repository secret**

| Secret Name           | Value                        | Example                    |
|-----------------------|------------------------------|----------------------------|
| `EC2_HOST`            | EC2 Public IP or domain      | `54.123.45.67`             |
| `EC2_USERNAME`        | SSH user on EC2              | `deploy`                   |
| `EC2_SSH_PRIVATE_KEY` | Contents of your private key | `-----BEGIN OPENSSH...`    |
| `EC2_DEPLOY_PATH`     | App path on server           | `/var/www/html/restaurant-app` |
| `DB_HOST`             | MySQL host                   | `localhost`                |
| `DB_USERNAME`         | MySQL user                   | `stayease_user`            |
| `DB_PASSWORD`         | MySQL password               | *(from setup script)*      |
| `DB_NAME`             | Database name                | `restaurant_db`            |
| `STRIPE_SECRET_KEY`   | Stripe secret key            | `sk_live_...`              |

---

## Step 5: Push to GitHub to Trigger the Pipeline

```bash
# Initialize git (if not already done):
git init
git remote add origin https://github.com/Knoweb/restaurant-app.git

# Stage, commit, and push:
git add .
git commit -m "feat: migrate CI/CD pipeline to AWS EC2"
git push -u origin main
```

---

## Step 6: Monitor the Pipeline

1. Go to your GitHub repo → **Actions** tab
2. You'll see two workflows running:
   - **CI — PHP Syntax & Dependency Check** ✅
   - **CD — Deploy to AWS EC2** ✅
3. Click any workflow run to see detailed logs

---

## Workflow Summary

### `ci.yml` — Runs on every push & PR

| Step                  | What it does                              |
|-----------------------|-------------------------------------------|
| PHP Syntax Check      | Runs `php -l` on every `.php` file        |
| Composer Validate     | Validates `composer.json`                 |
| Composer Install      | Installs all dependencies                 |

### `deploy.yml` — Runs on push to `main` only

| Step                  | What it does                              |
|-----------------------|-------------------------------------------|
| CI Gate               | Runs PHP syntax check before deploying    |
| `git pull`            | Pulls latest code on the EC2 instance    |
| Write `.env`          | Creates `.env` from GitHub Secrets        |
| `composer install`    | Installs production dependencies          |
| Set permissions       | `chown www-data`, `chmod 755/775`         |
| Reload Apache         | `systemctl reload apache2`                |

---

## Troubleshooting

### SSH Connection Failed

```bash
# Test manually from your machine:
ssh -i ~/.ssh/github_deploy_key -p 22 deploy@YOUR_EC2_IP

# On EC2, check authorized_keys:
cat /home/deploy/.ssh/authorized_keys

# Verify EC2 Security Group allows port 22
```

### Composer Install Fails on Server

```bash
# SSH in and run manually:
cd /var/www/html/restaurant-app/menus
composer install --no-dev -v
```

### Apache 403 / 404 Error

```bash
# Check Apache error log:
sudo tail -f /var/log/apache2/restaurant-app-error.log

# Check mod_rewrite is enabled:
sudo a2enmod rewrite && sudo systemctl reload apache2
```

### Database Connection Error

```bash
# Verify .env was written correctly:
cat /var/www/html/restaurant-app/menus/.env

# Test MySQL connection:
mysql -u stayease_user -p restaurant_db
```

### `.env` Not Updating

The `deploy.yml` workflow overwrites `.env` on every deployment from GitHub Secrets.
Check that all `DB_*` and `STRIPE_SECRET_KEY` secrets are set in your GitHub repo settings.

---

## Security Notes

- ✅ `.env` is **never** committed to Git (in `.gitignore`)
- ✅ `.env` is written on the server from GitHub Secrets at every deploy
- ✅ `menus/.env` has `chmod 600` (only readable by owner)
- ✅ Apache blocks direct access to `.env`, `.sql`, `.log` files
- ✅ Dedicated `deploy` user with minimal sudo permissions
- ✅ EC2 Security Group restricts unnecessary ports
- ⚠️ Rotate your Stripe key if it was ever committed to Git history
- ⚠️ Use an Elastic IP on AWS if you want a permanent, static IP address

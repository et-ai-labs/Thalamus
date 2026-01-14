# Thalamus Combined Deployment Guide

Deploy both Thalamus Server and Web App on a single VM using Docker Compose and Cloudflare Tunnel.

## Prerequisites

- **Ubuntu/Debian VM** with root access
- **Docker and Docker Compose** installed
- **Cloudflare account** with a domain
- **Git** for cloning the repository

## 1. Install Docker

```bash
# Update system
sudo apt update && sudo apt upgrade -y

# Install Docker
curl -fsSL https://get.docker.com -o get-docker.sh
sudo sh get-docker.sh

# Add your user to docker group (avoid needing sudo)
sudo usermod -aG docker $USER
newgrp docker

# Verify installation
docker --version
docker compose version
```

## 2. Install Cloudflare Tunnel

```bash
# Add Cloudflare GPG key and repository
curl -fsSL https://pkg.cloudflare.com/cloudflare-main.gpg | sudo tee /usr/share/keyrings/cloudflare-main.gpg >/dev/null
echo "deb [signed-by=/usr/share/keyrings/cloudflare-main.gpg] https://pkg.cloudflare.com/cloudflared $(lsb_release -cs) main" | sudo tee /etc/apt/sources.list.d/cloudflared.list

# Install cloudflared
sudo apt update
sudo apt install -y cloudflared
```

### Authenticate with Cloudflare

```bash
cloudflared tunnel login
# This opens a browser - select your domain
```

### Create Tunnel

```bash
cloudflared tunnel create thalamus
# Save the Tunnel ID shown in output
```

### Configure Tunnel

Replace `TUNNEL_ID` with your actual tunnel ID and `YOUR_DOMAIN` with your domain (e.g., `thalamus.example.com`):

```bash
sudo mkdir -p /etc/cloudflared
sudo mkdir -p /root/.cloudflared

sudo tee /etc/cloudflared/config.yml << EOF
tunnel: TUNNEL_ID
credentials-file: /root/.cloudflared/TUNNEL_ID.json

ingress:
  # API routes
  - hostname: YOUR_DOMAIN
    path: /v1/*
    service: http://localhost:3000
    originRequest:
      noTLSVerify: true

  # S3/MinIO routes
  - hostname: YOUR_DOMAIN
    path: /s3/*
    service: http://localhost:9000
    originRequest:
      noTLSVerify: true

  # Web app (catch-all)
  - hostname: YOUR_DOMAIN
    service: http://localhost:8080
    originRequest:
      noTLSVerify: true

  # Fallback
  - service: http_status:404
EOF

# Copy credentials
sudo cp ~/.cloudflared/TUNNEL_ID.json /root/.cloudflared/
```

### Create DNS Record

```bash
cloudflared tunnel route dns thalamus YOUR_DOMAIN
```

### Start Tunnel Service

```bash
sudo cloudflared service install
sudo systemctl start cloudflared
sudo systemctl enable cloudflared
sudo systemctl status cloudflared
```

Your domain is now ready: `https://YOUR_DOMAIN`

## 3. Install and Authenticate GitHub CLI

Since the Thalamus repositories are private, we'll use the GitHub CLI for authentication:

```bash
# Install GitHub CLI
curl -fsSL https://cli.github.com/packages/githubcli-archive-keyring.gpg | sudo dd of=/usr/share/keyrings/githubcli-archive-keyring.gpg
sudo chmod go+r /usr/share/keyrings/githubcli-archive-keyring.gpg
echo "deb [arch=$(dpkg --print-architecture) signed-by=/usr/share/keyrings/githubcli-archive-keyring.gpg] https://cli.github.com/packages stable main" | sudo tee /etc/apt/sources.list.d/github-cli.list > /dev/null
sudo apt update
sudo apt install -y gh

# Authenticate with GitHub
gh auth login
```

Follow the prompts:
1. Choose **GitHub.com**
2. Choose **HTTPS** as preferred protocol
3. Authenticate with your **token** or via **web browser**
4. Grant access to private repositories

Verify authentication:
```bash
gh auth status
```

## 4. Clone and Configure Repository

### Clone Repository

```bash
cd /opt
sudo gh repo clone Et-Ai-Labs/Thalamus
sudo chown -R $USER:$USER Thalamus
cd Thalamus

# Initialize and clone submodules
git submodule update --init --recursive
```

### Create Environment File

Generate a secure master secret:

```bash
SECRET=$(openssl rand -hex 32)
echo "Your master secret: $SECRET"
echo "IMPORTANT: Back up this secret!"
```

Create `.env` file (replace `YOUR_DOMAIN` with your actual domain):

```bash
cat > .env << EOF
# Required: Master encryption secret
THALAMUS_MASTER_SECRET=$SECRET

# Required: Public URL for S3 files
S3_PUBLIC_URL=https://localhost:9000/thalamus

# Optional: GitHub OAuth (create at https://github.com/settings/developers)
# GITHUB_CLIENT_ID=
# GITHUB_CLIENT_SECRET=
# GITHUB_REDIRECT_URL=https://YOUR_DOMAIN/v1/connect/github/callback

# Optional: Voice features (get key at https://elevenlabs.io)
# ELEVENLABS_API_KEY=

# Optional: Analytics (PostHog)
# POSTHOG_API_KEY=

# Optional: Revenue Cat (Stripe)
# REVENUE_CAT_STRIPE=
EOF
```

## 5. Deploy Services

### Build and Start

```bash
# Build all services (first time only)
docker compose build

# Start all services
docker compose up -d

# Run database migrations
docker compose run --rm migrate

# Verify all services are running
docker compose ps
```

Expected output - all services should show "Up" status:
```
NAME                STATUS
postgres            Up (healthy)
redis               Up (healthy)
minio               Up (healthy)
server              Up (healthy)
app                 Up
```

### Check Logs

```bash
# View all logs
docker compose logs -f

# View specific service logs
docker compose logs -f server
docker compose logs -f app
```

## 6. Verify Deployment

Visit your domain:

- **Web App**: `https://YOUR_DOMAIN` - Should show the Thalamus app
- **API Health**: `https://YOUR_DOMAIN/v1/version` - Should return JSON with version info
- **Cloudflare Tunnel**: Check `sudo journalctl -u cloudflared -f` for tunnel status

## Updating Your Deployment

### Update to Latest Version

```bash
cd /opt/Thalamus

# Pull latest changes from all repositories
git pull
git submodule update --remote --merge

# Rebuild and restart services
docker compose build
docker compose up -d

# Run any new migrations
docker compose run --rm migrate

# Verify services are healthy
docker compose ps
```

### Update Only Specific Component

**Update Server only:**
```bash
cd /opt/Thalamus
cd Server && git pull && cd ..
docker compose build server
docker compose up -d server
docker compose run --rm migrate
```

**Update App only:**
```bash
cd /opt/Thalamus
cd App && git pull && cd ..
docker compose build app
docker compose up -d app
```

### Rollback to Previous Version

```bash
cd /opt/Thalamus

# Find the commit hash you want to rollback to
git log --oneline

# Rollback to specific commit
git checkout <commit-hash>
git submodule update --init --recursive

# Rebuild and restart
docker compose build
docker compose up -d
docker compose run --rm migrate
```

## Maintenance Commands

### View Logs

```bash
cd /opt/Thalamus

# All services
docker compose logs -f

# Specific service (last 100 lines)
docker compose logs --tail=100 server

# Cloudflare Tunnel logs
sudo journalctl -u cloudflared -f
```

### Restart Services

```bash
cd /opt/Thalamus

# Restart all services
docker compose restart

# Restart specific service
docker compose restart server
docker compose restart app

# Restart Cloudflare Tunnel
sudo systemctl restart cloudflared
```

### Stop/Start Services

```bash
cd /opt/Thalamus

# Stop all services
docker compose down

# Start all services
docker compose up -d

# Stop but keep data
docker compose stop

# Start previously stopped services
docker compose start
```

### Database Backup

```bash
cd /opt/Thalamus

# Create backup
docker compose exec postgres pg_dump -U postgres thalamus > backup-$(date +%Y%m%d-%H%M%S).sql

# Restore from backup
cat backup-YYYYMMDD-HHMMSS.sql | docker compose exec -T postgres psql -U postgres thalamus
```

### Clean Up

```bash
# Remove stopped containers
docker compose down

# Remove everything including volumes (WARNING: deletes all data)
docker compose down -v

# Remove unused Docker images
docker image prune -a
```

## Architecture Overview

```
Internet → Cloudflare → Tunnel → VM
                                  ├─ localhost:8080 → app (nginx)
                                  ├─ localhost:3000 → server (Node.js)
                                  ├─ localhost:5432 → postgres (internal)
                                  ├─ localhost:6379 → redis (internal)
                                  └─ localhost:9000 → minio (internal)
```

| Service | Port | Access |
|---------|------|--------|
| Web App | 8080 | via tunnel (/) |
| API Server | 3000 | via tunnel (/v1/*) |
| MinIO | 9000 | via tunnel (/s3/*) |
| PostgreSQL | 5432 | internal only |
| Redis | 6379 | internal only |
| Metrics | 9090 | internal only |

**Security**: All services are internal except those exposed via Cloudflare Tunnel. No public ports are opened.

## Support

For issues and updates:
- **App**: https://github.com/Et-Ai-Labs/Thalamus-App
- **Server**: https://github.com/Et-Ai-Labs/Thalamus-Server
- **CLI**: https://github.com/Et-Ai-Labs/Thalamus-CLI

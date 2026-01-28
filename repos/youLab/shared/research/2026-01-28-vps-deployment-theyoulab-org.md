---
date: 2026-01-28T14:30:00-08:00
researcher: Claude (Opus 4.5)
git_commit: c5b6447f5fe1ab04cfd5cdac9dfef5d3c0ba7fdf
branch: ralph/ralph-wiggum-mvp
repository: YouLab
topic: "VPS Deployment for theyoulab.org with Cloudflare Tunnel"
tags: [research, deployment, cloudflare, docker, vps, production]
status: complete
last_updated: 2026-01-28
last_updated_by: Claude (Opus 4.5)
---

# Research: VPS Deployment for theyoulab.org with Cloudflare Tunnel

**Date**: 2026-01-28T14:30:00-08:00
**Researcher**: Claude (Opus 4.5)
**Git Commit**: c5b6447f5fe1ab04cfd5cdac9dfef5d3c0ba7fdf
**Branch**: ralph/ralph-wiggum-mvp
**Repository**: YouLab

## Research Question

How to deploy YouLab (Ralph stack) on an Ubuntu VPS and make it accessible via theyoulab.org using Cloudflare Tunnel? Looking for the simplest, most beginner-friendly approach.

## Summary

YouLab's Ralph stack consists of 3 services that need to be deployed:
1. **OpenWebUI** (frontend) - port 3000
2. **Ralph Server** (FastAPI backend) - port 8200
3. **Dolt** (MySQL-compatible database) - port 3307

The recommended deployment approach uses Docker Compose for all services and Cloudflare Tunnel for secure, firewall-friendly exposure to the internet. This approach requires no open inbound ports and provides free SSL/TLS and DDoS protection.

## Architecture Overview

```
Internet Users
      |
      v
theyoulab.org (Cloudflare DNS)
      |
      v
Cloudflare Edge (DDoS protection, SSL)
      |
      v
Cloudflare Tunnel (outbound connection)
      |
      v
Ubuntu VPS (localhost services)
  |-- OpenWebUI (port 3000) --> theyoulab.org
  |-- Ralph Server (port 8200) --> api.theyoulab.org (optional)
  |-- Dolt (port 3307) --> internal only
```

## Detailed Findings

### 1. Services to Deploy

#### OpenWebUI (Frontend)
- **Port**: 3000 (external) → 8080 (internal)
- **Image**: `ghcr.io/open-webui/open-webui:main`
- **Location in repo**: `OpenWebUI/open-webui/`
- **Docker compose**: `OpenWebUI/open-webui/docker-compose.yaml:11-28`
- **Purpose**: Web chat interface for users
- **Key config**: Needs `extra_hosts: host.docker.internal:host-gateway` to reach Ralph server

#### Ralph Server (Backend API)
- **Port**: 8200 (hardcoded at `src/ralph/server.py:352`)
- **Entry point**: `uv run ralph-server` (pyproject.toml:61)
- **Not containerized**: Currently runs natively with uvicorn
- **Dependencies**: OpenRouter API, Honcho, Dolt

#### Dolt (Database)
- **Port**: 3307 → 3306 (internal)
- **Image**: `dolthub/dolt-sql-server:latest`
- **Docker compose**: `docker-compose.yml:2-15`
- **Config files**: `config/dolt/config.yaml`, `config/dolt/init.sql`

### 2. Environment Variables Required

**Critical (must set for production)**:
```bash
# OpenRouter (LLM API)
RALPH_OPENROUTER_API_KEY=sk-or-v1-...  # Get from openrouter.ai

# Honcho (message persistence) - can use demo mode initially
RALPH_HONCHO_ENVIRONMENT=demo  # or "production" with API key
RALPH_HONCHO_API_KEY=hch-v2-... # Only needed for production

# Dolt (use defaults for Docker)
RALPH_DOLT_HOST=localhost  # or "dolt" if in same Docker network
RALPH_DOLT_PORT=3307
RALPH_DOLT_PASSWORD=devpassword  # Change for production!
```

**Optional but recommended**:
```bash
RALPH_OPENROUTER_MODEL=anthropic/claude-sonnet-4-20250514  # or preferred model
RALPH_USER_DATA_DIR=/data/ralph/users  # Per-user workspaces
RALPH_AGENT_WORKSPACE=/path/to/shared/codebase  # Optional shared workspace
```

### 3. Cloudflare Tunnel Setup

**Why Cloudflare Tunnel?**
- No open inbound ports needed (outbound connections only)
- Free SSL/TLS certificates
- Free DDoS protection
- Origin IP hidden from attackers
- Works behind NAT/firewalls
- Completely free tier

**Installation on Ubuntu**:
```bash
# Add Cloudflare repository
sudo mkdir -p --mode=0755 /usr/share/keyrings
curl -fsSL https://pkg.cloudflare.com/cloudflare-main.gpg | sudo tee /usr/share/keyrings/cloudflare-main.gpg >/dev/null
echo 'deb [signed-by=/usr/share/keyrings/cloudflare-main.gpg] https://pkg.cloudflare.com/cloudflared jammy main' | sudo tee /etc/apt/sources.list.d/cloudflared.list
sudo apt-get update && sudo apt-get install cloudflared

# Authenticate
cloudflared tunnel login  # Opens browser to select domain

# Create tunnel
cloudflared tunnel create youlab
```

**Tunnel Configuration** (`~/.cloudflared/config.yml`):
```yaml
tunnel: <TUNNEL-UUID>
credentials-file: /home/ubuntu/.cloudflared/<TUNNEL-UUID>.json

ingress:
  - hostname: theyoulab.org
    service: http://localhost:3000

  - hostname: api.theyoulab.org  # Optional separate API endpoint
    service: http://localhost:8200

  - service: http_status:404  # Required catch-all
```

**DNS routing**:
```bash
cloudflared tunnel route dns youlab theyoulab.org
cloudflared tunnel route dns youlab api.theyoulab.org
```

### 4. Production Docker Compose (Recommended)

Create a unified `docker-compose.prod.yml`:

```yaml
version: "3.9"

services:
  openwebui:
    image: ghcr.io/open-webui/open-webui:main
    container_name: openwebui
    ports:
      - "127.0.0.1:3000:8080"  # Localhost only
    volumes:
      - openwebui_data:/app/backend/data
    extra_hosts:
      - "host.docker.internal:host-gateway"
    environment:
      - WEBUI_SECRET_KEY=${WEBUI_SECRET_KEY:-changeme}
    restart: unless-stopped
    depends_on:
      - dolt

  ralph:
    build:
      context: .
      dockerfile: Dockerfile.ralph  # Need to create
    container_name: ralph
    ports:
      - "127.0.0.1:8200:8200"  # Localhost only
    env_file:
      - .env.production
    volumes:
      - ralph_data:/data/ralph
    restart: unless-stopped
    depends_on:
      - dolt

  dolt:
    image: dolthub/dolt-sql-server:latest
    container_name: dolt
    ports:
      - "127.0.0.1:3307:3306"  # Localhost only
    volumes:
      - dolt_data:/var/lib/dolt
      - ./config/dolt/config.yaml:/etc/dolt/servercfg.d/config.yaml:ro
      - ./config/dolt/init.sql:/docker-entrypoint-initdb.d/init.sql:ro
    environment:
      DOLT_ROOT_PASSWORD: ${DOLT_PASSWORD:-devpassword}
      DOLT_ROOT_HOST: "%"
    restart: unless-stopped

volumes:
  openwebui_data:
  ralph_data:
  dolt_data:
```

### 5. Deployment Steps (Beginner-Friendly)

#### Phase 1: VPS Setup

```bash
# 1. SSH into your VPS
ssh user@your-vps-ip

# 2. Update system
sudo apt update && sudo apt upgrade -y

# 3. Install Docker
sudo apt install ca-certificates curl
sudo install -m 0755 -d /etc/apt/keyrings
sudo curl -fsSL https://download.docker.com/linux/ubuntu/gpg -o /etc/apt/keyrings/docker.asc
sudo chmod a+r /etc/apt/keyrings/docker.asc

echo "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.asc] https://download.docker.com/linux/ubuntu $(. /etc/os-release && echo "$VERSION_CODENAME") stable" | sudo tee /etc/apt/sources.list.d/docker.list
sudo apt update
sudo apt install docker-ce docker-ce-cli containerd.io docker-compose-plugin

# 4. Add user to docker group
sudo usermod -aG docker $USER
newgrp docker

# 5. Enable Docker on boot
sudo systemctl enable docker
```

#### Phase 2: Cloudflare Tunnel

```bash
# 1. Install cloudflared
sudo apt install cloudflared

# 2. Authenticate (opens browser)
cloudflared tunnel login

# 3. Create tunnel
cloudflared tunnel create youlab

# 4. Create config
mkdir -p ~/.cloudflared
cat > ~/.cloudflared/config.yml << 'EOF'
tunnel: <YOUR-TUNNEL-UUID>
credentials-file: /home/ubuntu/.cloudflared/<YOUR-TUNNEL-UUID>.json

ingress:
  - hostname: theyoulab.org
    service: http://localhost:3000
  - service: http_status:404
EOF

# 5. Route DNS
cloudflared tunnel route dns youlab theyoulab.org

# 6. Install as system service
sudo cloudflared service install
sudo systemctl enable cloudflared
sudo systemctl start cloudflared
```

#### Phase 3: Deploy YouLab

```bash
# 1. Clone repository
git clone https://github.com/ariavasulin/YouLab.git
cd YouLab

# 2. Create production env file
cat > .env.production << 'EOF'
RALPH_OPENROUTER_API_KEY=your-api-key-here
RALPH_OPENROUTER_MODEL=anthropic/claude-sonnet-4-20250514
RALPH_HONCHO_ENVIRONMENT=demo
RALPH_DOLT_HOST=dolt
RALPH_DOLT_PORT=3306
RALPH_DOLT_PASSWORD=your-secure-password
RALPH_USER_DATA_DIR=/data/ralph/users
EOF

# 3. Start services
docker compose -f docker-compose.prod.yml up -d

# 4. Check logs
docker compose -f docker-compose.prod.yml logs -f
```

#### Phase 4: Configure OpenWebUI Pipe

1. Access OpenWebUI at http://localhost:3000 (or theyoulab.org after tunnel)
2. Create admin account (first user)
3. Go to Admin Panel → Settings → Functions → Pipes
4. Add Ralph pipe from `src/ralph/pipe.py`
5. Configure RALPH_SERVICE_URL valve:
   - Docker mode: `http://ralph:8200`
   - Host mode: `http://host.docker.internal:8200`

### 6. Security Considerations

**Essential**:
```bash
# Firewall - block all except SSH
sudo ufw default deny incoming
sudo ufw default allow outgoing
sudo ufw allow ssh
sudo ufw enable  # Cloudflare Tunnel needs no inbound ports!

# Fail2ban for SSH protection
sudo apt install fail2ban
sudo systemctl enable fail2ban

# Change default Dolt password in production
# Never commit .env.production to git
```

**Important files to gitignore**:
```
.env.production
*.json  # Cloudflare credentials
~/.cloudflared/cert.pem
```

## Code References

- `src/ralph/server.py:352` - Ralph server port (8200)
- `src/ralph/config.py:15-52` - All RALPH_* environment variables
- `src/ralph/pipe.py:24-27` - Pipe default URL configuration
- `docker-compose.yml:2-15` - Dolt Docker configuration
- `OpenWebUI/open-webui/docker-compose.yaml:11-28` - OpenWebUI Docker configuration
- `config/dolt/config.yaml` - Dolt server configuration
- `config/dolt/init.sql` - Database schema initialization

## Current Limitations

1. **Ralph not containerized**: The Ralph server runs natively, not in Docker. A Dockerfile needs to be created.
2. **No unified compose file**: Currently separate compose files for OpenWebUI and Dolt.
3. **Hardcoded port**: Ralph server port 8200 is hardcoded, not configurable via env var.
4. **No migrations system**: Dolt schema is in init.sql but migrations in `migrations/` aren't auto-applied.

## Recommended Next Steps

1. Create `Dockerfile.ralph` for containerizing Ralph server
2. Create unified `docker-compose.prod.yml` with all services
3. Make Ralph port configurable via `RALPH_PORT` env var
4. Set up automatic database migrations
5. Add health check endpoints for all services
6. Consider adding monitoring (Prometheus/Grafana) for production

## External Resources

- [Cloudflare Tunnel Documentation](https://developers.cloudflare.com/cloudflare-one/networks/connectors/cloudflare-tunnel/)
- [Docker Installation on Ubuntu](https://docs.docker.com/engine/install/ubuntu/)
- [OpenWebUI Documentation](https://docs.openwebui.com/)
- [Dolt Documentation](https://docs.dolthub.com/)
- [OpenRouter Documentation](https://openrouter.ai/docs)

## Open Questions

1. Should Ralph server use HTTPS internally or rely on Cloudflare for SSL termination?
2. What backup strategy for Dolt data volumes?
3. How to handle OpenWebUI user management (SSO/OAuth)?
4. Rate limiting strategy for the API?

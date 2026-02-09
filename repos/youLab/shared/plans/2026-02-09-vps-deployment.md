# VPS Deployment Plan — theyoulab.org

## Overview

Deploy the YouLab stack (OpenWebUI fork + Ralph + Dolt) to the RackNerd VPS (San Jose, 8GB RAM, Ubuntu 22.04) and expose it via Cloudflare Tunnel at `theyoulab.org`.

## Current State Analysis

- **VPS**: Provisioned, Ubuntu, SSH-ready
- **Domain**: `theyoulab.org` on Cloudflare (DNS + domain already set up)
- **OpenWebUI fork**: `github.com/ariavasulin/open-webui` — custom YouLab branding, memory block UI, "You" page, block diff approval
- **docker-compose.prod.yml**: Already written, builds OpenWebUI from fork, Ralph from local Dockerfile, Dolt from official image
- **Ralph Dockerfile**: Already written, multi-stage Python build with health check
- **.env.production.example**: Already written with all required vars documented

### Key Discoveries:
- `docker-compose.prod.yml` builds OpenWebUI directly from GitHub fork URL — no need to clone the fork separately on VPS
- Ralph pipe needs to be installed manually in OpenWebUI admin UI after first boot
- All ports bind to `127.0.0.1` only — Cloudflare Tunnel is the sole ingress
- OpenWebUI listens internally on 8080, mapped to host 3000

## Desired End State

- `theyoulab.org` serves the customized OpenWebUI frontend + backend
- Ralph API running on `localhost:8200` (internal, accessed by OpenWebUI pipe)
- Dolt DB running on `localhost:3307` (internal)
- Cloudflare Tunnel running as a systemd service
- UFW firewall allowing only SSH inbound
- Admin account created, public signups disabled

**Verification**: Visit `theyoulab.org`, log in, send a message through the Ralph pipe, get a response.

## What We're NOT Doing

- No Coolify — plain Docker Compose
- No CI/CD pipeline — manual `docker compose up` for now
- No monitoring/Prometheus — can add later
- No `openwebui-content-sync` service — that's for later
- No custom domain for Ralph API (`api.theyoulab.org`) — Ralph is internal only
- No database backups strategy yet — can add later

## Implementation Approach

SSH into VPS, install Docker, clone YouLab repo, configure env, bring up stack, set up Cloudflare Tunnel, configure OpenWebUI.

---

## Phase 1: VPS Base Setup

### Overview
Install Docker and clone the repo.

### Steps

```bash
# 1. SSH in
ssh root@<VPS-IP>

# 2. Update system
sudo apt update && sudo apt upgrade -y

# 3. Install Docker (official method)
sudo apt install -y ca-certificates curl
sudo install -m 0755 -d /etc/apt/keyrings
sudo curl -fsSL https://download.docker.com/linux/ubuntu/gpg -o /etc/apt/keyrings/docker.asc
sudo chmod a+r /etc/apt/keyrings/docker.asc

echo "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.asc] https://download.docker.com/linux/ubuntu $(. /etc/os-release && echo "$VERSION_CODENAME") stable" | sudo tee /etc/apt/sources.list.d/docker.list
sudo apt update
sudo apt install -y docker-ce docker-ce-cli containerd.io docker-compose-plugin

# 4. Add your user to docker group (if not root)
sudo usermod -aG docker $USER
newgrp docker

# 5. Enable Docker on boot
sudo systemctl enable docker

# 6. Verify
docker --version
docker compose version

# 7. Clone YouLab repo
cd /opt
sudo git clone https://github.com/ariavasulin/YouLab.git
cd YouLab
sudo chown -R $USER:$USER .
```

### Success Criteria

#### Automated Verification:
- [ ] `docker --version` returns a version
- [ ] `docker compose version` returns a version
- [ ] `/opt/YouLab` exists and contains `docker-compose.prod.yml`

#### Manual Verification:
- [ ] SSH access works

---

## Phase 2: Configure & Deploy Stack

### Overview
Set up production env vars and bring up all three services.

### Steps

```bash
cd /opt/YouLab

# 1. Create .env.production from example
cp .env.production.example .env.production

# 2. Edit with real values
nano .env.production
```

Fill in:
```bash
RALPH_OPENROUTER_API_KEY=sk-or-v1-<your-actual-key>
DOLT_ROOT_PASSWORD=<generate-a-strong-password>
WEBUI_SECRET_KEY=<run: openssl rand -hex 32>
```

Leave the rest as defaults (they're fine for initial deploy).

```bash
# 3. Build and start all services
docker compose -f docker-compose.prod.yml up -d --build

# 4. Watch logs until healthy
docker compose -f docker-compose.prod.yml logs -f
```

**Note**: The OpenWebUI build will take several minutes on first run (it does `npm ci` + `npm run build` + Python deps inside the container). Ralph build is faster.

### Success Criteria

#### Automated Verification:
- [ ] `docker compose -f docker-compose.prod.yml ps` shows all 3 containers running
- [ ] `curl -s http://localhost:3000` returns HTML
- [ ] `curl -s http://localhost:8200/health` returns healthy response
- [ ] `docker compose -f docker-compose.prod.yml logs ralph` shows no errors

#### Manual Verification:
- [ ] `http://localhost:3000` loads OpenWebUI in a browser (via SSH tunnel if needed: `ssh -L 3000:localhost:3000 root@VPS-IP`)

**Implementation Note**: Pause here for manual confirmation before proceeding to Phase 3.

---

## Phase 3: Cloudflare Tunnel

### Overview
Install cloudflared, create a tunnel, route `theyoulab.org` to localhost:3000, run as systemd service.

### Steps

```bash
# 1. Install cloudflared
sudo mkdir -p --mode=0755 /usr/share/keyrings
curl -fsSL https://pkg.cloudflare.com/cloudflare-main.gpg | sudo tee /usr/share/keyrings/cloudflare-main.gpg >/dev/null
echo 'deb [signed-by=/usr/share/keyrings/cloudflare-main.gpg] https://pkg.cloudflare.com/cloudflared jammy main' | sudo tee /etc/apt/sources.list.d/cloudflared.list
sudo apt update && sudo apt install -y cloudflared

# 2. Authenticate (this will print a URL — open it in your browser)
cloudflared tunnel login

# 3. Create tunnel
cloudflared tunnel create youlab
# Note the tunnel UUID from the output

# 4. Create tunnel config
cat > ~/.cloudflared/config.yml << 'EOF'
tunnel: <TUNNEL-UUID>
credentials-file: /root/.cloudflared/<TUNNEL-UUID>.json

ingress:
  - hostname: theyoulab.org
    service: http://localhost:3000
  - service: http_status:404
EOF

# 5. Route DNS
cloudflared tunnel route dns youlab theyoulab.org

# 6. Test it manually first
cloudflared tunnel run youlab
# Ctrl+C once confirmed working

# 7. Install as systemd service so it survives reboots
sudo cloudflared service install
sudo systemctl enable cloudflared
sudo systemctl start cloudflared
```

### Success Criteria

#### Automated Verification:
- [ ] `sudo systemctl status cloudflared` shows active/running
- [ ] `curl -s https://theyoulab.org` returns HTML

#### Manual Verification:
- [ ] Visit `https://theyoulab.org` in browser — OpenWebUI loads with YouLab branding

**Implementation Note**: Pause here for manual confirmation before proceeding to Phase 4.

---

## Phase 4: Configure OpenWebUI + Ralph Pipe

### Overview
Create admin account, install the Ralph pipe, and verify end-to-end chat.

### Steps

1. Go to `https://theyoulab.org`
2. Create admin account (first user automatically becomes admin)
3. Go to **Admin Panel → Settings → Functions → Pipes**
4. Click "+" to create a new pipe
5. Paste the contents of `src/ralph/pipe.py` into the editor
6. Save it
7. Configure the pipe valve: set `RALPH_SERVICE_URL` to `http://host.docker.internal:8200`
   - This works because `docker-compose.prod.yml` has `extra_hosts: host.docker.internal:host-gateway`
   - Alternative if that doesn't work: `http://youlab-ralph:8200` (container name on the `youlab` network)
8. **Disable public signups**: Admin Panel → Settings → General → toggle off "Enable New Sign Ups" (already set via `ENABLE_SIGNUP=false` in compose, but verify)

### Success Criteria

#### Manual Verification:
- [ ] Admin account created and logged in
- [ ] Ralph pipe visible in Pipes list
- [ ] Send a test message → get a response from Ralph (confirms full chain: OpenWebUI → Pipe → Ralph → OpenRouter → response)
- [ ] Sign-up page is disabled for new users

**Implementation Note**: Pause here for manual confirmation before proceeding to Phase 5.

---

## Phase 5: Harden

### Overview
Firewall and basic SSH protection.

### Steps

```bash
# 1. UFW firewall — only allow SSH, everything else via tunnel
sudo ufw default deny incoming
sudo ufw default allow outgoing
sudo ufw allow ssh
sudo ufw enable

# 2. Fail2ban for SSH brute force protection
sudo apt install -y fail2ban
sudo systemctl enable fail2ban
sudo systemctl start fail2ban
```

### Success Criteria

#### Automated Verification:
- [ ] `sudo ufw status` shows SSH allowed, default deny
- [ ] `sudo systemctl status fail2ban` shows active
- [ ] `https://theyoulab.org` still works (tunnel unaffected by firewall)

---

## Quick Reference — Post-Deploy Operations

```bash
# View logs
docker compose -f docker-compose.prod.yml logs -f

# Restart a service
docker compose -f docker-compose.prod.yml restart ralph

# Pull updates and redeploy
cd /opt/YouLab
git pull
docker compose -f docker-compose.prod.yml up -d --build

# Rebuild just OpenWebUI (after pushing to fork)
docker compose -f docker-compose.prod.yml build --no-cache openwebui
docker compose -f docker-compose.prod.yml up -d openwebui

# Check tunnel status
sudo systemctl status cloudflared
```

## References

- VPS research: `thoughts/shared/research/2026-01-28-vps-deployment-theyoulab-org.md`
- RackNerd handoff: `thoughts/shared/handoffs/general/2026-01-09_15-05-48_racknerd-deployment-setup.md`
- Production compose: `docker-compose.prod.yml`
- Ralph Dockerfile: `Dockerfile`
- Env template: `.env.production.example`
- Ralph pipe: `src/ralph/pipe.py`

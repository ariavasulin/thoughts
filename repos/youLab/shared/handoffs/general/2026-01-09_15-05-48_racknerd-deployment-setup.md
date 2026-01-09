---
date: 2026-01-09T15:05:48+07:00
researcher: ariasulin
git_commit: 3d8dbffc4b38f61f2e099982610a355a7360926a
branch: main
repository: YouLab
topic: "RackNerd VPS Deployment Setup"
tags: [deployment, infrastructure, racknerd, coolify, vps]
status: in_progress
last_updated: 2026-01-09
last_updated_by: claude
type: implementation_strategy
---

# Handoff: RackNerd VPS Deployment Setup for YouLab

## Task(s)

| Task | Status |
|------|--------|
| Research budget VPS options under $20/month | Completed |
| Find US-based (NorCal) alternative to Hetzner | Completed |
| Select provider and configure order | Completed |
| Provision server and install Coolify | Planned |
| Deploy YouLab stack | Planned |
| Configure domain + SSL | Planned |

**Decision Made:** RackNerd San Jose, 8GB RAM, $62.49/year (~$5.21/mo), Ubuntu 22.04 LTS

## Critical References

- `thoughts/shared/research/2026-01-04-team-deployment-options.md` - Comprehensive deployment research covering Railway, Fly.io, Hetzner, AWS options
- `docs/Quickstart.md` - Local setup documentation (reference for what needs deploying)

## Recent changes

No code changes - this was a research/planning session.

## Learnings

1. **Lightsail is NOT the best value** - Hetzner gives 4x the RAM for the same price. RackNerd gives 8GB for $5.21/mo vs Lightsail's 1GB for $7/mo.

2. **Hetzner now has US datacenters** at EU pricing ($5.99/mo for 8GB in Ashburn VA or Hillsboro OR).

3. **RackNerd is the only budget provider with NorCal servers** - San Jose datacenter specifically. Vultr has Silicon Valley but at $24/mo (not competitive).

4. **YouLab stack needs ~2-2.5GB RAM total:**
   - HTTP Service: 256MB
   - Letta Server: 1GB
   - OpenWebUI: 512MB-1GB

5. **Coolify simplifies deployment** - open-source Railway clone with git push deploy, auto SSL, one-click databases. Runs on any VPS with Docker.

6. **Ubuntu 22.04 LTS preferred** over 24.04 for Coolify/Docker - more battle-tested, though 24.04 works fine.

## Artifacts

- `thoughts/shared/research/2026-01-04-team-deployment-options.md` - Pre-existing comprehensive deployment research (read this for full context)
- User ordered RackNerd 8GB KVM VPS (Black Friday 2025) - San Jose, CA - $62.49/year

## Action Items & Next Steps

1. **Wait for RackNerd provisioning email** - will contain IP address and root password

2. **SSH into server and install Coolify:**
   ```bash
   ssh root@your-ip
   curl -fsSL https://cdn.coollabs.io/coolify/install.sh | bash
   ```

3. **Access Coolify** at `http://your-ip:8000` and create admin account

4. **Create Dockerfile for HTTP Service** (doesn't exist yet):
   - See template in `thoughts/shared/research/2026-01-04-team-deployment-options.md` lines 193-211

5. **Add YouLab repo to Coolify** and deploy services:
   - OpenWebUI
   - Letta Server (with volume for PostgreSQL persistence)
   - HTTP Service

6. **Configure domain:**
   - Point A record to server IP
   - Add domain in Coolify app settings
   - Enable SSL (Let's Encrypt auto-configured)
   - Optional: Use Cloudflare for CDN/DDoS protection (start with Proxy OFF, enable after SSL works)

7. **Configure OpenWebUI:**
   - Create admin account (first user)
   - Disable new registrations in Settings
   - Add Pipe from `src/letta_starter/pipelines/letta_pipe.py`
   - Set Pipe valve: `LETTA_SERVICE_URL=http://http-service:8100`

## Other Notes

**Provider Comparison (for reference):**

| Provider | RAM | Price | NorCal? |
|----------|-----|-------|---------|
| RackNerd | 8GB | $5.21/mo | San Jose |
| OVHcloud | 8GB | $4.20/mo | No (US East) |
| Hetzner | 8GB | $5.99/mo | No (OR, VA) |
| Vultr | 4GB | $24/mo | Silicon Valley |

**Useful Links:**
- [RackNerd San Jose Datacenter](https://www.racknerd.com/san-jose-datacenter)
- [Coolify Installation](https://coolify.io/docs/get-started/installation)
- [Coolify Docs](https://coolify.io/docs)

**Environment Variables Needed for Deployment:**
```bash
# HTTP Service
YOULAB_SERVICE_HOST=0.0.0.0
YOULAB_SERVICE_PORT=8100
YOULAB_SERVICE_LETTA_BASE_URL=http://letta:8283
OPENAI_API_KEY=sk-...

# Letta Server
OPENAI_API_KEY=sk-...
```

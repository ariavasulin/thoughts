---
date: 2026-01-04T17:07:29+07:00
researcher: ariasulin
git_commit: 29783e20ce6d230469eee40148d80b068d2fd021
branch: main
repository: YouLab
topic: "Cloud Deployment Options for YouLab"
tags: [research, deployment, infrastructure, docker, railway, cloud]
status: complete
last_updated: 2026-01-04
last_updated_by: ariasulin
---

# Research: Cloud Deployment Options for YouLab

**Date**: 2026-01-04T17:07:29+07:00
**Researcher**: ariasulin
**Git Commit**: 29783e20ce6d230469eee40148d80b068d2fd021
**Branch**: main
**Repository**: YouLab

## Research Question

How to deploy YouLab to the cloud for:
- Team of 4 to test/use the full-stack
- Easy development iteration (commit, test, deploy)
- Authentication via OpenWebUI (disable new account creation)
- Persistent Letta agent memory (non-negotiable)
- Foundation for eventual production launch

## Summary

YouLab requires **3 services** for cloud deployment (not 4 - Ollama is not needed):

| Service | Purpose | Persistence | Notes |
|---------|---------|-------------|-------|
| OpenWebUI | Chat UI + auth | Volume for SQLite | Has Railway one-click template |
| Letta Server | Agent memory | **Volume required** | PostgreSQL at `/var/lib/postgresql/data` |
| HTTP Service | API bridge | Stateless | Needs Dockerfile (doesn't exist yet) |

**Key finding**: Ollama is NOT used in the YouLab flow. The pipeline goes `OpenWebUI → HTTP Service → Letta → OpenAI/Claude API` directly.

**Recommended platform**: Railway - has one-click OpenWebUI template, supports persistent volumes, simple multi-service projects

## Detailed Findings

### Actual Data Flow (Ollama Not Used)

The YouLab pipeline bypasses Ollama entirely:

```
Browser → OpenWebUI:3000 → HTTP Service:8100 → Letta:8283 → OpenAI/Claude API
```

**Evidence** (from codebase analysis):
- `src/letta_starter/server/agents.py:110` - Agents hardcoded to `model="openai/gpt-4o-mini"`
- `src/letta_starter/pipelines/letta_pipe.py` - Pipe connects directly to HTTP Service, no Ollama reference
- Ollama only appears in docker-compose for OpenWebUI's built-in model support (unused by YouLab)

### Services Required for Cloud Deployment

| Service | Image | Port | Persistence | Railway Support |
|---------|-------|------|-------------|-----------------|
| OpenWebUI | `ghcr.io/open-webui/open-webui:main` | 8080 | `/app/backend/data` | One-click template |
| Letta Server | `letta/letta:latest` | 8283 | `/var/lib/postgresql/data` | Docker image |
| HTTP Service | Custom Dockerfile needed | 8100 | None | Dockerfile |

### Letta Persistence Details

From [Letta Docker documentation](https://docs.letta.com/guides/server/remote):

```bash
# Volume mount for persistent agent memory
-v ~/.letta/.persist/pgdata:/var/lib/postgresql/data
```

The Letta Docker image bundles PostgreSQL. Without this volume mount, **all agent data is lost on container restart**.

For cloud deployments, you can also use external PostgreSQL via `LETTA_PG_URI` environment variable.

### Environment Variables for Cloud

**HTTP Service**:
```bash
YOULAB_SERVICE_HOST=0.0.0.0              # Accept external connections
YOULAB_SERVICE_PORT=8100
YOULAB_SERVICE_LETTA_BASE_URL=http://letta:8283  # Internal service name
OPENAI_API_KEY=sk-...
```

**Letta Server**:
```bash
OPENAI_API_KEY=sk-...
# Optional: LETTA_PG_URI for external PostgreSQL
```

**OpenWebUI**:
```bash
# Disable Ollama (not needed)
OLLAMA_BASE_URL=  # Leave empty or don't set
# Point Pipe to HTTP Service
# (Configured in admin UI, not env var)
```

### OpenWebUI Authentication

OpenWebUI handles user authentication natively:
- First user becomes admin
- Admin can disable new account creation in Settings
- Users authenticate against OpenWebUI's internal user database
- No external auth service needed for team of 4

## Railway Deployment (Recommended)

### Why Railway

- [One-click OpenWebUI template](https://railway.com/deploy/open-webui) exists
- [Persistent volumes](https://docs.railway.com/reference/volumes) up to 250GB on Pro plan
- Multiple services in one project with internal networking
- Auto-HTTPS on all services
- GitHub integration for auto-deploy on push
- Cost: ~$5-20/month for this stack

### Deployment Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                     Railway Project                              │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  ┌──────────────────┐                                           │
│  │    OpenWebUI     │◄──── HTTPS (public domain)                │
│  │   (template)     │                                           │
│  │ Volume: /data    │                                           │
│  └────────┬─────────┘                                           │
│           │                                                      │
│           │ http://http-service.railway.internal:8100           │
│           ▼                                                      │
│  ┌──────────────────┐                                           │
│  │   HTTP Service   │◄──── Internal only (no public URL)        │
│  │   (Dockerfile)   │                                           │
│  │   Stateless      │                                           │
│  └────────┬─────────┘                                           │
│           │                                                      │
│           │ http://letta.railway.internal:8283                  │
│           ▼                                                      │
│  ┌──────────────────┐                                           │
│  │   Letta Server   │◄──── Internal only                        │
│  │   (Docker image) │                                           │
│  │ Volume: /pgdata  │◄──── CRITICAL: Agent memory               │
│  └──────────────────┘                                           │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

### Setup Steps

1. **Create Railway Project**
   - Start from OpenWebUI template: https://railway.com/deploy/open-webui
   - This creates OpenWebUI service with volume

2. **Add Letta Service**
   - Add new service → Docker Image → `letta/letta:latest`
   - Add volume mounted to `/var/lib/postgresql/data`
   - Set environment variables: `OPENAI_API_KEY`
   - No public URL needed (internal only)

3. **Add HTTP Service**
   - Add new service → GitHub repo (or Dockerfile)
   - Point to YouLab repo, Dockerfile at root
   - Set environment variables:
     - `YOULAB_SERVICE_HOST=0.0.0.0`
     - `YOULAB_SERVICE_LETTA_BASE_URL=http://letta.railway.internal:8283`
     - `OPENAI_API_KEY`
   - No public URL needed (internal only)

4. **Configure OpenWebUI**
   - Access via Railway-provided HTTPS URL
   - Create admin account (first user)
   - Disable new registrations in Settings
   - Add Pipe from `src/letta_starter/pipelines/letta_pipe.py`
   - Set Pipe valve: `LETTA_SERVICE_URL=http://http-service.railway.internal:8100`

5. **Invite Team**
   - Create accounts for team members manually in OpenWebUI admin
   - Share the Railway HTTPS URL

### What Needs to Be Created

**Dockerfile for HTTP Service** (doesn't exist yet):

```dockerfile
FROM python:3.11-slim

WORKDIR /app

# Install uv
RUN pip install uv

# Copy project files
COPY pyproject.toml uv.lock ./
COPY src/ ./src/

# Install dependencies
RUN uv sync --frozen

# Run the service
EXPOSE 8100
CMD ["uv", "run", "letta-server"]
```

### Development Workflow with Railway

**Option A: Auto-deploy from main**
- Push to `main` → Railway auto-deploys
- Team tests on production URL

**Option B: Staging environment**
- Create second Railway project or environment
- Deploy feature branches there
- Team tests on staging URL

**Option C: Local + Cloud hybrid**
- Develop locally as usual
- Push to branch when ready for team testing
- Railway deploys branch to staging

## Alternative: Fly.io

Fly.io is more complex but offers:
- Edge deployment (servers closer to users)
- More control over infrastructure
- Slightly cheaper for compute-heavy workloads

**Drawbacks**:
- No native docker-compose support
- Requires converting to `fly.toml` configuration
- Each service = separate Fly app
- More manual networking setup

See [Fly.io Docker documentation](https://fly.io/docs/app-guides/multiple-processes/)

## Alternative: Dev Server + Tunnel (Simpler Short-term)

If you want to start testing today without cloud setup:

1. Run stack on your machine (existing setup)
2. Expose via Tailscale or ngrok
3. Team accesses your machine directly
4. Migrate to Railway when ready for always-on

**Pros**: Works immediately, no new infrastructure
**Cons**: Your machine must be online, single point of failure

## Code References

- `src/letta_starter/pipelines/letta_pipe.py:25` - Pipe default URL: `http://host.docker.internal:8100`
- `src/letta_starter/server/agents.py:110` - Model hardcoded to `openai/gpt-4o-mini`
- `src/letta_starter/server/cli.py:8-17` - HTTP service entry point
- `src/letta_starter/config/settings.py:106-153` - Environment configuration
- `OpenWebUI/open-webui/docker-compose.yaml` - Local OpenWebUI compose (includes Ollama, not needed for cloud)
- `docs/Quickstart.md` - Current local setup documentation

## Historical Context (from thoughts/)

- `thoughts/shared/plans/2025-12-26-youlab-technical-foundation.md` - Original scope excluded production deployment
- `thoughts/shared/research/2025-12-31-open-webui-docker-setup.md` - Docker setup research

**Note**: The original plan scoped out production deployment for pilot. This research expands scope to support team testing and future launch.

## Next Steps

1. **Create Dockerfile** for HTTP Service (see template above)
2. **Test Railway deployment** with all three services
3. **Configure OpenWebUI** authentication (disable registration)
4. **Document deployment** in `/docs/Deployment.md`

## Web Sources

- [Railway Volumes Documentation](https://docs.railway.com/reference/volumes)
- [Railway Open WebUI Template](https://railway.com/deploy/open-webui)
- [Letta Remote Deployment Guide](https://docs.letta.com/guides/server/remote)
- [OpenWebUI Environment Configuration](https://docs.openwebui.com/getting-started/env-configuration/)
- [Fly.io Multiple Processes](https://fly.io/docs/app-guides/multiple-processes/)

## Open Questions

1. Should Letta use Railway's managed PostgreSQL instead of bundled PostgreSQL for better reliability?
2. Do you want staging + production environments, or just one deployment?
3. Should the HTTP Service have a public healthcheck URL for monitoring?

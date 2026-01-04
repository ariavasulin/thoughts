---
date: 2026-01-04T17:07:29+07:00
researcher: ariasulin
git_commit: 29783e20ce6d230469eee40148d80b068d2fd021
branch: main
repository: YouLab
topic: "Simple Team Deployment Options for YouLab"
tags: [research, deployment, infrastructure, docker, team-access]
status: complete
last_updated: 2026-01-04
last_updated_by: ariasulin
---

# Research: Simple Team Deployment Options for YouLab

**Date**: 2026-01-04T17:07:29+07:00
**Researcher**: ariasulin
**Git Commit**: 29783e20ce6d230469eee40148d80b068d2fd021
**Branch**: main
**Repository**: YouLab

## Research Question

How to deploy YouLab for a team of 4 to test/use the full-stack while:
- Easily committing and making changes
- Testing changes (potentially on a test branch)
- Not interfering with ongoing development

## Summary

YouLab currently has **no cloud deployment infrastructure** - this is intentional for the pilot phase. The application runs as a 4-service stack: OpenWebUI (port 3000), Ollama (port 11434), Letta Server (port 8283), and HTTP Service (port 8100). All services are designed for localhost with a "trust localhost" security model.

For simple team deployment, there are three practical paths:
1. **Shared development server** with tunnel access (simplest, recommended)
2. **Unified Docker Compose** with all services (moderate complexity)
3. **Cloud deployment** to Railway/Fly.io/Render (highest complexity)

## Detailed Findings

### Current Architecture

The full YouLab stack consists of 4 services:

```
Browser → OpenWebUI:3000 → HTTP Service:8100 → Letta:8283 → Claude API
```

| Service | Container/Process | Port | Data Persistence |
|---------|-------------------|------|------------------|
| OpenWebUI | Docker: `open-webui` | 3000 | Volume: `open-webui:/app/backend/data` |
| Ollama | Docker: `ollama` | 11434 | Volume: `ollama:/root/.ollama` |
| Letta Server | Docker: `letta` | 8283 | Volume: `letta-data:/root/.letta` |
| HTTP Service | Host process | 8100 | None (stateless, delegates to Letta) |

**Source**: `docs/Quickstart.md:15-21`

### Docker Configurations

**OpenWebUI docker-compose.yaml** (`OpenWebUI/open-webui/docker-compose.yaml`):
- Runs OpenWebUI + Ollama together
- OpenWebUI builds from local Dockerfile or pulls `ghcr.io/open-webui/open-webui:main`
- Uses `host.docker.internal:host-gateway` for container-to-host networking

**Letta Server** - runs as standalone container:
```bash
docker run -d --name letta -p 8283:8283 -v letta-data:/root/.letta letta/letta:latest
```

**HTTP Service** - runs on host (not containerized):
```bash
uv run letta-server
```

### External Service Dependencies

| Service | Requirement | Notes |
|---------|-------------|-------|
| Claude API | Required | `OPENAI_API_KEY` or `ANTHROPIC_API_KEY` |
| Langfuse | Optional | Tracing/observability, gracefully degrades |
| Honcho | Future | Theory-of-mind layer (Phase 3) |

### Environment Variables Required for Deployment

**Core (`.env.example`)**:
```bash
LETTA_BASE_URL=http://localhost:8283
OPENAI_API_KEY=sk-...
# or ANTHROPIC_API_KEY=sk-ant-...
```

**HTTP Service (`YOULAB_SERVICE_` prefix)**:
```bash
YOULAB_SERVICE_HOST=127.0.0.1
YOULAB_SERVICE_PORT=8100
YOULAB_SERVICE_LETTA_BASE_URL=http://localhost:8283
YOULAB_SERVICE_LANGFUSE_ENABLED=true  # optional
YOULAB_SERVICE_LANGFUSE_PUBLIC_KEY=...
YOULAB_SERVICE_LANGFUSE_SECRET_KEY=...
```

**Source**: `src/letta_starter/config/settings.py:106-153`

### Security Model

Current "trust localhost" approach (`thoughts/shared/plans/2025-12-26-youlab-technical-foundation.md`):
- No authentication between services
- API key authentication designed but not implemented
- All services assume local network

**Implication**: Any team deployment needs either:
1. VPN/tunnel to appear as localhost
2. Network isolation (private VPC)
3. Authentication implementation (more work)

## Deployment Options Analysis

### Option A: Shared Dev Server + Tunnel (Recommended)

**How it works**: Run the full stack on a single server (personal machine, EC2, DigitalOcean droplet). Team accesses via secure tunnel.

**Setup**:
1. Use existing local dev setup on a dedicated machine
2. Expose via Tailscale (VPN) or ngrok/Cloudflare Tunnel (public)
3. Team connects to `your-server.tailnet.ts.net:3000` or `xxx.ngrok.io`

**Pros**:
- No infrastructure changes needed
- Development workflow unchanged (edit code, restart service)
- Branch testing: checkout branch, restart services
- Works today with zero code changes

**Cons**:
- Single point of failure (your machine)
- Your machine must be running/online
- All team shares same instance (no isolation)

**Branch testing workflow**:
```bash
# On server
git checkout feature-branch
docker restart open-webui letta
uv run letta-server  # restart HTTP service
# Team tests immediately
```

### Option B: Unified Docker Compose

**How it works**: Create a single `docker-compose.yml` that runs all 4 services together.

**What's needed**:
1. Create Dockerfile for HTTP Service (doesn't exist yet)
2. Unified compose file with all services
3. Environment variable management
4. Networking between containers

**Pros**:
- Reproducible deployment
- Easy to spin up on any Docker host
- Can run multiple instances (staging, production)

**Cons**:
- Need to create Dockerfile for HTTP Service
- Need to rebuild container on code changes
- More moving parts than Option A

**Rough structure**:
```yaml
services:
  ollama: ...
  open-webui: ...
  letta: ...
  http-service:
    build: .
    ports: ["8100:8100"]
    environment:
      LETTA_BASE_URL: http://letta:8283
    depends_on: [letta]
```

### Option C: Cloud Platform (Railway/Fly.io/Render)

**How it works**: Deploy containerized services to a cloud platform.

**What's needed**:
1. Everything from Option B (Dockerfiles, compose)
2. Platform configuration (Railway.toml, fly.toml)
3. Environment variable management in platform
4. Consider persistent storage (volumes)
5. May need to split services across multiple apps

**Complexity considerations**:
- OpenWebUI: Large image, needs volume for SQLite
- Letta: Needs persistent volume for agent memory
- HTTP Service: Easiest (stateless)
- Ollama: Very large, may want to skip (use OpenAI directly)

**Pros**:
- Professional deployment
- Always available
- Multiple environments possible
- CI/CD integration

**Cons**:
- Cost ($5-50/month depending on platform)
- More complex setup and debugging
- Code changes require deploy pipeline
- Overkill for team of 4 testing

## Code References

- `OpenWebUI/open-webui/docker-compose.yaml` - Existing OpenWebUI compose
- `docs/Quickstart.md` - Current local setup documentation
- `src/letta_starter/server/cli.py:8-17` - HTTP service entry point
- `src/letta_starter/config/settings.py:106-153` - Environment configuration
- `.env.example` - Environment template

## Architecture Documentation

### Current Local Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                        localhost                                 │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  ┌──────────────┐    ┌──────────────┐    ┌──────────────┐       │
│  │  OpenWebUI   │    │    Ollama    │    │    Letta     │       │
│  │   :3000      │    │   :11434     │    │    :8283     │       │
│  │   (Docker)   │    │   (Docker)   │    │   (Docker)   │       │
│  └──────┬───────┘    └──────────────┘    └──────▲───────┘       │
│         │                                        │               │
│         │ http://host.docker.internal:8100       │               │
│         ▼                                        │               │
│  ┌──────────────┐                               │               │
│  │ HTTP Service │───────────────────────────────┘               │
│  │    :8100     │   http://localhost:8283                       │
│  │   (Host)     │                                               │
│  └──────────────┘                                               │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

### Team Deployment with Tunnel (Option A)

```
┌──────────────┐     ┌──────────────────────────────────────────────┐
│  Team Member │     │            Development Server                 │
│              │     ├──────────────────────────────────────────────┤
│  Browser     │────▶│  Tailscale/ngrok                             │
│              │     │       │                                       │
└──────────────┘     │       ▼                                       │
                     │  ┌──────────────┐  ┌──────────────┐          │
                     │  │  OpenWebUI   │  │    Letta     │          │
                     │  │   :3000      │  │    :8283     │          │
                     │  └──────┬───────┘  └──────▲───────┘          │
                     │         │                  │                  │
                     │         ▼                  │                  │
                     │  ┌──────────────┐         │                  │
                     │  │ HTTP Service │─────────┘                  │
                     │  │    :8100     │                            │
                     │  └──────────────┘                            │
                     │                                               │
                     │  Git repo: /Users/you/Git/YouLab             │
                     └──────────────────────────────────────────────┘
```

## Historical Context (from thoughts/)

- `thoughts/shared/plans/2025-12-26-youlab-technical-foundation.md` - Technical foundation explicitly scopes out production deployment for pilot
- `thoughts/shared/research/2025-12-31-open-webui-docker-setup.md` - Docker setup research
- `thoughts/shared/research/2026-01-04-documentation-launch-instructions-assessment.md` - Recent launch instructions assessment

**Key Decision**: "Production deployment infrastructure (staying local for pilot)" was explicitly excluded from Phase 1-2 scope.

## Recommendation

**For a team of 4 testing during active development**: Use **Option A (Shared Dev Server + Tunnel)**

Reasoning:
1. Zero infrastructure changes needed
2. Development workflow stays exactly the same
3. Branch testing is trivial (git checkout + restart)
4. Can migrate to Option B/C later when needed
5. Pilot phase explicitly scoped local-only

**Suggested setup**:
1. Use Tailscale (free for personal use, 100 devices)
2. Team members join your Tailscale network
3. Share the Tailscale hostname: `your-machine.tailnet.ts.net:3000`
4. Keep developing normally; team tests in real-time

**For branch testing**:
1. Create a simple script to switch branches and restart services
2. Or: Run a second instance on different ports for "staging"

## Open Questions

1. Should the HTTP Service be containerized for easier deployment?
2. Is persistent agent state important (Letta volumes) or can team start fresh?
3. Should authentication be implemented before team access?
4. Would team benefit from separate Langfuse project for tracing visibility?

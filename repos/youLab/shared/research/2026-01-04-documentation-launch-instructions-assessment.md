---
date: 2026-01-04T16:48:29+07:00
researcher: ariasulin
git_commit: 6f90b14dc7563475fc38c2e81a89b6c7c8ebafa9
branch: main
repository: YouLab
topic: "Assessment of Launch/Docker Documentation Clarity"
tags: [research, documentation, docker, launch, openwebui, letta]
status: complete
last_updated: 2026-01-04
last_updated_by: ariasulin
---

# Research: Assessment of Launch/Docker Documentation Clarity

**Date**: 2026-01-04T16:48:29+07:00
**Researcher**: ariasulin
**Git Commit**: 6f90b14dc7563475fc38c2e81a89b6c7c8ebafa9
**Branch**: main
**Repository**: YouLab

## Research Question

How clear are the current instructions in CLAUDE.md and /docs about launching the full service stack (OpenWebUI, Letta Server, HTTP Service) with Docker?

## Summary

The current documentation has **significant gaps** for launching the full service stack. While individual component documentation is good, there's no unified "launch everything" guide. A developer starting fresh would struggle to get the complete system running without trial and error.

## Detailed Findings

### What's Currently Documented

| Location | What It Covers | What's Missing |
|----------|----------------|----------------|
| `CLAUDE.md` | Basic commands, `uv run letta-server` | No Docker instructions for OpenWebUI |
| `docs/Quickstart.md` | Letta Docker, HTTP service, basic API usage | OpenWebUI setup, full stack orchestration |
| `docs/Development.md` | `pip install letta && letta server` | Docker approach for Letta |
| `docs/Architecture.md` | System flow diagrams | Launch instructions |
| `docs/Pipeline.md` | OpenWebUI pipe integration | How to get OpenWebUI running first |

### Critical Gaps

#### 1. No Unified Launch Guide

There's no document that says: "To run YouLab end-to-end, do these 4 things in order."

The actual launch sequence learned from this session:
```bash
# 1. Start OpenWebUI + Ollama (from existing docker-compose)
cd OpenWebUI/open-webui && docker compose up -d

# 2. Start Letta Server (Docker)
docker start letta  # or docker run... if first time

# 3. Start HTTP Service (host)
uv run letta-server

# 4. (Optional) Unpause containers if paused
docker unpause open-webui ollama
```

None of this is documented together.

#### 2. OpenWebUI Docker Setup Undiscoverable

- OpenWebUI exists at `OpenWebUI/open-webui/` with working `docker-compose.yaml`
- This is **never mentioned** in CLAUDE.md or /docs
- A research doc exists in `thoughts/shared/research/2025-12-31-open-webui-docker-setup.md` with excellent info, but it's not linked anywhere

#### 3. Letta Docker Image Name Mismatch

- `docs/Quickstart.md` uses: `lettaai/letta:latest`
- Actual running container uses: `letta/letta:latest`
- This would cause confusion for anyone following the docs

#### 4. Port Conflicts Not Documented

- OpenWebUI runs on port 3000
- Docs server (`npx serve docs`) defaults to 3000
- No warning about this conflict

#### 5. Container State Issues Not Covered

- Containers can be paused (not stopped)
- `docker ps` shows them but they're unreachable
- Need `docker unpause` — not intuitive

### What Exists But Is Fragmented

| Info | Location | Issue |
|------|----------|-------|
| OpenWebUI docker-compose | `OpenWebUI/open-webui/docker-compose.yaml` | Not referenced in main docs |
| Full Docker research | `thoughts/shared/research/2025-12-31-open-webui-docker-setup.md` | Not in /docs, not linked |
| Letta Docker run | `docs/Quickstart.md:30-37` | Wrong image name |
| Port 8100 for HTTP service | `docs/HTTP-Service.md:9-11` | Clear |
| Port 8283 for Letta | `docs/Quickstart.md:39-44` | Clear |

## Full Service Stack (As Learned This Session)

| Service | Container Name | Port | How to Start |
|---------|----------------|------|--------------|
| OpenWebUI | `open-webui` | 3000 | `cd OpenWebUI/open-webui && docker compose up -d` |
| Ollama | `ollama` | 11434 | (started with OpenWebUI compose) |
| Letta Server | `letta` | 8283 | `docker run -d --name letta -p 8283:8283 letta/letta:latest` |
| HTTP Service | (host process) | 8100 | `uv run letta-server` |
| Docs | (host process) | 3001 | `npx serve docs -p 3001` |

## Code References

- `CLAUDE.md:33-50` - Commands section (lacks Docker details)
- `docs/Quickstart.md:26-44` - Letta Docker (wrong image name)
- `docs/Development.md:204-207` - Uses pip install approach
- `OpenWebUI/open-webui/docker-compose.yaml` - Undocumented OpenWebUI setup

## Architecture Documentation

The architecture docs are excellent at explaining *what* the system does but not *how* to run it:
- `docs/Architecture.md` - Great flow diagrams
- `docs/README.md` - Good overview with status table

## Historical Context (from thoughts/)

- `thoughts/shared/research/2025-12-31-open-webui-docker-setup.md` - Comprehensive Docker guide that should be integrated into main docs

## Open Questions

1. Should there be a root-level `docker-compose.yaml` that orchestrates everything?
2. Should the OpenWebUI research doc be promoted to `/docs/Docker-Setup.md`?
3. Should CLAUDE.md include a "Quick Launch" section with Docker commands?

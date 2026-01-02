---
date: 2025-12-31T16:59:35-08:00
researcher: ariasulin
git_commit: 9edb6a07380fd0c3e5cf1b011ef6199a225d96e4
branch: main
repository: YouLab
topic: "Running Open WebUI in Docker for local testing"
tags: [research, open-webui, docker, local-development]
status: complete
last_updated: 2025-12-31
last_updated_by: ariasulin
---

# Research: Running Open WebUI in Docker for Local Testing

**Date**: 2025-12-31T16:59:35-08:00
**Researcher**: ariasulin
**Git Commit**: 9edb6a07380fd0c3e5cf1b011ef6199a225d96e4
**Branch**: main
**Repository**: YouLab

## Research Question

How to start Open WebUI in Docker for local testing.

## Summary

Open WebUI is already checked out in this repository at `OpenWebUI/open-webui/` with a working docker-compose configuration. You can start it immediately with a single command. The setup includes Ollama for local LLM inference and Open WebUI as the chat frontend.

## Quick Start (Recommended)

### Option 1: Use Existing Repository Setup

```bash
cd OpenWebUI/open-webui
docker compose up -d
```

Access at: http://localhost:3000

### Option 2: Standalone Docker Command (OpenAI API Only)

```bash
docker run -d -p 3000:8080 \
  -e OPENAI_API_KEY="$OPENAI_API_KEY" \
  -v open-webui:/app/backend/data \
  --name open-webui \
  --restart always \
  ghcr.io/open-webui/open-webui:main
```

## Detailed Findings

### Existing Configuration in This Repository

The project already includes Open WebUI checked out at `OpenWebUI/open-webui/` with these Docker files:

| File | Purpose |
|------|---------|
| `docker-compose.yaml` | Standard Ollama + Open WebUI stack |
| `docker-compose.api.yaml` | API-focused configuration |
| `docker-compose.gpu.yaml` | GPU-enabled setup |
| `docker-compose.amdgpu.yaml` | AMD GPU support |
| `docker-compose.data.yaml` | Data persistence focus |
| `docker-compose.otel.yaml` | OpenTelemetry observability |

### Main docker-compose.yaml Configuration

```yaml
services:
  ollama:
    image: ollama/ollama:${OLLAMA_DOCKER_TAG-latest}
    container_name: ollama
    volumes:
      - ollama:/root/.ollama
    restart: unless-stopped

  open-webui:
    image: ghcr.io/open-webui/open-webui:${WEBUI_DOCKER_TAG-main}
    container_name: open-webui
    volumes:
      - open-webui:/app/backend/data
    ports:
      - ${OPEN_WEBUI_PORT-3000}:8080
    environment:
      - 'OLLAMA_BASE_URL=http://ollama:11434'
    depends_on:
      - ollama
    extra_hosts:
      - host.docker.internal:host-gateway
    restart: unless-stopped

volumes:
  ollama: {}
  open-webui: {}
```

### Environment Variables

| Variable | Description | Default |
|----------|-------------|---------|
| `OPEN_WEBUI_PORT` | External port for web UI | `3000` |
| `WEBUI_DOCKER_TAG` | Open WebUI image tag | `main` |
| `OLLAMA_DOCKER_TAG` | Ollama image tag | `latest` |
| `OLLAMA_BASE_URL` | Internal Ollama URL | `http://ollama:11434` |
| `OPENAI_API_KEY` | OpenAI API key (optional) | - |
| `WEBUI_SECRET_KEY` | Session secret | auto-generated |

### Post-Startup Steps

1. **Pull an LLM model** (after containers are running):
   ```bash
   docker exec ollama ollama pull llama3.2
   ```

2. **First-time setup**:
   - Navigate to http://localhost:3000
   - Create an admin account (first user becomes admin)
   - Configure additional connections in Admin Panel > Settings > Connections

### Adding Pipelines (for LettaStarter integration)

To integrate with the LettaStarter HTTP service:

```yaml
services:
  pipelines:
    image: ghcr.io/open-webui/pipelines:main
    container_name: pipelines
    ports:
      - "9099:9099"
    volumes:
      - pipelines:/app/pipelines
    extra_hosts:
      - host.docker.internal:host-gateway
    environment:
      - PIPELINES_API_KEY=your-secure-key
    restart: unless-stopped
```

Then configure in Open WebUI:
1. Admin Panel > Settings > Connections
2. Add new connection with URL: `http://pipelines:9099`

### Useful Commands

```bash
# Start the stack
cd OpenWebUI/open-webui && docker compose up -d

# View logs
docker compose logs -f

# Stop the stack
docker compose down

# Update to latest images
docker compose pull && docker compose up -d

# Pull a model
docker exec ollama ollama pull llama3.2

# List downloaded models
docker exec ollama ollama list
```

### Connecting to Host Services

If running Ollama or LettaStarter on the host (not in Docker):

```bash
docker run -d -p 3000:8080 \
  --add-host=host.docker.internal:host-gateway \
  -e OLLAMA_BASE_URL=http://host.docker.internal:11434 \
  -v open-webui:/app/backend/data \
  --name open-webui \
  ghcr.io/open-webui/open-webui:main
```

## Code References

- `OpenWebUI/open-webui/docker-compose.yaml` - Main Docker Compose configuration
- `OpenWebUI/open-webui/Dockerfile` - Container build definition
- `src/letta_starter/pipelines/letta_pipe.py` - Pipeline integration for Open WebUI

## Architecture Documentation

```
Docker Stack:
┌─────────────────────────────────────────┐
│  Browser → http://localhost:3000        │
└─────────────────┬───────────────────────┘
                  │
┌─────────────────▼───────────────────────┐
│  open-webui container (port 8080→3000)  │
│  - Chat UI                              │
│  - User management                      │
│  - Pipe extension system                │
└─────────────────┬───────────────────────┘
                  │
┌─────────────────▼───────────────────────┐
│  ollama container (port 11434)          │
│  - Local LLM inference                  │
│  - Model management                     │
└─────────────────────────────────────────┘

Optional:
┌─────────────────────────────────────────┐
│  pipelines container (port 9099)        │
│  - Custom pipeline logic                │
│  - External API integration             │
│  - LettaStarter connection              │
└─────────────────────────────────────────┘
```

## External Documentation

- [Open WebUI Quick Start](https://docs.openwebui.com/getting-started/quick-start/)
- [Open WebUI GitHub](https://github.com/open-webui/open-webui)
- [Pipelines Documentation](https://docs.openwebui.com/features/pipelines/)
- [Ollama Docker Guide](https://geshan.com.np/blog/2025/02/ollama-docker-compose/)

## Open Questions

- Should we create a project-level docker-compose.yaml that includes both Open WebUI and LettaStarter?
- What's the best way to configure the Pipe to connect to LettaStarter HTTP service?

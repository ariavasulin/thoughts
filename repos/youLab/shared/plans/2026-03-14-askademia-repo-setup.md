# Askademia Repo Setup — Implementation Plan

## Overview

Set up the Askademia monorepo using **git subtree** to house forks of Open Web UI and Open Terminal, plus a new FastAPI service (askademia-api). The goal is a single repo where two developers can clone once, run a few commands, and start coding with hot-reload on everything.

## Current State

- Fresh git repo at `/Users/ariasulin/Git/Askademia/` with no commits
- `open-webui/` and `open-terminal/` are full git clones (their own `.git` dirs) pointing at upstream remotes
- No GitHub remote for Askademia yet
- No `.gitignore`, no dev tooling, no docker-compose

## Desired End State

```
Askademia/
├── open-webui/          # git subtree from fork (SvelteKit + FastAPI)
├── open-terminal/       # git subtree from fork (FastAPI terminal API)
├── askademia-api/       # new FastAPI service (video stream + context filter endpoints)
├── docker-compose.yml   # production-like setup (all 3 services)
├── docker-compose.dev.yml  # dev overrides (volume mounts, hot-reload)
├── Makefile             # convenience commands
├── .gitignore
└── README.md
```

**Verification**: After completing this plan:
1. `git clone <askademia-repo>` gets everything in one step
2. `make dev` starts all services with hot-reload
3. Open Web UI frontend at localhost:5173 with HMR
4. Open Web UI backend at localhost:8080 with auto-reload
5. Askademia API at localhost:8000 with auto-reload
6. Open Terminal at localhost:8888 (if needed)

## What We're NOT Doing

- Building the askademia-api endpoints (video stream, context filter) — just scaffolding
- Setting up CI/CD
- Production deployment
- Database migrations or Postgres setup (SQLite for dev)
- Modifying Open Web UI or Open Terminal source code

## Implementation Approach

Use git subtree to pull in forks of both upstream projects. This keeps all code in one repo, lets contributors clone once, and still allows pulling upstream changes. The dev workflow runs services natively (no Docker) for fast hot-reload.

---

## Phase 1: GitHub Setup & Git Subtree Integration

### Overview
Create GitHub forks, set up the Askademia repo, and integrate both projects as git subtrees.

### Steps:

#### 1. Create GitHub Forks
- Fork `open-webui/open-webui` to your GitHub account
- Fork `open-webui/open-terminal` to your GitHub account

#### 2. Clean Up Existing Clones
The current `open-webui/` and `open-terminal/` directories are standalone git repos. They need to be removed before adding as subtrees.

```bash
cd /Users/ariasulin/Git/Askademia

# Remove existing clones (their git history is upstream's, we'll get it via subtree)
rm -rf open-webui open-terminal
```

#### 3. Create Initial Commit
```bash
# Create .gitignore
cat > .gitignore << 'EOF'
# Python
__pycache__/
*.py[cod]
*.egg-info/
.venv/
venv/
.env

# Node
node_modules/
.svelte-kit/
build/

# IDE
.idea/
.vscode/
*.swp

# OS
.DS_Store
Thumbs.db

# Data
*.db
*.sqlite
EOF

git add .gitignore
git commit -m "Initial commit"
```

#### 4. Create GitHub Repo & Push
```bash
gh repo create ariasulin/Askademia --private --source=. --push
```

#### 5. Add Subtrees
```bash
# Add your forks as remotes
git remote add open-webui-fork https://github.com/<your-username>/open-webui.git
git remote add open-terminal-fork https://github.com/<your-username>/open-terminal.git

# Also add upstream remotes for pulling upstream changes later
git remote add open-webui-upstream https://github.com/open-webui/open-webui.git
git remote add open-terminal-upstream https://github.com/open-webui/open-terminal.git

# Add as subtrees (--squash compresses upstream history into one commit)
git subtree add --prefix=open-webui open-webui-fork main --squash
git subtree add --prefix=open-terminal open-terminal-fork main --squash
```

#### 6. Push
```bash
git push origin main
```

### Cheat Sheet for Future Upstream Syncing

```bash
# Pull latest upstream changes into your subtree
git subtree pull --prefix=open-webui open-webui-upstream main --squash

# Push your modifications back to your fork
git subtree push --prefix=open-webui open-webui-fork my-feature-branch
```

### Success Criteria:

#### Automated Verification:
- [ ] `git remote -v` shows 5 remotes: origin, open-webui-fork, open-terminal-fork, open-webui-upstream, open-terminal-upstream
- [ ] `ls open-webui/src open-webui/backend open-terminal/open_terminal` all exist
- [ ] `git log --oneline` shows initial commit + two subtree merge commits
- [ ] `git status` is clean

#### Manual Verification:
- [ ] GitHub repo exists and is accessible
- [ ] Collaborator (your friend) can clone and see everything

---

## Phase 2: Askademia API Scaffold

### Overview
Create the minimal FastAPI service structure so it's ready for you to add endpoints.

### Changes Required:

#### 1. Create `askademia-api/` directory structure

```
askademia-api/
├── pyproject.toml
├── askademia/
│   ├── __init__.py
│   └── main.py
└── dev.sh
```

**File**: `askademia-api/pyproject.toml`
```toml
[project]
name = "askademia-api"
version = "0.1.0"
description = "Askademia API service"
requires-python = ">=3.11"
dependencies = [
    "fastapi>=0.115.0",
    "uvicorn[standard]>=0.34.0",
    "httpx>=0.27.0",
    "python-multipart>=0.0.22",
]

[build-system]
requires = ["hatchling"]
build-backend = "hatchling.build"
```

**File**: `askademia-api/askademia/main.py`
```python
from fastapi import FastAPI
from fastapi.middleware.cors import CORSMiddleware

app = FastAPI(title="Askademia API", version="0.1.0")

app.add_middleware(
    CORSMiddleware,
    allow_origins=["http://localhost:5173", "http://localhost:8080"],
    allow_credentials=True,
    allow_methods=["*"],
    allow_headers=["*"],
)


@app.get("/health")
async def health():
    return {"status": "ok"}


# TODO: POST /video-stream — send video stream link to Open Web UI
# TODO: POST /context — send context to be injected via filter
```

**File**: `askademia-api/dev.sh`
```bash
#!/bin/bash
PORT="${PORT:-8000}"
uvicorn askademia.main:app --port $PORT --host 0.0.0.0 --reload
```

### Success Criteria:

#### Automated Verification:
- [ ] `cd askademia-api && pip install -e . && sh dev.sh` starts server on port 8000
- [ ] `curl http://localhost:8000/health` returns `{"status":"ok"}`

---

## Phase 3: Makefile & Dev Workflow

### Overview
Create a Makefile with convenience commands so both developers can get started quickly.

### Changes Required:

**File**: `Makefile`
```makefile
.PHONY: setup dev dev-frontend dev-backend dev-api dev-terminal help

help: ## Show this help
	@grep -E '^[a-zA-Z_-]+:.*?## .*$$' $(MAKEFILE_LIST) | sort | awk 'BEGIN {FS = ":.*?## "}; {printf "\033[36m%-20s\033[0m %s\n", $$1, $$2}'

# ── Setup ──────────────────────────────────────────────

setup: ## First-time setup: install all dependencies
	@echo "==> Installing Open Web UI frontend dependencies..."
	cd open-webui && npm install
	@echo ""
	@echo "==> Installing Open Web UI backend dependencies..."
	cd open-webui/backend && pip install -r requirements.txt
	@echo ""
	@echo "==> Installing Askademia API dependencies..."
	cd askademia-api && pip install -e .
	@echo ""
	@echo "==> Done! Run 'make dev' to start all services."

# ── Dev Servers (run each in a separate terminal) ──────

dev-frontend: ## Start Open Web UI frontend (port 5173, hot-reload)
	cd open-webui && npm run dev

dev-backend: ## Start Open Web UI backend (port 8080, auto-reload)
	cd open-webui/backend && sh dev.sh

dev-api: ## Start Askademia API (port 8000, auto-reload)
	cd askademia-api && sh dev.sh

dev-terminal: ## Start Open Terminal (port 8888)
	cd open-terminal && uvicorn open_terminal.main:app --port 8888 --host 0.0.0.0 --reload

# ── Upstream Sync ──────────────────────────────────────

sync-webui: ## Pull latest upstream Open Web UI changes
	git subtree pull --prefix=open-webui open-webui-upstream main --squash

sync-terminal: ## Pull latest upstream Open Terminal changes
	git subtree pull --prefix=open-terminal open-terminal-upstream main --squash
```

**File**: `README.md`
```markdown
# Askademia

## Quick Start

### Prerequisites
- Python 3.11+
- Node.js 18-22
- npm 6+

### Setup
git clone https://github.com/<your-username>/Askademia.git
cd Askademia
python3 -m venv .venv && source .venv/bin/activate
make setup

### Development
Run each in a separate terminal (or use tmux):

make dev-backend    # Open Web UI backend → localhost:8080
make dev-frontend   # Open Web UI frontend → localhost:5173 (HMR)
make dev-api        # Askademia API → localhost:8000

### Syncing Upstream
make sync-webui     # Pull latest Open Web UI changes
make sync-terminal  # Pull latest Open Terminal changes
```

### Success Criteria:

#### Automated Verification:
- [ ] `make help` prints available commands
- [ ] `make setup` installs all dependencies without errors

#### Manual Verification:
- [ ] `make dev-backend` + `make dev-frontend` in separate terminals loads Open Web UI at localhost:5173
- [ ] Editing a `.svelte` file in `open-webui/src/` reflects in browser within seconds
- [ ] `make dev-api` starts askademia-api and `/health` responds

**Implementation Note**: After completing this phase, pause for manual confirmation that the dev workflow works end-to-end before proceeding.

---

## Phase 4: Docker Compose (Production-like)

### Overview
Docker Compose for when you eventually want to run everything containerized (deployment prep, CI testing). Not needed for day-to-day dev.

### Changes Required:

**File**: `docker-compose.yml`
```yaml
services:
  open-webui:
    build:
      context: ./open-webui
    ports:
      - "8080:8080"
    volumes:
      - open-webui-data:/app/backend/data
    environment:
      - WEBUI_SECRET_KEY=dev-secret-change-me
    restart: unless-stopped

  askademia-api:
    build:
      context: ./askademia-api
      dockerfile: Dockerfile
    ports:
      - "8000:8000"
    restart: unless-stopped

  open-terminal:
    build:
      context: ./open-terminal
    ports:
      - "8888:8888"
    restart: unless-stopped

volumes:
  open-webui-data:
```

**File**: `askademia-api/Dockerfile`
```dockerfile
FROM python:3.11-slim

WORKDIR /app
COPY pyproject.toml .
COPY askademia/ askademia/

RUN pip install --no-cache-dir .

EXPOSE 8000
CMD ["uvicorn", "askademia.main:app", "--host", "0.0.0.0", "--port", "8000"]
```

### Success Criteria:

#### Automated Verification:
- [ ] `docker compose build` completes without errors
- [ ] `docker compose up` starts all 3 services

#### Manual Verification:
- [ ] Open Web UI accessible at localhost:8080
- [ ] Askademia API health check at localhost:8000/health
- [ ] Open Terminal accessible at localhost:8888

---

## Summary of Dev Workflow

### Daily Development (no Docker)
```
Terminal 1: make dev-backend     # Open Web UI Python backend (port 8080)
Terminal 2: make dev-frontend    # SvelteKit with HMR (port 5173)
Terminal 3: make dev-api         # Your Askademia FastAPI (port 8000)
```

- Frontend edits: instant HMR via Vite at localhost:5173
- Backend edits: auto-reload via uvicorn `--reload`
- SQLite database: stored in `open-webui/backend/data/`
- No Docker needed for development

### Pulling Upstream Changes
```
make sync-webui      # git subtree pull from upstream
make sync-terminal   # git subtree pull from upstream
```

### Onboarding a New Developer
```
git clone https://github.com/<username>/Askademia.git
cd Askademia
python3 -m venv .venv && source .venv/bin/activate
make setup
# Start coding
```

## References

- Open Web UI dev setup: https://docs.openwebui.com/getting-started/advanced-topics/development/
- Open Web UI dev discussion: https://github.com/open-webui/open-webui/discussions/8288
- Git subtree docs: https://www.atlassian.com/git/tutorials/git-subtree

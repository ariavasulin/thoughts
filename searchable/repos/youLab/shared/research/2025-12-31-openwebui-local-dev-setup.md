---
date: 2025-12-31T14:52:07-08:00
researcher: ariasulin
git_commit: 9edb6a07380fd0c3e5cf1b011ef6199a225d96e4
branch: main
repository: YouLab
topic: "OpenWebUI Local Development Setup for YouLab Integration"
tags: [research, openwebui, local-dev, fastapi, integration, pipelines]
status: complete
last_updated: 2025-12-31
last_updated_by: ariasulin
---

# Research: OpenWebUI Local Development Setup for YouLab Integration

**Date**: 2025-12-31T14:52:07-08:00
**Researcher**: ariasulin
**Git Commit**: 9edb6a07380fd0c3e5cf1b011ef6199a225d96e4
**Branch**: main
**Repository**: YouLab

## Research Question

How to set up a local OpenWebUI development environment to create students and interact with the YouLab HTTP server?

## Summary

OpenWebUI is a SvelteKit frontend + FastAPI backend application that can connect to OpenAI-compatible APIs. To integrate with the YouLab FastAPI server:

1. **Local dev setup**: Run backend via `./backend/dev.sh` (port 8080) and frontend via `npm run dev` (port 5173)
2. **Create students**: First user gets admin role automatically; subsequent users can be created via signup or admin API
3. **Connect to YouLab**: Configure OpenWebUI to use the YouLab HTTP server as an "OpenAI-compatible" backend via environment variables or admin UI

## Quick Start Guide

### Prerequisites

- **Python**: >= 3.11, < 3.13
- **Node.js**: >= 18.13.0, <= 22.x.x
- **npm**: >= 6.0.0

### Step 1: Install Dependencies

```bash
cd OpenWebUI/open-webui

# Backend
cd backend
pip install -r requirements.txt

# Frontend
cd ..
npm install
```

### Step 2: Configure Environment

Create `.env` in the OpenWebUI root:

```bash
# Point to YouLab HTTP server
OPENAI_API_BASE_URL='http://localhost:8000'
OPENAI_API_KEY='your-api-key-if-needed'

# CORS for local dev
CORS_ALLOW_ORIGIN='http://localhost:5173;http://localhost:8080'

# Disable telemetry
SCARF_NO_ANALYTICS=true
DO_NOT_TRACK=true
ANONYMIZED_TELEMETRY=false
```

### Step 3: Start Development Servers

**Terminal 1 - Backend** (port 8080):
```bash
cd OpenWebUI/open-webui/backend
./dev.sh
```

**Terminal 2 - Frontend** (port 5173):
```bash
cd OpenWebUI/open-webui
npm run dev
```

### Step 4: Create First User (Admin)

1. Navigate to `http://localhost:5173`
2. Click "Sign up" (or navigate to signup page)
3. First user created automatically becomes **admin**
4. Signup is auto-disabled after first user is created

### Step 5: Configure YouLab Connection

**Option A: Via Admin UI**
1. Login as admin
2. Go to Settings → Connections → OpenAI API
3. Click "Add Connection"
4. URL: `http://localhost:8000` (YouLab server)
5. Auth Type: Bearer (if YouLab requires auth)
6. Enable the connection

**Option B: Via Environment Variables**
```bash
OPENAI_API_BASE_URL='http://localhost:8000'
OPENAI_API_KEY='optional-key'
```

---

## Detailed Findings

### 1. Local Development Setup

#### Backend Configuration

**Dev script**: `OpenWebUI/open-webui/backend/dev.sh:1-3`
```bash
export CORS_ALLOW_ORIGIN="http://localhost:5173;http://localhost:8080"
PORT="${PORT:-8080}"
uvicorn open_webui.main:app --port $PORT --host 0.0.0.0 --forwarded-allow-ips '*' --reload
```

**Python requirements**: `OpenWebUI/open-webui/pyproject.toml:122`
- Python >= 3.11, < 3.13
- FastAPI 0.126.0
- uvicorn 0.37.0
- SQLAlchemy 2.0.45
- Default SQLite database at `backend/data/webui.db`

**CLI entry points**: `OpenWebUI/open-webui/backend/open_webui/__init__.py`
- `open-webui serve` - Production mode
- `open-webui dev` - Development mode with hot reload

#### Frontend Configuration

**Node requirements**: `OpenWebUI/open-webui/package.json:149-152`
- Node.js >= 18.13.0, <= 22.x.x
- npm >= 6.0.0

**Dev scripts**: `OpenWebUI/open-webui/package.json:5-12`
- `npm run dev` - Starts Vite on port 5173 with host binding
- `npm run dev:5050` - Alternative port 5050
- `npm run build` - Production build

**Pre-build step**: Downloads Pyodide files before dev/build via `scripts/prepare-pyodide.js`

#### Database Setup

OpenWebUI uses dual migration systems:

1. **Peewee migrations** (legacy): `OpenWebUI/open-webui/backend/open_webui/internal/migrations/`
2. **Alembic migrations** (current): `OpenWebUI/open-webui/backend/open_webui/migrations/versions/`

Database is auto-created on first run at `backend/data/webui.db` (SQLite) or configurable via:
```bash
DATABASE_URL='postgresql://user:pass@host:port/db'
```

---

### 2. User/Student Creation

#### First User (Admin)

**File**: `OpenWebUI/open-webui/backend/open_webui/routers/auths.py:677`

The first user to sign up automatically becomes admin:
```python
role = "admin" if not has_users else request.app.state.config.DEFAULT_USER_ROLE
```

After first user creation, signup is auto-disabled (`auths.py:729-731`).

#### Subsequent Users

**Option 1: Enable Signup**
Set environment variable:
```bash
ENABLE_SIGNUP=True
DEFAULT_USER_ROLE=user  # or "pending" for approval workflow
```

**Option 2: Admin Creates Users**
Admin can create users via:
- **API**: `POST /api/v1/auths/add` with email, password, name, role
- **UI**: Admin Settings → Users → Add User

#### User Roles

Three roles exist (`OpenWebUI/open-webui/backend/open_webui/models/users.py:51`):
- `admin` - Full system access
- `user` - Standard verified user
- `pending` - Awaiting admin approval

#### API Endpoints for User Management

| Endpoint | Method | Auth | Description |
|----------|--------|------|-------------|
| `/api/v1/auths/signup` | POST | None | User registration |
| `/api/v1/auths/signin` | POST | None | Password login |
| `/api/v1/auths/add` | POST | Admin | Create user (admin only) |
| `/api/v1/users/` | GET | Admin | List users |
| `/api/v1/users/{user_id}/update` | POST | Admin | Update user |

---

### 3. Connecting to YouLab HTTP Server

#### Environment Variables

**File**: `OpenWebUI/open-webui/backend/open_webui/config.py:1042-1090`

```bash
# Single connection
OPENAI_API_BASE_URL='http://localhost:8000'
OPENAI_API_KEY='optional-key'

# Multiple connections (semicolon-separated)
OPENAI_API_BASE_URLS='http://localhost:8000;https://api.openai.com/v1'
OPENAI_API_KEYS='youlab-key;openai-key'
```

#### Admin UI Configuration

**File**: `OpenWebUI/open-webui/src/lib/components/admin/Settings/Connections.svelte`

Via admin UI at Settings → Connections → OpenAI API:
- Add connection with URL and API key
- Configure auth type (bearer, none, session)
- Optionally specify model IDs to expose
- Add prefix to model names to avoid conflicts

#### Connection Config Options

Each connection supports:
- `enable` - Toggle connection on/off
- `auth_type` - `'bearer'` (default), `'none'`, `'session'`
- `prefix_id` - Prefix added to model names
- `model_ids` - Explicit list of models (empty = auto-discover)
- `headers` - Custom HTTP headers (JSON)
- `azure` - Azure OpenAI mode (auto-detected from URL)

#### YouLab Server Requirements

For OpenWebUI to connect, the YouLab FastAPI server must implement:

1. **`GET /models`** - Return OpenAI-compatible model list:
```json
{
  "data": [
    {"id": "tutor", "object": "model", "owned_by": "youlab"}
  ]
}
```

2. **`POST /chat/completions`** - Accept OpenAI-compatible chat format:
```json
{
  "model": "tutor",
  "messages": [{"role": "user", "content": "Hello"}],
  "stream": true
}
```

---

### 4. Pipes/Pipelines Integration (Alternative Approach)

OpenWebUI has a Pipes system for custom chat handlers:

#### What is a Pipe?

**File**: `OpenWebUI/open-webui/backend/open_webui/utils/plugin.py:149-156`

A Pipe is a Python class that replaces the model backend:
```python
class Pipe:
    class Valves(BaseModel):
        YOULAB_URL: str = "http://localhost:8000"

    def __init__(self):
        self.valves = self.Valves()

    def pipe(self, body: dict) -> str:
        # Custom logic to call YouLab server
        return response
```

#### When to Use Pipes vs OpenAI Config

| Approach | Use When |
|----------|----------|
| OpenAI-compatible config | YouLab exposes `/models` and `/chat/completions` endpoints |
| Custom Pipe | Need custom request transformation or non-OpenAI protocol |

#### Creating a Pipe

1. Go to Workspace → Functions → Create
2. Select "Pipe" type
3. Implement `pipe(body: dict)` method
4. Configure Valves for settings like URL
5. Pipe appears as a model in chat

---

## Code References

### Configuration Files
- `OpenWebUI/open-webui/pyproject.toml` - Python dependencies
- `OpenWebUI/open-webui/package.json` - Node dependencies
- `OpenWebUI/open-webui/.env.example` - Environment template
- `OpenWebUI/open-webui/backend/open_webui/config.py` - PersistentConfig system

### Startup Scripts
- `OpenWebUI/open-webui/backend/dev.sh` - Backend dev server
- `OpenWebUI/open-webui/backend/start.sh` - Backend production server

### Authentication
- `OpenWebUI/open-webui/backend/open_webui/routers/auths.py` - Auth endpoints
- `OpenWebUI/open-webui/backend/open_webui/models/users.py` - User model
- `OpenWebUI/open-webui/backend/open_webui/utils/auth.py` - JWT/password handling

### OpenAI Integration
- `OpenWebUI/open-webui/backend/open_webui/routers/openai.py` - API routing
- `OpenWebUI/open-webui/backend/open_webui/utils/chat.py` - Chat completion flow
- `OpenWebUI/open-webui/src/lib/components/admin/Settings/Connections.svelte` - Admin UI

### Pipes System
- `OpenWebUI/open-webui/backend/open_webui/functions.py` - Pipe execution
- `OpenWebUI/open-webui/backend/open_webui/utils/plugin.py` - Module loading
- `OpenWebUI/open-webui/backend/open_webui/routers/pipelines.py` - Pipeline router

---

## Architecture Documentation

### Request Flow: Chat to YouLab

```
1. User sends message in OpenWebUI chat
   ↓
2. POST /api/chat/completions (OpenWebUI backend)
   ↓
3. generate_chat_completion() routes based on model type
   ↓
4. If OpenAI-compatible model (YouLab):
   - Get URL/key from config
   - Transform request if needed
   - POST {youlab_url}/chat/completions
   ↓
5. Stream response back to frontend
```

### Authentication Flow

```
1. User navigates to OpenWebUI
   ↓
2. If first user: Sign up → becomes admin
   ↓
3. JWT token created, stored in httponly cookie
   ↓
4. Subsequent requests include token
   ↓
5. get_current_user() dependency validates token
```

### Configuration Persistence

```
1. Environment variables loaded at startup
   ↓
2. PersistentConfig checks database for overrides
   ↓
3. If ENABLE_PERSISTENT_CONFIG=true, database wins
   ↓
4. Admin UI updates → saved to database
   ↓
5. Changes reflected immediately via app.state.config
```

---

## Open Questions

1. **YouLab API Schema**: Does the current YouLab HTTP server at `src/letta_starter/server/main.py` implement OpenAI-compatible `/models` and `/chat/completions` endpoints?

2. **User Identity**: How will OpenWebUI user identity (email, user_id) be passed to YouLab for per-student agent routing? Options:
   - OpenWebUI can forward `user` object in pipeline mode
   - Custom Pipe can extract and forward user info
   - Custom header injection via connection config

3. **Authentication**: Does YouLab need API key auth, or will it trust requests from OpenWebUI?

4. **Streaming**: Does YouLab's chat endpoint support streaming responses (`text/event-stream`)?

---

## Related Research

- `thoughts/shared/plans/2025-12-26-youlab-technical-foundation.md` - Full implementation plan
- `thoughts/shared/youlab-project-context.md` - Architecture decisions and next steps

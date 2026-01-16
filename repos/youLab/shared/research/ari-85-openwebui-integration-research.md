# OpenWebUI User/Course Assignment Research

**Ticket**: ARI-85
**Research Date**: 2026-01-16
**Author**: Claude (Sonnet 4.5)

## Executive Summary

This research investigates how courses are assigned to users in the YouLab platform's OpenWebUI integration. Key findings:

1. **Admin User Found**: `ea1ac2d7-9442-400f-b1ba-759891b3f015` (ariav2002@gmail.com, role: admin)
2. **Course Assignment**: Currently done via global `AGENT_TYPE` valve in letta_pipe.py
3. **No Per-User Course Assignment**: System lacks dynamic user->course mapping
4. **Pipeline Integration**: OpenWebUI → letta_pipe.py → YouLab HTTP Service → Letta

---

## 1. OpenWebUI User Management

### Admin User Discovery

**Database Location**: `OpenWebUI/open-webui/backend/data/webui.db`

**Admin User**:
- ID: `ea1ac2d7-9442-400f-b1ba-759891b3f015`
- Email: `ariav2002@gmail.com`
- Name: `Ari Asulin`
- Role: `admin`

### User Model Structure

**File**: `OpenWebUI/open-webui/backend/open_webui/models/users.py`

```python
class User(Base):
    id = Column(String, primary_key=True)
    email = Column(String)
    username = Column(String(50), nullable=True)
    role = Column(String)  # "admin", "user", "pending"
    name = Column(String)

    # Extensible JSON fields
    info = Column(JSON, nullable=True)      # Custom metadata
    settings = Column(JSON, nullable=True)  # UI/tool settings
    oauth = Column(JSON, nullable=True)     # OAuth providers

    # Timestamps
    last_active_at = Column(BigInteger)
    created_at = Column(BigInteger)
    updated_at = Column(BigInteger)
```

**Key Capabilities**:
- Users identified by UUID `id` field
- JSON `info` and `settings` fields allow arbitrary metadata storage
- Role-based access: first user auto-assigned "admin" role
- No dedicated `course_id` field currently exists

### Authentication Flow

1. **First User**: Auto-assigned "admin" role (auths.py:677)
2. **Subsequent Users**: Get `DEFAULT_USER_ROLE` (typically "user" or "pending")
3. **Admin Retrieval**: `Users.get_super_admin_user()` returns first admin user

---

## 2. Pipeline Integration Architecture

### Data Flow

```
OpenWebUI Frontend
    ↓ (__user__, __metadata__, message)
letta_pipe.py (Pipe class)
    ↓ Extract user_id, user_name, chat_id
HTTP POST /agents (ensure agent exists)
HTTP POST /chat/stream
    ↓ (agent_id, message, chat_id, chat_title)
youlab_server/main.py
    ↓ Extract user_id from agent metadata
    ↓ set_user_context(agent_id, user_id)
    ↓ Persist to Honcho (optional)
Letta Agent
    ↓ Tools access user_id from context
    ↓ query_honcho, edit_memory_block, etc.
Stream SSE events back to OpenWebUI
```

### File: `src/youlab_server/pipelines/letta_pipe.py`

**Pipe Configuration (Valves)**:

```python
class Valves(BaseModel):
    LETTA_SERVICE_URL: str = "http://host.docker.internal:8100"
    AGENT_TYPE: str = "college-essay"  # ← Global course assignment
    ENABLE_LOGGING: bool = True
    ENABLE_THINKING: bool = True
```

**User Context Extraction**:

```python
async def pipe(
    body: dict[str, Any],
    __user__: dict[str, Any] | None,     # OpenWebUI user data
    __metadata__: dict[str, Any] | None, # Chat metadata
    __event_emitter__: Callable,        # Stream events back
):
    user_id = __user__.get("id")        # REQUIRED - UUID
    user_name = __user__.get("name")    # Display name
    chat_id = __metadata__.get("chat_id")
    chat_title = self._get_chat_title(chat_id)
```

**Agent Lookup/Creation** (lines 99-136):

1. Query existing agents: `GET /agents?user_id={user_id}`
2. Find agent matching `AGENT_TYPE` valve setting
3. If not found, create: `POST /agents` with `{user_id, agent_type, user_name}`

---

## 3. Course Assignment Mechanism

### Current Implementation: Global Valve

**Problem**: All OpenWebUI users get the same course via `AGENT_TYPE` valve.

**File**: `src/youlab_server/pipelines/letta_pipe.py:28-31`

```python
AGENT_TYPE: str = Field(
    default="college-essay",
    description="Agent type to use (course_id)",
)
```

This is a **global setting** - every user accessing through this pipe gets the same course.

### Agent Creation Flow

**File**: `src/youlab_server/server/agents.py`

```python
async def create_agent(
    user_id: str,
    agent_type: str = "tutor",
    user_name: str | None = None,
) -> str:
    # Map legacy "tutor" to "default" course
    course_id = "default" if agent_type == "tutor" else agent_type

    return await create_agent_from_curriculum(
        user_id=user_id,
        course_id=course_id,
        user_name=user_name,
    )
```

**Agent Naming Convention**:
- Name: `youlab_{user_id}_{agent_type}`
- Example: `youlab_ea1ac2d7-9442-400f-b1ba-759891b3f015_college-essay`

**Agent Metadata**:

```python
metadata = {
    "youlab_user_id": user_id,
    "youlab_agent_type": agent_type,
    "course_id": course_id,
    "course_version": course.version,
}
```

### Course Configuration Loading

**File**: `src/youlab_server/curriculum/loader.py`

**Location**: `config/courses/{course_id}/course.toml`

**Available Courses**:
- `college-essay` - Main college essay coaching course
- `default` - Default course
- Additional courses can be added as TOML files

**Example** (`config/courses/college-essay/course.toml`):

```toml
[agent]
id = "college-essay"
name = "College Essay Coaching"
version = "2.0.0"
model = "openai/gpt-4o"
embedding = "openai/text-embedding-3-small"
modules = ["01-first-impression", "02-topic-development", "03-drafting"]

[block.student]
label = "human"
shared = false
description = "Rich narrative understanding of who this student is"

[block.journey]
label = "journey"
shared = false
description = "Curriculum progress and grader state"
```

---

## 4. How Agent Selection Works Per User

### Agent-per-User-per-Course Model

Each user can have **multiple agents** (one per course):

```python
# Cache structure in AgentManager
_cache: dict[tuple[str, str], str] = {}  # (user_id, agent_type) -> agent_id
```

**Lookup Flow**:

1. Extract `user_id` from OpenWebUI's `__user__` parameter
2. Use pipe's `AGENT_TYPE` valve as the course identifier
3. Query: `GET /agents?user_id={user_id}`
4. Filter by: `agent.agent_type == AGENT_TYPE`
5. If not found: Create new agent with that course

**Multi-Course Support**:
- A user with ID `alice123` could have:
  - `youlab_alice123_college-essay` (college essay course)
  - `youlab_alice123_career-prep` (career prep course)
  - `youlab_alice123_default` (default course)

---

## 5. Missing Functionality: Per-User Course Assignment

### What Doesn't Exist

**No mechanism for**:
- Storing course assignment in OpenWebUI user record
- Looking up user's assigned course from database
- Dynamic course assignment based on user attributes
- Course assignment via user groups/cohorts
- Admin UI to assign courses to users

### How It Would Need to Work

**Option 1: Extend OpenWebUI User Model**

Add course assignment to user's `info` JSON field:

```python
user.info = {
    "youlab": {
        "course_id": "college-essay",
        "cohort": "2026-spring",
        "enrollment_date": "2026-01-15"
    }
}
```

**Option 2: Create Course Assignment Service**

```python
# New endpoint: /users/{user_id}/course
async def get_user_course(user_id: str) -> str:
    # Query OpenWebUI database or separate assignment table
    user = openwebui_db.get_user_by_id(user_id)
    return user.info.get("youlab", {}).get("course_id", "default")
```

**Option 3: Pipe Valve Per-User Override**

Extend pipe to check user-specific settings:

```python
# In letta_pipe.py
def get_user_agent_type(self, user_id: str) -> str:
    # Check OpenWebUI user settings/info
    user = Users.get_user_by_id(user_id)
    return user.settings.get("youlab_course", self.valves.AGENT_TYPE)
```

---

## 6. Assigning Course to Admin User

### Current State

The admin user (`ea1ac2d7-9442-400f-b1ba-759891b3f015`) currently:
- **Has no explicit course assignment** in OpenWebUI database
- **Will get the global AGENT_TYPE** ("college-essay") when accessing via pipe
- **Can be assigned any course** via direct API call to `/agents`

### How to Assign a Course (Manual)

**Method 1: Via HTTP API**

```bash
# Create agent for admin user with specific course
curl -X POST http://localhost:8100/agents \
  -H "Content-Type: application/json" \
  -d '{
    "user_id": "ea1ac2d7-9442-400f-b1ba-759891b3f015",
    "agent_type": "college-essay",
    "user_name": "Ari Asulin"
  }'
```

**Method 2: Via Pipe Valve (Global)**

1. Access OpenWebUI admin panel
2. Navigate to Pipe settings
3. Update `AGENT_TYPE` valve to desired course
4. Next time admin user chats, they'll get that course

**Method 3: Via User Settings (Requires Implementation)**

Store course in user's OpenWebUI settings:

```python
# Update user in OpenWebUI database
Users.update_user_settings_by_id(
    "ea1ac2d7-9442-400f-b1ba-759891b3f015",
    {"youlab_course": "college-essay"}
)
```

Then modify `letta_pipe.py` to read this setting.

---

## 7. Implementation Recommendations for ARI-85

### For Single-Module POC

**Simplest approach**: Use existing global `AGENT_TYPE` valve:

1. Set `AGENT_TYPE` valve to the new single-module course ID
2. All users (including admin) automatically get this course
3. No code changes needed

### For Multi-User Course Assignment

**Recommended approach**:

1. **Add course assignment to OpenWebUI user info**:
   ```python
   # In user initialization or admin UI
   Users.update_user_by_id(user_id, {
       "info": {"youlab_course": "college-essay"}
   })
   ```

2. **Extend letta_pipe.py to check user info**:
   ```python
   def _get_agent_type_for_user(self, user_id: str) -> str:
       try:
           from open_webui.models.users import Users
           user = Users.get_user_by_id(user_id)
           if user and user.info:
               return user.info.get("youlab_course", self.valves.AGENT_TYPE)
       except:
           pass
       return self.valves.AGENT_TYPE
   ```

3. **Update agent lookup**:
   ```python
   # In _ensure_agent_exists()
   agent_type = self._get_agent_type_for_user(user_id)
   ```

---

## 8. Key Files Reference

### OpenWebUI Integration
- `OpenWebUI/open-webui/backend/open_webui/models/users.py` - User model
- `OpenWebUI/open-webui/backend/open_webui/routers/users.py` - User routes
- `OpenWebUI/open-webui/backend/data/webui.db` - SQLite database

### YouLab Pipeline
- `src/youlab_server/pipelines/letta_pipe.py` - OpenWebUI pipe (lines 28-31: AGENT_TYPE valve)
- `src/youlab_server/server/agents.py` - Agent creation (lines 190-367)
- `src/youlab_server/server/main.py` - HTTP endpoints
- `src/youlab_server/server/schemas.py` - Request/response models

### Course Configuration
- `src/youlab_server/curriculum/__init__.py` - Curriculum registry
- `src/youlab_server/curriculum/loader.py` - TOML loader
- `src/youlab_server/curriculum/schema.py` - Course schema
- `config/courses/*/course.toml` - Course definitions

---

## Conclusion

**Current System**:
- Uses global `AGENT_TYPE` valve in pipe for all users
- No per-user course assignment mechanism
- Admin user will get whatever course is set in valve
- Agents cached per (user_id, agent_type) tuple

**To Assign Admin User to Course**:
1. Set `AGENT_TYPE` valve to desired course_id, OR
2. Call `/agents` API with admin's user_id and desired course, OR
3. Implement per-user course assignment in OpenWebUI user info

**For Production**:
- Recommend implementing user-level course assignment
- Store in OpenWebUI user `info` JSON field
- Extend pipe to check user info before falling back to valve
- Allows different users to be in different courses simultaneously

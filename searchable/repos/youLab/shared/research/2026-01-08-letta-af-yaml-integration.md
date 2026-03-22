---
date: 2026-01-08T11:46:03+07:00
researcher: Claude
git_commit: 864e1dcf90c15a7988ba7cc667c00680ee6edbf2
branch: main
repository: YouLab
topic: "Letta .AF AgentFile Format and YAML Configuration Integration"
tags: [research, letta, agentfile, yaml, configuration, agent-creation]
status: complete
last_updated: 2026-01-08
last_updated_by: Claude
---

# Research: Letta .AF AgentFile Format and YAML Configuration Integration

**Date**: 2026-01-08T11:46:03+07:00
**Researcher**: Claude
**Git Commit**: 864e1dcf90c15a7988ba7cc667c00680ee6edbf2
**Branch**: main
**Repository**: YouLab

## Research Question

How does the Letta .AF agentfile format work, and how can our YAML course configuration integrate with or compile to the AF format?

## Summary

The **.af (AgentFile)** format is Letta's open standard for serializing **stateful agents**—complete with conversation history, memory state, and tool source code. Your **YAML configuration** serves a different purpose: it's a **template/factory config** for creating new agents. These two formats are complementary rather than competing:

- **AF** = Portable snapshot of a running agent (includes state)
- **YAML** = Configuration template for creating new agents (stateless)

**Recommendation**: Keep YAML as your course configuration format and have it compile to Letta API calls (your current plan). AF integration is optional and would be useful only for importing pre-built agents or exporting agent snapshots.

## Detailed Findings

### Letta .AF AgentFile Format

The AF format was introduced by Letta (the MemGPT team) in April 2025 as an open standard for agent portability.

#### Schema Structure

The root `AgentFileSchema` contains:

```python
AgentFileSchema:
  agents: List[AgentSchema]         # Core agent definitions
  groups: List[GroupSchema]         # Agent groups
  blocks: List[BlockSchema]         # Memory blocks
  files: List[FileSchema]           # Associated files
  sources: List[SourceSchema]       # Data sources
  tools: List[ToolSchema]           # Tool definitions with source code
  mcp_servers: List[MCPServerSchema]  # MCP server configs
  metadata: dict                    # Export info
  created_at: datetime
```

Each `AgentSchema` includes:

| Field | Description |
|-------|-------------|
| `id` | Human-readable identifier |
| `name` | Agent name |
| `llm_config` | Model configuration (model name, context window) |
| `embedding_config` | Embedding model settings |
| `system` | System prompt |
| `core_memory` | Memory blocks (persona, human) |
| `messages` | Complete conversation history |
| `in_context_message_ids` | Which messages are in current context |
| `tools` | Tool definitions |
| `tool_rules` | Tool sequencing constraints |
| `tool_exec_environment_variables` | Env vars for tool execution |

#### What AF Captures

- **Model configuration**: Context window limits, model names
- **System prompt**: Initial behavioral instructions
- **Memory blocks**: Persona and human info blocks
- **Message history**: Complete chat with `in_context` tracking
- **Tools**: Full source code and JSON schemas
- **Tool rules**: Sequencing and constraints
- **Environment variables**: For tool execution

#### What AF Does NOT Capture (yet)

- Archival memory passages (on roadmap)

#### AF Import/Export

```python
# Import
client.agents.import_file(file=open("agent.af", "rb"))

# Export
client.agents.export_file(agent_id)
```

### Your YAML Course Configuration

From your implementation plan (`thoughts/shared/plans/2026-01-07-yaml-course-configuration.md`):

```yaml
course:
  id: college-essay
  name: College Essay Mastery

tutor:
  name: YouLab Essay Coach
  role: AI tutor specializing in college application essays
  tone: warm
  verbosity: adaptive
  capabilities: [...]
  expertise: [...]
  constraints: [...]

messages:
  welcome_first: "..."
  error_not_logged_in: "..."

background_tasks:
  tasks:
    - trigger: "0 3 * * *"
      human:
        - query: What learning style works best?
          field: context_notes
```

### Field Mapping: YAML → AF

| Your YAML | AF Equivalent | Notes |
|-----------|---------------|-------|
| `course.id` | `agent.id` | Direct mapping |
| `course.name` | `agent.name` | Direct mapping |
| `tutor.*` | `core_memory[label="persona"]` | Serializes via `PersonaBlock.to_memory_string()` |
| `messages.*` | ❌ Not in AF | These are application-level messages, not agent state |
| `background_tasks.*` | ❌ Not in AF | Honcho-specific, not Letta-native |
| ❌ Not in YAML | `llm_config` | Hardcoded in `AgentManager` |
| ❌ Not in YAML | `tools` | Not yet configurable |
| ❌ Not in YAML | `messages` | State, not config |

### Current Agent Creation Flow

**Location**: `src/letta_starter/server/agents.py:108-117`

```python
agent = self.client.agents.create(
    name=agent_name,
    model="openai/gpt-4o-mini",
    embedding="openai/text-embedding-ada-002",
    memory_blocks=[
        {"label": "persona", "value": template.persona.to_memory_string()},
        {"label": "human", "value": human_block.to_memory_string()},
    ],
    metadata=metadata,
)
```

Your YAML feeds into `AgentTemplate.persona` which becomes the `memory_blocks[0]`.

### Key Architectural Difference

```
AF (.af file):
┌─────────────────────────────────────────┐
│ Complete Agent Snapshot                 │
│ ├── Identity (id, name)                 │
│ ├── Model Config (llm, embedding)       │
│ ├── System Prompt                       │
│ ├── Memory Blocks (current state)       │
│ ├── Message History (with state)        │
│ ├── Tools (source code + schemas)       │
│ └── Tool Rules                          │
└─────────────────────────────────────────┘

Your YAML:
┌─────────────────────────────────────────┐
│ Agent Template Config                   │
│ ├── Course Metadata                     │
│ ├── Tutor Persona Definition            │
│ ├── UI Messages (welcome, errors)       │
│ └── Background Tasks (Honcho)           │
└─────────────────────────────────────────┘
```

## Integration Options

### Option A: Keep YAML as Template (Recommended)

**How it works**: YAML defines agent configuration, compiles to Letta API calls at runtime.

```
YAML → CourseConfig → AgentTemplate → client.agents.create()
```

**Pros**:
- Simplest approach
- Matches your existing plan
- YAML includes app-specific fields (messages, background_tasks) that AF can't represent
- Course designers only need to know YAML

**Cons**:
- Can't share pre-configured agents as files
- No standardized export format

**When to use**: Primary path for creating new per-user agents.

### Option B: Generate Partial AF for Import

**How it works**: Compile YAML to a minimal .af file and import it.

```yaml
# config/college-essay.yaml
→ generates →
# agents/college-essay.af (minimal, no conversation state)
```

**Pros**:
- Could share agent templates as .af files
- Aligns with Letta ecosystem

**Cons**:
- AF expects state (messages, context) that templates don't have
- Doesn't help with messages/background_tasks (app-specific)
- Extra complexity for no clear benefit
- AF import creates one agent; you need per-user agents

**When to use**: If you want to distribute pre-built agent templates externally.

### Option C: Extend YAML with AF Fields

**How it works**: Add optional AF-native fields to your YAML schema.

```yaml
course:
  id: college-essay

tutor:
  name: YouLab Essay Coach
  # ... existing fields

# Optional: AF-native fields
letta:
  model: openai/gpt-4o-mini
  embedding: openai/text-embedding-ada-002
  tools:
    - name: web_search
      source: letta://tools/web_search
  tool_rules:
    - type: max_count_per_step
      tool: web_search
      count: 3
```

**Pros**:
- Full control over Letta agent configuration
- Could generate valid .af files if needed
- Future-proofs for tool configuration

**Cons**:
- Increases YAML complexity
- Requires course designers to understand Letta concepts
- Still need app-specific fields (messages, background_tasks)

**When to use**: When you need fine-grained control over model/tool configuration per course.

### Option D: Hybrid - YAML for Templates, AF for Snapshots

**How it works**: Use YAML for creating new agents, AF for importing/exporting snapshots.

```
New Agent:  YAML → AgentTemplate → client.agents.create()
Export:     client.agents.export_file(agent_id) → .af
Import:     .af → client.agents.import_file() → Running agent
```

**Pros**:
- Best of both worlds
- Can checkpoint and share trained agents
- Useful for debugging (export problematic agent state)

**Cons**:
- Two formats to understand
- AF export includes sensitive conversation data

**When to use**: For advanced use cases like agent checkpointing or debugging.

## Recommendation

**Go with Option A (current plan)** with awareness of AF for future features:

1. **Now**: Implement YAML configuration as planned
   - Your YAML schema is well-designed for course configuration
   - It includes app-specific fields (messages, background_tasks) that AF can't represent
   - Course designers don't need to know about AF

2. **Future (optional)**: Add AF export for debugging
   - `GET /agents/{id}/export` → Downloads .af file
   - Useful for debugging or sharing specific agent snapshots

3. **Future (optional)**: Extend YAML with `letta:` section
   - Add model/embedding config per course
   - Add tool configuration when you implement custom tools

## Code References

- `src/letta_starter/server/agents.py:108-117` - Where YAML config becomes Letta API call
- `src/letta_starter/agents/templates.py:8-16` - AgentTemplate schema
- `src/letta_starter/memory/blocks.py:75-101` - PersonaBlock.to_memory_string() serialization
- `thoughts/shared/plans/2026-01-07-yaml-course-configuration.md` - Your YAML implementation plan

## Historical Context (from thoughts/)

- `thoughts/shared/plans/2026-01-07-yaml-course-configuration.md` - Comprehensive YAML implementation plan
- `thoughts/shared/research/2026-01-07-toml-configuration-opportunities.md` - Earlier config format research (referenced in plan)

## Related Research

- [Letta AgentFile Documentation](https://docs.letta.com/guides/agents/agent-file/)
- [Agent File GitHub Repository](https://github.com/letta-ai/agent-file)
- [Letta Blog: Introducing Agent File](https://www.letta.com/blog/agent-file)

## Extended YAML Schema Proposal

Based on your request to configure all Letta agent properties in YAML, here's a comprehensive schema that maps to the Letta API:

### Complete YAML Schema

```yaml
# config/college-essay.yaml
# =============================================================================
# COURSE METADATA
# =============================================================================
course:
  id: college-essay
  name: College Essay Mastery
  description: 8-week college essay coaching program

# =============================================================================
# LETTA AGENT CONFIGURATION
# =============================================================================
# These fields map directly to client.agents.create() parameters

agent:
  # Model configuration (currently hardcoded in agents.py:110-111)
  model: openai/gpt-4o-mini
  embedding: openai/text-embedding-3-small

  # System prompt - injected into Letta's system prompt
  # This is SEPARATE from the persona block
  system_prompt: |
    You are a college essay coach. Your job is to guide students through
    self-discovery and help them craft authentic personal narratives.

    IMPORTANT GUIDELINES:
    - Never write essays for students
    - Use Socratic questioning
    - Celebrate progress and small wins

  # Memory blocks - defines what goes into agent's editable memory
  memory_blocks:
    persona:
      name: YouLab Essay Coach
      role: AI tutor specializing in college application essays
      tone: warm
      verbosity: adaptive
      capabilities:
        - Guide students through self-discovery exercises
        - Help brainstorm and develop essay topics
        - Provide constructive feedback on drafts
      expertise:
        - College admissions
        - Personal narrative
        - Reflective writing
      constraints:
        - Never write essays for students
        - Always ask clarifying questions before giving advice

    human:
      # Initial human block (empty by default, populated during onboarding)
      name: null
      role: student

  # Tools - list of tool names to attach
  tools:
    - send_message        # Default, always included
    - conversation_search # Search conversation history
    - archival_memory_insert
    - archival_memory_search
    # - web_search        # Uncomment if TAVILY_API_KEY is set
    # - run_code          # Uncomment if E2B_API_KEY is set

  # Tool rules - control agent execution flow
  tool_rules:
    - tool_name: send_message
      type: exit_loop      # Terminal - agent stops after sending message

    # Example: require web_search before answering factual questions
    # - tool_name: web_search
    #   type: max_count_per_step
    #   max_count: 3

  # Secrets - tool-specific API keys (optional)
  secrets:
    # TAVILY_API_KEY: ${TAVILY_API_KEY}  # Reference env var
    # E2B_API_KEY: ${E2B_API_KEY}

# =============================================================================
# APP-SPECIFIC CONFIGURATION (Not part of Letta)
# =============================================================================

messages:
  welcome_first: |
    Welcome to YouLab! I'm your personal college essay coach.
    What would you like to work on today?
  welcome_returning: Welcome back! Ready to continue your essay journey?
  error_not_logged_in: Please log in to access your coach.
  error_service_unavailable: I'm taking a short break. Please try again.

background_tasks:
  scope: [tutor]
  batch_size: 50
  tasks:
    - trigger: "0 3 * * *"
      human:
        - query: What learning style works best for this student?
          field: context_notes
```

### Field Mapping to Letta API

| YAML Field | Letta API Parameter | Notes |
|------------|---------------------|-------|
| `agent.model` | `model` | Currently hardcoded in `agents.py:110` |
| `agent.embedding` | `embedding` | Currently hardcoded in `agents.py:111` |
| `agent.system_prompt` | `system` | NEW - not currently used |
| `agent.memory_blocks.persona` | `memory_blocks[0]` | Via `PersonaBlock.to_memory_string()` |
| `agent.memory_blocks.human` | `memory_blocks[1]` | Via `HumanBlock.to_memory_string()` |
| `agent.tools` | `tools` | NEW - not currently used |
| `agent.tool_rules` | `tool_rules` | NEW - not currently used |
| `agent.secrets` | `secrets` | NEW - not currently used |

### Implementation Changes Required

**1. Extend `CourseConfig` schema** (`config/course_config.py`):

```python
class AgentConfig(BaseModel):
    """Letta agent configuration."""
    model: str = "openai/gpt-4o-mini"
    embedding: str = "openai/text-embedding-3-small"
    system_prompt: str | None = None
    memory_blocks: MemoryBlocksConfig
    tools: list[str] = Field(default_factory=lambda: ["send_message"])
    tool_rules: list[ToolRuleConfig] = Field(default_factory=list)
    secrets: dict[str, str] = Field(default_factory=dict)

class ToolRuleConfig(BaseModel):
    """Tool rule configuration."""
    tool_name: str
    type: Literal["exit_loop", "run_first", "continue_loop", "max_count_per_step"]
    max_count: int | None = None  # For max_count_per_step
    children: list[str] | None = None  # For child rules

class MemoryBlocksConfig(BaseModel):
    """Memory blocks configuration."""
    persona: TutorConfig  # Reuse existing TutorConfig
    human: HumanBlockConfig = Field(default_factory=HumanBlockConfig)
```

**2. Update `AgentManager.create_agent()`** (`server/agents.py`):

```python
def create_agent(self, user_id: str, agent_type: str = "tutor", ...) -> str:
    config = load_course_config(agent_type)

    agent = self.client.agents.create(
        name=agent_name,
        model=config.agent.model,                    # From YAML
        embedding=config.agent.embedding,            # From YAML
        system=config.agent.system_prompt,           # NEW
        memory_blocks=[
            {"label": "persona", "value": persona_block.to_memory_string()},
            {"label": "human", "value": human_block.to_memory_string()},
        ],
        tools=config.agent.tools,                    # NEW
        tool_rules=[                                 # NEW
            {"tool_name": r.tool_name, "type": r.type, ...}
            for r in config.agent.tool_rules
        ],
        secrets=config.agent.secrets,                # NEW
        metadata=metadata,
    )
```

### Comparison: Your YAML vs AF

| Aspect | Your YAML | AF (.af) |
|--------|-----------|----------|
| Purpose | Template for creating agents | Snapshot of existing agent |
| State | Stateless config | Includes message history |
| Tools | References by name | Includes full source code |
| System prompt | Configurable | Captured at export time |
| Memory blocks | Template values | Current state values |
| Format | YAML (human-editable) | JSON (machine-portable) |

**Your YAML is the "factory config" - AF is the "serialized instance".**

## Open Questions

1. **Custom Tools**: Should YAML reference tools by name (requiring separate registration) or include tool definitions inline?
2. **System Prompt vs Persona**: Should system_prompt be separate or merged with persona block content?
3. **Secrets Management**: Should secrets reference env vars (`${VAR}`) or be stored elsewhere?

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

## Open Questions

1. **Tool Configuration**: When you add custom tools, should they be defined in YAML or registered programmatically?
2. **Model per Course**: Should different courses use different models? (Currently hardcoded to gpt-4o-mini)
3. **AF Export**: Would an agent export feature be useful for debugging or user data portability?

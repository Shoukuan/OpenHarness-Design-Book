# Chapter 9: Coordinator — Multi-Agent Definition & Registration Hub

**Code Location**: `src/openharness/coordinator/` (3 files, 1,506 lines)

---

## 9.1 Why Do We Need Coordinator?

Swarm handles multi-agent execution scheduling; **Coordinator handles definition and management**:

- Which predefined Agent roles exist (planner, reviewer, executor...)
- Each role's system prompt, model config, permission level
- How teams are formed, member join/exit
- Global singleton: Coordinator is sole entry point for querying Agent definitions

---

## 9.2 Core Data Structures

### 9.2.1 AgentDefinition

```python
class AgentDefinition(BaseModel):
    name: str                    # Role name, e.g., "senior-reviewer"
    description: str             # Human-readable description
    prompt: str                  # System prompt
    model: Optional[str] = None  # Override global model setting
    color: Optional[str] = None  # UI display color (TUI use)
    team: Optional[str] = None   # Team it belongs to (optional)
    permissions: Optional[AgentPermissions] = None
    isolation: Optional[IsolationConfig] = None  # Workspace isolation
```

**Example**:

```yaml
agentDefinitions:
  - name: "planner"
    description: "Task planning expert, decomposes complex requirements"
    prompt: |
      You are planning expert. Decompose user needs into executable task list.
      Output format: JSON array, each item with description, tool_hints.
    model: "claude-3-opus"

  - name: "reviewer"
    description: "Code review expert"
    prompt: "Review code security, performance, readability."
    permissions:
      file_read: true
      bash: false  # Reviewer Agent not allowed to execute commands
```

---

### 9.2.2 TeamRegistry (Team Registration)

**Global singleton**, only one instance in entire process:

```python
class TeamRegistry:
    def create_team(self, name: str, description: str = "") -> TeamRecord: ...
    def delete_team(self, name: str): ...
    def add_agent(self, team_name: str, agent_def: AgentDefinition) -> AgentRecord: ...
    def remove_agent(self, team_name: str, agent_name: str): ...
    def send_message(self, team_name: str, message: str, to: str = None): ...
    def list_teams(self) -> list[TeamRecord]: ...
    def get_team(self, name: str) -> TeamRecord | None: ...
```

---

### 9.2.3 TeamRecord (Team Data)

```python
class TeamRecord(BaseModel):
    name: str
    description: str
    members: list[AgentRecord]
    created_at: datetime
    metadata: dict = {}  # Extension fields
```

---

## 9.3 Initialization Flow

At startup:

1. Load default Agent definitions (built-in `assistant`, `planner`, `reviewer`)
2. Scan project `.openharness/coordinator/` directory for YAML files
3. Register `TeamRegistry` as global singleton (`get_registry()`)

```python
# coordinator/__init__.py
_registry: TeamRegistry | None = None

def initialize():
    global _registry
    _registry = TeamRegistry()
    _registry.load_builtin_agents()
    _registry.load_project_agents()

def get_registry() -> TeamRegistry:
    if _registry is None:
        raise RuntimeError("Coordinator not initialized")
    return _registry
```

---

## 9.4 Collaboration with Swarm

Coordinator only **defines**, Swarm only **executes**:

```
Coordinator Layer                    Swarm Layer
─────────────────┼─────────────────────────────
 Define Agents    │   spawn(agent_def) → Actually start process
 Define Teams     │   create worktree + mailbox
 Query member list │   manage Agent lifecycle
─────────────────┼─────────────────────────────
    One-way dependency: Swarm reads Coordinator definitions
```

**Typical usage**:

```python
# Create team (definition)
registry = coordinator.get_registry()
registry.create_team("dev-team", "Development collaboration team")
registry.add_agent("dev-team", AgentDefinition(name="frontend", ...))
registry.add_agent("dev-team", AgentDefinition(name="backend", ...))

# Swarm starts team execution
await swarm.spawn_team("dev-team", task="Implement login feature")
```

---

## 9.5 Persistence Design

Team state stored on disk:

```
~/.openharness/teams/my-team/
├── team.json           # TeamRecord + members list
├── permissions/        # Permission snapshots
├── worktrees/          # Each Agent's independent cwd
└── mailboxes/          # Message mailboxes (for Agent communication)
```

After restart, Swarm can restore team state from disk.

---

## 9.6 Comparison with OpenClaw

| Comparison | OpenHarness Coordinator | OpenClaw |
|------------|------------------------|----------|
| Definition method | YAML + Python classes | AGENTS.md + hot-reload |
| Global singleton | ✅ TeamRegistry singleton | No explicit singleton, permeates everywhere |
| Team creation | CLI command `/team create` | sessions_spawn + label grouping |
| Persistence | `~/.openharness/teams/` | sessions stored as files |
| Permission isolation | AgentPermissions object | Inherits parent session permissions |

**OpenHarness advantages**:
- Clear concepts, separation of definition and execution
- Team state can persist and restore

**OpenClaw advantages**:
- Lighter weight, no need to pre-define teams, spawn anytime
- Deep OpenViking memory integration

---

## 9.7 Extending with Custom Agent

Create `.openharness/coordinator/agents.yaml` in project:

```yaml
agentDefinitions:
  - name: "senior-python-dev"
    description: "Senior Python engineer, handles complex backend logic"
    prompt: |
      You are Python engineer with 10 years experience.
      Familiar with FastAPI, SQLAlchemy, Celery.
      Code must include type annotations and exception handling.
    model: "claude-3-opus"
    permissions:
      bash: true
      file_write: true
```

After restart, this Agent appears in `/agents` command list, can be activated with `/agent senior-python-dev <task description>`.

---

Next Chapter: [Chapter 10: Swarm System — 4,899 Lines Deep Dive](10-swarm-system.md)
# Chapter 14: Commands System — 54 CLI Commands, 1,456 Lines

**Module**: `src/openharness/commands/` (2 files, 1,456 lines)

---

## 14.1 Command Overview

OpenHarness provides 54 slash commands (`/help`, `/plan`, `/commit`...), usable in CLI/TUI, also triggerable via API.

Commands registered in `commands/registry.py`, implementations scattered in `commands/` subdirectories.

---

## 14.2 Command List

| Category | Command | Description | Implementation File |
|----------|---------|-------------|---------------------|
| Basic | `/help` | Show help | `cmd_help.py` |
|       | `/exit` | Exit | `cmd_exit.py` |
| Session | `/resume` | Resume last session | `cmd_resume.py` |
|         | `/export` | Export conversation as Markdown | `cmd_export.py` |
|         | `/session` | Session management (list/delete) | `cmd_session.py` |
| Planning | `/plan` | Enter plan mode (planner Agent) | `cmd_plan.py` |
|          | `/plan-commit` | Submit plan | `cmd_plan_commit.py` |
|          | `/plan-cancel` | Cancel plan | `cmd_plan_cancel.py` |
| Task | `/task` | Background task management (list/create/stop) | `cmd_task.py` |
|      | `/cron` | Scheduled task management | `cmd_cron.py` |
| Team | `/team` | Team management (create/add/remove) | `cmd_team.py` |
|      | `/agent` | Spawn sub-Agent | `cmd_agent.py` |
| Skills | `/skill` | Skill management (list/reload) | `cmd_skill.py` |
|       | `/plugin` | Plugin management (install/uninstall) | `cmd_plugin.py` |
| System | `/config` | View/modify config | `cmd_config.py` |
|        | `/auth` | Auth management (login/logout) | `cmd_auth.py` |
|        | `/mcp` | MCP server management | `cmd_mcp.py` |
|        | `/hook` | Hook management (list/reload) | `cmd_hook.py` |
| Tools | `/brief` | Generate session summary | `cmd_brief.py` |
|       | `/sleep` | Pause for duration (debug) | `cmd_sleep.py` |
|       | `/todo` | Todo management | `cmd_todo.py` |
|       | `/git` | Git operations (commit/push) | `cmd_git.py` |
|       | `/lsp` | Language Server Protocol (go to definition) | `cmd_lsp.py` |

**Total 54 commands** (table shows 40+ representatives).

---

## 14.3 Command Registration Mechanism

```python
# commands/registry.py
class CommandRegistry:
    def __init__(self):
        self._commands: dict[str, CommandDef] = {}

    def register(self, cmd: CommandDef):
        self._commands[cmd.name] = cmd

    def get(self, name: str) -> CommandDef | None:
        return self._commands.get(name)
```

At startup, auto-import all `.py` modules in `commands/` directory (except `__init__.py`), each module provides `register(registry)` function.

---

## 14.4 Command Execution Flow

```
User input "/plan Implement login feature"
    │
    ▼
CommandRegistry.get("plan")
    │
    ▼
cmd_plan.execute(args="Implement login feature", context)
    │
    ├─ Check permissions (permission_checker)
    ├─ Parse arguments (simple split)
    ├─ Call Coordinator to create planner Agent
    └─ Return CommandOutput (text + metadata)
```

---

## 14.5 Typical Command Implementation: /plan

```python
# commands/plan/cmd_plan.py
async def execute(args: str, context: CommandContext) -> CommandOutput:
    # 1. Parse arguments
    task = args.strip()
    if not task:
        return CommandOutput(error=True, message="❌ Please provide task description")

    # 2. Check if planner Agent exists
    registry = coordinator.get_registry()
    planner = registry.get_agent("planner")
    if not planner:
        return CommandOutput(error=True, message="❌ planner Agent not defined")

    # 3. Spawn planner Agent (async, don't wait for completion)
    await swarm.spawn_agent(
        agent_def=planner,
        task=task,
        background=True,  # run in background
    )

    return CommandOutput(
        message=f"✅ Started planner Agent processing task: {task}\nUse /task list to check progress"
    )
```

---

## 14.6 Command Permission Control

Commands can declare required permissions:

```python
@command(
    name="/cron",
    requires=["cron:create", "cron:list"],  # Which permissions needed
    category="task",
)
async def cmd_cron(...):
    ...
```

`permission_checker.evaluate_command("/cron")` runs before execution.

---

## 14.7 Comparison with OpenClaw

| Comparison | OpenHarness Commands | OpenClaw |
|------------|---------------------|----------|
| Command count | 54 | 40+ |
| Implementation | Separate modules + registry | TUI command system |
| Permission integration | Each command can declare requires | Based on tool permissions |
| Extensibility | New command = new file + register() | New command = new Command class |
| Hot reload | ✅ hot_reload command | ⚠️ Requires restart |

---

## 14.8 Custom Commands

Create `mycommands/cmd_hello.py`:

```python
from openharness.commands import register

@register(
    name="/hello",
    description="Say hello",
    category="misc",
)
async def cmd_hello(args: str, context):
    name = args or "World"
    return CommandOutput(message=f"Hello, {name}!")

# Activate: oh plugin load mycommands
```

---

Next Chapter: [Chapter 15: Full Comparison — OpenHarness vs OpenClaw vs Other Frameworks](15-full-comparison.md)
# Chapter 4: Tools System — Design and Implementation of 43 Tools

**Core Location**: `src/openharness/tools/` (42 files, 2,542 lines)

---

## 4.1 Tool Panorama

OpenHarness includes **43 built-in tools**, categorized into 11 major groups:

| Category | Count | Example Tools | Purpose |
|-----------|-------|---------------|---------|
| **File Operations** | 5 | FileRead, FileWrite, Edit, Glob, Grep | Read/write filesystem |
| **Shell Execution** | 1 | Bash | Run arbitrary shell commands |
| **Task Management** | 6 | TaskCreate, TaskList, TaskStop, TaskInfo, TaskDone | Manage todo items |
| **Agent Coordination** | 4 | Agent, SendMessage, TeamCreate, TeamDelete | Multi-agent interaction |
| **Cron Scheduling** | 4 | CronCreate, CronList, CronDelete, CronToggle | Scheduled tasks |
| **Planning & Workspace** | 4 | EnterPlanMode, ExitPlanMode, EnterWorktree, ExitWorktree | Mode switching |
| **Web Capabilities** | 2 | WebFetch, WebSearch | Fetch web pages, search internet |
| **MCP Protocol** | 4 | McpTool, McpAuth, McpListResources, McpReadResource | External tool integration |
| **Meta Operations** | 4 | Skill, Config, Brief, Sleep | System bootstrapping |
| **User Interaction** | 2 | AskUserQuestion, TodoWrite | Ask user, write todos |
| **Other** | 7 | Lsp, RemoteTrigger, NotebookEdit, … | Editor integration, etc. |

**Total: 43 tools** (some are adapters like McpToolAdapter that dynamically load external tools)

---

## 4.2 Core Abstraction: BaseTool

Base class defined in `tools/base.py` (75 lines):

```python
class BaseTool(ABC):
    name: str                  # Tool name, e.g., "Bash"
    description: str           # For LLM to understand purpose
    input_model: type[BaseModel]  # Pydantic model defining input parameters
    output_model: type[BaseModel] | None = None  # Optional output model

    async def execute(
        self,
        arguments: BaseModel,
        context: ToolExecutionContext
    ) -> ToolResult: ...

    def is_read_only(self, arguments: BaseModel) -> bool:
        """For permission check: whether read-only operation"""
        return False
```

**Design Highlights**:

1. **Pydantic auto-validation** — `input_model` defines JSON Schema, passed to LLM to produce correct tool_use; automatically `model_validate()` at execution, preventing invalid parameters.
2. **cwd context** — `ToolExecutionContext` contains `cwd`, tools run in Agent's working directory, file operations won't pollute other locations.
3. **Read/write flag** — `is_read_only()` provides permission system risk assessment; file read tools should return `True`.

---

## 4.3 ToolRegistry: Tool Registration Center

`tools/registry.py` implements a simple in-memory registry:

```python
class ToolRegistry:
    def __init__(self):
        self._tools: dict[str, BaseTool] = {}

    def register(self, tool: BaseTool):
        self._tools[tool.name] = tool

    def get(self, name: str) -> BaseTool | None:
        return self._tools.get(name)

    def to_api_schema(self) -> list[dict]:
        """Convert to Anthropic API tool description format"""
        return [t.to_api_schema() for t in self._tools.values()]
```

### Default Initialization

```python
def create_default_tool_registry(mcp_manager=None) -> ToolRegistry:
    registry = ToolRegistry()
    # Register all 40+ built-in tools at once
    for tool in (
        BashTool(),
        FileReadTool(),
        FileWriteTool(),
        GrepTool(),
        TaskCreateTool(),
        CronCreateTool(),
        # ... 40+ total
    ):
        registry.register(tool)

    # If MCP manager provided, dynamically register remote tools
    if mcp_manager:
        for tool_info in mcp_manager.list_tools():
            registry.register(McpToolAdapter(mcp_manager, tool_info))

    return registry
```

---

## 4.4 Typical Tool Implementation Analysis

### 4.4.1 BashTool — Simplest Tool

`tools/bash.py`: 72 lines

```python
class BashTool(BaseTool):
    name = "Bash"
    description = "Execute a shell command"
    input_model = BashInput   # {"command": str, "timeout": int?}

    async def execute(self, arguments, context):
        proc = await asyncio.create_subprocess_exec(
            "bash", "-c", arguments.command,
            cwd=context.cwd,
            stdout=asyncio.subprocess.PIPE,
            stderr=asyncio.subprocess.PIPE,
        )
        try:
            stdout, stderr = await asyncio.wait_for(
                proc.communicate(),
                timeout=arguments.timeout or 30
            )
        except asyncio.TimeoutError:
            proc.kill()
            raise

        return ToolResult(
            output=stdout.decode(),
            error=stderr.decode(),
            exit_code=proc.returncode,
        )
```

**Key Points**:
- Uses `asyncio.create_subprocess_exec` — non-blocking I/O, key to parallel execution
- `context.cwd` ensures running in Agent's working directory
- Timeout control (default 30s) prevents hangs

---

### 4.4.2 GrepTool — Most Complex Tool

`tools/grep.py`: 248 lines

Function: Full-text search across files, supports regex, file type filters, result truncation.

```python
class GrepTool(BaseTool):
    async def execute(self, arguments, context):
        path = Path(arguments.path).resolve()
        pattern = arguments.pattern

        # 1. Build find command to list all files (exclude .git, binaries)
        find_cmd = ["find", str(path), "-type", "f", "-not", "-path", "*/\.git/*"]
        if arguments.include:
            find_cmd += ["-name", arguments.include]
        if arguments.exclude:
            find_cmd += ["-not", "-name", arguments.exclude]

        # 2. Get file list
        files = subprocess.run(find_cmd, capture_output=True, text=True).stdout.splitlines()

        # 3. Batch parallel grep (using xargs -P to limit concurrency)
        grep_cmd = ["xargs", "-P", "4", "grep", "-nH", pattern]
        results = subprocess.run(grep_cmd, input="\\n".join(files), capture_output=True, text=True)

        # 4. If too many results, truncate to max_results
        lines = results.stdout.splitlines()[:arguments.max_results]

        return ToolResult(output="\\n".join(lines), matches=len(lines))
```

**Why complex**: Needs performance optimization (don't load all files into memory), skip binary files, handle symlinks, result truncation.

---

## 4.5 Tool Execution Pipeline (From Engine to Tool)

```
Engine.run_query() gets tool_use
    │
    ▼
_execute_tool_call(tool_use, context)
    │
    ├─ 1. HookExecutor.execute(PRE_TOOL_USE)  ← Can intercept
    │   if blocked: return ToolResult(reason)
    │
    ├─ 2. permission_checker.evaluate()      ← Permission check
    │   if denied: return ToolResult(reason)
    │   if requires_confirmation:
    │       confirmed = await context.permission_prompt(...)
    │       if not confirmed: return denial
    │
    ├─ 3. tool_registry.get(name)             ← Get tool instance
    │
    ├─ 4. input_model.model_validate(arguments)  ← Pydantic validation
    │   if invalid: return ToolResult(error message)
    │
    ├─ 5. tool.execute(arguments, context)   ← Actual execution (async)
    │   # Returns ToolResult(output, error, exit_code)
    │
    ├─ 6. HookExecutor.execute(POST_TOOL_USE) ← Audit logging, metrics
    │
    └─ 7. Return ToolResult → append to messages → next LLM round
```

---

## 4.6 Permission Integration Details

Every tool execution passes permission check:

```python
decision = context.permission_checker.evaluate(
    tool_name=tool.name,
    is_read_only=tool.is_read_only(parsed_input),
    file_path=_extract_file_path(parsed_input),  # Tool-specific logic
    command=_extract_command(parsed_input),
)

if not decision.allowed:
    # Immediate denial (high risk)
    return ToolResult(content=decision.reason, is_error=True)

if decision.requires_confirmation:
    confirmed = await context.permission_prompt(
        f"About to execute {tool.name}: {decision.reason}"
    )
    if not confirmed:
        return ToolResult(content="User denied", is_error=True)
```

**Design philosophy**: Permission check decoupled from tools. Tools only need to declare `is_read_only()`, path/command extraction done by PermissionChecker by convention (e.g., `FileReadTool`'s `path` parameter automatically treated as file path).

See [Chapter 5: Permissions Permission System](05-permissions.md) for details.

---

## 4.7 Comparison with OpenClaw Tool System

| Comparison | OpenHarness | OpenClaw |
|------------|-------------|----------|
| Tool count | 43 built-in | ~60+ (depends on skill loading) |
| Registration | ToolRegistry (in-memory at startup) | Dynamic skill discovery (each skill defines tools) |
| Execution isolation | cwd isolation + permission | Sandbox container (stricter) |
| Permission model | Three-tier (deny/confirm/allow) | Path whitelist + command blacklist |
| Tool definition | One class per tool, `execute()` method | tools description + handler function (flexible) |
| Tool selection | All registered → send schema to LLM → LLM picks | Similar but supports hot skill loading |

**OpenHarness advantages**:
- Cleaner code organization, each tool independent file, easy to maintain/extend
- Async execution non-blocking, good performance

**OpenClaw advantages**:
- Skill mechanism more flexible, hot-load new tools (no restart needed)
- OpenViking vector memory can optimize tool selection (choose based on historical success rate)

---

## 4.8 Custom Development Guide

Add a new tool in three steps:

```python
# my_tool.py
from openharness.tools import BaseTool, ToolResult, ToolExecutionContext
from pydantic import BaseModel

class MyInput(BaseModel):
    param1: str
    param2: int

class MyTool(BaseTool):
    name = "MyTool"
    description = "Do something useful"
    input_model = MyInput

    async def execute(self, args: MyInput, context: ToolExecutionContext) -> ToolResult:
        # Your logic here
        result = do_something(args.param1, args.param2)
        return ToolResult(output=result)

# Register (typically in main.py)
registry.register(MyTool())
```

**Advanced extension**: Override `is_read_only()` for permission control; use `context.hook_executor` for Hook interactions.

---

> **Previous**: [Chapter 3: Engine Core Loop](03-engine-loop.md)  
> **Next**: [Chapter 5: Permissions Permission System — Enterprise Security Sandbox](05-permissions.md)
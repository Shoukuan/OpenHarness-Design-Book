# 第4章：Tools 工具系统 —— 43 种工具的设计与实现

**核心位置**：`src/openharness/tools/`（42 文件，2,542 行）

---

## 4.1 工具全景图

OpenHarness 内置 **43 种工具**，分为 11 个大类：

| 类别 | 工具数 | 代表工具 | 用途 |
|------|--------|----------|------|
| **文件操作** | 5 | FileRead, FileWrite, Edit, Glob, Grep | 读写文件系统 |
| **Shell 执行** | 1 | Bash | 运行任意 shell 命令 |
| **任务管理** | 6 | TaskCreate, TaskList, TaskStop, TaskInfo, TaskDone | 管理待办事项 |
| **Agent 协调** | 4 | Agent, SendMessage, TeamCreate, TeamDelete | 多 Agent 交互 |
| **Cron 调度** | 4 | CronCreate, CronList, CronDelete, CronToggle | 定时任务 |
| **计划与工作区** | 4 | EnterPlanMode, ExitPlanMode, EnterWorktree, ExitWorktree | 模式切换 |
| **Web 能力** | 2 | WebFetch, WebSearch | 抓网页、搜网络 |
| **MCP 协议** | 4 | McpTool, McpAuth, McpListResources, McpReadResource | 外部工具集成 |
| **元操作** | 4 | Skill, Config, Brief, Sleep | 系统自举 |
| **用户交互** | 2 | AskUserQuestion, TodoWrite | 询问用户、写待办 |
| **其他** | 7 | Lsp, RemoteTrigger, NotebookEdit, … | 编辑器集成等 |

**总计：43 个工具**（部分为适配器，如 McpToolAdapter 可动态加载外部工具）

---

## 4.2 核心抽象：BaseTool

所有工具的基类定义在 `tools/base.py`（75 行）：

```python
class BaseTool(ABC):
    name: str                  # 工具名称，如 "Bash"
    description: str           # 用于 LLM 理解用途
    input_model: type[BaseModel]  # Pydantic 模型定义输入参数
    output_model: type[BaseModel] | None = None  # 可选输出模型

    async def execute(
        self,
        arguments: BaseModel,
        context: ToolExecutionContext
    ) -> ToolResult: ...

    def is_read_only(self, arguments: BaseModel) -> bool:
        """用于权限检查：是否只读操作"""
        return False
```

**设计亮点**：

1. **Pydantic 自动校验** —— `input_model` 定义了 JSON Schema，传给 LLM 产生正确的 tool_use；执行时自动 `model_validate()`，避免无效参数。
2. **cwd 上下文** —— `ToolExecutionContext` 包含 `cwd`，工具在 Agent 的工作目录中运行，文件操作不会污染其他地方。
3. **读写标记** —— `is_read_only()` 供权限系统判断风险等级，文件读工具应返回 `True`。

---

## 4.3 ToolRegistry：工具注册中心

`tools/registry.py` 实现一个简单的内存注册表：

```python
class ToolRegistry:
    def __init__(self):
        self._tools: dict[str, BaseTool] = {}

    def register(self, tool: BaseTool):
        self._tools[tool.name] = tool

    def get(self, name: str) -> BaseTool | None:
        return self._tools.get(name)

    def to_api_schema(self) -> list[dict]:
        """转换为 Anthropic API 的工具描述格式"""
        return [t.to_api_schema() for t in self._tools.values()]
```

### 默认初始化函数

```python
def create_default_tool_registry(mcp_manager=None) -> ToolRegistry:
    registry = ToolRegistry()
    # 一次性注册 40+ 个内置工具
    for tool in (
        BashTool(),
        FileReadTool(),
        FileWriteTool(),
        GrepTool(),
        TaskCreateTool(),
        CronCreateTool(),
        # ... 共 40+ 个
    ):
        registry.register(tool)

    # 如果提供 MCP 管理器，动态注册远程工具
    if mcp_manager:
        for tool_info in mcp_manager.list_tools():
            registry.register(McpToolAdapter(mcp_manager, tool_info))

    return registry
```

---

## 4.4 典型工具实现分析

### 4.4.1 BashTool —— 最简单的工具

`tools/bash.py`：72 行

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

**要点**：
- 使用 `asyncio.create_subprocess_exec` —— 非阻塞 I/O，并行执行的关键
- `context.cwd` 确保在 Agent 的工作目录运行
- 超时控制（默认 30 秒）防死锁

---

### 4.4.2 GrepTool —— 最复杂的工具

`tools/grep.py`：248 行

功能：跨文件全文搜索，支持正则、文件类型过滤、结果截断。

```python
class GrepTool(BaseTool):
    async def execute(self, arguments, context):
        path = Path(arguments.path).resolve()
        pattern = arguments.pattern

        # 1. 构建 find 命令找出所有文件（排除 .git、二进制等）
        find_cmd = ["find", str(path), "-type", "f", "-not", "-path", "*/\.git/*"]
        if arguments.include:
            find_cmd += ["-name", arguments.include]
        if arguments.exclude:
            find_cmd += ["-not", "-name", arguments.exclude]

        # 2. 获得文件列表
        files = subprocess.run(find_cmd, capture_output=True, text=True).stdout.splitlines()

        # 3. 批量并行 grep（用 xargs -P 限制并发数）
        grep_cmd = ["xargs", "-P", "4", "grep", "-nH", pattern]
        results = subprocess.run(grep_cmd, input="\\n".join(files), capture_output=True, text=True)

        # 4. 如果结果太多，截断到 max_results
        lines = results.stdout.splitlines()[:arguments.max_results]

        return ToolResult(output="\\n".join(lines), matches=len(lines))
```

**为什么复杂**：需要优化性能（不要一次加载所有文件到内存）、跳过二进制文件、处理符号链接、结果截断。

---

## 4.5 工具执行管线（从 Engine 到 Tool）

```
Engine.run_query() 得到 tool_use
    │
    ▼
_execute_tool_call(tool_use, context)
    │
    ├─ 1. HookExecutor.execute(PRE_TOOL_USE)  ← 可拦截
    │   if blocked: return ToolResult(reason)
    │
    ├─ 2. permission_checker.evaluate()      ← 权限检查
    │   if denied: return ToolResult(reason)
    │   if requires_confirmation:
    │       confirmed = await context.permission_prompt(...)
    │       if not confirmed: return 拒绝
    │
    ├─ 3. tool_registry.get(name)             ← 获取工具实例
    │
    ├─ 4. input_model.model_validate(arguments)  ← Pydantic 校验
    │   if invalid: return ToolResult(错误信息)
    │
    ├─ 5. tool.execute(arguments, context)   ← 实际执行（异步）
    │   # 返回 ToolResult(output, error, exit_code)
    │
    ├─ 6. HookExecutor.execute(POST_TOOL_USE) ← 审计日志、指标收集
    │
    └─ 7. 返回 ToolResult → 追加到 messages → 下一轮 LLM
```

---

## 4.6 权限集成详解

每个工具执行前都要过权限检查：

```python
decision = context.permission_checker.evaluate(
    tool_name=tool.name,
    is_read_only=tool.is_read_only(parsed_input),
    file_path=_extract_file_path(parsed_input),  # 工具特定逻辑
    command=_extract_command(parsed_input),
)

if not decision.allowed:
    # 立即拒绝（高风险）
    return ToolResult(content=decision.reason, is_error=True)

if decision.requires_confirmation:
    confirmed = await context.permission_prompt(
        f"即将执行 {tool.name}: {decision.reason}"
    )
    if not confirmed:
        return ToolResult(content="用户拒绝", is_error=True)
```

**设计思想**：权限检查不与工具耦合。工具只需声明 `is_read_only()`，路径/命令提取由 PermissionChecker 通过约定完成（如 `FileReadTool` 的 `path` 参数自动被视为文件路径）。

详细信息见 [第5章：Permissions 权限系统](05-permissions.md)。

---

## 4.7 与 OpenClaw 工具系统对比

| 对比项 | OpenHarness | OpenClaw |
|--------|------------|----------|
| 工具数量 | 43 个内置 | ~60+（依赖技能动态加载） |
| 注册机制 | ToolRegistry（内存启动时注册） | 动态技能发现（每个 skill 定义工具） |
| 执行隔离 | cwd 隔离 + permission | 沙箱容器（更严格） |
| 权限模型 | 三级（拒绝/确认/允许） | 路径白名单 + 命令黒名单 |
| 工具定义 | 每个工具一个类，`execute()` 方法 | tools 描述 + handler 函数（灵活） |
| 工具选择 | 全部注册 → 传 schema 给 LLM → LLM 选 | 类似，但支持 skill 热加载 |

**OpenHarness 优势**：
- 代码组织更规范，每个工具独立文件，易于维护和扩展
- Async 执行无阻塞，性能好

**OpenClaw 优势**：
- Skill 机制更灵活，可热加载新工具（无需重启）
- OpenViking 向量记忆可用于工具选择优化（根据历史成功率选工具）

---

## 4.8 自定义开发指南

添加一个新工具只需三步：

```python
# my_tool.py
from openharness.tools import BaseTool, ToolResult, ToolExecutionContext
from pydantic import BaseModel

class MyInput(BaseModel):
    param1: str
    param2: int

class MyTool(BaseTool):
    name = "MyTool"
    description = "做某件有用的事"
    input_model = MyInput

    async def execute(self, args: MyInput, context: ToolExecutionContext) -> ToolResult:
        # 你的逻辑
        result = do_something(args.param1, args.param2)
        return ToolResult(output=result)

# 注册 (通常在 main.py 中)
registry.register(MyTool())
```

**高级扩展**：如需权限控制，重写 `is_read_only()`；如需 Hook 交互，使用 `context.hook_executor`。

---

> **上一章**：[第3章 Engine 核心循环](03-engine-loop.md)  
> **下一章**：[第5章 Permissions 权限系统 —— 企业级安全沙箱](05-permissions.md)

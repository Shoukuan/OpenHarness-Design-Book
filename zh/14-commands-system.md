# 第14章：Commands 系统 —— 54 个 CLI 命令详解

**模块**：`src/openharness/commands/`（2 文件，1,456 行）

---

## 14.1 命令概览

OpenHarness 提供 54 个斜杠命令（`/help`、`/plan`、`/commit`…），在 CLI/TUI 中使用，也可通过 API 触发。

命令注册在 `commands/registry.py`，实现分散在 `commands/` 子目录。

---

## 14.2 命令清单

| 分类 | 命令 | 描述 | 实现文件 |
|------|------|------|----------|
| 基础 | `/help` | 显示帮助 | `cmd_help.py` |
|      | `/exit` | 退出 | `cmd_exit.py` |
| 会话 | `/resume` | 恢复上次会话 | `cmd_resume.py` |
|      | `/export` | 导出对话为 Markdown | `cmd_export.py` |
|      | `/session` | 会话管理（list/delete） | `cmd_session.py` |
| 计划 | `/plan` | 进入计划模式（planner Agent） | `cmd_plan.py` |
|      | `/plan-commit` | 提交计划 | `cmd_plan_commit.py` |
|      | `/plan-cancel` | 取消计划 | `cmd_plan_cancel.py` |
| 任务 | `/task` | 后台任务管理（list/create/stop） | `cmd_task.py` |
|      | `/cron` | 定时任务管理 | `cmd_cron.py` |
| 团队 | `/team` | 团队管理（create/add/remove） | `cmd_team.py` |
|      | `/agent` | 生成子 Agent | `cmd_agent.py` |
| 技能 | `/skill` | 技能管理（list/reload） | `cmd_skill.py` |
|      | `/plugin` | 插件管理（install/uninstall） | `cmd_plugin.py` |
| 系统 | `/config` | 查看/修改配置 | `cmd_config.py` |
|      | `/auth` | 认证管理（login/logout） | `cmd_auth.py` |
|      | `/mcp` | MCP 服务器管理 | `cmd_mcp.py` |
|      | `/hook` | Hook 管理（list/reload） | `cmd_hook.py` |
| 工具 | `/brief` | 生成会话简报 | `cmd_brief.py` |
|      | `/sleep` | 暂停指定时间（调试用） | `cmd_sleep.py` |
|      | `/todo` | 待办事项管理 | `cmd_todo.py` |
|      | `/git` | Git 操作（commit/push） | `cmd_git.py` |
|      | `/lsp` | 语言服务器协议（跳转定义） | `cmd_lsp.py` |

**总计 54 个命令**（上表列出 40+ 个代表）。

---

## 14.3 命令注册机制

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

启动时自动导入 `commands/` 目录所有 `.py` 模块（除 `__init__.py`），每个模块提供 `register(registry)` 函数。

---

## 14.4 命令执行流程

```
用户输入 "/plan 实现登录功能"
    │
    ▼
CommandRegistry.get("plan")
    │
    ▼
cmd_plan.execute(args="实现登录功能", context)
    │
    ├─ 检查权限（permission_checker）
    ├─ 解析参数（简单的 split）
    ├─ 调用 Coordinator 创建 planner Agent
    └─ 返回 CommandOutput（text + metadata）
```

---

## 14.5 典型命令实现：/plan

```python
# commands/plan/cmd_plan.py
async def execute(args: str, context: CommandContext) -> CommandOutput:
    # 1. 解析参数
    task = args.strip()
    if not task:
        return CommandOutput(error=True, message="❌ 请提供任务描述")

    # 2. 检查 planner Agent 是否存在
    registry = coordinator.get_registry()
    planner = registry.get_agent("planner")
    if not planner:
        return CommandOutput(error=True, message="❌ planner Agent 未定义")

    # 3. Spawn planner Agent（异步，不等待完成）
    await swarm.spawn_agent(
        agent_def=planner,
        task=task,
        background=True,  # 后台运行
    )

    return CommandOutput(
        message=f"✅ 已启动 planner Agent 处理任务：{task}\n使用 /task list 查看进度"
    )
```

---

## 14.6 命令权限控制

命令可声明所需权限：

```python
@command(
    name="/cron",
    requires=["cron:create", "cron:list"],  # 需要哪些权限
    category="task",
)
async def cmd_cron(...):
    ...
```

执行前 `permission_checker.evaluate_command("/cron")`。

---

## 14.7 与 OpenClaw 对比

| 对比项 | OpenHarness Commands | OpenClaw |
|--------|---------------------|----------|
| 命令数量 | 54 个 | 40+ 个 |
| 实现方式 | 独立模块 + registry | TUI 命令系统 |
| 权限集成 | 每命令可声明 requires | 基于 tool 权限 |
| 扩展性 | 新命令 = 新文件 + register() | 新命令 = new Command 类 |
| 热重载 | ✅ hot_reload 命令 | ⚠️ 需重启 |

---

## 14.8 自定义命令

新建 `mycommands/cmd_hello.py`：

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

# 激活：oh plugin load mycommands
```

---

下一章：[第15章 Full Comparison —— OpenHarness vs OpenClaw vs 其他框架](15-full-comparison.md)

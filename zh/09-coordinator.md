# 第9章：Coordinator —— 多 Agent 定义与注册中枢

**代码位置**：`src/openharness/coordinator/`（3 文件，1,506 行）

---

## 9.1 为什么需要 Coordinator？

Swarm 负责多 Agent 的执行调度，**Coordinator 负责定义和管理**：

- 有哪些预定义 Agent 角色（planner、reviewer、executor…）
- 每个角色的系统提示词、模型配置、权限级别
- 团队（Team）如何组建、成员加入/退出
- 全局单例：Coordinator 是查询 Agent 定义的唯一入口

---

## 9.2 核心数据结构

### 9.2.1 AgentDefinition

```python
class AgentDefinition(BaseModel):
    name: str                    # 角色名，如 "senior-reviewer"
    description: str             # 人话描述
    prompt: str                  # 系统提示词
    model: Optional[str] = None  # 覆盖全局模型设置
    color: Optional[str] = None  # UI 显示颜色（TUI 用）
    team: Optional[str] = None   # 所属团队（可选）
    permissions: Optional[AgentPermissions] = None
    isolation: Optional[IsolationConfig] = None  # 工作区隔离
```

**示例**：

```yaml
agentDefinitions:
  - name: "planner"
    description: "任务规划专家，拆解复杂需求"
    prompt: |
      你是规划专家。把用户需求拆为可执行的任务列表。
      输出格式：JSON 数组，每项含 description、tool_hints。
    model: "claude-3-opus"

  - name: "reviewer"
    description: "代码审查专家"
    prompt: "审查代码安全性、性能、可读性。"
    permissions:
      file_read: true
      bash: false  # 审查 Agent 不允许执行命令
```

---

### 9.2.2 TeamRegistry（团队注册表）

**全局单例**，整个进程只有一份：

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

### 9.2.3 TeamRecord（团队数据）

```python
class TeamRecord(BaseModel):
    name: str
    description: str
    members: list[AgentRecord]
    created_at: datetime
    metadata: dict = {}  # 扩展字段
```

---

## 9.3 初始化流程

启动时：

1. 加载默认 Agent 定义（内置 `assistant`, `planner`, `reviewer`）
2. 扫描项目 `.openharness/coordinator/` 目录的 YAML
3. 注册 `TeamRegistry` 为全局单例（`get_registry()`）

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

## 9.4 与 Swarm 的协作

Coordinator 只负责**定义**，Swarm 负责**执行**：

```
Coordinator 层                      Swarm 层
─────────────────┼─────────────────────────────
 定义 Agent       │   spawn(agent_def) → 实际启动进程
 定义 Team        │   创建 worktree + mailbox
 查询成员列表      │   管理 Agent lifecycle
─────────────────┼─────────────────────────────
     单向依赖：Swarm 读取 Coordinator 的定义
```

**典型使用**：

```python
# 创建团队（定义）
registry = coordinator.get_registry()
registry.create_team("dev-team", "开发协作团队")
registry.add_agent("dev-team", AgentDefinition(name="frontend", ...))
registry.add_agent("dev-team", AgentDefinition(name="backend", ...))

# Swarm 启动团队执行
await swarm.spawn_team("dev-team", task="实现登录功能")
```

---

## 9.5 持久化设计

团队状态存在磁盘：

```
~/.openharness/teams/my-team/
├── team.json           # TeamRecord + members 列表
├── permissions/        # 权限快照
├── worktrees/          # 每个 Agent 的独立 cwd
└── mailboxes/          # 消息邮箱（用于 Agent 通信）
```

重启后 Swarm 可以从磁盘恢复团队状态。

---

## 9.6 与 OpenClaw 对比

| 对比项 | OpenHarness Coordinator | OpenClaw |
|--------|------------------------|----------|
| 定义方式 | YAML + Python 类 | AGENTS.md + 热加载 |
| 全局单例 | ✅ TeamRegistry singleton | 无显式单例，渗透到各处 |
| 团队创建 | 命令行 `/team create` | sessions_spawn + label 分组 |
| 持久化 | `~/.openharness/teams/` | session 以文件存储 |
| 权限隔离 | AgentPermissions 对象 | 继承父 session 权限 |

**OpenHarness 优势**：
- 概念清晰，定义与执行分离
- 团队状态可持久化恢复

**OpenClaw 优势**：
- 更轻量，不需要预定义团队，随时 spawn
- 与 OpenViking 记忆深度集成

---

## 9.7 扩展自定义 Agent

在项目中创建 `.openharness/coordinator/agents.yaml`：

```yaml
agentDefinitions:
  - name: "senior-python-dev"
    description: "资深 Python 工程师，负责复杂后端逻辑"
    prompt: |
      你是 10 年经验的 Python 工程师。
      熟悉 FastAPI、SQLAlchemy、Celery。
      写代码必须加类型标注和异常处理。
    model: "claude-3-opus"
    permissions:
      bash: true
      file_write: true
```

重启后，这个 Agent 在 `/agents` 命令列表可见，可用 `/agent senior-python-dev 描述任务` 激活。

---

下一章：[第10章 Swarm 集群协调 —— 4,899 行深度解剖](10-swarm-system.md)

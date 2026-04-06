# 第8章：Hooks 生命周期系统 —— 拦截、审计、自定义逻辑

**核心模块**：`src/openharness/hooks/`（7 文件，475 行）

---

## 8.1 钩子事件总览

HookEvent 枚举支持的 7 个生命周期事件：

| 事件名 | 触发时机 | 典型用途 |
|--------|----------|----------|
| `PRE_TOOL_USE` | 工具执行**前** | 安全审计、权限二次确认、速率限制 |
| `POST_TOOL_USE` | 工具执行**后** | 日志记录、指标统计、失败告警 |
| `PRE_MESSAGE` | 调用 LLM **前** | 注入水印、PII 脱敏、修改 system prompt |
| `POST_MESSAGE` | LLM 返回 **后** | 过滤敏感信息、结构化解析 |
| `AGENT_START` | Agent 回合**开始** | 初始化资源、开启事务 |
| `AGENT_END` | Agent 回合**结束** | 清理临时数据、提交事务 |
| `COMMAND_REGISTERED` | 命令注册时 | 扩展 / 修改命令系统行为 |

---

## 8.2 四种钩子类型

### 8.2.1 CommandHook —— 执行 shell 命令

```yaml
# .openharness/hooks/audit.hook.yaml
id: audit-tool-use
event: PRE_TOOL_USE
type: command
command: "echo '[{{timestamp}}] {{user.id}} 尝试执行 {{tool_name}}: {{payload.tool_input | json}}' >> ~/tool-audit.log"
```

**变量注入**：`{{timestamp}}`、`{{user.id}}`、`{{tool_name}}`、`{{payload}}` 等。

---

### 8.2.2 HttpHook —— HTTP 回调

```yaml
id: push-metrics
event: POST_TOOL_USE
type: http
url: https://analytics.example.com/tool-log
method: POST
headers:
  Authorization: "Bearer {{env.ANALYTICS_TOKEN}}"
body: '{"tool":"{{tool_name}}","latency_ms":{{payload.latency}},"user":"{{user.id}}"}'
```

**适用场景**：推送审计日志到外部 SIEM、Prometheus、Datadog。

---

### 8.2.3 PromptHook —— 动态修改提示

```yaml
id: add-watermark
event: PRE_MESSAGE
type: prompt
template: |
  [系统内部] 当前用户：{{user.id}} | 会话：{{session.id}}
  ---
  {{original_system}}
  (请始终保持专业语气)
```

前置注入到 system prompt。

---

### 8.2.4 AgentHook —— 触发子 Agent

```yaml
id: auto-review
event: POST_TOOL_USE
type: agent
agent: code-reviewer
message: |
  请审查以下 {{tool_name}} 输出：

  {{payload.tool_output}}

  关注点：安全性、性能、可读性。
```

自动触发另一个（预注册的）Agent 来处理结果。

---

## 8.3 HookExecutor 执行引擎（~200 行）

核心类：

```python
class HookExecutor:
    def __init__(self, registry: HookRegistry, context: HookExecutionContext):
        self._registry = registry
        self._context = context

    async def execute(self, event: HookEvent, payload: dict) -> AggregatedHookResult:
        results = []
        for hook in self._registry.get(event):
            # 条件过滤
            if not _matches_hook(hook, payload):
                continue

            if hook.type == "command":
                results.append(await self._run_command(hook, payload))
            elif hook.type == "http":
                results.append(await self._run_http(hook, payload))
            elif hook.type in ("prompt", "agent"):
                results.append(await self._run_prompt_like(hook, payload))
        
        return AggregatedHookResult(results)
```

---

### 8.3.1 命令钩子执行细节

```python
async def _run_command(self, hook: CommandHook, payload: dict) -> HookResult:
    # 模板替换
    command = _inject_placeholders(hook.command, payload)
    
    # 沙箱执行（sandbox.py）
    proc = await create_sandboxed_subprocess(
        command,
        cwd=self._context.cwd,
        env=hook.env,
        timeout=hook.timeout,
    )
    stdout, stderr = await proc.communicate()
    
    return HookResult(
        hook_id=hook.id,
        type="command",
        exit_code=proc.returncode,
        stdout=stdout.decode(),
        stderr=stderr.decode(),
        duration_ms=...,  # 计时
    )
```

**沙箱限制**：
- 网络：默认禁用（可配置 allow_network）
- 文件：继承 cwd，但可限制只读
- 时间：超时强制 Kill

---

### 8.3.2 HTTP 钩子执行

```python
async def _run_http(self, hook: HttpHook, payload: dict) -> HookResult:
    body = _inject_placeholders(hook.body, payload)
    start = time.time()
    
    async with httpx.AsyncClient() as client:
        resp = await client.request(
            hook.method, hook.url,
            headers=hook.headers,
            content=body,
            timeout=hook.timeout,
        )
    
    return HookResult(
        hook_id=hook.id,
        type="http",
        status_code=resp.status_code,
        response_body=await resp.aread(),
        duration_ms=(time.time() - start) * 1000,
    )
```

**失败处理**：网络错误或 4xx/5xx 不阻塞主流程，只记录日志。

---

## 8.4 Hook 注册与发现

Hook 定义放在两个位置：

```
.openharness/hooks/          ← 项目级（版本控制）
~/.config/openharness/hooks/  ← 用户级（个人配置）
```

每个钩子一个文件 `.hook.yaml`：

```yaml
id: unique-hook-id
event: PRE_TOOL_USE
type: command | http | prompt | agent
# 以下字段依 type 而定
command: "..."          # type=command
url: "..."              # type=http
template: "..."         # type=prompt/agent
agent: "reviewer"       # type=agent
# 可选过滤条件
when:
  tool_name: "Bash"
  user.id: "ou_12345"
```

启动时 `load_hook_registry()` 递归扫描目录，加载所有钩子到内存。

---

## 8.5 Engine 集成示例

在 `engine/query.py` 中：

```python
async def _execute_tool_call(self, tool_use, context):
    # PRE_TOOL_USE
    if context.hook_executor:
        pre = await context.hook_executor.execute(
            HookEvent.PRE_TOOL_USE,
            {"tool_name": tool_use.name, "tool_input": tool_use.input, ...}
        )
        if pre.blocked:  # 某个钩子设定了 block=True
            return ToolResultBlock(content=pre.reason, is_error=True)

    # 执行工具（略）

    # POST_TOOL_USE
    if context.hook_executor:
        await context.hook_executor.execute(
            HookEvent.POST_TOOL_USE,
            {"tool_name": tool_use.name, "tool_output": result.content, ...}
        )
```

---

## 8.6 典型使用场景

### 场景 1：调用前审计

记录：谁、什么时候、尝试执行哪个工具、传了什么参数。

```yaml
event: PRE_TOOL_USE
type: http
url: https://siem/audit
body: '{"user":"{{user.id}}","tool":"{{tool_name}}","input":{{payload.tool_input | json}}}'
```

---

### 场景 2：敏感信息过滤

POST_MESSAGE 钩子扫描 LLM 输出，脱敏 API Key、密码。

```yaml
event: POST_MESSAGE
type: command
command: "python scripts/redact.py '{{payload.text}}'"
```

---

### 场景 3：自动代码审查

POST_TOOL_USE 钩子：如果工具是 `WriteFile` 且是 `.py` 文件，触发子 Agent 审查。

```yaml
event: POST_TOOL_USE
type: agent
when:
  tool_name: "FileWrite"
  file_path: ".*\\.py$"
agent: code-reviewer
message: "审查这段代码：{{payload.tool_output}}"
```

---

## 8.7 与 OpenClaw 对比

| 对比项 | OpenHarness Hooks | OpenClaw |
|--------|------------------|----------|
| 触发点 | Engine 层（pre/post tool, message） | 核心层（类似） |
| 拦截能力 | `blocked=True` 可中断执行 | `block` 返回 True 可中断 |
| 扩展类型 | Command / HTTP / Prompt / Agent | Skill Hook + Plugin Hook |
| 配置方式 | YAML 文件扫描（灵活） | Skill 内定义（耦合） |
| 热重载 | ✅ `hook hot_reload` 命令 | ❌ 需重启 Agent |

**OpenHarness 优势**：
- 配置化（YAML），运维友好
- 支持 HTTP 远程钩子，方便对接外部系统
- 命令钩子运行在沙箱中，安全

---

> 下一章：[第9章 Coordinator 多 Agent 协调 —— 定义、注册、生命周期](09-coordinator.md)

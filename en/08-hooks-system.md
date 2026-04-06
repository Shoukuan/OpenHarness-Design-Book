# Chapter 8: Hooks Lifecycle System — Interception, Auditing, Custom Logic

**Core Module**: `src/openharness/hooks/` (7 files, 475 lines)

---

## 8.1 Hook Events Overview

HookEvent enum supports 7 lifecycle events:

| Event | Trigger Timing | Typical Use |
|-------|----------------|-------------|
| `PRE_TOOL_USE` | **Before** tool execution | Security audit, secondary confirmation, rate limiting |
| `POST_TOOL_USE` | **After** tool execution | Logging, metrics collection, failure alerting |
| `PRE_MESSAGE` | **Before** calling LLM | Inject watermark, PII masking, modify system prompt |
| `POST_MESSAGE` | **After** LLM returns | Filter sensitive info, structured parsing |
| `AGENT_START` | Agent turn **start** | Initialize resources, start transaction |
| `AGENT_END` | Agent turn **end** | Clean temp data, commit transaction |
| `COMMAND_REGISTERED` | When command registers | Extend/modify command system behavior |

---

## 8.2 Four Hook Types

### 8.2.1 CommandHook — Execute shell commands

```yaml
# .openharness/hooks/audit.hook.yaml
id: audit-tool-use
event: PRE_TOOL_USE
type: command
command: "echo '[{{timestamp}}] {{user.id}} attempted {{tool_name}}: {{payload.tool_input \| json}}' >> ~/tool-audit.log"
```

**Variable injection**: `{{timestamp}}`, `{{user.id}}`, `{{tool_name}}`, `{{payload}}`, etc.

---

### 8.2.2 HttpHook — HTTP callback

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

**Use case**: Push audit logs to external SIEM, Prometheus, Datadog.

---

### 8.2.3 PromptHook — Dynamically modify prompt

```yaml
id: add-watermark
event: PRE_MESSAGE
type: prompt
template: |
  [Internal] Current user: {{user.id}} | Session: {{session.id}}
  ---
  {{original_system}}
  (Always maintain professional tone)
```

Injected into system prompt beforehand.

---

### 8.2.4 AgentHook — Trigger sub-Agent

```yaml
id: auto-review
event: POST_TOOL_USE
type: agent
agent: code-reviewer
message: |
  Please review this {{tool_name}} output:

  {{payload.tool_output}}

  Focus: security, performance, readability.
```

Automatically triggers another (pre-registered) Agent to process result.

---

## 8.3 HookExecutor Engine (~200 lines)

Core class:

```python
class HookExecutor:
    def __init__(self, registry: HookRegistry, context: HookExecutionContext):
        self._registry = registry
        self._context = context

    async def execute(self, event: HookEvent, payload: dict) -> AggregatedHookResult:
        results = []
        for hook in self._registry.get(event):
            # Condition filtering
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

### 8.3.1 Command Hook Execution Details

```python
async def _run_command(self, hook: CommandHook, payload: dict) -> HookResult:
    # Template substitution
    command = _inject_placeholders(hook.command, payload)

    # Sandbox execution (sandbox.py)
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
        duration_ms=...,  # timing
    )
```

**Sandbox restrictions**:
- Network: disabled by default (configurable allow_network)
- Files: inherit cwd, but can restrict read-only
- Time: timeout forces Kill

---

### 8.3.2 HTTP Hook Execution

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

**Failure handling**: Network errors or 4xx/5xx don't block main flow, only log.

---

## 8.4 Hook Registration & Discovery

Hook definitions placed in two locations:

```
.openharness/hooks/          ← Project-level (version controlled)
~/.config/openharness/hooks/  ← User-level (personal config)
```

One file per hook `.hook.yaml`:

```yaml
id: unique-hook-id
event: PRE_TOOL_USE
type: command | http | prompt | agent
# Following fields depend on type
command: "..."          # type=command
url: "..."              # type=http
template: "..."         # type=prompt/agent
agent: "reviewer"       # type=agent
# Optional filter conditions
when:
  tool_name: "Bash"
  user.id: "ou_12345"
```

At startup, `load_hook_registry()` recursively scans directories, loads all hooks into memory.

---

## 8.5 Engine Integration Example

In `engine/query.py`:

```python
async def _execute_tool_call(self, tool_use, context):
    # PRE_TOOL_USE
    if context.hook_executor:
        pre = await context.hook_executor.execute(
            HookEvent.PRE_TOOL_USE,
            {"tool_name": tool_use.name, "tool_input": tool_use.input, ...}
        )
        if pre.blocked:  # Some hook set blocked=True
            return ToolResultBlock(content=pre.reason, is_error=True)

    # Execute tool (omitted)

    # POST_TOOL_USE
    if context.hook_executor:
        await context.hook_executor.execute(
            HookEvent.POST_TOOL_USE,
            {"tool_name": tool_use.name, "tool_output": result.content, ...}
        )
```

---

## 8.6 Typical Use Cases

### Scenario 1: Pre-execution audit

Record: who, when, which tool attempted, what parameters passed.

```yaml
event: PRE_TOOL_USE
type: http
url: https://siem/audit
body: '{"user":"{{user.id}}","tool":"{{tool_name}}","input":{{payload.tool_input | json}}}'
```

---

### Scenario 2: Sensitive info filtering

POST_MESSAGE hook scans LLM output, redacts API keys, passwords.

```yaml
event: POST_MESSAGE
type: command
command: "python scripts/redact.py '{{payload.text}}'"
```

---

### Scenario 3: Automatic code review

POST_TOOL_USE hook: if tool is `WriteFile` and file is `.py`, trigger sub-Agent review.

```yaml
event: POST_TOOL_USE
type: agent
when:
  tool_name: "FileWrite"
  file_path: ".*\\.py$"
agent: code-reviewer
message: "Review this code: {{payload.tool_output}}"
```

---

## 8.7 Comparison with OpenClaw

| Comparison | OpenHarness Hooks | OpenClaw |
|------------|-------------------|----------|
| Trigger points | Engine layer (pre/post tool, message) | Core layer (similar) |
| Interception | `blocked=True` can interrupt execution | `block` returns True can interrupt |
| Extension types | Command / HTTP / Prompt / Agent | Skill Hook + Plugin Hook |
| Configuration | YAML file scanning (flexible) | Defined within Skill (coupled) |
| Hot reload | ✅ `hook hot_reload` command | ❌ Requires Agent restart |

**OpenHarness advantages**:
- Configuration-based (YAML), operations-friendly
- Supports HTTP remote hooks, easy integration with external systems
- Command hooks run in sandbox, secure

---

Next Chapter: [Chapter 9: Coordinator Multi-Agent Orchestration — Definition, Registration, Lifecycle](09-coordinator.md)
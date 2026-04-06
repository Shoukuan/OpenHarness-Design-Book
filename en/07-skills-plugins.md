# Chapter 7: Skills & Plugins Extensibility Mechanisms

**Modules**:
- `src/openharness/skills/` (6 files, 234 lines)
- `src/openharness/plugins/` (6 files, 328 lines)

---

## 7.1 Difference Between Two Extension Mechanisms

| Dimension | Skills | Plugins |
|-----------|--------|---------|
| **Format** | `.md` knowledge base (simple) | `plugin.json` + Python/JS module (complex) |
| **Purpose** | Inject domain knowledge into system prompt | Add new tools, commands, capabilities |
| **Complexity** | Low (users can write manually) | High (requires coding) |
| **Loading** | Auto-scan (on-demand) | Explicit install (pip/npm) |
| **Compatibility** | OpenHarness native format | Compatible with Anthropic Skills + Claude Code Plugins |

**Simply put**:
- Want to "teach Agent knowledge" → Skills (write Markdown)
- Want to "give Agent new tools" → Plugins (write code)

---

## 7.2 Skills: Knowledge Base as Code

### 7.2.1 Skill Definition (.md format)

```markdown
# Your Skill Name

## Description
One sentence explaining what this skill does.

## System Prompt
When user asks about related topics, I automatically inject this into model's system prompt:

```
You are a data analysis expert. When responding, prefer Python pandas, output tables.
```

## Rules
- Must include type annotations
- Use `NaN` for missing values
- Charts use matplotlib, colors use tab10 palette

## Examples
User: Analyze sales data
Assistant: `df.describe()`
```

---

### 7.2.2 Loading Mechanism (loader.py: ~150 lines)

```python
def load_skill_registry(cwd: Path, user_skills_dir: Path) -> SkillRegistry:
    registry = SkillRegistry()

    # 1. Scan project .openharness/skills/
    for skill_dir in (cwd / ".openharness" / "skills").glob("*"):
        if (skill_dir / "skill.md").exists():
            definition = _parse_skill_markdown(skill_dir / "skill.md")
            registry.register(definition)

    # 2. Scan global user skills directory
    for skill_dir in user_skills_dir.glob("*"):
        if (skill_dir / "skill.md").exists():
            definition = _parse_skill_markdown(skill_dir / "skill.md")
            registry.register(definition)

    return registry
```

**Trigger timing**:
- Every `QueryContext` construction, before Engine starts, match relevant skill from SkillRegistry, attach its `system_prompt`.

```python
# engine/query.py
system_prompt = base_system + "\n\n" + skill_registry.match(user_input)
```

---

### 7.2.3 SkillDefinition Structure

```python
@dataclass
class SkillDefinition:
    name: str
    description: str
    system_prompt: str  # Injected prompt
    rules: list[str]    # Rule list (optional)
    examples: list[Example]  # Example dialogues (optional)
```

---

## 7.3 Plugins: Plugin Ecosystem

### 7.3.1 Plugin Manifest (plugin.json)

```json
{
  "name": "my-plugin",
  "version": "0.1.0",
  "description": "Add custom tools for my workflow",
  "tools": [
    {
      "name": "MyTool",
      "description": "Does something useful",
      "input_schema": { ... },
      "execute": "module.my_tool.execute"
    }
  ],
  "commands": [
    {
      "name": "/mytool",
      "description": "Run my tool",
      "handler": "module.my_tool.command"
    }
  ],
  "dependencies": {
    "python": ["requests>=2.28"]
  }
}
```

---

### 7.3.2 Plugin Loader (loader.py: ~200 lines)

`discover_plugin_paths()` → find plugin.json → `load_plugin()` → instantiate `LoadedPlugin`:

```python
class LoadedPlugin:
    manifest: PluginManifest
    tool_instances: list[BaseTool]
    command_handlers: dict[str, Callable]
```

Then:
- `tool_instances` → register to `ToolRegistry`
- `command_handlers` → register to `CommandRegistry`

---

### 7.3.3 Compatibility

Supports two plugin formats:
1. **Anthropic Skills** (directory structure + prompts)
2. **Claude Code Plugins** (plugin.json specification)

OpenHarness's loader supports both, unifies into internal `LoadedPlugin` object.

---

## 7.4 Comparison with OpenClaw Skill System

| Comparison | OpenHarness | OpenClaw |
|------------|-------------|----------|
| Knowledge injection | Auto-match skill → append to system prompt | Auto-match + MEMORY search results |
| Tool extension | Plugin explicitly declares tools → register to ToolRegistry | Skill can dynamically return tools (more flexible) |
| Format | .md (simple) | JSON Schema + handler (rigorous) |
| Discovery | Scan directories (.openharness/skills + ~/.config/openharness/skills) | Scan workspace/skills/ subdirectories |
| Complexity | Low (suitable for non-programmers) | Medium (requires writing JSON + handler functions) |

**OpenClaw more powerful**:
- Skills can dynamically return tools (based on context)
- Memory system (OpenViking) can integrate skill search results

**OpenHarness simpler**:
- Markdown-only for knowledge, low barrier
- Plugins directly compatible with Claude Code ecosystem (existing plugins usable)

---

## 7.5 Practical: Adding Custom Skill to OpenHarness

Want to add "code review skill" to project:

1. Create `.openharness/skills/code-reviewer/skill.md`
2. Content:

```markdown
# Code Review Expert

## Description
When user requests PR or code review, I provide structured feedback.

## System Prompt
You are a senior Code Reviewer focusing on:
1. Security vulnerabilities (SQL injection, XSS)
2. Performance issues (O(n²), N+1 queries)
3. Readability (naming, comments)

## Output Format
```json
{
  "summary": "Overall assessment",
  "issues": [
    {"severity": "high", "file": "x.py", "line": 12, "desc": "..."}
  ]
}
```
```

3. Restart or new session, takes effect automatically.

---

## 7.6 Practical: Creating a Plugin

A plugin needs:
- `plugin.json` manifest
- One entry module `module.py`

Example: `ssh-tool` plugin, adds remote SSH execution capability.

Directory structure:
```
~/.config/openharness/plugins/ssh-tool/
├── plugin.json
├── module.py
└── requirements.txt  (optional)
```

`plugin.json`:

```json
{
  "name": "ssh-tool",
  "tools": [
    {
      "name": "SshExec",
      "description": "Execute command on remote host via SSH",
      "input_schema": {
        "type": "object",
        "properties": {
          "host": {"type": "string"},
          "command": {"type": "string"}
        },
        "required": ["host", "command"]
      },
      "execute": "module.SshExecTool.execute"
    }
  ]
}
```

`module.py`:

```python
from openharness.tools.base import BaseTool, ToolExecutionContext, ToolResult

class SshExecTool(BaseTool):
    name = "SshExec"
    description = "Execute command on remote host via SSH"
    input_model = SshInput  # Pydantic model

    async def execute(self, arguments, context):
        # Actual execution using paramiko SSH client
        ...
        return ToolResult(output=stdout, is_error=False)
```

---

Next Chapter: Hooks Lifecycle System (PRE/POST_TOOL_USE hooks for auditing, metrics)
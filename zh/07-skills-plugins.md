# 第7章：Skills & Plugins 可扩展机制

**模块**：
- `src/openharness/skills/`（6 文件，234 行）
- `src/openharness/plugins/`（6 文件，328 行）

---

## 7.1 两种扩展机制的区别

| 维度 | Skills | Plugins |
|------|--------|---------|
| **格式** | `.md` 知识库文件（简单） | `plugin.json` + Python/JS 模块（复杂） |
| **用途** | 注入领域知识到系统 prompt | 添加新工具、命令、能力 |
| **复杂度** | 低（用户可手写） | 高（需要编码） |
| **加载方式** | 自动扫描（按需加载） | 显式安装（pip/npm） |
| **兼容性** | OpenHarness 自有格式 | 兼容 Anthropic Skills + Claude Code Plugins |

**简单说**：
- 想「教 Agent 知识」 → Skills（写 Markdown）
- 想「给 Agent 加新工具」 → Plugins（写代码）

---

## 7.2 Skills：知识库即代码

### 7.2.1 Skill 定义（.md 格式）

```markdown
# 你的技能名称

## 描述
一句话说明这个技能是干什么的。

## 系统提示
当用户询问相关主题时，我会自动把以下内容注入到模型 system prompt：

```
你是数据分析专家。回答时优先使用 Python pandas，输出表格。
```

## 规则
- 必须使用类型标注
- 遇到缺失值用 `NaN` 填充
- 图表用 matplotlib，颜色用 tab10 调色板

## 示例
用户：统计一下销售数据
Assistant：`df.describe()`
```

---

### 7.2.2 加载机制（loader.py: ~150 行）

```python
def load_skill_registry(cwd: Path, user_skills_dir: Path) -> SkillRegistry:
    registry = SkillRegistry()
    
    # 1. 扫描项目内 .openharness/skills/
    for skill_dir in (cwd / ".openharness" / "skills").glob("*"):
        if (skill_dir / "skill.md").exists():
            definition = _parse_skill_markdown(skill_dir / "skill.md")
            registry.register(definition)
    
    # 2. 扫描全局用户技能目录
    for skill_dir in user_skills_dir.glob("*"):
        if (skill_dir / "skill.md").exists():
            definition = _parse_skill_markdown(skill_dir / "skill.md")
            registry.register(definition)
    
    return registry
```

**触发时机**：
- 每次 `QueryContext` 构建时，Engine 前会从 SkillRegistry 匹配相关 skill，将其 `system_prompt` 附加。

```python
# engine/query.py
system_prompt = base_system + "\n\n" + skill_registry.match(user_input)
```

---

### 7.2.3 SkillDefinition 结构

```python
@dataclass
class SkillDefinition:
    name: str
    description: str
    system_prompt: str  # 注入的提示词
    rules: list[str]    # 规则列表（可选）
    examples: list[Example]  # 示例对话（可选）
```

---

## 7.3 Plugins：插件生态

### 7.3.1 插件清单（plugin.json）

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

### 7.3.2 插件加载器（loader.py: ~200 行）

`discover_plugin_paths()` → 查找 plugin.json → `load_plugin()` → 实例化 `LoadedPlugin`：

```python
class LoadedPlugin:
    manifest: PluginManifest
    tool_instances: list[BaseTool]
    command_handlers: dict[str, Callable]
```

然后：
- `tool_instances` → 注册到 `ToolRegistry`
- `command_handlers` → 注册到 `CommandRegistry`

---

### 7.3.3 兼容性

支持两种插件格式：
1. **Anthropic Skills**（基于目录结构 + prompts）
2. **Claude Code Plugins**（plugin.json 规范）

OpenHarness 的 loader 同时支持两者，统一转为内部的 `LoadedPlugin` 对象。

---

## 7.4 与 OpenClaw 的技能系统对比

| 对比项 | OpenHarness | OpenClaw |
|--------|------------|----------|
| 知识注入 | 自动匹配 skill → 附加到 system prompt | 自动匹配 + MEMORY 搜索结果 |
| 工具扩展 | 插件明确声明 tools → 注册到 ToolRegistry | Skill 可以动态返回工具（更灵活） |
| 格式规范 | .md（简单） | JSON Schema + handler（严谨） |
| 发现机制 | 扫描目录（.openharness/skills + ~/.config/openharness/skills） | 扫描 workspace/skills/ 子目录 |
| 复杂度 | 低（适合非程序员） | 中（需要写 JSON + handler 函数） |

**OpenClaw 更强大**：
- 技能可以动态返回工具（根据上下文）
- 记忆系统（OpenViking）可接技能搜索结果

**OpenHarness 更简洁**：
- Markdown 就能写知识，门槛低
- 插件直接兼容 Claude Code 生态（已有现成插件可用）

---

## 7.5 实战：为 OpenHarness 添加自定义技能

想在项目里加一个「代码审查技能」：

1. 创建 `.openharness/skills/code-reviewer/skill.md`
2. 内容：

```markdown
# 代码审查专家

## 描述
当用户请求 review PR 或代码时，我会给出结构化的审查意见。

## 系统提示
你是资深 Code Reviewer，关注：
1. 安全漏洞（SQL 注入、XSS）
2. 性能问题（O(n²)、N+1 查询）
3. 可读性（命名、注释）

## 输出格式
```json
{
  "summary": "总体评价",
  "issues": [
    {"severity": "high", "file": "x.py", "line": 12, "desc": "..."}
  ]
}
```
```

3. 重启或新建会话，自动生效。

---

## 7.6 实战：创建插件

一个插件需要：
- `plugin.json` 清单
- 一个入口模块 `module.py`

示例：`ssh-tool` 插件，添加远程 SSH 执行功能。

目录结构：
```
~/.config/openharness/plugins/ssh-tool/
├── plugin.json
├── module.py
└── requirements.txt  (可选)
```

`plugin.json`：

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

`module.py`：

```python
from openharness.tools.base import BaseTool, ToolExecutionContext, ToolResult

class SshExecTool(BaseTool):
    name = "SshExec"
    description = "Execute command on remote host via SSH"
    input_model = SshInput  # Pydantic model

    async def execute(self, arguments, context):
        # 实际执行 paramiko SSH 客户端
        ...
        return ToolResult(output=stdout, is_error=False)
```

---

下一章：Hooks 生命周期系统（PRE/POST_TOOL_USE 钩子，用于审计、指标）

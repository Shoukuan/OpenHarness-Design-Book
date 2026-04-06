# 第6章：Memory 系统 —— CLAUDE.md + MEMORY.md + Auto-Compact 三件套

**核心模块**：
- `src/openharness/memory/`（7 文件，274 行）
- `src/openharness/services/compact/`（1 文件，492 行）

---

## 6.1 Memory 的三层架构

OpenHarness 的 Memory 系统分成三个独立职责的组件：

| 组件 | 职责 | 文件位置 | 代码行 |
|------|------|----------|--------|
| **CLAUDE.md 自动注入** | 项目级自定义指令（编码风格、规则） | `memory/loader.py` | ~80 |
| **MEMORY.md 持久化** | 跨会话记忆（用户偏好、事实、决策） | `memory/writer.py` | ~120 |
| **Auto-Compact 压缩** | 防止上下文溢出窗口 | `services/compact/` | 492 |

**设计哲学**：
- **短期记忆**：对话历史存在 Engine 的 `messages` 中，轮数多了就压缩
- **长期记忆**：跨会话的偏好和事实写到 `MEMORY.md`，下次启动自动加载
- **项目指令**：每个项目可以有 `CLAUDE.md`，自动注入 system prompt

---

## 6.2 CLAUDE.md：项目级别的 Custom Instructions

### 机制

Agent 启动时，从当前工作目录（cwd）**向上遍历**父目录，查找 `CLAUDE.md`。找到的第一个生效（类似 `.git` 的查找逻辑）。

```python
def load_claude_instructions(cwd: Path) -> str | None:
    for parent in [cwd] + list(cwd.parents):
        candidate = parent / "CLAUDE.md"
        if candidate.exists():
            return candidate.read_text()
    return None
```

### 内容格式

`CLAUDE.md` 是纯 Markdown，内容会原样注入到 system prompt：

```markdown
# 本项目规范

## 编码风格
- 所有 Python 文件必须包含类型标注
- 使用 black 格式化，行宽 88
- 异常处理必须记录到日志

## 测试要求
- 新功能必须写单测
- 使用 pytest 框架
- 覆盖率 > 80%
```

**注入位置**：在 Engine 构造 `messages` 时，第一条 message 的 `system` 字段会加上 CLAUDE.md 的内容。

---

## 6.3 MEMORY.md：持久化跨会话记忆

### 存储格式

使用 **YAML Frontmatter** + 正文：

```markdown
---
id: memory-2026-04-06-abc123
type: preference           # 类型：preference | decision | fact | project
created: 2026-04-06T08:42:00+08:00
---

用户偏好使用 web-search-plus 而不是 DuckDuckGo 直接搜索。

原因：搜索结果质量更高，且有 Serper/Tavily 等高质量提供商。
```

### 写入时机

| 触发场景 | 写入内容 |
|----------|----------|
| 用户明确说"记住" | 原话写入 MEMORY.md |
| 系统检测到偏好（如总是选某个选项） | 提炼后写入 |
| 用户纠正错误 | 记录正确做法 |
| 决策完成后 | 记录决策原因 |

写入函数：`memory/writer.py: append_memory(type, content)`，自动生成 UUID 和 timestamp。

### 读取时机

Agent 启动时，自动加载 `MEMORY.md` 所有记忆（解析 YAML frontmatter），注入到 system prompt：

```python
memories = parse_memory_file("MEMORY.md")
system_prompt += "\n\n# 历史记忆\n" + format_memories(memories)
```

---

## 6.4 Auto-Compact：自动上下文压缩

问题：对话历史会无限增长，而 LLM 有上下文窗口限制（如 200K tokens）。解决方案：压缩旧消息。

### 两种策略

| 策略 | 做法 | LLM 调用成本 | 效果 |
|------|------|-------------|------|
| **Micro-Compact** | 清除 `tool_result` 的 `output` 字段，替换为 "[Output truncated to save context]" | ❌ 0 | 节省 30-50% tokens，但丢失细节 |
| **Full Compact** | 调用 LLM 生成摘要，用一条 `Assistant` 消息替换多轮历史 | ✅ 付费 | 语义保留，节省 70%+ tokens |

### 触发条件

```python
def estimated_tokens(text: str) -> int:
    return int(len(text) / 4 * 1.33)  # 字符数 /4 加 33% 缓冲

if estimated_tokens(messages_text) > model.context_window * 0.8:
    # 超过 80% 窗口 → 开始压缩
```

### 压缩顺序

从最早的非系统消息开始，逐批压缩：

1. 先尝试 Micro-Compact（清除 tool_result）
2. 如果还不够，Full Compact（摘要）
3. **始终保留最近 5 条完整消息**（防止丢失核心上下文）

### 安全保护

- 只压缩 `tool_result` 内容，不删除 `user` / `assistant` 文本意图
- Full Compact 使用配置的小模型（如 `claude-3-haiku`），降低成本
- 压缩后记录一条 system 事件："context auto-compacted"

---

## 6.5 与 OpenViking（OpenClaw）的对比

OpenHarness 和 OpenClaw 都用 OpenViking 作为长期记忆向量库，但**短记忆策略不同**：

| 维度 | OpenHarness Compact | OpenViking (OpenClaw) |
|------|---------------------|-----------------------|
| **目标** | 减少 token 消耗（省钱） | 长期语义记忆召回（知识复用） |
| **粒度** | 整段对话摘要 | 片段级向量检索（按语义找回） |
| **存储** | 单条摘要消息留在消息列表 | 独立向量库 + 原始文件（MEMORY.md） |
| **成本** | LLM 摘要调用（几 cents） | 向量索引存储与检索（计算资源） |
| **互补性** | ✅ 两者可共存 | ✅ 两者可共存 |

**联合使用方案**：
- OpenHarness Auto-Compact 处理短期窗口压力
- OpenViking 向量库提供跨会话长期记忆检索
- 既省钱又保留知识

---

## 6.6 Token 估算算法（为什么不用精确 tokenizer）

OpenHarness 使用快速估算：

```python
def estimate_tokens(text: str) -> int:
    return int(len(text) / 4 * 1.33)
```

**原因**：
- 精确 tokenizer（tiktoken）需要加载模型文件，慢
- 压缩是高频操作，必须快
- 误差 10-20% 可接受，阈值设得宽松（80% 窗口才压缩）

---

## 6.7 技术细节：压缩顺序算法

```python
# services/compact/__init__.py 的核心逻辑
def compact_loop(messages, context_window, keep_recent=5):
    """从最早的消息开始压缩，直到满足窗口要求"""
    result = messages.copy()
    # 1. 跳过系统消息和最近 N 条
    for i in range(0, len(result) - keep_recent):
        if not should_compact(result[i]):
            continue
        # Micro-Compact: 只清 tool_result.output
        if micro_compact(result[i]):
            continue
        # Full Compact: 摘要一段连续的对话
        result[i] = full_compact(result[i:i+5])
        if estimated_tokens(result) < context_window * 0.8:
            break
    return result
```

---

第6章总结：三个组件各司其职——CLAUDE.md 做项目指令注入，MEMORY.md 做跨会话持久化，Auto-Compact 做窗口管理。三者结合，既记住长期知识，又不爆窗口。

下一章：[第7章 Skills & Plugins —— 两种扩展机制详解](07-skills-plugins.md)

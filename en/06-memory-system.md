# Chapter 6: Memory System — CLAUDE.md + MEMORY.md + Auto-Compact Trio

**Core Modules**:
- `src/openharness/memory/` (7 files, 274 lines)
- `src/openharness/services/compact/` (1 file, 492 lines)

---

## 6.1 Three-Layer Memory Architecture

OpenHarness's Memory system has three independently responsible components:

| Component | Responsibility | File Location | Lines |
|-----------|----------------|---------------|-------|
| **CLAUDE.md Auto-Injection** | Project-level custom instructions (coding style, rules) | `memory/loader.py` | ~80 |
| **MEMORY.md Persistence** | Cross-session memory (user preferences, facts, decisions) | `memory/writer.py` | ~120 |
| **Auto-Compact Compression** | Prevent context overflow | `services/compact/` | 492 |

**Design Philosophy**:
- **Short-term memory**: Conversation history exists in Engine's `messages`; compress when too many turns
- **Long-term memory**: Cross-session preferences and facts written to `MEMORY.md`, auto-loaded on next startup
- **Project instructions**: Each project can have `CLAUDE.md`, auto-injected into system prompt

---

## 6.2 CLAUDE.md: Project-Level Custom Instructions

### Mechanism

When Agent starts, traverse upward from cwd through parent directories, searching for `CLAUDE.md`. First found takes effect (similar to `.git` logic).

```python
def load_claude_instructions(cwd: Path) -> str | None:
    for parent in [cwd] + list(cwd.parents):
        candidate = parent / "CLAUDE.md"
        if candidate.exists():
            return candidate.read_text()
    return None
```

### Content Format

`CLAUDE.md` is plain Markdown, content is injected verbatim into system prompt:

```markdown
# Project Standards

## Coding Style
- All Python files must include type annotations
- Use black formatting, line width 88
- Exception handling must be logged

## Testing Requirements
- New features must have unit tests
- Use pytest framework
- Coverage > 80%
```

**Injection position**: When Engine constructs `messages`, the first message's `system` field includes CLAUDE.md content.

---

## 6.3 MEMORY.md: Persistent Cross-Session Memory

### Storage Format

Uses **YAML Frontmatter** + body:

```markdown
---
id: memory-2026-04-06-abc123
type: preference           # Types: preference | decision | fact | project
created: 2026-04-06T08:42:00+08:00
---

User prefers web-search-plus over direct DuckDuckGo searches.

Reason: Higher result quality with Serper/Tavily providers.
```

### Write Triggers

| Trigger Scenario | Written Content |
|------------------|-----------------|
| User explicitly says "remember" | Original text to MEMORY.md |
| System detects preference (e.g., always chooses option) | Refined entry |
| User corrects error | Record correct approach |
| After decision completion | Record decision rationale |

Write function: `memory/writer.py: append_memory(type, content)`, auto-generates UUID and timestamp.

### Read Timing

At Agent startup, automatically load all memories from `MEMORY.md` (parse YAML frontmatter), inject into system prompt:

```python
memories = parse_memory_file("MEMORY.md")
system_prompt += "\n\n# Historical Memories\n" + format_memories(memories)
```

---

## 6.4 Auto-Compact: Automatic Context Compression

Problem: Conversation history grows infinitely, but LLMs have context window limits (e.g., 200K tokens). Solution: compress old messages.

### Two Strategies

| Strategy | Approach | LLM Call Cost | Effectiveness |
|----------|----------|---------------|--------------|
| **Micro-Compact** | Clear `tool_result` `output` fields, replace with "[Output truncated to save context]" | ❌ 0 | Saves 30-50% tokens, but loses detail |
| **Full Compact** | Call LLM to generate summary, replaces old message list with one `Assistant` message | ✅ Paid | Semantic preservation, saves 70%+ tokens |

### Trigger Condition

```python
def estimated_tokens(text: str) -> int:
    return int(len(text) / 4 * 1.33)  # char count /4 plus 33% buffer

if estimated_tokens(messages_text) > model.context_window * 0.8:
    # Exceeds 80% of window → start compaction
```

### Compaction Order

Start from earliest non-system message, compress in batches:

1. Try Micro-Compact first (clear tool_results)
2. If still not enough, Full Compact (summary)
3. **Always keep last 5 complete messages** (prevent losing core context)

### Safety Protections

- Only compress `tool_result` content, don't delete `user`/`assistant` text intent
- Full Compact uses configured small model (e.g., `claude-3-haiku`) to reduce cost
- After compaction, record one system event: "context auto-compacted"

---

## 6.5 Comparison with OpenViking (OpenClaw)

Both OpenHarness and OpenClaw use OpenViking as long-term memory vector store, but **short-term memory strategies differ**:

| Dimension | OpenHarness Compact | OpenViking (OpenClaw) |
|-----------|--------------------|----------------------|
| **Goal** | Reduce token consumption (save money) | Long-term semantic memory recall (knowledge reuse) |
| **Granularity** | Whole conversation summary | Fragment-level vector search (retrieve by semantics) |
| **Storage** | Single summary message remains in message list | Independent vector db + raw file (MEMORY.md) |
| **Cost** | LLM summary call (few cents) | Vector indexing storage & retrieval (compute) |
| **Complementarity** | ✅ Both can coexist | ✅ Both can coexist |

**Combined usage approach**:
- OpenHarness Auto-Compact handles short-term window pressure
- OpenViking vector db provides cross-session long-term memory retrieval
- Both save money and preserve knowledge

---

## 6.6 Token Estimation Algorithm (Why Not Use Exact Tokenizer)

OpenHarness uses fast estimation:

```python
def estimate_tokens(text: str) -> int:
    return int(len(text) / 4 * 1.33)
```

**Reasoning**:
- Exact tokenizer (tiktoken) needs to load model files, slow
- Compaction is frequent operation, must be fast
- 10-20% error margin acceptable, threshold set loose (80% window before compress)

---

## 6.7 Technical Details: Compaction Order Algorithm

```python
# Core logic in services/compact/__init__.py
def compact_loop(messages, context_window, keep_recent=5):
    """Start from earliest messages, compress until window satisfied"""
    result = messages.copy()
    # 1. Skip system messages and last N
    for i in range(0, len(result) - keep_recent):
        if not should_compact(result[i]):
            continue
        # Micro-Compact: only clear tool_result.output
        if micro_compact(result[i]):
            continue
        # Full Compact: summarize a batch of consecutive dialogue
        result[i] = full_compact(result[i:i+5])
        if estimated_tokens(result) < context_window * 0.8:
            break
    return result
```

---

**Chapter Summary**: Three components each do their job — CLAUDE.md for project instruction injection, MEMORY.md for cross-session persistence, Auto-Compact for window management. Combined, they remember long-term knowledge without overflowing context.

Next Chapter: [Chapter 7: Skills & Plugins — Two Extension Mechanisms Explained](07-skills-plugins.md)
# Chapter 13: API Layer & OpenAI Compatibility (342 Lines)

**Module**: `src/openharness/api/` (9 files, 1,468 lines)

---

## 13.1 Why an API Layer?

OpenHarness internally uses Anthropic Messages API format (content_blocks, tool_use...), but most LLM providers (OpenAI, DashScope, DeepSeek, GitHub Models) provide **OpenAI-compatible interfaces**.

API layer's role: **format conversion + multi-provider routing**.

---

## 13.2 Core Abstraction

```python
class SupportsStreamingMessages(Protocol):
    """Any OpenAI-compatible client must implement this protocol"""
    async def stream_message(self, request: ApiMessageRequest) -> AsyncIterator[ApiStreamEvent]:
        ...
```

`ApiMessageRequest` is OpenHarness's internal request format:

```python
@dataclass
class ApiMessageRequest:
    model: str
    messages: list[ConversationMessage]
    system_prompt: str
    max_tokens: int
    tools: list[dict]  # Anthropic tool schema
```

---

## 13.3 OpenAI Client Adapter (openai_client.py: ~260 lines)

### 13.3.1 Format Conversion

OpenHarness → OpenAI conversion needed:

1. **System Prompt** (Anthropic separate parameter) → OpenAI's `{"role":"system", "content":...}`
2. **Tool format** (Anthropic input_schema) → OpenAI function-calling (`parameters` field)
3. **Message content** (Anthropic ContentBlock array) → OpenAI's `content` field (`text` or `tool_calls`)

```python
def _convert_tools_to_openai(tools: list[dict]) -> list[dict]:
    return [{
        "type": "function",
        "function": {
            "name": t["name"],
            "description": t.get("description", ""),
            "parameters": t.get("input_schema", {}),
        }
    } for t in tools]
```

---

### 13.3.2 Assistant Message Conversion (Most Complex)

Anthropic's assistant message:

```json
{
  "role": "assistant",
  "content": [
    {"type": "text", "text": "OK, I'll execute"},
    {"type": "tool_use", "id": "toolu_01", "name": "Bash", "input": {"command": "ls"}}
  ]
}
```

OpenAI format:

```json
{
  "role": "assistant",
  "content": "OK, I'll execute",
  "tool_calls": [{
    "id": "call_abc123",
    "type": "function",
    "function": {"name": "Bash", "arguments": "{\"command\":\"ls\"}"}
  }]
}
```

Conversion logic:

```python
def _convert_assistant_message(msg: ConversationMessage) -> dict:
    text_parts = []
    tool_calls = []
    for block in msg.content:
        if isinstance(block, TextBlock):
            text_parts.append(block.text)
        elif isinstance(block, ToolUseBlock):
            tool_calls.append({
                "id": block.id,
                "type": "function",
                "function": {
                    "name": block.name,
                    "arguments": json.dumps(block.input),
                }
            })
    return {
        "role": "assistant",
        "content": "\n".join(text_parts) or None,
        "tool_calls": tool_calls or None,
    }
```

---

### 13.3.3 Streaming Response Conversion

OpenAI returns SSE (`data: {"choices":[...]}`), OpenHarness needs to convert to internal `ApiStreamEvent`:

```python
async def stream_message(self, request: ApiMessageRequest) -> AsyncIterator[ApiStreamEvent]:
    openai_req = convert_to_openai_request(request)
    stream = await self._client.chat.completions.create(**openai_req)

    async for chunk in stream:
        if chunk.choices and chunk.choices[0].delta.content:
            yield ApiTextDeltaEvent(text=chunk.choices[0].delta.content)
        if chunk.choices and chunk.choices[0].delta.tool_calls:
            # Accumulate tool_calls, emit ApiMessageCompleteEvent when done
            ...
```

---

### 13.3.4 Provider Support

Dynamic configuration via `openai.AsyncOpenAI(base_url=..., api_key=...)`:

| Provider | base_url example |
|----------|------------------|
| DashScope | `https://dashscope.aliyuncs.com/compatible-mode/v1` |
| GitHub Models | `https://models.inference.ai.azure.com` |
| Ollama local | `http://localhost:11434/v1` |
| OpenAI official | `https://api.openai.com/v1` |

---

## 13.4 Copilot Auth Adapter (244 lines)

Use GitHub OAuth to access Copilot models (GPT-4.1, Claude, etc.) for free.

Flow:
1. `FeishuConfig` provides `client_id/secret`
2. User authorizes in browser, gets `access_token`
3. `CopilotAuthClient` uses `access_token` to call GitHub Models API

---

## 13.5 Comparison with OpenClaw API Layer

| Comparison | OpenHarness | OpenClaw |
|------------|-------------|----------|
| Target providers | OpenAI compatible + Copilot | Any Anthropic compatible |
| Auth method | API Key + OAuth (GitHub) | API Key (per platform) |
| Model discovery | List all models at startup, map | Hardcoded model IDs |
| Error handling | Retry (3 times) + backoff | Single provider |
| Streaming | ✅ Standard SSE | ✅ Standard SSE |

OpenHarness's API layer is more like a "universal adapter", OpenClaw is "direct connection".

---

## 13.6 Practical: Add a New Provider

Say add **SiliconFlow**:

1. Confirm it's OpenAI-compatible (most are)
2. Add provider in config file, no code changes needed:

```yaml
models:
  siliconflow:
    baseUrl: "https://api.siliconflow.com/v1"
    apiKey: "sk-..."
```

3. Test `oh --model siliconflow/Pro/DeepSeek-R1` works directly.

---

Next Chapter: [Chapter 14: Commands System — 54 CLI Commands, 1,456 Lines](14-commands-system.md)
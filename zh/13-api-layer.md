# 第13章：API 层与 OpenAI 兼容（342 行）

**模块**：`src/openharness/api/`（9 文件，1,468 行）

---

## 13.1 为什么需要 API 层？

OpenHarness 内部使用 Anthropic Messages API 格式（content_blocks, tool_use...），但市面上大多数 LLM 提供商（OpenAI、DashScope、DeepSeek、GitHub Models）都提供 **OpenAI 兼容接口**。

API 层的作用：**格式转换 + 多提供商路由**。

---

## 13.2 核心抽象

```python
class SupportsStreamingMessages(Protocol):
    """任何兼容 OpenAI 接口的客户端需实现此协议"""
    async def stream_message(self, request: ApiMessageRequest) -> AsyncIterator[ApiStreamEvent]:
        ...
```

`ApiMessageRequest` 是 OpenHarness 内部请求格式：

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

## 13.3 OpenAI 客户端适配器（openai_client.py: ~260 行）

### 13.3.1 格式转换

OpenHarness → OpenAI 需要转换：

1. **System Prompt**（Anthropic 是独立参数）→ OpenAI 的 `{"role":"system", "content":...}`
2. **工具格式**（Anthropic input_schema）→ OpenAI function-calling (`parameters` 字段)
3. **消息内容**（Anthropic ContentBlock 数组）→ OpenAI 的 `content` 字段（`text` 或 `tool_calls`）

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

### 13.3.2 Assistant 消息转换（最复杂）

Anthropic 的助手消息：

```json
{
  "role": "assistant",
  "content": [
    {"type": "text", "text": "好的，我来执行"},
    {"type": "tool_use", "id": "toolu_01", "name": "Bash", "input": {"command": "ls"}}
  ]
}
```

OpenAI 格式：

```json
{
  "role": "assistant",
  "content": "好的，我来执行",
  "tool_calls": [{
    "id": "call_abc123",
    "type": "function",
    "function": {"name": "Bash", "arguments": "{\"command\":\"ls\"}"}
  }]
}
```

转换逻辑：

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

### 13.3.3 流式响应转换

OpenAI 返回 SSE（`data: {"choices":[...]}`），OpenHarness 需要转为内部 `ApiStreamEvent`：

```python
async def stream_message(self, request: ApiMessageRequest) -> AsyncIterator[ApiStreamEvent]:
    openai_req = convert_to_openai_request(request)
    stream = await self._client.chat.completions.create(**openai_req)
    
    async for chunk in stream:
        if chunk.choices and chunk.choices[0].delta.content:
            yield ApiTextDeltaEvent(text=chunk.choices[0].delta.content)
        if chunk.choices and chunk.choices[0].delta.tool_calls:
            # 累积 tool_calls，完成时发出 ApiMessageCompleteEvent
            ...
```

---

### 13.3.4 提供商支持

通过 `openai.AsyncOpenAI(base_url=..., api_key=...)` 动态配置：

| 提供商 | base_url 示例 |
|--------|--------------|
| DashScope | `https://dashscope.aliyuncs.com/compatible-mode/v1` |
| GitHub Models | `https://models.inference.ai.azure.com` |
| Ollama 本地 | `http://localhost:11434/v1` |
| OpenAI 官方 | `https://api.openai.com/v1` |

---

## 13.4 Copilot Auth 适配器（244 行）

通过 GitHub OAuth 免费用 Copilot 模型（GPT-4.1、Claude 等）。

流程：
1. `FeishuConfig` 提供 `client_id/secret`
2. 用户在浏览器授权，获得 `access_token`
3. `CopilotAuthClient` 使用 `access_token` 调用 GitHub Models API

---

## 13.5 与 OpenClaw 的 API 层对比

| 对比项 | OpenHarness | OpenClaw |
|--------|------------|----------|
| 目标提供商 | OpenAI 兼容 + Copilot | 任何 Anthropic 兼容 |
| 认证方式 | API Key + OAuth (GitHub) | API Key (各平台) |
| 模型发现 | 启动时列表所有模型，映射 | 硬编码模型 ID |
| 错误处理 | 重试（3 次）+ 退避 | 单一 provider |
| 流式 | ✅ 标准 SSE | ✅ 标准 SSE |

OpenHarness 的 API 层更像「通用适配器」，OpenClaw 是「直接对接」。

---

## 13.6 实战：添加一个新提供商

比如要支持 **硅基流动 (SiliconFlow)**：

1. 确认其兼容 OpenAI 接口（大部分都兼容）
2. 在配置文件中添加provider，不改变代码即可使用：

```yaml
models:
  siliconflow:
    baseUrl: "https://api.siliconflow.com/v1"
    apiKey: "sk-..."
```

3. 测试 `oh --model siliconflow/Pro/DeepSeek-R1` 直接可用。

---

下一章：Commands 系统（54 个命令，1,456 行）

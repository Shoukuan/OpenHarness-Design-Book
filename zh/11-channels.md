# 第11章：Channels 渠道层（13 平台，5,183 行）

**模块**：`src/openharness/channels/`（18 文件，5,424 行）

---

## 11.1 渠道架构总览

```
channels/
├── base.py           : BaseChannel 抽象
├── bus/              : 消息总线（OutboundMessage, Queue）
└── impl/
    ├── feishu.py     : 945 行（飞书 + Lark）
    ├── mochat.py     : 897 行（企业微信）
    ├── matrix.py     : 700 行（Matrix）
    ├── telegram.py   : 509 行
    ├── dingtalk.py   : 445 行
    ├── email.py      : 410 行
    ├── discord.py    : 313 行
    ├── slack.py      : 281 行
    └── ...           : 其他
```

---

### 渠道矩阵


```mermaid
flowchart LR
    subgraph BUS["📦 消息总线 (bus/)"]
        OM["OutboundMessage"]
        Q["MessageQueue"]
        DISP["dispatch()"]
    end
    subgraph BASE["🔌 BaseChannel 抽象"]
        BC["parse_message()"]
        BS["send_message()"]
        GN["get_name()"]
    end
    subgraph IMPL["🏗️ 13 平台实现"]
        F["飞书 945行"]
        M["企业微信 897行"]
        MT["Matrix 700行"]
        T["Telegram 509行"]
        D["钉钉 445行"]
        E["Email 410行"]
        S["Slack 310行"]
        O["其他6平台"]
    end
    OM --> Q --> DISP --> BASE --> IMPL
    classDef bus fill:#e3f2fd,stroke:#1565c0,stroke-width:2px
    classDef impl fill:#fff3e0,stroke:#e65100,stroke-width:2px
    class BUS,BASE bus
    class IMPL impl
```


## 11.2 统一接口

```python
class BaseChannel(Protocol):
    async def start(self) -> None: ...
    async def stop(self) -> None: ...
    async def send(self, message: str, **kwargs) -> None: ...
    async def listen(self) -> AsyncIterator[ChannelEvent]: ...
```

所有渠道实现 `listen()` 方法，将外部 IM 消息转为内部 `ChannelEvent` 再送入 Agent Loop。

---

## 11.3 消息总线（MessageBus）

```python
class MessageBus:
    """中心化消息队列，渠道 → Agent 的中转站"""
    def __init__(self):
        self._queue: asyncio.Queue[ChannelEvent] = asyncio.Queue()
    
    async def publish(self, event: ChannelEvent):
        await self._queue.put(event)
    
    async def consume(self) -> AsyncIterator[ChannelEvent]:
        while True:
            yield await self._queue.get()
```

---

## 11.4 飞书实现详解（945 行）

### 11.4.1 SDK 选择

使用字节跳动的 `lark-oapi`，支持：
- WebSocket 长连接（事件实时推送）
- 消息类型全面（text, image, file, interactive, share...）

---

### 11.4.2 核心类

```python
class FeishuChannel(BaseChannel):
    def __init__(self, config: FeishuConfig, message_bus: MessageBus):
        self.config = config
        self.bus = message_bus
        self._client: lark.Client | None = None
        self._background_tasks: set[asyncio.Task] = set()
```

---

### 11.4.3 启动流程 (`start()`)

```python
async def start(self):
    # 1. 初始化 SDK
    self._client = lark.Client(
        app_id=self.config.app_id,
        app_secret=self.config.app_secret,
        enable_store_token=True,  # 自动管理 tenant_access_token
    )

    # 2. 启动 WebSocket 事件监听（后台任务）
    task = asyncio.create_task(self._event_loop())
    self._background_tasks.add(task)
    task.add_done_callback(self._background_tasks.discard)
```

---

### 11.4.4 事件循环 (`_event_loop()`)

```python
async def _event_loop(self):
    ws_client = self._client.ws_client()
    async for event in ws_client.iter_events():
        # 事件类型：im.message.received_v1, p2p.message.create, ...
        if event.type == "im.message.received_v1":
            await self._handle_message(event)
```

---

### 11.4.5 消息解析 (`_handle_message()`)

飞书消息 `content` 是 JSON 字符串，需根据 `msg_type` 解析：

```python
def _parse_content(content: dict, msg_type: str) -> str:
    if msg_type == "text":
        return content.get("text", "")
    elif msg_type == "image":
        return "[图片]"
    elif msg_type == "file":
        return f"[文件: {content.get('file_name')}]"
    elif msg_type == "interactive":
        return _extract_interactive_content(content)  # 提取卡片按钮、标题
    elif msg_type == "share_chat":
        return f"[分享群聊: {content.get('chat_id')}]"
    ...
```

---

### 11.4.6 发送消息 (`send()`)

```python
async def send(self, message: str, chat_id: str, **kwargs):
    # 支持 @人、富文本卡片
    if kwargs.get("mention_user_id"):
        message = f"<at user_id=\"{kwargs['mention_user_id']}\"></at> {message}"
    
    req = lark.im.MessageReq(
        receive_id_type="chat_id",
        receive_id=chat_id,
        msg_type="text",
        content=json.dumps({"text": message}),
    )
    resp = self._client.im.msg.send(req)
    if not resp.success:
        logger.error(f"Feishu send failed: {resp.msg}")
```

---

### 11.4.7 错误处理与重连

- WebSocket 异常自动重连（指数退避）
- `tenant_access_token` 过期自动刷新（SDK 内置）
- 消息发送失败记录到日志，不阻塞

---

## 11.5 企业微信实现（mochat.py: 897 行）

企业微信无官方 WebSocket API，实现更复杂：

- 轮询模式：定时 GET /getmsg?key=...
- 消息去重：`msgid` 缓存 24h
- 文件下载：先 `media_id` → 再 `/media/download`
- 机器人回复：需 `chatid` 且开启机器人权限

---

## 11.6 渠道对比表

| 渠道 | 实现行数 | API 类型 | 特点 |
|------|---------|----------|------|
| 飞书 | 945 | WebSocket | 实时性好，消息类型全 |
| 企业微信 | 897 | Polling | 需轮询，文件下载繁琐 |
| Matrix | 700 | WebSocket | 开源协议，去中心化 |
| Telegram | 509 | WebSocket + Polling | 本地化好，中国可用性差 |
| 钉钉 | 445 | Webhook (HTTP) | 简单但频率限制 |
| 邮件 | 410 | IMAP/SMTP | 异步，延迟高 |
| Discord | 313 | WebSocket | 国外社区适用 |
| Slack | 281 | WebSocket | 企业用，国内难访问 |

---

## 11.7 添加新渠道的步骤（实战）

假设要添加 **钉钉** 渠道（DingTalk）：

1. 创建 `src/openharness/channels/impl/dingtalk.py`（已有，可参考）
2. 继承 `BaseChannel`，实现 `start`, `stop`, `send`, `listen`
3. 注册到 `tools/channels/__init__.py` 的 `create_default_channel_registry()`
4. 添加配置模型 `FeishuConfig` 同级 `DingTalkConfig` 到 `config/schemas.py`
5. 更新 `config/schema.py` 的 `feishu.accounts` 同级添加 `dingtalk.accounts`
6. 写测试：`tests/channels/test_dingtalk.py`

---

## 11.8 与 OpenClaw 的渠道对比

OpenClaw 的 Feishu 实现在 `feishu-im` npm 包 + `bot` 目录，约 800 行 TS。

| 对比项 | OpenHarness | OpenClaw |
|--------|------------|----------|
| 协议 | WebSocket 长连接 | Webhook HTTP (webhooks.feishu.cn) |
| 消息去重 | 自动（msg_id 缓存） | 事件幂等处理 |
| 文件接收 | 需手动 `/media/download` | 自动下载到 `/tmp` |
| 扩展性 | 统一 BaseChannel，易添加新平台 | 需改多个模块 |

**OpenHarness 优势**：
- 统一抽象，代码结构清晰
- 消息总线解耦，渠道不影响 Engine

**OpenClaw 优势**：
- Webhook 模式对 Serverless 友好（无长连接）
- 文件自动下载（通过 feishu_im_bot_image）

---

下一章：MCP 协议集成（340 行实现 Model Context Protocol 客户端）

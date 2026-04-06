# 第12章：MCP 协议集成（340 行）

**模块**：`src/openharness/mcp/`（4 文件，340 行）

---

## 12.1 什么是 MCP？

**M**odel **C**ontext **P**rotocol（由 Anthropic 推出），标准化模型与外部工具的通信协议。

类似：
- OpenAI 的 Function Calling
- Anthropic 的 Tool Use
但 **MCP 是双向协议 + 独立进程**，通过 stdio 或 SSE 传输。

**MCP 服务器**：提供工具/资源的独立进程（如 filesystem、github、docker）
**MCP 客户端**：Agent 连接到服务器，动态获取工具集

---

## 12.2 OpenHarness 的 MCP 架构

```
OpenHarness Agent
       │
       ├─ McpClientManager (mcp/client.py)
       │      ├─ connect_all() → 启动 MCP 服务器子进程
       │      ├─ list_tools() → 返回工具列表
       │      └─ call_tool(server, name, arguments)
       │
       └─ McpToolAdapter (tools/mcp_tool.py)
               └─ 将 MCP 工具包装为 BaseTool 注册到 ToolRegistry
```

**效果**：一个 MCP 服务器提供的 100 个工具，自动变成 100 个 OpenHarness 工具。

---

## 12.3 配置 MCP

```yaml
# ~/.config/openharness/mcp.yaml
servers:
  filesystem:
    type: stdio
    command: "npx"
    args: ["-y", "@modelcontextprotocol/server-filesystem", "/path/to/allowed"]
    env:
      SOME_VAR: "value"
  github:
    type: stdio
    command: "node"
    args: ["/path/to/github-mcp-server", "--token", "ghp_xxx"]
```

启动时加载到 `McpClientManager`。

---

## 12.4 McpClientManager（~150 行）

核心方法：

```python
class McpClientManager:
    async def connect_all(self) -> None:
        for name, config in self._server_configs.items():
            if isinstance(config, McpStdioServerConfig):
                await self._connect_stdio(name, config)
    
    async def _connect_stdio(self, name: str, config: McpStdioServerConfig):
        # 启动子进程（MCP 服务器）
        process = await asyncio.create_subprocess_exec(
            config.command, *config.args,
            stdin=asyncio.subprocess.PIPE,
            stdout=asyncio.subprocess.PIPE,
        )
        # 通过 stdio 协议初始化会话
        session = ClientSession(process.stdin, process.stdout)
        await session.initialize()
        self._sessions[name] = session
        self._statuses[name] = McpConnectionStatus(state="connected")
    
    def list_tools(self) -> list[McpToolInfo]:
        """枚举所有已连接服务器的工具"""
        tools = []
        for name, session in self._sessions.items():
            response = sync_wrap(session.list_tools)()  # 同步转异步
            for tool in response.tools:
                tools.append(McpToolInfo(
                    server_name=name,
                    name=tool.name,
                    description=tool.description,
                    input_schema=tool.inputSchema,
                ))
        return tools
    
    async def call_tool(self, server_name: str, tool_name: str, arguments: dict) -> str:
        session = self._sessions[server_name]
        result: CallToolResult = await session.call_tool(tool_name, arguments)
        return result.content[0].text  # 假设文本输出
```

---

## 12.5 McpToolAdapter：MCP → OpenHarness 桥梁

```python
class McpToolAdapter(BaseTool):
    def __init__(self, manager: McpClientManager, tool_info: McpToolInfo) -> None:
        self._manager = manager
        self._tool_info = tool_info
        # 工具名格式：mcp__filesystem__read_file
        self.name = f"mcp__{server_segment}__{tool_segment}"
        self.description = tool_info.description
        # 动态生成 Pydantic Input Model
        self.input_model = _input_model_from_schema(self.name, tool_info.input_schema)

    async def execute(self, arguments: BaseModel, context: ToolExecutionContext) -> ToolResult:
        return await self._manager.call_tool(
            self._tool_info.server_name,
            self._tool_info.name,
            arguments.model_dump(),
        )
```

注册时：

```python
for tool_info in mcp_manager.list_tools():
    registry.register(McpToolAdapter(mcp_manager, tool_info))
```

---

## 12.6 与 OpenClaw 的工具生态对比

| 对比项 | OpenHarness (MCP) | OpenClaw (Skills) |
|--------|-------------------|-------------------|
| 扩展机制 | 标准协议（MCP） | 自定义 JSON + Handler |
| 工具来源 | 独立进程（文件系统、数据库、GitHub...） | Python 函数 |
| 动态性 | 运行时连接，实时获取工具 | 启动时扫描，静态注册 |
| 安全性 | MCP 服务器进程隔离 | 直接在同一进程执行（需额外沙箱） |
| 生态 | 已有几十个 MCP 服务器（filesystem, github, sqlite...） | 依赖 Skill 开发者 |

**OpenHarness 赢在标准**：任何符合 MCP 的工具都能接入，生态爆炸可能。

---

## 12.7 实战：连接文件系统 MCP 服务器

1. 安装文件系统 MCP 服务器：

```bash
npm install -g @modelcontextprotocol/server-filesystem
```

2. 配置 OpenHarness：

```yaml
mcp:
  servers:
    myfs:
      type: stdio
      command: "npx"
      args: ["@modelcontextprotocol/server-filesystem", "/Users/you/projects"]
```

3. 启动 OpenHarness：

```bash
oh  # 自动连接 MCP，工具名变成：mcp__myfs__read_file
```

4. Agent 使用：

```
Agent: 列出 ~/projects 目录下的所有 .py 文件
→ 调用 mcp__myfs__list_directory 工具
```

---

## 12.8 设计评价

**优点**：
- 标准化：一个协议对接无限工具
- 隔离：MCP 服务器独立进程，崩溃不影响 Agent
- 动态：启动不同服务器得到不同工具集

**缺点**：
- 依赖第三方 MCP 服务器（需 Node.js 环境）
- 无内置权限检查（MCP 服务器需自行实现）
- 中文文档少，学习曲线陡

---

下一章：API 层与 OpenAI 兼容（342 行实现与对比）

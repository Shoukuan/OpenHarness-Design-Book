# Chapter 12: MCP Integration (340 Lines)

**Module**: `src/openharness/mcp/` (4 files, 340 lines)

---

## 12.1 What is MCP?

**M**odel **C**ontext **P**rotocol (launched by Anthropic) standardizes communication between models and external tools.

Similar to:
- OpenAI's Function Calling
- Anthropic's Tool Use
But **MCP is a bidirectional protocol + separate process**, transmitting via stdio or SSE.

**MCP Server**: Independent process providing tools/resources (e.g., filesystem, GitHub, Docker)
**MCP Client**: Agent connects to server, dynamically fetches toolset

---

## 12.2 OpenHarness's MCP Architecture

```
OpenHarness Agent
       │
       ├─ McpClientManager (mcp/client.py)
       │      ├─ connect_all() → start MCP server subprocesses
       │      ├─ list_tools() → return tool list
       │      └─ call_tool(server, name, arguments)
       │
       └─ McpToolAdapter (tools/mcp_tool.py)
               └─ Wraps MCP tools as BaseTool registered in ToolRegistry
```

**Effect**: 100 tools from one MCP server automatically become 100 OpenHarness tools.

---

## 12.3 Configure MCP

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

Loaded into `McpClientManager` at startup.

---

## 12.4 McpClientManager (~150 lines)

Core methods:

```python
class McpClientManager:
    async def connect_all(self) -> None:
        for name, config in self._server_configs.items():
            if isinstance(config, McpStdioServerConfig):
                await self._connect_stdio(name, config)

    async def _connect_stdio(self, name: str, config: McpStdioServerConfig):
        # Start subprocess (MCP server)
        process = await asyncio.create_subprocess_exec(
            config.command, *config.args,
            stdin=asyncio.subprocess.PIPE,
            stdout=asyncio.subprocess.PIPE,
        )
        # Initialize session via stdio protocol
        session = ClientSession(process.stdin, process.stdout)
        await session.initialize()
        self._sessions[name] = session
        self._statuses[name] = McpConnectionStatus(state="connected")

    def list_tools(self) -> list[McpToolInfo]:
        """Enumerate all connected servers' tools"""
        tools = []
        for name, session in self._sessions.items():
            response = sync_wrap(session.list_tools)()  # sync→async
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
        return result.content[0].text  # assume text output
```

---

## 12.5 McpToolAdapter: MCP → OpenHarness Bridge

```python
class McpToolAdapter(BaseTool):
    def __init__(self, manager: McpClientManager, tool_info: McpToolInfo) -> None:
        self._manager = manager
        self._tool_info = tool_info
        # Tool name format: mcp__filesystem__read_file
        self.name = f"mcp__{server_segment}__{tool_segment}"
        self.description = tool_info.description
        # Dynamically generate Pydantic Input Model
        self.input_model = _input_model_from_schema(self.name, tool_info.input_schema)

    async def execute(self, arguments: BaseModel, context: ToolExecutionContext) -> ToolResult:
        return await self._manager.call_tool(
            self._tool_info.server_name,
            self._tool_info.name,
            arguments.model_dump(),
        )
```

Registration:

```python
for tool_info in mcp_manager.list_tools():
    registry.register(McpToolAdapter(mcp_manager, tool_info))
```

---

## 12.6 Comparison with OpenClaw Tool Ecosystem

| Comparison | OpenHarness (MCP) | OpenClaw (Skills) |
|------------|-------------------|-------------------|
| Extension mechanism | Standard protocol (MCP) | Custom JSON + Handler |
| Tool source | Independent processes (filesystem, database, GitHub...) | Python functions |
| Dynamicity | Runtime connection, real-time tool fetch | Scan at startup, static registration |
| Security | MCP server process isolation | Direct same-process execution (needs extra sandbox) |
| Ecosystem | Dozens of existing MCP servers (filesystem, github, sqlite...) | Depends on Skill developers |

**OpenHarness wins on standard**: Any MCP-compliant tool can connect, ecosystem explosion potential.

---

## 12.7 Practical: Connect Filesystem MCP Server

1. Install filesystem MCP server:

```bash
npm install -g @modelcontextprotocol/server-filesystem
```

2. Configure OpenHarness:

```yaml
mcp:
  servers:
    myfs:
      type: stdio
      command: "npx"
      args: ["@modelcontextprotocol/server-filesystem", "/Users/you/projects"]
```

3. Start OpenHarness:

```bash
oh  # Auto-connect MCP, tool names become: mcp__myfs__read_file
```

4. Agent usage:

```
Agent: List all .py files in ~/projects
→ calls mcp__myfs__list_directory tool
```

---

## 12.8 Design Evaluation

**Pros**:
- Standardization: One protocol connects unlimited tools
- Isolation: MCP server separate process, crash won't affect Agent
- Dynamic: Start different servers get different tool sets

**Cons**:
- Depends on third-party MCP servers (requires Node.js)
- No built-in permission check (MCP server must implement itself)
- Limited Chinese documentation, steep learning curve

---

Next Chapter: [Chapter 13: API Layer & OpenAI Compatibility (342 Lines Implementation & Comparison)](13-api-layer.md)
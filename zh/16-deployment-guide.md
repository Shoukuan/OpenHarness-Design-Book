# 第16章：实战部署指南 —— 从零到线上可用

---

## 16.1 快速开始（30 分钟）

### 1. 环境准备

```bash
# Python
python3 --version  # >= 3.10

# Optional: Node.js for TUI
node --version     # >= 18.x

# 创建虚拟环境
python3 -m venv .venh
source .venv/bin/activate

# 安装 OpenHarness
pip install openharness

# Optional: 安装 TUI
npm install -g @openharness/tui
```

---

### 2. 初始化配置

```bash
# 生成默认配置文件
oh init

# 编辑配置文件
vim ~/.config/openharness/config.yaml
```

**最小配置**（AI + 渠道）：

```yaml
models:
  default: "qwen/qwen3.6-plus:free"  # 通义千问免费模型

channels:
  feishu:
    accounts:
      main:
        appId: "cli_xxxxxx"
        appSecret: "xxxxxxxx"
        encryptionKey: "base64:xxxx"
```

**模型说明**：
- `qwen/qwen3.6-plus:free` —— 通义千问免费 tier（OpenRouter 路由）
- 替换为 `claude-3-opus` 或 `deepseek-chat` 也可

---

### 3. 启动

**CLI 模式**（一次性问题）：

```bash
oh -p "用 Python 写一个冒泡排序"
```

输出：

🤖 回答：
```python
def bubble_sort(arr):
    n = len(arr)
    for i in range(n):
        for j in range(0, n-i-1):
            if arr[j] > arr[j+1]:
                arr[j], arr[j+1] = arr[j+1], arr[j]
    return arr
```

Usage: ～85K tokens (估算)
---

**TUI 模式**（交互式）：

```bash
oh
```

进入全屏终端 UI，类似 Claude Code 的界面。

---

### 4. 飞书 Bot 配置

1. 打开飞书开发者后台 → 创建机器人
2. 权限：`im:message`、`im:resource`、`contact:user`
3. Webhook 地址：`https://你的域名/openharness/feishu/callback`
4. 加密密钥：生成随机 32 字节 base64
5. 订阅事件：`im.message.receive.v1`
6. 保存到 `config.yaml`（见上）

---

## 16.2 二次开发：添加自定义工具

**目标**：实现一个查询公司 CRM 的工具。

---

### 步骤 1：创建 Python 包

```bash
mkdir my-company-tools && cd my-company-tools
cat > pyproject.toml << 'EOF'
[build-system]
requires = ["setuptools>=61"]
build-backend = "setuptools.build_meta"

[project]
name = "my-company-tools"
version = "0.1.0"
dependencies = ["openharness"]
EOF
```

---

### 步骤 2：实现工具

`my_company_tools/__init__.py`：

```python
from openharness.tools.base import BaseTool, ToolRegistry, ToolResult
from pydantic import BaseModel
import os
import httpx

class CrmLookupInput(BaseModel):
    customer_id: str

class CrmLookupTool(BaseTool):
    name = "CrmLookup"
    description = "查询公司 CRM 系统客户信息（客户ID、合同金额、联系人）"
    input_model = CrmLookupInput

    async def execute(self, arguments: CrmLookupInput, context):
        api_key = os.getenv("CRM_API_KEY")
        if not api_key:
            return ToolResult(
                output="",
                error="CRM_API_KEY 环境变量未设置",
                is_error=True,
            )

        async with httpx.AsyncClient() as client:
            resp = await client.get(
                f"https://crm.example.com/api/customers/{arguments.customer_id}",
                headers={"Authorization": f"Bearer {api_key}"},
                timeout=10,
            )
            if resp.status_code == 200:
                data = resp.json()
                return ToolResult(
                    output=f"客户ID: {data['id']}\n"
                           f"名称: {data['name']}\n"
                           f"合同金额: ¥{data['contract_value']}\n"
                           f"联系人: {data['contact_name']}",
                )
            else:
                return ToolResult(
                    output=f"HTTP {resp.status_code}",
                    error=resp.text,
                    is_error=True,
                )

def register(registry: ToolRegistry):
    registry.register(CrmLookupTool())
```

---

### 步骤 3：安装与激活

```bash
# 开发模式安装
pip install -e .

# 在 config.yaml 中启用插件
plugins:
  - module: "my_company_tools"
    config: {}   # 可选配置
```

重启 `oh` 服务，`CrmLookup` 工具自动注册。

---

## 16.3 Docker 容器化部署

```dockerfile
# Dockerfile
FROM python:3.11-slim

WORKDIR /app

# 安装依赖
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

# 复制代码
COPY . .

# 配置文件（运行时挂载）
VOLUME /root/.config/openharness

# 启动
CMD ["oh"]
```

```bash
# 构建
docker build -t openharness:latest .

# 运行
docker run -d \
  -v ~/.config/openharness:/root/.config/openharness \
  -e CRM_API_KEY=xxx \
  -p 3000:3000 \
  --name oh-agent \
  openharness:latest
```

---

## 16.4 K8s 部署（企业级）

```yaml
# deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: openharness-agent
spec:
  replicas: 1
  selector:
    matchLabels:
      app: openharness
  template:
    metadata:
      labels:
        app: openharness
    spec:
      containers:
      - name: agent
        image: openharness:latest
        ports:
        - containerPort: 3000
        env:
        - name: CRM_API_KEY
          valueFrom:
            secretKeyRef:
              name: openharness-secrets
              key: crm-api-key
        volumeMounts:
        - name: config
          mountPath: /root/.config/openharness
        resources:
          requests:
            memory: "512Mi"
            cpu: "500m"
          limits:
            memory: "2Gi"
            cpu: "2000m"
---
apiVersion: v1
kind: Service
metadata:
  name: openharness-service
spec:
  selector:
    app: openharness
  ports:
  - protocol: TCP
    port: 80
    targetPort: 3000
```

---

## 16.5 监控与日志

**日志**：OpenHarness 使用 structlog，输出 JSON 格式。

```yaml
# config.yaml
logging:
  level: "INFO"
  format: "json"
  output: "/var/log/openharness/agent.log"
```

**Prometheus 指标**（可选插件）：

```bash
pip install openharness[metrics]
```

```yaml
metrics:
  enabled: true
  port: 9090
  path: /metrics
```

指标包括：
- `oh_requests_total`
- `oh_request_duration_seconds`
- `oh_tool_usage_total`

---

## 16.6 常见问题与排错

| 问题 | 可能原因 | 解决 |
|------|---------|------|
| 模型调用 429 限流 | API key 额度耗尽 | 更换提供商或升级套餐 |
| 飞书收不到消息 | Webhook URL 错误或签名验证失败 | 检查 `verification_token`、`encryption_key` |
| 工具执行超时 | 命令耗时太长 | 增大工具超时配置或优化命令 |
| 内存泄漏 | 历史消息未压缩 | 检查 `auto_compact_if_needed` 是否触发 |
| 权限拒绝 | `permissions.yaml` 太严格 | 放宽规则或添加白名单路径 |

---

## 16.7 性能优化建议

1. **模型选择**：用 `qwen1.5-14b` 等轻量模型处理简单任务
2. **并行工具**：OpenHarness 默认并行执行多工具调用，确保模型返回多个 tool_use
3. **Auto-Compact 阈值**：调低到 `0.7` 更激进压缩
4. **本地模型**：Ollama 运行 `llama3:70b` 零 API 费用
5. **渠道选择**：不需要的渠道（如 email）在 config 里禁用

---

第16章总结：部署 OpenHarness 只需 30 分钟，从 pip install 到配置渠道。自定义工具只需继承 BaseTool 并注册，Docker + K8s 部署成熟。监控、日志、排错指南齐全。

---

**全书完** 🎉

**全书目录**：
- 第1章：OpenHarness 是什么
- 第2章：架构全景图
- 第3章：Engine 核心循环
- 第4章：Tools 系统
- 第5章：Permissions 权限系统
- 第6章：Memory 持久化
- 第7章：Skills & Plugins
- 第8章：Hooks 生命周期
- 第9章：Coordinator 多 Agent 协调
- 第10章：Swarm 集群协调
- 第11章：Channels 渠道层
- 第12章：MCP 协议集成
- 第13章：API 层
- 第14章：Commands 系统
- 第15章：全面对比
- 第16章：实战部署指南

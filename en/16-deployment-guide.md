# Chapter 16: Practical Deployment Guide — From Zero to Live

---

## 16.1 Quick Start (30 minutes)

### 1. Environment Setup

```bash
# Python
python3 --version  # >= 3.10

# Optional: Node.js for TUI
node --version     # >= 18.x

# Create virtual environment
python3 -m venv .venv
source .venv/bin/activate

# Install OpenHarness
pip install openharness

# Optional: Install TUI
npm install -g @openharness/tui
```

---

### 2. Initialize Configuration

```bash
# Generate default config
oh init

# Edit config file
vim ~/.config/openharness/config.yaml
```

**Minimum config** (AI + channels):

```yaml
models:
  default: "qwen/qwen3.6-plus:free"  # Qwen free tier (OpenRouter routed)

channels:
  feishu:
    accounts:
      main:
        appId: "cli_xxxxxx"
        appSecret: "xxxxxxxx"
        encryptionKey: "base64:xxxx"
```

**Model notes**:
- `qwen/qwen3.6-plus:free` — Qwen free tier (OpenRouter routing)
- Replace with `claude-3-opus` or `deepseek-chat` also works

---

### 3. Start Up

**CLI mode** (one-off questions):

```bash
oh -p "Write bubble sort in Python"
```

Output:

🤖 Answer:
```python
def bubble_sort(arr):
    n = len(arr)
    for i in range(n):
        for j in range(0, n-i-1):
            if arr[j] > arr[j+1]:
                arr[j], arr[j+1] = arr[j+1], arr[j]
    return arr
```

Usage: ~85K tokens (estimated)
---

**TUI mode** (interactive):

```bash
oh
```

Enters fullscreen terminal UI, similar to Claude Code experience.

---

### 4. Feishu Bot Configuration

1. Open Feishu Developer Console → Create bot
2. Permissions: `im:message`, `im:resource`, `contact:user`
3. Webhook URL: `https://your-domain/openharness/feishu/callback`
4. Encryption key: Generate random 32-byte base64
5. Subscribe events: `im.message.receive.v1`
6. Save to `config.yaml` (see above)

---

## 16.2 Secondary Development: Add Custom Tool

**Goal**: Implement a company CRM query tool.

---

### Step 1: Create Python Package

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

### Step 2: Implement Tool

`my_company_tools/__init__.py`:

```python
from openharness.tools.base import BaseTool, ToolRegistry, ToolResult
from pydantic import BaseModel
import os
import httpx

class CrmLookupInput(BaseModel):
    customer_id: str

class CrmLookupTool(BaseTool):
    name = "CrmLookup"
    description = "Query company CRM customer info (customer ID, contract value, contact person)"
    input_model = CrmLookupInput

    async def execute(self, arguments: CrmLookupInput, context):
        api_key = os.getenv("CRM_API_KEY")
        if not api_key:
            return ToolResult(
                output="",
                error="CRM_API_KEY environment variable not set",
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
                    output=f"Customer ID: {data['id']}\n"
                           f"Name: {data['name']}\n"
                           f"Contract Value: ¥{data['contract_value']}\n"
                           f"Contact: {data['contact_name']}",
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

### Step 3: Install & Activate

```bash
# Install in development mode
pip install -e .

# Enable plugin in config.yaml
plugins:
  - module: "my_company_tools"
    config: {}   # Optional config
```

Restart `oh` service, `CrmLookup` tool auto-registered.

---

## 16.3 Docker Containerized Deployment

```dockerfile
# Dockerfile
FROM python:3.11-slim

WORKDIR /app

# Install dependencies
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

# Copy code
COPY . .

# Config file (mounted at runtime)
VOLUME /root/.config/openharness

# Start
CMD ["oh"]
```

```bash
# Build
docker build -t openharness:latest .

# Run
docker run -d \
  -v ~/.config/openharness:/root/.config/openharness \
  -e CRM_API_KEY=xxx \
  -p 3000:3000 \
  --name oh-agent \
  openharness:latest
```

---

## 16.4 K8s Deployment (Enterprise-Grade)

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

## 16.5 Monitoring & Logging

**Logging**: OpenHarness uses structlog, outputs JSON format.

```yaml
# config.yaml
logging:
  level: "INFO"
  format: "json"
  output: "/var/log/openharness/agent.log"
```

**Prometheus metrics** (optional plugin):

```bash
pip install openharness[metrics]
```

```yaml
metrics:
  enabled: true
  port: 9090
  path: /metrics
```

Metrics include:
- `oh_requests_total`
- `oh_request_duration_seconds`
- `oh_tool_usage_total`

---

## 16.6 Troubleshooting

| Issue | Possible Cause | Solution |
|-------|----------------|----------|
| Model call 429 rate limit | API key quota exhausted | Switch provider or upgrade plan |
| Feishu not receiving messages | Webhook URL wrong or signature verification fails | Check `verification_token`, `encryption_key` |
| Tool execution timeout | Command takes too long | Increase tool timeout config or optimize command |
| Memory leak | History messages not compacted | Check if `auto_compact_if_needed` triggers |
| Permission denied | `permissions.yaml` too strict | Loosen rules or add whitelist paths |

---

## 16.7 Performance Optimization Recommendations

1. **Model selection**: Use light models like `qwen1.5-14b` for simple tasks
2. **Parallel tools**: OpenHarness parallelizes multi-tool calls by default, ensure model returns multiple tool_uses
3. **Auto-Compact threshold**: Lower to `0.7` for more aggressive compaction
4. **Local models**: Ollama with `llama3:70b` zero API cost
5. **Channel selection**: Disable unneeded channels (e.g., email) in config

---

**Chapter 16 Summary**: Deploying OpenHarness takes just 30 minutes from pip install to channel configuration. Custom tools only need to inherit BaseTool and register. Docker + K8s deployment mature. Monitoring, logging, troubleshooting guides complete.

---

**Book Complete** 🎉

**Table of Contents**:
- Chapter 1: What is OpenHarness
- Chapter 2: Architecture Panorama
- Chapter 3: Engine Core Loop
- Chapter 4: Tools System
- Chapter 5: Permissions System
- Chapter 6: Memory Persistence
- Chapter 7: Skills & Plugins
- Chapter 8: Hooks Lifecycle
- Chapter 9: Coordinator Multi-Agent Orchestration
- Chapter 10: Swarm Cluster Coordination
- Chapter 11: Channels Layer
- Chapter 12: MCP Protocol Integration
- Chapter 13: API Layer
- Chapter 14: Commands System
- Chapter 15: Full Comparison
- Chapter 16: Practical Deployment Guide
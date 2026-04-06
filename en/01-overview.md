# Chapter 1: What Is OpenHarness

## 1.1 Project Positioning and Vision

OpenHarness is an **enterprise-grade open-source Agent framework**, released by the University of Hong Kong in 2025 under the MIT license. Its goal is to wrap large language models (LLMs) into **fully functional intelligent agents that can actually get work done**.

**One-sentence summary:** An open-source alternative to Claude Code, with comprehensive coverage of the Chinese market including Feishu, DingTalk, and WeCom.

---

### Why Should You Care About OpenHarness?

The market is full of Agent frameworks (LangChain, AutoGPT, OpenManus...), but OpenHarness is unique in these ways:

| Dimension | Other Frameworks | OpenHarness |
|-----------|------------------|-------------|
| **Completeness** | Most are half-finished, requiring extensive customization | Ready-to-use, 114 unit tests passing + 6 E2E test suites |
| **Channel Integration** | Basically only Slack/Discord | **Feishu, DingTalk, WeCom** comprehensive coverage (essential for Chinese market) |
| **Tool Coverage** | Scattered implementations, missing this and that | Covers **98%** of Claude Code's tool capabilities |
| **Enterprise-Ready** | Lacks permissions, context management | Multi-tier permission sandbox, context compression, session recovery complete package |
| **Code Size** | Often tens of thousands of lines, hard to understand | Core only **11,000 lines** (engine 666 lines), highly readable source |

---

## 1.2 Technical Architecture Overview

### Tech Stack

| Component | Technology Choice | Notes |
|-----------|-------------------|-------|
| Language | **Python 3.10+** | Low barrier, easy for teams to extend |
| Frontend/CLI | React TUI (based on Ink) | In-terminal interaction, claude-code-like experience |
| Protocol | **MCP** (Model Context Protocol) | Open standard led by Anthropic |
| Testing | pytest | 114 unit tests passed + 6 end-to-end suites |
| License | **MIT** | Commercially friendly, no AGPL contagion |

### Core Architecture Diagram

```
                    ┌─────────────────────────┐
                    │   User (Multi-channel)  │
                    │ Feishu/DingTalk/WeCom/CLI │
                    └──────────┬──────────────┘
                               │
                    ┌──────────▼──────────────┐
                    │   Channel Layer         │  ← 5,183 lines, 13 implementations
                    │ (Message send/receive,  │
                    │  session management)    │
                    └──────────┬──────────────┘
                               │
                    ┌──────────▼──────────────┐
                    │   Engine Core Loop      │  ← 666 lines, the heart
                    │  (Tool Use Loop)        │
                    └──┬─────┬───┬────┬───────┘
                       │     │   │    │
            ┌──────────┘     │   │    └──────────┐
            ▼                ▼   ▼               ▼
    ┌──────────────┐ ┌──────────┐ ┌────────────┐  ┌──────────────┐
    │  Tools        │ │Permissions│ │ Memory     │  │   Swarm      │
    │  (43+)        │ │Permissions│ │ System     │  │ Multi-Agent  │
    └──────────────┘ └──────────┘ └────────────┘  └──────────────┘
            │
            ▼
    ┌─────────────────────────────┐
    │    Hooks Subsystem          │  ← Pre/Post Tool Use
    │    Skills Library           │  ← On-demand .md loading
    │    Plugins Compatibility    │  ← Anthropic format
    │    Cron Scheduler           │  ← Built-in scheduling
    └─────────────────────────────┘
```

---

## 1.3 Code Size Distribution (Precise Source Data)

Using `wc` for exact counting: **194 Python files, 26,666 lines of code**.

| Module | Files | Lines | Percentage | One-line Description |
|--------|-------|-------|------------|-----------------------|
| **channels/impl** | 13 | 5,183 | 19.4% | Multi-channel messaging — the project's moat |
| **swarm** | 11 | 4,899 | 18.4% | Multi-agent coordination scheduler |
| **tools** | 42 | 2,542 | 9.5% | 43 tool implementations |
| **ui** | 10 | 2,006 | 7.5% | CLI terminal interaction |
| **api** | 9 | 1,468 | 5.5% | RESTful API |
| **commands** | 2 | 1,456 | 5.5% | 54 CLI commands |
| **coordinator** | 3 | 1,506 | 5.6% | Multi-agent orchestration |
| **services** | 8 | 1,388 | 5.2% | Common service layer |
| **plugins** | 6 | 328 | 1.2% | Plugin loader |
| **config** | 3 | 362 | 1.4% | Config loading & validation |
| **engine** | 6 | **666** | 2.5% | ⭐ Core loop — protagonist of this book |
| Others | 33 | 4,875 | 18.3% | Utils, exceptions, constants |
| **Total** | **194** | **26,666** | **100%** | |

**Key Insights:**

> 🔑 **The largest module isn't engine, it's channels (channel layer)**. This shows OpenHarness's core competitiveness isn't "yet another Agent loop implementation" but **"ability to deliver Agents to all platforms users interact with daily"**.

> 🔑 **Engine is only 666 lines** — meaning the core loop is extremely simple and easy to understand. This is the focus of Chapter 3.

> 🔑 **Swarm (18.4%) surpasses tools (9.5%)** — multi-agent coordination is the project's strategic focus, not just an add-on feature.

---

## 1.4 Core Feature Checklist

| Feature | Description | Enterprise Value |
|---------|-------------|------------------|
| 🔄 **Agent Loop** | Streaming tool-calling loop with parallel tool execution | Low latency, high throughput |
| 🔧 **43+ Tools** | File I/O, Shell, Web search, MCP, task management... | Covers everyday enterprise operations |
| 🧠 **Memory** | CLAUDE.md auto-injection + MEMORY.md persistence + session recovery | Agent has "long-term memory" |
| 🛡️ **Permissions** | Strict/Mild/Disabled three-tier modes, path rules, command blacklist | Zero-trust security model |
| 📚 **Skills** | On-demand loading of `.md` knowledge bases into system prompt | Teaches Agents professional knowledge |
| 🔌 **Plugins** | Compatible with Anthropic Skills + Claude Code Plugins | Reuses ecosystem |
| 🕊️ **Swarm** | Multi-agent coordination, sub-agent spawn, result aggregation | Complex task decomposition & execution |
| ⚙️ **Hooks** | Pre/Post Tool Use and other lifecycle interceptors | Auditing, logging, custom logic |
| 🕐 **Cron** | Built-in scheduler with cron expression support | Scheduled reports, patrols |
| 💬 **Commands** | 54 slash commands (/help, /commit, /plan...) | User interaction experience |
| 🌍 **MCP** | Model Context Protocol client support | Integrates with external tool ecosystem |

---

## 1.5 Model Compatibility

| Protocol Format | Supported Providers | Price Reference |
|----------------|---------------------|-----------------|
| **Anthropic Native** | Claude series, Moonshot (Kimi) | Kimi ~¥0.01/per 1K tokens |
| **OpenAI Compatible** | Qwen (DashScope), DeepSeek, OpenAI, SiliconFlow, Groq | DeepSeek has free tier |
| **Local Models** | Ollama (supports 1000+ open-source models) | Zero API cost, hardware only |
| **GitHub Copilot** | OAuth login, no API key needed | Enterprise already has Copilot licenses |

**Top Three for Chinese Developers:**

1. **Qwen** — Provided by DashScope, excellent cost-performance, top-tier Chinese language capability
2. **Moonshot (Kimi)** — Strong at Chinese long-text processing, ideal for knowledge base scenarios
3. **DeepSeek** — Free tier available, suitable for low-cost experimentation

---

## 1.6 Comparison with Similar Frameworks

| Dimension | OpenHarness | OpenClaw | LangChain | AutoGPT |
|-----------|-------------|----------|-----------|---------|
| Core Code Size | 26K lines | 8K lines | 100K+ lines | 15K+ lines |
| LLM Providers | 8+ | 10+ | 20+ | 1 |
| Channel Integrations | 7 types | 8 types | ❌ none | ❌ none |
| China Market Channels | ✅ Feishu/DingTalk/WeCom | ✅ Feishu | ❌ | ❌ |
| Multi-Agent | ✅ Swarm | ✅ Sub-Agent | ✅ LangGraph | ⚠️ Limited |
| Permission System | ✅ Three-tier | ✅ exec/allowlist | ❌ | ❌ |
| Memory Persistence | ✅ CLAUDE.md + MEMORY.md | ✅ AGENTS.md + MEMORY.md | ⚠️ manual | ⚠️ limited |
| Commercial Ready | 🟡 Medium | 🟡 Medium | 🟢 High | 🔴 Low |
| License | MIT | MIT | MIT | AGPL-3.0 |

---

## 1.7 Significance for OpenClaw Team

Studying OpenHarness source code brings four direct values to OpenClaw:

| Value | Description |
|-------|-------------|
| 📐 **Technical Reference** | Highly similar architecture design ("channel → engine → tools" three-layer); mutual learning opportunities |
| 🎣 **Open-source Benchmark** | Recently open-sourced competitor from HKU; commercialization path unclear — **our window of opportunity** |
| 📱 **Feishu Channel Reference** | `channels/impl/feishu.py` 945-line implementation, serves as a direct reference for Feishu integration approach |
| 📖 **Learning Resource** | Full 26K-line engineering implementation for team capability improvement and technical interviews |

---

> **Next Chapter Preview:** **[Chapter 2: Architecture Panorama](02-architecture.md)** — Deep dive into OpenHarness's overall architecture design, including data flow, inter-module communication, and how it delivers Agents to virtually every chat platform.

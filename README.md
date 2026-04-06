<div align="center">

# 📘 OpenHarness 深度设计解析

**基于 HKUDS Open Agent Harness v0.1.0 源码的架构设计与实现剖析**

🇺🇸 [English](README_EN.md) · [源码: OpenHarness (HKUDS)](https://github.com/HKUDS/OpenHarness) · [Issues](https://github.com/Shoukuan/OpenHarness-Design-Book/issues)

[![GitHub stars](https://img.shields.io/github/stars/Shoukuan/OpenHarness-Design-Book?style=for-the-badge&logo=github&color=yellow&labelColor=1a1a2e)](https://github.com/Shoukuan/OpenHarness-Design-Book/stargazers)
[![GitHub forks](https://img.shields.io/github/forks/Shoukuan/OpenHarness-Design-Book?style=for-the-badge&logo=github&color=blue&labelColor=1a1a2e)](https://github.com/Shoukuan/OpenHarness-Design-Book/network/members)
[![last commit](https://img.shields.io/github/last-commit/Shoukuan/OpenHarness-Design-Book?style=for-the-badge&logo=git&color=green&labelColor=1a1a2e)](https://github.com/Shoukuan/OpenHarness-Design-Book/commits/main)
[![license](https://img.shields.io/github/license/Shoukuan/OpenHarness-Design-Book?style=for-the-badge&logo=opensourceinitiative&color=red&labelColor=1a1a2e)](./LICENSE)
[![language](https://img.shields.io/github/languages/top/Shoukuan/OpenHarness-Design-Book?style=for-the-badge&logo=markdown&color=orange&labelColor=1a1a2e)](.)

📖 **16 章完整深度解析** · 🎨 **6 张 Mermaid 架构图** · 🌐 **中英双语** · 📊 **300 KB+ 技术文档**

</div>

---

<!-- toc -->
- [⚠️ 声明](#️-声明)
- [✨ 特性](#-特性)
- [📁 仓库结构](#-仓库结构)
- [📑 章节目录](#-章节目录)
- [📊 统计数据](#-统计数据)
- [🔧 Mermaid 图索引](#-mermaid-图索引)
- [🚀 快速开始](#-快速开始)
- [🎯 典型使用场景](#-典型使用场景)
- [🤝 贡献指南](#-贡献指南)
- [📄 License](#-license)
- [🙏 致谢](#-致谢)
<!-- tocstop -->

---

## ⚠️ 声明

> 本设计解析书**基于并衍生自**开源项目 [HKUDS/OpenHarness](https://github.com/HKUDS/OpenHarness)（v0.1.0，MIT 许可证）。
> 所有分析、代码引用和架构描述均来源于 OpenHarness 原始代码库。本仓库**仅包含设计分析与文档**，不包含任何 OpenHarness 原始源代码。

---

## ✨ 特性

| 特性 | 描述 |
|------|------|
| 📚 **全面覆盖** | 16 章覆盖 OpenHarness 完整架构（194 个 Python 文件，26,666 行源码） |
| 🎨 **可视化图表** | 6 张 Mermaid 架构图（流程图、序列图、拓扑图、矩阵图）内嵌 Markdown |
| 🌐 **中英双语** | 中文版（zh/）+ 英文版（en/）对照翻译，逐章精译 |
| 📦 **开箱即用** | GitHub-ready，结构清晰、徽标完整、文档齐全 |
| 🔧 **企业导向** | 深入机制：权限安全沙箱、多 Agent 协调、渠道层、部署实践 |

---

## 📁 仓库结构

```
OpenHarness-Design-Book/
├── README.md          ← 中文 README（本文件）
├── README_EN.md       ← 英文 README
├── BOOK.md            ← 目录索引（中）
├── zh/                ← 中文版（16 章，约 148 KB）
│   ├── 01-overview.md
│   ├── 02-architecture.md   (含 Mermaid 架构图)
│   ├── 03-engine-loop.md    (含 Mermaid 序列图)
│   ├── 04-tools-system.md
│   ├── 05-permissions.md    (含 Mermaid 决策流)
│   ├── 06-memory-system.md
│   ├── 07-skills-plugins.md
│   ├── 08-hooks-system.md
│   ├── 09-coordinator.md
│   ├── 10-swarm-system.md   (含 Mermaid 集群拓扑)
│   ├── 11-channels.md       (含 Mermaid 渠道矩阵)
│   ├── 12-mcp-integration.md
│   ├── 13-api-layer.md
│   ├── 14-commands-system.md
│   ├── 15-full-comparison.md(含 Mermaid 对比图)
│   └── 16-deployment-guide.md
├── en/                ← 英文版（16 章，约 152 KB）
│   └── ...（与 zh/ 结构一致）
├── assets/            ← 资源文件
└── .gitignore
```

---

## 📑 章节目录

| # | 标题 | 图表 | 内容概览 |
|---|------|------|----------|
| 01 | OpenHarness 是什么 | — | 框架定位、设计目标、核心概念 |
| 02 | 架构全景图与核心设计哲学 | 🎨 | 8 层架构、设计原则与设计哲学 |
| 03 | Engine 核心循环源码解析 | 📊 | 查询→LLM→工具→循环全链路 |
| 04 | Tools 系统（43 个工具） | — | Tool 注册、执行、权限校验 |
| 05 | 权限系统（企业级安全沙箱） | 📊 | 零信任决策树、权限模型 |
| 06 | Memory 持久化与上下文管理 | — | 记忆存储、上下文裁剪与恢复 |
| 07 | Skills & Plugins 可扩展机制 | — | 技能加载、插件化架构 |
| 08 | Hooks 生命周期系统 | — | Hook 触发点、生命周期管理 |
| 09 | Coordinator 多 Agent 协调 | — | 协作流程、任务分发策略 |
| 10 | Swarm 集群协调（4,899 行） | 🎨 | Leader-Worker 拓扑、状态同步 |
| 11 | Channels 渠道层（13 平台） | 📊 | 统一消息总线、平台适配 |
| 12 | MCP 协议集成（340 行） | — | 协议适配、通信格式 |
| 13 | API 层与 OpenAI 兼容 | — | 接口映射、认证转换 |
| 14 | Commands 系统（54 个命令） | — | 注册、执行、权限控制 |
| 15 | 与 OpenClaw 全面对比 | 📊 | 代码规模、功能覆盖差异 |
| 16 | 实战与部署指南 | — | 生产环境部署、运维要点 |

---

## 📊 统计数据

| 指标 | 数值 |
|------|------|
| 章节数 | **16** |
| 总行数 | ~7,300+ |
| 总文件大小 | ~300 KB |
| Mermaid 图表 | **6 张** |
| 分析源码 | 194 个 Python 文件，26,666 行 |
| OpenHarness 版本 | v0.1.0（MIT License） |
| 作者 | [Shoukuan](https://github.com/Shoukuan) |
| 创建日期 | 2026-04-06 |

---

## 🔧 Mermaid 图索引

| 章节 | 图表类型 | 描述 |
|------|----------|------|
| [第 2 章](zh/02-architecture.md) | 架构流程图 | 8 层自上而下架构全景 |
| [第 3 章](zh/03-engine-loop.md) | 引擎序列图 | 查询→LLM→工具→循环 |
| [第 5 章](zh/05-permissions.md) | 权限决策流程图 | 零信任决策树 |
| [第 10 章](zh/10-swarm-system.md) | Swarm 集群拓扑 | Leader-Worker 多 Agent 协调 |
| [第 11 章](zh/11-channels.md) | 渠道矩阵图 | 13 平台消息总线架构 |
| [第 15 章](zh/15-full-comparison.md) | 代码规模对比柱状图 | OpenHarness vs OpenClaw |

---

## 🚀 快速开始

### 在线阅读

直接在 GitHub 浏览，点击 `zh/` 目录阅读中文，`en/` 目录阅读英文。GitHub 自动渲染 Mermaid 图表。

### 克隆到本地

```bash
git clone https://github.com/Shoukuan/OpenHarness-Design-Book.git
cd OpenHarness-Design-Book
```

使用支持 Mermaid 的 Markdown 阅读器打开章节：VS Code（Mermaid 插件）、Obsidian、Typora 等。

### 生成 PDF（可选）

```bash
pandoc zh/*.md -o OpenHarness-Design-Book-zh.pdf
pandoc en/*.md -o OpenHarness-Design-Book-en.pdf
```

---

## 🎯 典型使用场景

| 用户角色 | 推荐阅读章节 | 预期收益 |
|----------|-------------|----------|
| 企业架构师 | [第 16 章](zh/16-deployment-guide.md) | 获取私有化部署最佳实践 |
| AI Agent 开发者 | [第 4 章](zh/04-tools-system.md) / [第 7 章](zh/07-skills-plugins.md) | 快速扩展自有框架 |
| 技术决策者 | [第 15 章](zh/15-full-comparison.md) | 框架选型对比分析 |
| 学术研究者 | [第 2 章](zh/02-architecture.md) / [第 3 章](zh/03-engine-loop.md) | 直观理解系统设计 |
| 开源布道师 | 本项目整体结构 | 参考文档组织与写作模板 |

---

## 🤝 贡献指南

我们欢迎社区贡献：

- **🐛 报告问题**：在 [Issues](https://github.com/Shoukuan/OpenHarness-Design-Book/issues) 提交 Bug 或建议
- **📝 提交 PR**：请先开 Issue 讨论，确保改动方向与社区共识一致
- **✏️ 内容勘误**：欢迎修正错别字、翻译错误、图表缺失或逻辑不一致
- **🔄 版本同步**：如果 OpenHarness 发布新版本，欢迎同步更新对应章节

---

## 📄 License

- **OpenHarness 源代码** → [MIT License](https://github.com/HKUDS/OpenHarness)（© HKUDS）
- **本设计解析书** → 由 [Shoukuan](https://github.com/Shoukuan) 创作，仅供学习与参考，非 OpenHarness 官方产品。

---

## 🙏 致谢

- [HKUDS](https://github.com/HKUDS) — OpenHarness 原作者团队
- [OpenViking](https://github.com/volcengine/OpenViking) — README 排版风格启示

---

<div align="center">

⭐ 如果这个项目对你有帮助，欢迎 Star！

**作者**: [Shoukuan](https://github.com/Shoukuan) · **版本**: 1.0 · **创建**: 2026-04-06

</div>

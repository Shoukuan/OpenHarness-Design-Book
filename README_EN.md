<div align="center">

# 📘 OpenHarness Deep Design Analysis

**Architecture Design & Implementation Analysis based on HKUDS Open Agent Harness v0.1.0**

🇨🇳 [中文](README.md) · [Source: OpenHarness (HKUDS)](https://github.com/HKUDS/OpenHarness) · [Issues](https://github.com/Shoukuan/OpenHarness-Design-Book/issues)

[![GitHub stars](https://img.shields.io/github/stars/Shoukuan/OpenHarness-Design-Book?style=for-the-badge&logo=github&color=yellow&labelColor=1a1a2e)](https://github.com/Shoukuan/OpenHarness-Design-Book/stargazers)
[![GitHub forks](https://img.shields.io/github/forks/Shoukuan/OpenHarness-Design-Book?style=for-the-badge&logo=github&color=blue&labelColor=1a1a2e)](https://github.com/Shoukuan/OpenHarness-Design-Book/network/members)
[![last commit](https://img.shields.io/github/last-commit/Shoukuan/OpenHarness-Design-Book?style=for-the-badge&logo=git&color=green&labelColor=1a1a2e)](https://github.com/Shoukuan/OpenHarness-Design-Book/commits/main)
[![license](https://img.shields.io/github/license/Shoukuan/OpenHarness-Design-Book?style=for-the-badge&logo=opensourceinitiative&color=red&labelColor=1a1a2e)](./LICENSE)
[![language](https://img.shields.io/github/languages/top/Shoukuan/OpenHarness-Design-Book?style=for-the-badge&logo=markdown&color=orange&labelColor=1a1a2e)](.)

📖 **16 Chapters** · 🎨 **6 Mermaid Diagrams** · 🌐 **Bilingual (zh/en)** · 📊 **300 KB+ Documentation**

</div>

---

<!-- toc -->
- [⚠️ Disclaimer](#️-disclaimer)
- [✨ Features](#-features)
- [📁 Repository Structure](#-repository-structure)
- [📑 Chapters](#-chapters)
- [📊 Statistics](#-statistics)
- [🔧 Mermaid Diagrams](#-mermaid-diagrams)
- [🚀 Quick Start](#-quick-start)
- [🎯 Typical Use Cases](#-typical-use-cases)
- [🤝 Contributing](#-contributing)
- [📄 License](#-license)
- [🙏 Acknowledgments](#-acknowledgments)
<!-- tocstop -->

---

## ⚠️ Disclaimer

> This design analysis book is **based on and derived from** the open-source project [HKUDS/OpenHarness](https://github.com/HKUDS/OpenHarness) (v0.1.0, MIT License).
> All analysis, code references, and architecture descriptions are sourced from the original OpenHarness codebase.
> This repository contains **only design analysis and documentation** — no original OpenHarness source code is included.

---

## ✨ Features

| Feature | Description |
|---------|-------------|
| 📚 **Comprehensive** | 16 chapters covering the complete OpenHarness architecture (194 Python files, 26,666 LOC) |
| 🎨 **Visual Diagrams** | 6 Mermaid diagrams (flowcharts, sequences, topology, matrix) embedded in Markdown |
| 🌐 **Bilingual** | Chinese (zh/) and English (en/) versions, meticulously translated |
| 📦 **Ready to Use** | GitHub-ready with proper structure, badges, and documentation |
| 🔧 **Enterprise Focus** | Deep-dive into permissions, security sandbox, multi-agent coordination, channels, deployment |

---

## 📁 Repository Structure

```
OpenHarness-Design-Book/
├── README.md          ← Chinese README (中文版)
├── README_EN.md       ← English README (this file)
├── BOOK.md            ← Index (Chinese)
├── zh/                ← Chinese version (16 chapters, ~148 KB)
│   ├── 01-overview.md
│   ├── 02-architecture.md   (with Mermaid)
│   ├── 03-engine-loop.md    (with Mermaid)
│   ├── 04-tools-system.md
│   ├── 05-permissions.md    (with Mermaid)
│   ├── 06-memory-system.md
│   ├── 07-skills-plugins.md
│   ├── 08-hooks-system.md
│   ├── 09-coordinator.md
│   ├── 10-swarm-system.md   (with Mermaid)
│   ├── 11-channels.md       (with Mermaid)
│   ├── 12-mcp-integration.md
│   ├── 13-api-layer.md
│   ├── 14-commands-system.md
│   ├── 15-full-comparison.md(with Mermaid)
│   └── 16-deployment-guide.md
├── en/                ← English version (16 chapters, ~152 KB)
│   └── ... (same structure as zh/)
├── assets/            ← Asset files
└── .gitignore
```

---

## 📑 Chapters

| # | Title | Diagram | Overview |
|---|-------|---------|----------|
| 01 | What Is OpenHarness | — | Framework positioning, design goals, core concepts |
| 02 | Architecture Overview & Core Design Philosophy | 🎨 | 8-layer architecture, design principles |
| 03 | Engine Core Loop Source Analysis | 📊 | Query → LLM → Tool → cycle |
| 04 | Tools System (43 Tools) | — | Tool registration, execution, permission check |
| 05 | Permissions System (Enterprise Security Sandbox) | 📊 | Zero-trust decision tree, permission model |
| 06 | Memory Persistence & Context Management | — | Memory storage, context truncation & recovery |
| 07 | Skills & Plugins Extensibility | — | Skill loading, plugin architecture |
| 08 | Hooks Lifecycle System | — | Hook triggers, lifecycle management |
| 09 | Multi-Agent Coordinator | — | Collaboration flow, task distribution |
| 10 | Swarm Cluster Coordination (4,899 lines) | 🎨 | Leader-Worker topology, state sync |
| 11 | Channels Layer (13 platforms) | 📊 | Unified message bus, platform adapters |
| 12 | MCP Protocol Integration (340 lines) | — | Protocol adaptation, message format |
| 13 | API Layer & OpenAI Compatibility | — | Interface mapping, auth transformation |
| 14 | Commands System (54 commands) | — | Registration, execution, permissions |
| 15 | Full Comparison with OpenClaw | 📊 | Code size, feature coverage differences |
| 16 | Deployment Guide | — | Production deployment, ops checklist |

---

## 📊 Statistics

| Metric | Value |
|--------|-------|
| Chapters | **16** |
| Total Lines | ~7,300+ |
| Total Size | ~300 KB |
| Mermaid Diagrams | **6** |
| Source Files Analyzed | 194 Python files, 26,666 lines |
| OpenHarness Version | v0.1.0 (MIT License) |
| Author | [Shoukuan](https://github.com/Shoukuan) |
| Created | 2026-04-06 |

---

## 🔧 Mermaid Diagrams

| Chapter | Type | Description |
|---------|------|-------------|
| [Ch 2](en/02-architecture.md) | Architecture Flowchart | 8-layer top-down visual structure |
| [Ch 3](en/03-engine-loop.md) | Engine Loop Sequence | Full query → LLM → tool → cycle |
| [Ch 5](en/05-permissions.md) | Permission Decision Flow | Zero-trust decision tree |
| [Ch 10](en/10-swarm-system.md) | Swarm Cluster Topology | Leader-Workers multi-agent flow |
| [Ch 11](en/11-channels.md) | Channels Matrix | 13-platform message bus architecture |
| [Ch 15](en/15-full-comparison.md) | Code Size Comparison | OpenHarness vs OpenClaw bar chart |

---

## 🚀 Quick Start

### Read Online

Browse `zh/` for Chinese or `en/` for English directly on GitHub. Mermaid diagrams render automatically.

### Clone Locally

```bash
git clone https://github.com/Shoukuan/OpenHarness-Design-Book.git
cd OpenHarness-Design-Book
```

Open chapters in a Markdown viewer with Mermaid support: VS Code (Mermaid plugin), Obsidian, Typora, etc.

### Generate PDFs (Optional)

```bash
pandoc zh/*.md -o OpenHarness-Design-Book-zh.pdf
pandoc en/*.md -o OpenHarness-Design-Book-en.pdf
```

---

## 🎯 Typical Use Cases

| User | Recommended Chapters | Benefit |
|------|---------------------|---------|
| Enterprise Architects | [Ch 16](en/16-deployment-guide.md) | Best practices for private deployment |
| AI Agent Developers | [Ch 4](en/04-tools-system.md) / [Ch 7](en/07-skills-plugins.md) | Extend your own frameworks quickly |
| Technical Decision-Makers | [Ch 15](en/15-full-comparison.md) | Framework comparison for selection |
| Academic Researchers | [Ch 2](en/02-architecture.md) / [Ch 3](en/03-engine-loop.md) | Visual understanding of system design |
| Open-Source Advocates | Overall project structure | Reference documentation template |

---

## 🤝 Contributing

We welcome contributions from the community:

- **🐛 Report Issues**: For bug reports or suggestions, open an issue on [GitHub Issues](https://github.com/Shoukuan/OpenHarness-Design-Book/issues).
- **📝 Submit PRs**: Please discuss in an issue first to ensure alignment with community consensus.
- **✏️ Content Fixes**: Typos, translation errors, missing diagrams, or logical inconsistencies are all welcome.
- **🔄 Version Sync**: If OpenHarness releases a new major version, help keep chapters up-to-date.

---

## 📄 License

- **OpenHarness source code** → [MIT License](https://github.com/HKUDS/OpenHarness) (© HKUDS)
- **This design analysis book** → Created by [Shoukuan](https://github.com/Shoukuan) for educational and reference purposes. Not an official OpenHarness product.

---

## 🙏 Acknowledgments

- [HKUDS](https://github.com/HKUDS) — Original authors of OpenHarness
- [OpenViking](https://github.com/volcengine/OpenViking) — README layout inspiration

---

<div align="center">

⭐ **Star this repository** if you find this analysis helpful!

**Author**: [Shoukuan](https://github.com/Shoukuan) · **Version**: 1.0 · **Created**: 2026-04-06

</div>

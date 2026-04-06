# OpenHarness Deep Design Analysis

> Architecture Design & Implementation Analysis based on HKUDS Open Agent Harness v0.1.0 source code

---

⚠️ **Disclaimer**: This book is **based on and derived from** the open-source project [OpenHarness](https://github.com/HKUDS/OpenHarness) by HKUDS. All analysis, code references, and architecture descriptions are sourced from the original OpenHarness codebase (v0.1.0, MIT License). This repository contains **only design analysis and documentation** — no original OpenHarness source code is included.

---

## 📖 About This Book

This is a comprehensive deep-dive into the OpenHarness architecture, covering:

- **16 chapters** of detailed analysis across 148 KB (Chinese) / 152 KB (English)
- **6 Mermaid diagrams** (architecture, sequence flows, permission logic, swarm topology, channel matrix, code comparison)
- **Complete source code analysis** — 194 Python files, 26,666 lines examined
- **Enterprise deployment guide** with practical examples

## 📁 Structure

| Directory | Description | Language |
|-----------|-------------|----------|
| `zh/` | High-quality analysis, enterprise-grade documentation | 🇨🇳 Chinese |
| `en/` | Complete English translation | 🇬🇧 English |

Each directory contains:
- `BOOK.md` — Index page with table of contents, statistics, and notes
- `chapters/01-16.md` — 16 chapters of detailed analysis

## 📑 Chapters

| # | Chinese Title | English Title |
|---|---------------|---------------|
| 1 | OpenHarness 是什么 | What Is OpenHarness |
| 2 | 架构全景图与核心设计哲学 | Architecture Overview & Core Design Philosophy |
| 3 | Engine 核心循环源码解析 | Engine Core Loop Source Analysis |
| 4 | Tools 系统（43 个工具的实现与分派）| Tools System (43 Tools Implementation & Dispatch) |
| 5 | Permissions 权限系统（企业级安全沙箱）| Permissions System (Enterprise Security Sandbox) |
| 6 | Memory 持久化与上下文管理 | Memory Persistence & Context Management |
| 7 | Skills & Plugins 可扩展机制 | Skills & Plugins Extensibility |
| 8 | Hooks 生命周期系统 | Hooks Lifecycle System |
| 9 | Coordinator 多 Agent 协调 | Multi-Agent Coordinator |
| 10 | Swarm 集群协调（4,899 行深度解析）| Swarm Cluster Coordination |
| 11 | Channels 渠道层（13 平台）| Channels Layer (13 Platforms) |
| 12 | MCP 协议集成 | MCP Protocol Integration |
| 13 | API 层与 OpenAI 兼容 | API Layer & OpenAI Compatibility |
| 14 | Commands 系统（54 个命令）| Commands System (54 Commands) |
| 15 | 与 OpenClaw 全面对比 | Full Comparison with OpenClaw |
| 16 | 实战与部署指南 | Deployment Guide |

## 📊 Statistics

- **Chapters**: 16
- **Total Lines**: ~3,600+ (Chinese) / ~3,650+ (English)
- **Total Size**: ~148 KB (Chinese) / ~152 KB (English)
- **Mermaid Diagrams**: 6
- **Source Files Analyzed**: 194 Python files, 26,666 lines
- **OpenHarness Version**: v0.1.0 (MIT License)

## 🔧 Mermaid Diagrams

The following chapters include Mermaid diagrams for visual understanding:

- **Ch 2** — Architecture Overview (8-layer top-down flowchart)
- **Ch 3** — Engine Core Loop (sequence diagram)
- **Ch 5** — Permission Decision Flow (flowchart)
- **Ch 10** — Swarm Cluster Topology (multi-agent flow)
- **Ch 11** — Channel Matrix (13-platform architecture)
- **Ch 15** — Code Size Comparison (bar chart: OpenHarness vs OpenClaw)

## 📄 License

- **OpenHarness source code** → MIT License (© HKUDS)
- **This design analysis book** → Created by Shoukuan, for educational and reference purposes

---

**Author**: Shoukuan  
**Date**: 2026-04-06  
**Version**: 1.0  

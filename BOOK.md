# OpenHarness 深度设计解析

> 基于 HKUDS Open Agent Harness v0.1.0 源码的架构设计与实现剖析

**作者**: Shoukuan
**日期**: 2026-04-06
**版本**: 1.0-draft

---

## 目录

- [第1章:OpenHarness 是什么](chapters/01-overview.md)
- [第2章:架构全景图与核心设计哲学](chapters/02-architecture.md)
- [第3章:Engine 核心循环源码解析](chapters/03-engine-loop.md)
- [第4章:Tools 系统(43 个工具的实现与分派)](chapters/04-tools-system.md)
- [第5章:Permissions 权限系统(企业级安全沙箱)](chapters/05-permissions.md)
- [第6章:Memory 持久化与上下文管理](chapters/06-memory-system.md)
- [第7章:Skills & Plugins 可扩展机制](chapters/07-skills-plugins.md)
- [第8章:Hooks 生命周期系统](chapters/08-hooks-system.md)
- [第9章:Coordinator 多 Agent 协调](chapters/09-coordinator.md)
- [第10章:Swarm 集群协调(4,899 行深度解析)](chapters/10-swarm-system.md)
- [第11章:Channels 渠道层(13 平台,5,183 行)](chapters/11-channels.md)
- [第12章:MCP 协议集成(340 行)](chapters/12-mcp-integration.md)
- [第13章:API 层与 OpenAI 兼容(342 行)](chapters/13-api-layer.md)
- [第14章:Commands 系统(54 个命令,1,456 行)](chapters/14-commands-system.md)
- [第15章:与 OpenClaw 全面对比](chapters/15-full-comparison.md)
- [第16章:实战与部署指南](chapters/16-deployment-guide.md)

---

## 统计数据

- 章节数:16 章
- 总行数:约 2,600+ 行
- 总字数:约 9,800 词
- 参考源码:194 文件,26,666 行(OpenHarness v0.1.1)

---

## 说明

本书为 **草稿版**,基于对 OpenHarness 源码的快速扫描和关键模块的深入阅读。
内容覆盖架构设计、核心流程、代码实现、与 OpenClaw 对比,以及企业部署实践。

可扩展方向:
1. 每个工具的详细参数说明
2. MCP 服务器实战案例(不止 filesystem)
3. 飞书渠道 OAuth 流程详解
4. Swarm 故障恢复与 Leader 选举
5. 性能基准测试数据

---

*文档位置:`~/Desktop/OpenHarness-Design-Book/`*
*生成时间:2026-04-06 09:30 CST*
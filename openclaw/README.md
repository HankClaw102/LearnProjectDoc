# OpenClaw 深度学习系列 - 总目录

> 本系列文章深入剖析 OpenClaw 项目，帮助开发者全面理解其架构设计与实现细节。

## 项目简介

**OpenClaw** 是一个多通道 AI 网关系统，支持 WhatsApp、Telegram、Slack、Discord、Signal、iMessage 等 20+ 消息平台，提供本地优先的 AI 助手体验。

- **版本**: 2026.4.9-beta.1
- **技术栈**: TypeScript + Node.js 24
- **代码规模**: 6200+ 核心源文件 + 3700+ 扩展文件
- **License**: MIT

---

## 系列文章目录

### [第一章：项目概览与环境搭建](./01-overview.md)

- 项目定位与核心价值
- 技术栈分析
- 环境搭建指南
- 快速启动流程

### [第二章：架构设计深度解析](./02-architecture.md)

- 整体架构设计
- 核心模块划分
- 数据流向分析
- 关键设计模式

### [第三章：核心功能详解](./03-core-features.md)

- Gateway 网关服务
- Agent 代理系统
- Channel 通道管理
- Session 会话机制

### [第四章：关键模块深入](./04-key-modules.md)

- Plugin SDK 插件开发
- 配置系统设计
- 安全机制实现
- 存储与持久化

### [第五章：扩展系统与技能框架](./05-extensions-skills.md)

- Extension 扩展机制
- Skills 技能系统
- Channel 插件开发
- Provider 适配器

### [第六章：实战开发指南](./06-practical-guide.md)

- 开发自定义技能
- 扩展新消息通道
- 调试与测试
- 部署与运维

---

## 项目亮点

| 特性 | 说明 |
|------|------|
| 多通道支持 | WhatsApp、Telegram、Discord、Slack 等 20+ 平台 |
| 本地优先 | 数据存储在本地，隐私安全 |
| 扩展性强 | Plugin SDK 支持自定义扩展 |
| AI 模型丰富 | 支持 OpenAI、Anthropic、Google 等主流模型 |
| 跨平台 | macOS、Linux、Windows、iOS、Android |

---

## 相关链接

- 官网: https://openclaw.ai
- 文档: https://docs.openclaw.ai
- GitHub: https://github.com/openclaw/openclaw
- Discord: https://discord.gg/clawd

---

*最后更新: 2026-04-09*
# OpenClaw 深度学习 - 第一章：项目概览与环境搭建

## 1.1 项目定位与核心价值

### 什么是 OpenClaw？

OpenClaw 是一个**多通道 AI 网关系统**，核心理念是让 AI 助手在你已有的消息平台上工作。

**核心价值**:
- 🏠 **本地优先**: 数据存储在本地设备，隐私安全
- 🔌 **多通道统一**: 一套系统连接 20+ 消息平台
- 🤖 **AI 无缝集成**: 支持多种 AI 模型，统一接口
- 🧩 **高度可扩展**: 插件系统支持自定义功能

### 支持的消息平台

| 平台类型 | 支持的渠道 |
|---------|-----------|
| 即时通讯 | WhatsApp、Telegram、Signal、iMessage |
| 企业协作 | Slack、Discord、Microsoft Teams、Google Chat |
| 社交平台 | Matrix、Mattermost、Nostr、Twitch |
| 国内平台 | 飞书、LINE、Zalo、微信 |
| 其他 | IRC、Nextcloud Talk、BlueBubbles |

---

## 1.2 技术栈分析

### 核心技术

```
运行时:     Node.js 24 (推荐) / Node.js 22.16+
语言:       TypeScript 6.0
包管理:     pnpm 10.32
构建工具:   tsdown
测试框架:   Vitest 4.1
```

### 关键依赖

| 类别 | 依赖项 |
|------|--------|
| AI SDK | OpenAI、Anthropic、Google GenAI、Mistral |
| 消息平台 | grammy (Telegram)、@slack/bolt、discord-api-types |
| 存储 | @lancedb/lancedb、sqlite-vec |
| 浏览器 | playwright-core |
| 其他 | express、hono、ws、zod |

### 项目结构

```
openclaw/
├── src/                    # 核心源码 (6200+ 文件)
│   ├── agents/            # AI 代理系统
│   ├── gateway/           # 网关服务
│   ├── channels/          # 通道管理
│   ├── plugins/           # 插件系统
│   ├── plugin-sdk/        # 插件开发 SDK
│   ├── config/            # 配置系统
│   └── ...
├── extensions/            # 扩展模块 (3700+ 文件)
│   ├── openai/           # OpenAI 提供者
│   ├── anthropic/        # Anthropic 提供者
│   ├── telegram/         # Telegram 通道
│   ├── discord/          # Discord 通道
│   └── ...
├── skills/               # 技能模块 (58 个)
│   ├── weather/          # 天气查询
│   ├── summarize/        # 内容总结
│   ├── github/           # GitHub 操作
│   └── ...
├── apps/                 # 移动应用
│   ├── android/          # Android 客户端
│   ├── ios/              # iOS 客户端
│   └── macos/            # macOS 客户端
└── docs/                 # 文档
```

---

## 1.3 环境搭建指南

### 系统要求

- **操作系统**: macOS、Linux、Windows (WSL2)
- **Node.js**: 24.x (推荐) 或 22.16+
- **内存**: 至少 4GB RAM

### 安装方式

#### 方式一：NPM 全局安装（推荐）

```bash
# 安装
npm install -g openclaw@latest

# 或使用 pnpm
pnpm add -g openclaw@latest
```

#### 方式二：从源码构建

```bash
# 克隆仓库
git clone https://github.com/openclaw/openclaw.git
cd openclaw

# 安装依赖
pnpm install

# 构建项目
pnpm build

# 安装 UI 依赖
pnpm ui:build
```

### 初始化配置

```bash
# 运行引导程序
openclaw onboard --install-daemon
```

这会：
1. 安装 Gateway 守护进程（launchd/systemd 服务）
2. 引导配置模型提供商
3. 设置工作空间

---

## 1.4 快速启动流程

### 启动 Gateway

```bash
# 前台运行（调试用）
openclaw gateway --port 18789 --verbose

# 或使用守护进程
openclaw gateway start
```

### 发送消息

```bash
# 发送测试消息
openclaw message send --to +1234567890 --message "Hello from OpenClaw"
```

### 与 AI 对话

```bash
# 命令行对话
openclaw agent --message "帮我分析今天的日程"

# 带深度思考
openclaw agent --message "复杂的决策问题" --thinking high
```

### 开发模式

```bash
# 热重载开发
pnpm gateway:watch
```

---

## 1.5 常用命令速查

| 命令 | 说明 |
|------|------|
| `openclaw onboard` | 引导配置 |
| `openclaw gateway` | 启动网关 |
| `openclaw agent` | AI 对话 |
| `openclaw message` | 消息操作 |
| `openclaw doctor` | 诊断检查 |
| `openclaw status` | 查看状态 |
| `openclaw pairing` | 设备配对 |
| `openclaw cron` | 定时任务 |

---

## 小结

本章介绍了 OpenClaw 的定位、技术栈和环境搭建。下一章我们将深入分析其架构设计。

**下一章**: [架构设计深度解析](./02-architecture.md)
# OpenClaw 深度学习 - 第二章：架构设计深度解析

## 2.1 整体架构设计

### 架构概览

OpenClaw 采用**分层模块化架构**，核心分为四层：

```
┌─────────────────────────────────────────────────────────────┐
│                    应用层 (Apps)                              │
│     macOS App │ iOS App │ Android App │ CLI │ Web UI        │
├─────────────────────────────────────────────────────────────┤
│                    服务层 (Services)                          │
│     Gateway │ Agent Runtime │ Session Manager │ Cron         │
├─────────────────────────────────────────────────────────────┤
│                    核心层 (Core)                              │
│     Plugin SDK │ Config │ Security │ Storage │ Memory        │
├─────────────────────────────────────────────────────────────┤
│                    通道层 (Channels)                          │
│     Telegram │ Discord │ Slack │ WhatsApp │ ...              │
└─────────────────────────────────────────────────────────────┘
```

### 核心设计原则

1. **本地优先**: 所有数据存储在本地，不依赖云服务
2. **插件化**: 功能通过插件扩展，核心保持精简
3. **统一接口**: 不同消息平台通过统一的 Channel 接口抽象
4. **事件驱动**: 基于 WebSocket 的实时通信

---

## 2.2 核心模块划分

### 2.2.1 Gateway 网关模块

**位置**: `src/gateway/`

Gateway 是 OpenClaw 的控制中心，负责：

```
Gateway
├── server/           # HTTP/WebSocket 服务器
├── auth.ts           # 认证系统
├── client.ts         # 客户端管理
├── sessions/         # 会话管理
├── hooks/            # 钩子系统
├── cron/             # 定时任务
└── protocol/         # 通信协议
```

**关键组件**:

| 组件 | 职责 |
|------|------|
| `server.impl.ts` | 主服务器实现 (67KB) |
| `client.ts` | 客户端连接管理 (31KB) |
| `auth.ts` | 认证与授权 (19KB) |
| `server-chat.ts` | 聊天消息处理 (32KB) |

### 2.2.2 Agents 代理模块

**位置**: `src/agents/`

Agents 负责 AI 模型的调用和工具执行：

```
Agents
├── agent-command.ts      # 代理命令处理 (34KB)
├── acp-spawn.ts         # ACP 协议实现 (39KB)
├── anthropic-*.ts       # Anthropic 适配
├── models.*.ts          # 模型配置
└── tools/               # 工具定义
```

**核心流程**:
```
用户消息 → Agent 接收 → 模型调用 → 工具执行 → 响应返回
```

### 2.2.3 Channels 通道模块

**位置**: `src/channels/`

Channels 实现消息平台的统一抽象：

```
Channels
├── plugins/            # 通道插件实现
│   ├── telegram/
│   ├── discord/
│   ├── slack/
│   └── ...
├── registry.ts         # 通道注册表
├── session.ts          # 会话绑定
└── targets.ts          # 目标解析
```

### 2.2.4 Plugin SDK

**位置**: `src/plugin-sdk/`

Plugin SDK 是扩展开发的核心，提供：

```
Plugin SDK (200+ 导出)
├── runtime/            # 运行时支持
├── channel-*/          # 通道开发接口
├── provider-*/         # 提供者接口
├── approval-*/         # 审批系统
├── memory-*/           # 记忆系统
└── ...
```

---

## 2.3 数据流向分析

### 消息处理流程

```
1. 消息接收
   用户消息 → Channel Plugin → Gateway
   
2. 消息处理
   Gateway → Auth 验证 → Session 绑定 → Agent 分发

3. AI 处理
   Agent → Model Provider → 工具调用 → 响应生成

4. 消息返回
   响应 → Channel Plugin → 用户
```

### WebSocket 通信协议

```
客户端 ←→ Gateway
  ├── connect        # 连接建立
  ├── authenticate   # 认证
  ├── session/join   # 加入会话
  ├── message/send   # 发送消息
  ├── message/stream # 流式响应
  └── tool/invoke    # 工具调用
```

---

## 2.4 关键设计模式

### 2.4.1 插件系统模式

```typescript
// 插件定义示例
interface Plugin {
  name: string;
  version: string;
  setup(context: PluginContext): Promise<void>;
  teardown(): Promise<void>;
}
```

**特点**:
- 支持热加载
- 沙箱隔离
- 依赖管理

### 2.4.2 事件总线模式

```typescript
// 事件订阅
gateway.on('message:received', async (msg) => {
  // 处理消息
});

// 事件发布
gateway.emit('response:sent', response);
```

### 2.4.3 工厂模式

```typescript
// Channel 工厂
class ChannelFactory {
  static create(type: ChannelType): Channel {
    switch (type) {
      case 'telegram': return new TelegramChannel();
      case 'discord': return new DiscordChannel();
      // ...
    }
  }
}
```

### 2.4.4 策略模式

```typescript
// 认证策略
interface AuthStrategy {
  authenticate(credentials: Credentials): Promise<AuthResult>;
}

class TokenAuthStrategy implements AuthStrategy { ... }
class OAuthStrategy implements AuthStrategy { ... }
```

---

## 2.5 配置系统设计

### 配置层次

```
配置优先级 (高→低):
1. 命令行参数
2. 环境变量
3. 用户配置文件 (~/.openclaw/config.json)
4. 工作空间配置 (workspace/config.json)
5. 默认配置
```

### 配置结构

```json
{
  "gateway": {
    "port": 18789,
    "auth": { ... }
  },
  "channels": {
    "telegram": { ... },
    "discord": { ... }
  },
  "models": {
    "default": "openai/gpt-4",
    "providers": { ... }
  },
  "skills": [ ... ]
}
```

---

## 小结

本章分析了 OpenClaw 的整体架构设计。下一章我们将深入核心功能实现。

**下一章**: [核心功能详解](./03-core-features.md)
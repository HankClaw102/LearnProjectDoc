# OpenClaw 深度学习 - 第三章：核心功能详解

## 3.1 Gateway 网关服务

### 3.1.1 服务架构

Gateway 是 OpenClaw 的核心服务，提供：

```
Gateway Server
├── HTTP Server        # REST API
├── WebSocket Server   # 实时通信
├── Control UI         # Web 控制台
├── OpenAI Compatible  # OpenAI 兼容 API
└── MCP Server         # Model Context Protocol
```

### 3.1.2 认证系统

**认证模式**:

| 模式 | 说明 | 适用场景 |
|------|------|----------|
| `token` | 静态 Token | 开发测试 |
| `pairing` | 设备配对 | 移动端 |
| `oauth` | OAuth 2.0 | 第三方集成 |

**认证流程**:
```
1. 客户端发起连接
2. 服务端返回配对码
3. 用户执行 `openclaw pairing approve`
4. 连接建立成功
```

### 3.1.3 会话管理

**会话类型**:
- **main**: 主会话，直接对话
- **group**: 群组会话，隔离处理
- **subagent**: 子代理会话

**会话持久化**:
```
~/.openclaw/
├── sessions/
│   ├── main/
│   └── subagents/
└── transcripts/    # 会话记录
```

---

## 3.2 Agent 代理系统

### 3.2.1 Agent 架构

Agent 负责 AI 模型的调用编排：

```
Agent
├── Model Selection      # 模型选择
├── Tool Execution       # 工具执行
├── Context Management   # 上下文管理
├── Stream Processing    # 流式处理
└── Error Handling       # 错误处理
```

### 3.2.2 模型提供商

**支持的 AI 模型**:

| 提供商 | 模型示例 |
|--------|----------|
| OpenAI | gpt-4o, gpt-4-turbo |
| Anthropic | claude-sonnet-4, claude-opus |
| Google | gemini-2.0-flash, gemini-2.1-pro |
| Mistral | mistral-large, codestral |
| 本地模型 | ollama, vllm |

**模型配置**:
```json
{
  "models": {
    "default": "openai/gpt-4o",
    "fallback": "anthropic/claude-sonnet-4",
    "profiles": {
      "fast": "openai/gpt-4o-mini",
      "smart": "anthropic/claude-opus-4"
    }
  }
}
```

### 3.2.3 工具系统

**内置工具**:

| 工具 | 功能 |
|------|------|
| `read` | 读取文件 |
| `write` | 写入文件 |
| `edit` | 编辑文件 |
| `exec` | 执行命令 |
| `browser` | 浏览器控制 |
| `web_search` | 网络搜索 |
| `message` | 消息操作 |

**工具权限控制**:
```json
{
  "tools": {
    "exec": {
      "security": "allowlist",
      "allowlist": ["ls", "cat", "git status"]
    }
  }
}
```

---

## 3.3 Channel 通道管理

### 3.3.1 Channel 接口

所有消息平台实现统一的 Channel 接口：

```typescript
interface Channel {
  // 发送消息
  send(message: Message): Promise<SendResult>;
  
  // 接收消息回调
  onMessage(handler: MessageHandler): void;
  
  // 获取用户信息
  getUser(userId: string): Promise<User>;
  
  // 群组操作
  getChatMembers(chatId: string): Promise<Member[]>;
}
```

### 3.3.2 通道配置

**Telegram 示例**:
```json
{
  "channels": {
    "telegram": {
      "botToken": "your-bot-token",
      "dmPolicy": "pairing",
      "allowFrom": ["user_id_1", "user_id_2"]
    }
  }
}
```

**Discord 示例**:
```json
{
  "channels": {
    "discord": {
      "botToken": "your-bot-token",
      "applicationId": "app-id",
      "dmPolicy": "pairing"
    }
  }
}
```

### 3.3.3 消息格式转换

不同平台的消息格式统一转换为内部格式：

```
平台消息 → InboundEnvelope → 统一处理
         ↓
统一响应 → OutboundPayload → 平台格式
```

---

## 3.4 Session 会话机制

### 3.4.1 会话生命周期

```
创建 → 激活 → 运行 → 休眠 → 销毁
```

**会话状态**:
```typescript
interface SessionState {
  status: 'active' | 'idle' | 'paused' | 'terminated';
  messages: Message[];
  context: Context;
  bindings: SessionBinding[];
}
```

### 3.4.2 会话绑定

会话可以绑定到特定的通道/群组/用户：

```typescript
interface SessionBinding {
  type: 'channel' | 'group' | 'dm';
  channelId: string;
  accountId: string;
  peerId?: string;
}
```

### 3.4.3 会话持久化

**存储位置**:
```
~/.openclaw/
├── sessions/
│   ├── main/session.db
│   └── {session-key}/
│       ├── messages.jsonl
│       └── context.json
└── transcripts/
    └── {date}.jsonl
```

---

## 3.5 Cron 定时任务

### 3.5.1 任务配置

```json
{
  "cron": {
    "tasks": [
      {
        "id": "daily-summary",
        "schedule": "0 9 * * *",
        "action": {
          "type": "message",
          "to": "user@telegram",
          "template": "daily-summary"
        }
      }
    ]
  }
}
```

### 3.5.2 任务管理命令

```bash
# 列出任务
openclaw cron list

# 添加任务
openclaw cron add --schedule "0 9 * * *" --message "早安"

# 删除任务
openclaw cron remove <task-id>

# 手动触发
openclaw cron trigger <task-id>
```

---

## 小结

本章详细介绍了 Gateway、Agent、Channel、Session 等核心功能。下一章我们将深入关键模块。

**下一章**: [关键模块深入](./04-key-modules.md)
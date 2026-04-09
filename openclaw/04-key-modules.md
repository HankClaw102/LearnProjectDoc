# OpenClaw 深度学习 - 第四章：关键模块深入

## 4.1 Plugin SDK 插件开发

### 4.1.1 SDK 架构

Plugin SDK 是 OpenClaw 扩展性的核心，提供 200+ 导出接口：

```
Plugin SDK
├── runtime/              # 运行时
│   ├── agent-runtime     # Agent 运行时
│   ├── channel-runtime   # Channel 运行时
│   └── config-runtime    # 配置运行时
├── channel-*/            # Channel 开发接口
├── provider-*/           # Provider 开发接口
├── approval-*/           # 审批系统
├── memory-*/             # 记忆系统
├── speech-*/             # 语音系统
└── browser-*/            # 浏览器控制
```

### 4.1.2 开发插件

**基本结构**:
```typescript
// my-plugin/index.ts
import { definePlugin } from 'openclaw/plugin-sdk';

export default definePlugin({
  name: 'my-plugin',
  version: '1.0.0',
  
  async setup(context) {
    // 注册工具
    context.registerTool({
      name: 'my_tool',
      description: '我的工具',
      parameters: { ... },
      handler: async (params) => {
        return { result: 'success' };
      }
    });
  },
  
  async teardown() {
    // 清理资源
  }
});
```

### 4.1.3 插件配置

```json
{
  "plugins": {
    "my-plugin": {
      "enabled": true,
      "config": {
        "option1": "value1"
      }
    }
  }
}
```

---

## 4.2 配置系统设计

### 4.2.1 配置层级

```
优先级 (高→低):
1. 命令行参数
2. 环境变量 (OPENCLAW_*)
3. 运行时配置 (~/.openclaw/runtime.json)
4. 工作空间配置 (workspace/config.json)
5. 默认配置
```

### 4.2.2 配置热重载

```typescript
// 监听配置变化
gateway.on('config:changed', (changes) => {
  console.log('配置已更新:', changes);
});
```

### 4.2.3 配置验证

使用 JSON Schema 进行配置验证：

```typescript
const configSchema = {
  type: 'object',
  properties: {
    gateway: { ... },
    channels: { ... },
    models: { ... }
  },
  required: ['gateway']
};
```

---

## 4.3 安全机制实现

### 4.3.1 DM 访问控制

**配对机制**:
```
1. 陌生用户发送消息
2. 系统返回配对码
3. 管理员执行: openclaw pairing approve <code>
4. 用户加入白名单
```

**配置示例**:
```json
{
  "channels": {
    "telegram": {
      "dmPolicy": "pairing",
      "allowFrom": ["user_123", "user_456"]
    }
  }
}
```

### 4.3.2 命令执行审批

**审批模式**:

| 模式 | 说明 |
|------|------|
| `deny` | 拒绝所有 |
| `allowlist` | 白名单模式 |
| `full` | 允许所有（危险） |

**配置示例**:
```json
{
  "security": {
    "exec": {
      "mode": "allowlist",
      "allowlist": ["ls", "cat", "git"]
    }
  }
}
```

### 4.3.3 敏感数据保护

**密钥存储**:
```bash
# 存储密钥
openclaw secrets set OPENAI_API_KEY sk-xxx

# 查看密钥
openclaw secrets list

# 审计密钥使用
openclaw secrets audit
```

---

## 4.4 存储与持久化

### 4.4.1 存储架构

```
~/.openclaw/
├── config.json           # 主配置
├── runtime.json          # 运行时配置
├── auth/                 # 认证数据
│   ├── tokens.db
│   └── pairing.db
├── sessions/             # 会话数据
│   └── {session-key}/
├── memory/               # 记忆数据
│   └── lancedb/
├── transcripts/          # 会话记录
└── logs/                 # 日志文件
```

### 4.4.2 LanceDB 记忆系统

OpenClaw 使用 LanceDB 实现向量记忆：

```typescript
// 记忆存储
await memory.store({
  content: '用户喜欢咖啡',
  embedding: await embed('用户喜欢咖啡'),
  metadata: { source: 'chat' }
});

// 记忆检索
const results = await memory.search('用户喜欢什么?', { limit: 5 });
```

### 4.4.3 会话记录

```typescript
// 会话记录格式
interface Transcript {
  sessionId: string;
  timestamp: Date;
  role: 'user' | 'assistant' | 'tool';
  content: string;
  metadata?: Record<string, unknown>;
}
```

---

## 4.5 日志系统

### 4.5.1 日志级别

```
TRACE < DEBUG < INFO < WARN < ERROR < FATAL
```

### 4.5.2 日志配置

```json
{
  "logging": {
    "level": "info",
    "file": "~/.openclaw/logs/openclaw.log",
    "rotation": {
      "maxSize": "10MB",
      "maxFiles": 5
    }
  }
}
```

### 4.5.3 查看日志

```bash
# 查看实时日志
openclaw logs --follow

# 过滤日志
openclaw logs --level error

# 查看特定模块
openclaw logs --module gateway
```

---

## 小结

本章深入分析了 Plugin SDK、配置系统、安全机制和存储系统。下一章我们将介绍扩展系统和技能框架。

**下一章**: [扩展系统与技能框架](./05-extensions-skills.md)
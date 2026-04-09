# OpenClaw 深度学习 - 第五章：扩展系统与技能框架

## 5.1 Extension 扩展机制

### 5.1.1 扩展类型

OpenClaw 支持多种扩展类型：

| 类型 | 位置 | 说明 |
|------|------|------|
| Channel | extensions/*/ | 消息通道扩展 |
| Provider | extensions/*/ | AI 模型提供者 |
| Tool | extensions/*/ | 工具扩展 |
| Integration | extensions/*/ | 第三方集成 |

### 5.1.2 扩展目录结构

```
extensions/
├── telegram/            # Telegram 通道
│   ├── index.ts
│   ├── config.ts
│   └── runtime.ts
├── openai/              # OpenAI 提供者
│   ├── index.ts
│   ├── models.ts
│   └── stream.ts
├── brave/               # Brave 搜索
├── browser/             # 浏览器控制
├── feishu/              # 飞书集成
└── ... (100+ 扩展)
```

### 5.1.3 扩展清单

**已支持的扩展**（部分）：

**消息通道**:
- telegram, discord, slack, whatsapp
- signal, matrix, irc, msteams
- googlechat, feishu, line, zalo

**AI 提供者**:
- openai, anthropic, google
- mistral, groq, deepseek
- ollama, vllm, openrouter

**工具扩展**:
- browser, brave, tavily
- memory-core, memory-lancedb
- diffs, speech-core

---

## 5.2 Skills 技能系统

### 5.2.1 技能定义

技能是预定义的功能模块，通过 SKILL.md 描述：

```
skills/
├── weather/SKILL.md
├── summarize/SKILL.md
├── github/SKILL.md
├── tmux/SKILL.md
└── ... (58 个技能)
```

### 5.2.2 技能结构

```markdown
# SKILL.md

## 描述
技能的功能描述

## 触发条件
- 关键词匹配
- 意图识别
- 手动调用

## 依赖
- 需要的工具
- 需要的配置

## 使用示例
示例命令和预期结果
```

### 5.2.3 内置技能列表

| 技能 | 功能 |
|------|------|
| weather | 天气查询与预报 |
| summarize | URL/文件总结 |
| github | GitHub 操作 |
| tmux | tmux 会话管理 |
| video-frames | 视频帧提取 |
| healthcheck | 系统健康检查 |
| skill-creator | 创建新技能 |
| clawhub | 技能市场 |

---

## 5.3 Channel 插件开发

### 5.3.1 开发模板

```typescript
// extensions/my-channel/index.ts
import { defineChannelPlugin } from 'openclaw/plugin-sdk';

export default defineChannelPlugin({
  name: 'my-channel',
  version: '1.0.0',
  
  // 配置定义
  configSchema: {
    type: 'object',
    properties: {
      apiKey: { type: 'string' },
      dmPolicy: { type: 'string', default: 'pairing' }
    },
    required: ['apiKey']
  },
  
  // 初始化
  async setup(context) {
    const config = context.config;
    
    // 创建客户端
    const client = new MyClient(config.apiKey);
    
    // 注册消息处理器
    context.onMessage(async (envelope) => {
      // 处理入站消息
      const result = await context.agent.send(envelope.message);
      
      // 发送响应
      await client.sendMessage(envelope.from, result);
    });
  }
});
```

### 5.3.2 消息格式

**入站消息**:
```typescript
interface InboundEnvelope {
  messageId: string;
  timestamp: Date;
  sender: {
    id: string;
    name?: string;
    isBot?: boolean;
  };
  chat: {
    id: string;
    type: 'dm' | 'group' | 'channel';
  };
  content: {
    text?: string;
    media?: MediaAttachment[];
    replyTo?: string;
  };
}
```

**出站消息**:
```typescript
interface OutboundPayload {
  text: string;
  media?: MediaPayload[];
  replyTo?: string;
  silent?: boolean;
}
```

---

## 5.4 Provider 适配器

### 5.4.1 Provider 接口

```typescript
interface Provider {
  name: string;
  
  // 模型列表
  listModels(): Promise<Model[]>;
  
  // 聊天补全
  chatCompletion(request: ChatRequest): Promise<ChatResponse>;
  
  // 流式聊天
  streamChat(request: ChatRequest): AsyncIterable<ChatChunk>;
  
  // 嵌入向量
  embeddings(text: string): Promise<number[]>;
}
```

### 5.4.2 开发 Provider

```typescript
// extensions/my-provider/index.ts
import { defineProvider } from 'openclaw/plugin-sdk';

export default defineProvider({
  name: 'my-provider',
  
  async listModels() {
    return [
      { id: 'model-1', name: 'Model 1' },
      { id: 'model-2', name: 'Model 2' }
    ];
  },
  
  async chatCompletion(request) {
    // 实现聊天逻辑
    const response = await fetch('https://api.my-provider.com/chat', {
      method: 'POST',
      body: JSON.stringify(request)
    });
    return response.json();
  }
});
```

---

## 5.5 ClawHub 技能市场

### 5.5.1 查找技能

```bash
# 搜索技能
openclaw skill search <keyword>

# 浏览市场
openclaw skill browse
```

### 5.5.2 安装技能

```bash
# 从 ClawHub 安装
openclaw skill install <skill-name>

# 从 URL 安装
openclaw skill install https://github.com/user/skill
```

### 5.5.3 发布技能

```bash
# 发布到 ClawHub
openclaw skill publish ./my-skill
```

---

## 小结

本章介绍了扩展系统、技能框架、Channel 插件和 Provider 适配器的开发。下一章我们将进入实战开发。

**下一章**: [实战开发指南](./06-practical-guide.md)
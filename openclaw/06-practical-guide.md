# OpenClaw 深度学习 - 第六章：实战开发指南

## 6.1 开发自定义技能

### 6.1.1 创建技能目录

```bash
# 创建技能目录
mkdir -p ~/.openclaw/workspace/skills/my-skill
cd ~/.openclaw/workspace/skills/my-skill
```

### 6.1.2 编写 SKILL.md

```markdown
# my-skill

## 描述
我的自定义技能，用于 [功能描述]

## 触发条件
- 用户提到 "XXX" 关键词
- 手动调用 `/my-skill` 命令

## 依赖工具
- read: 读取配置文件
- exec: 执行系统命令

## 配置
\`\`\`json
{
  "skills": {
    "my-skill": {
      "enabled": true,
      "option1": "value1"
    }
  }
}
\`\`\`

## 使用示例
用户: 帮我执行 my-skill
助手: [执行技能逻辑]
```

### 6.1.3 添加脚本（可选）

```bash
# scripts/main.sh
#!/bin/bash
echo "执行自定义技能逻辑"
```

---

## 6.2 扩展新消息通道

### 6.2.1 创建扩展

```bash
# 在项目 extensions 目录
mkdir -p extensions/my-channel
```

### 6.2.2 实现核心代码

```typescript
// extensions/my-channel/index.ts
import {
  defineChannelPlugin,
  type ChannelRuntimeContext
} from 'openclaw/plugin-sdk';

export default defineChannelPlugin({
  name: 'my-channel',
  version: '1.0.0',
  
  configSchema: {
    type: 'object',
    properties: {
      webhookUrl: { type: 'string' }
    },
    required: ['webhookUrl']
  },
  
  async setup(context: ChannelRuntimeContext) {
    // 1. 初始化客户端
    const client = new MyChannelClient(context.config.webhookUrl);
    
    // 2. 注册消息处理器
    context.onInboundMessage(async (envelope) => {
      // 发送给 Agent 处理
      const response = await context.sendToAgent({
        message: envelope.content.text,
        context: {
          channelId: envelope.chat.id,
          senderId: envelope.sender.id
        }
      });
      
      // 发送响应
      await client.sendMessage(envelope.chat.id, response.text);
    });
    
    // 3. 处理出站消息
    context.onOutboundMessage(async (payload) => {
      await client.sendMessage(payload.target, payload.text);
    });
  }
});
```

### 6.2.3 配置通道

```json
{
  "channels": {
    "my-channel": {
      "webhookUrl": "https://api.example.com/webhook",
      "dmPolicy": "pairing",
      "allowFrom": []
    }
  }
}
```

---

## 6.3 调试与测试

### 6.3.1 调试模式

```bash
# 启用详细日志
openclaw gateway --verbose

# 或设置环境变量
DEBUG=openclaw:* openclaw gateway
```

### 6.3.2 测试扩展

```bash
# 运行特定测试
pnpm test:extensions --filter my-channel

# 运行所有测试
pnpm test
```

### 6.3.3 使用 Doctor 诊断

```bash
# 检查系统状态
openclaw doctor

# 检查特定模块
openclaw doctor --check channels
```

### 6.3.4 日志分析

```bash
# 查看错误日志
openclaw logs --level error

# 实时监控
openclaw logs --follow
```

---

## 6.4 部署与运维

### 6.4.1 守护进程模式

```bash
# 安装为系统服务
openclaw onboard --install-daemon

# 启动服务
openclaw gateway start

# 停止服务
openclaw gateway stop

# 重启服务
openclaw gateway restart

# 查看状态
openclaw gateway status
```

### 6.4.2 Docker 部署

```bash
# 构建镜像
docker build -t openclaw .

# 运行容器
docker run -d \
  --name openclaw \
  -p 18789:18789 \
  -v ~/.openclaw:/root/.openclaw \
  openclaw
```

### 6.4.3 生产环境配置

```json
{
  "gateway": {
    "port": 18789,
    "auth": {
      "mode": "token",
      "token": "${GATEWAY_TOKEN}"
    }
  },
  "logging": {
    "level": "info",
    "file": "/var/log/openclaw/openclaw.log"
  },
  "security": {
    "exec": {
      "mode": "allowlist",
      "allowlist": ["git", "npm", "node"]
    }
  }
}
```

### 6.4.4 监控与告警

```bash
# 健康检查
curl http://localhost:18789/health

# Prometheus 指标
curl http://localhost:18789/metrics
```

---

## 6.5 常见问题解决

### Q1: Gateway 启动失败

```bash
# 检查端口占用
lsof -i :18789

# 检查权限
openclaw doctor --check permissions
```

### Q2: 消息发送失败

```bash
# 检查通道配置
openclaw doctor --check channels

# 检查认证
openclaw pairing list
```

### Q3: AI 模型调用失败

```bash
# 检查模型配置
openclaw doctor --check models

# 测试模型连接
openclaw agent --message "test" --model openai/gpt-4o
```

---

## 6.6 进阶话题

### 多模型配置

```json
{
  "models": {
    "default": "openai/gpt-4o",
    "profiles": {
      "code": "anthropic/claude-sonnet-4",
      "fast": "openai/gpt-4o-mini",
      "local": "ollama/llama3"
    }
  }
}
```

### 自定义工具

```typescript
context.registerTool({
  name: 'my_tool',
  description: '自定义工具',
  parameters: {
    type: 'object',
    properties: {
      input: { type: 'string' }
    }
  },
  handler: async (params) => {
    // 实现工具逻辑
    return { result: 'success' };
  }
});
```

---

## 系列总结

恭喜您完成 OpenClaw 深度学习系列！您已经学习了：

1. ✅ 项目概览与环境搭建
2. ✅ 架构设计深度解析
3. ✅ 核心功能详解
4. ✅ 关键模块深入
5. ✅ 扩展系统与技能框架
6. ✅ 实战开发指南

**下一步建议**:
- 阅读官方文档: https://docs.openclaw.ai
- 加入社区讨论: https://discord.gg/clawd
- 贡献代码: https://github.com/openclaw/openclaw

---

*系列完成于 2026-04-09*

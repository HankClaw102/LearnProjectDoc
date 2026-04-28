# Hermes Agent vs OpenClaw 深度对比分析

> 本文档从功能、设计思想、架构、源码实现等多个维度深入对比 Hermes Agent 和 OpenClaw 两个开源 AI Agent 框架。

## 目录

1. [项目概述](#项目概述)
2. [核心架构对比](#核心架构对比)
3. [技术栈差异](#技术栈差异)
4. [消息网关设计](#消息网关设计)
5. [工具系统对比](#工具系统对比)
6. [技能系统对比](#技能系统对比)
7. [记忆系统对比](#记忆系统对比)
8. [设计哲学差异](#设计哲学差异)
9. [适用场景分析](#适用场景分析)
10. [总结与建议](#总结与建议)

---

## 项目概述

### Hermes Agent

| 属性 | 值 |
|------|-----|
| **开发者** | Nous Research |
| **语言** | Python 3.11+ |
| **定位** | Self-improving AI Agent（自我进化的 AI Agent） |
| **核心特色** | 自主学习循环、技能自动创建、持久化记忆 |
| **许可证** | MIT |
| **GitHub** | https://github.com/nousresearch/hermes-agent |

**核心宣传语**：
> The agent that grows with you — creates skills from experience, improves them during use.

### OpenClaw

| 属性 | 值 |
|------|-----|
| **开发者** | OpenClaw Community |
| **语言** | TypeScript / Node.js 24+ |
| **定位** | Multi-channel AI Gateway（多渠道 AI 网关） |
| **核心特色** | 多平台消息集成、插件化架构、Canvas 交互 |
| **许可证** | MIT |
| **GitHub** | https://github.com/openclaw/openclaw |

**核心宣传语**：
> Personal AI Assistant — answers you on the channels you already use.

---

## 核心架构对比

### 整体架构差异

```
┌─────────────────────────────────────────────────────────────────┐
│                    Hermes Agent 架构                            │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │              run_agent.py (676KB 单体)                   │   │
│  │  ┌─────────────┬─────────────┬─────────────┬──────────┐ │   │
│  │  │ Agent Loop  │ Tool System │ Memory Mgr  │ Gateway  │ │   │
│  │  └─────────────┴─────────────┴─────────────┴──────────┘ │   │
│  └─────────────────────────────────────────────────────────┘   │
│                              │                                  │
│              ┌───────────────┼───────────────┐                  │
│              ▼               ▼               ▼                  │
│         ┌────────┐     ┌──────────┐    ┌──────────┐             │
│         │ agent/ │     │  tools/  │    │ gateway/ │             │
│         └────────┘     └──────────┘    └──────────┘             │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────┐
│                    OpenClaw 架构                                │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────────────────┐ │
│  │   src/cli/  │  │src/gateway/ │  │    src/agents/          │ │
│  │  CLI 入口   │──│  消息网关   │──│ Agent 命令调度          │ │
│  └─────────────┘  └─────────────┘  └─────────────────────────┘ │
│         │               │                      │                │
│         ▼               ▼                      ▼                │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────────────────┐ │
│  │src/channels/│  │src/config/  │  │src/memory-host-sdk/     │ │
│  │ 平台适配器  │  │  配置管理   │  │ 记忆引擎 SDK            │ │
│  └─────────────┘  └─────────────┘  └─────────────────────────┘ │
│                                                                 │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │                    extensions/                           │   │
│  │  插件系统 - 按需加载，独立开发部署                        │   │
│  └─────────────────────────────────────────────────────────┘   │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### 源码规模对比

| 指标 | Hermes Agent | OpenClaw |
|------|--------------|----------|
| **核心 agent 文件** | `run_agent.py` 676KB (~16,000 行) | `src/agents/*.ts` ~150 个文件 |
| **CLI 文件** | `cli.py` 515KB (~12,000 行) | `src/cli/*.ts` ~100 个文件 |
| **总代码行数** | ~145,000 行 Python | ~360,000 行 TypeScript |
| **模块化程度** | 单体为主，部分拆分 | 高度模块化，插件化 |

---

## 技术栈差异

### 语言与运行时

| 方面 | Hermes Agent | OpenClaw |
|------|--------------|----------|
| **核心语言** | Python 3.11+ | TypeScript 5.x |
| **运行时** | CPython / uv 包管理 | Node.js 24+ / pnpm |
| **包管理** | uv / pip | pnpm workspace |
| **构建工具** | setuptools | tsdown / esbuild |

### 依赖生态

**Hermes Agent 核心依赖**：
```python
# AI 模型
openai>=2.21.0
anthropic>=0.39.0

# CLI
prompt_toolkit>=3.0.52
rich>=14.3.3
fire>=0.7.1

# 消息平台
python-telegram-bot>=22.6
discord.py>=2.7.1
slack-bolt>=1.18.0
```

**OpenClaw 核心依赖**：
```json
{
  "@discordjs/*": "Discord.js 生态",
  "grammy": "Telegram 框架",
  "@slack/web-api": "Slack SDK",
  "@anthropic-ai/sdk": "Anthropic API",
  "openai": "OpenAI API"
}
```

### 设计理念差异

| 方面 | Hermes Agent | OpenClaw |
|------|--------------|----------|
| **打包分发** | PyPI / pip install | npm / npx |
| **开发体验** | Python 脚本即工具 | TypeScript 类型安全 + 插件 SDK |
| **性能优化** | 同步 I/O 为主，async/await 辅助 | 全异步架构，事件驱动 |
| **可扩展性** | Python 插件/技能系统 | TypeScript 插件 + extensions 目录 |

---

## 消息网关设计

### 平台支持对比

| 平台 | Hermes Agent | OpenClaw |
|------|:------------:|:--------:|
| Telegram | ✅ | ✅ |
| Discord | ✅ | ✅ |
| Slack | ✅ | ✅ |
| WhatsApp | ✅ | ✅ |
| Signal | ✅ | ✅ |
| Feishu | ✅ | ✅ |
| WeChat | ✅ (企业微信) | ✅ |
| Matrix | ✅ | ❌ |
| iMessage | ❌ | ✅ (BlueBubbles) |
| IRC | ❌ | ✅ |
| Google Chat | ❌ | ✅ |
| LINE | ❌ | ✅ |
| Nostr | ❌ | ✅ |

### 网关架构实现

**Hermes Agent Gateway**:
```python
# gateway/platforms/base.py - 基类设计
class BasePlatformAdapter(ABC):
    @abstractmethod
    async def start(self): ...
    
    @abstractmethod
    async def send_message(self, chat_id, text): ...
    
    @abstractmethod
    async def receive_message(self): ...
```

- **设计模式**: 继承式适配器模式
- **文件组织**: 每个平台一个 Python 文件
- **消息格式**: 统一转换为 OpenAI 消息格式

**OpenClaw Gateway**:
```typescript
// src/channels/ - 平台适配器目录
// src/gateway/ - 网关核心逻辑

// 插件化平台适配器
export interface ChannelAdapter {
  platformId: string;
  start(config: ChannelConfig): Promise<void>;
  send(opts: SendOptions): Promise<void>;
}
```

- **设计模式**: 插件化适配器 + 依赖注入
- **文件组织**: 每个平台独立目录，支持配置驱动
- **消息格式**: 统一 ChatContent 类型

### 关键差异

| 方面 | Hermes Agent | OpenClaw |
|------|--------------|----------|
| **架构模式** | 单进程，各平台适配器继承 BasePlatformAdapter | 插件化，extensions/ 独立加载 |
| **配置管理** | YAML 配置文件 | TypeScript 配置 + JSON Schema |
| **热重载** | 需重启 | 支持部分热更新 |
| **多实例** | 单实例多平台 | 支持分布式部署 |

---

## 工具系统对比

### 工具定义方式

**Hermes Agent**:
```python
# tools/registry.py
@registry.register(
    name="terminal",
    description="Execute shell commands"
)
def terminal(command: str, timeout: int = 30) -> str:
    ...

# toolsets.py - 工具集管理
TOOLSETS = {
    "web": {
        "description": "Web research tools",
        "tools": ["web_search", "web_extract"],
    }
}
```

**OpenClaw**:
```typescript
// src/agents/agent-command.ts
// 工具通过 ACP (Agent Communication Protocol) 动态加载

// 工具定义使用 TypeScript 类型
interface ToolDefinition {
  name: string;
  description: string;
  parameters: JSONSchema;
}
```

### 工具系统架构

```
Hermes Agent 工具系统:
┌─────────────────────────────────────────┐
│           model_tools.py                │
│  ┌────────────────────────────────────┐ │
│  │  get_tool_definitions()            │ │
│  │  handle_function_call()            │ │
│  │  check_toolset_requirements()      │ │
│  └────────────────────────────────────┘ │
└─────────────────────────────────────────┘
                    │
        ┌──────────┼──────────┐
        ▼          ▼          ▼
   ┌─────────┐ ┌─────────┐ ┌─────────┐
   │terminal │ │ web_*   │ │ skills  │
   │_tool.py │ │ .py     │ │ _tool.py│
   └─────────┘ └─────────┘ └─────────┘

OpenClaw 工具系统:
┌─────────────────────────────────────────┐
│        src/agents/agent-command.ts      │
│  ┌────────────────────────────────────┐ │
│  │  resolveAgentRuntimeConfig()       │ │
│  │  runAgentAttempt()                 │ │
│  └────────────────────────────────────┘ │
└─────────────────────────────────────────┘
                    │
        ┌──────────┼──────────┐
        ▼          ▼          ▼
   ┌─────────┐ ┌─────────┐ ┌─────────┐
   │src/acp/ │ │src/mcp/ │ │extensions│
   │ ACP协议 │ │ MCP协议 │ │ 插件工具 │
   └─────────┘ └─────────┘ └─────────┘
```

### 工具隔离与安全

| 方面 | Hermes Agent | OpenClaw |
|------|--------------|----------|
| **审批机制** | 内置 approval.py | ACP policy.ts |
| **沙箱隔离** | Docker/SSH/Modal 环境 | Docker + sandbox 配置 |
| **权限控制** | toolset_distributions.py | policy + permissions |

---

## 技能系统对比

### 技能目录结构

**Hermes Agent**:
```
~/.hermes/skills/
├── my-skill/
│   ├── SKILL.md          # 主指令文件
│   ├── references/       # 参考文档
│   ├── templates/        # 模板文件
│   └── scripts/          # Python 脚本
└── category/
    └── another-skill/
        └── SKILL.md
```

**OpenClaw**:
```
skills/
├── my-skill/
│   ├── SKILL.md
│   ├── references/
│   └── scripts/
└── ...
```

### SKILL.md 格式对比

两者都遵循 [agentskills.io](https://agentskills.io) 标准：

```markdown
---
name: skill-name
description: Brief description
version: 1.0.0
platforms: [macos, linux]
prerequisites:
  env_vars: [API_KEY]
  commands: [curl]
---

# Skill Instructions

Your skill content here...
```

### 技能自动创建

**Hermes Agent 特色功能**：
- 任务完成后自动创建技能
- 技能使用中自我改进
- Skills Hub 集成

```python
# tools/skill_manager_tool.py
def auto_create_skill(task_description: str, steps: List[str]):
    """Agent 自动从任务中提取技能"""
    ...
```

**OpenClaw**:
- 手动创建技能
- 支持 clawhub 安装
- 无自动学习功能

---

## 记忆系统对比

### Hermes Agent 记忆系统

```python
# agent/memory_manager.py
class MemoryManager:
    """统一管理内置 + 外部记忆提供者"""
    
    def __init__(self):
        self._providers: List[MemoryProvider] = []
        
    def build_system_prompt(self) -> str:
        """构建记忆上下文"""
        ...
        
    def prefetch_all(self, user_message: str) -> str:
        """预取相关记忆"""
        ...
        
    def sync_all(self, user_msg, assistant_response):
        """同步记忆存储"""
        ...
```

**支持的 Memory Provider**:
- `BuiltinMemoryProvider` - 内置文件存储
- `HonchoProvider` - Honcho 用户建模
- `Mem0Provider` - Mem0 记忆系统
- `SupermemoryProvider` - Supermemory
- `HolographicProvider` - 向量检索

### OpenClaw 记忆系统

```typescript
// packages/memory-host-sdk/src/engine.ts
export class MemoryEngine {
  // 向量检索引擎
  async query(qmd: string): Promise<MemoryResult[]>
  
  // 批量嵌入
  async embedBatch(texts: string[]): Promise<number[][]>
  
  // 存储管理
  async store(entry: MemoryEntry): Promise<void>
}
```

**记忆架构**:
```
src/memory-host-sdk/
├── engine.ts              # 记忆引擎核心
├── engine-embeddings.ts   # 嵌入向量
├── engine-qmd.ts          # 查询语言
├── engine-storage.ts      # 存储后端
└── host/
    ├── sqlite-vec.ts      # SQLite 向量扩展
    ├── embeddings-remote-client.ts  # 远程嵌入
    └── batch-http.ts      # 批量处理
```

### 关键差异

| 方面 | Hermes Agent | OpenClaw |
|------|--------------|----------|
| **架构** | Manager + Provider 模式 | Engine + SDK 模式 |
| **向量存储** | 可插拔 Provider | 内置 sqlite-vec |
| **用户建模** | Honcho 辩证建模 | 无内置用户建模 |
| **会话搜索** | FTS5 + LLM 总结 | QMD 查询语言 |
| **记忆同步** | Post-turn 自动同步 | 手动 + 定期同步 |

---

## 设计哲学差异

### Hermes Agent: "Growth-Oriented"

```
┌─────────────────────────────────────────────────────────────┐
│                    Hermes 设计哲学                           │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  1. 自我进化为核心                                           │
│     - 自动技能创建                                          │
│     - 技能使用中改进                                        │
│     - 持续学习循环                                          │
│                                                             │
│  2. 用户建模                                                 │
│     - Honcho 辩证用户画像                                   │
│     - 跨会话记忆                                            │
│     - 个性化理解                                            │
│                                                             │
│  3. 研究导向                                                 │
│     - 批量轨迹生成                                          │
│     - Atropos RL 环境                                       │
│     - 训练下一代模型                                        │
│                                                             │
│  核心问题: "这个 Agent 如何变得更懂用户？"                   │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

### OpenClaw: "Integration-First"

```
┌─────────────────────────────────────────────────────────────┐
│                    OpenClaw 设计哲学                         │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  1. 多渠道统一                                               │
│     - 20+ 消息平台                                          │
│     - 统一消息格式                                          │
│     - 无缝切换                                              │
│                                                             │
│  2. 插件化架构                                               │
│     - extensions/ 独立开发                                   │
│     - plugin-sdk 类型安全                                   │
│     - 动态加载                                              │
│                                                             │
│  3. 企业级特性                                               │
│     - Canvas 交互界面                                       │
│     - ACP 协议支持                                          │
│     - 分布式部署                                            │
│                                                             │
│  核心问题: "如何让 Agent 无处不在？"                         │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

### 设计决策对比

| 决策点 | Hermes Agent 选择 | OpenClaw 选择 |
|--------|-------------------|---------------|
| **架构风格** | 单体脚本（run_agent.py） | 模块化 + 插件 |
| **语言选择** | Python（AI 研究友好） | TypeScript（工程健壮） |
| **扩展方式** | Skills + Plugins | Extensions + Skills |
| **分发模式** | PyPI 包 | npm 包 + Docker |
| **目标用户** | 研究者 + 高级用户 | 普通用户 + 企业 |

---

## 适用场景分析

### Hermes Agent 适合

✅ **研究场景**:
- Agent 学习与进化研究
- 用户建模实验
- RL 训练数据生成

✅ **高级用户**:
- 需要深度定制 Agent 行为
- 熟悉 Python 生态
- 追求"越用越懂我"的体验

✅ **特定需求**:
- 自动技能创建
- Honcho 用户建模
- 批量轨迹分析

### OpenClaw 适合

✅ **生产环境**:
- 多渠道消息统一
- 企业级部署
- 高可用性要求

✅ **普通用户**:
- 开箱即用
- 可视化 Canvas 界面
- 低门槛配置

✅ **企业场景**:
- 合规性要求
- 多租户支持
- 插件化扩展

### 不适合场景

❌ **Hermes Agent 不适合**:
- 需要高度模块化的企业架构
- TypeScript 技术栈团队
- 需要 Canvas 可视化界面

❌ **OpenClaw 不适合**:
- 需要自动学习能力的场景
- Python AI 研究工作流
- 需要 Honcho 用户建模

---

## 总结与建议

### 核心差异一览表

| 维度 | Hermes Agent | OpenClaw |
|------|:------------:|:--------:|
| **语言** | Python | TypeScript |
| **架构** | 单体为主 | 模块化+插件 |
| **学习** | ✅ 自动 | ❌ 手动 |
| **平台数** | ~12 | ~20+ |
| **Canvas** | ❌ | ✅ |
| **研究** | ✅ RL/Atropos | ❌ |
| **企业** | ⚠️ 需定制 | ✅ 原生支持 |
| **上手难度** | 中等 | 较低 |

### 迁移建议

Hermes Agent 内置了 OpenClaw 迁移工具：

```bash
hermes claw migrate --dry-run
hermes claw migrate --preset user-data
```

可迁移内容：
- SOUL.md 人格文件
- MEMORY.md / USER.md 记忆
- 自定义技能
- 命令审批列表
- 平台配置

### 选择建议

```
你需要什么样的 Agent？
│
├─ "我希望它越用越懂我"
│    └─ 选择 Hermes Agent
│
├─ "我需要在 20+ 平台使用"
│    └─ 选择 OpenClaw
│
├─ "我是 AI 研究者"
│    └─ 选择 Hermes Agent
│
├─ "我是企业 IT 管理员"
│    └─ 选择 OpenClaw
│
└─ "我两个都想要"
     └─ 可以共存：Hermes 用于研究，OpenClaw 用于生产
```

---

## 附录：源码关键路径

### Hermes Agent 关键文件

```
hermes-agent/
├── run_agent.py          # 核心 Agent Loop (676KB)
├── cli.py                # CLI 入口 (515KB)
├── toolsets.py           # 工具集定义
├── model_tools.py        # 工具处理
├── agent/
│   ├── memory_manager.py # 记忆管理
│   ├── prompt_builder.py # Prompt 构建
│   └── skill_utils.py    # 技能工具
├── tools/
│   ├── terminal_tool.py  # 终端工具
│   ├── skills_tool.py    # 技能工具
│   └── *.py              # 其他工具
├── gateway/
│   └── platforms/        # 平台适配器
└── skills/               # 技能库
```

### OpenClaw 关键文件

```
openclaw/
├── src/
│   ├── agents/           # Agent 核心
│   │   ├── agent-command.ts
│   │   └── skills.ts
│   ├── gateway/          # 网关核心
│   ├── channels/         # 平台适配器
│   ├── cli/              # CLI 入口
│   ├── acp/              # ACP 协议
│   ├── mcp/              # MCP 协议
│   └── memory-host-sdk/  # 记忆 SDK
├── extensions/           # 插件目录
├── skills/               # 技能库
└── packages/
    ├── plugin-sdk/       # 插件 SDK
    └── memory-host-sdk/  # 记忆 SDK
```

---

*文档生成时间: 2026-04-28*
*作者: JARVIS AI Assistant*

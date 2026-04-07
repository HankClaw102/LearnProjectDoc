# Google AI Edge Gallery 项目学习指南

> 本系列文章将带你从零开始深入理解 Google AI Edge Gallery 项目，学完后能够独立修改项目代码、修复 Issue、开发自定义功能。

---

## 项目简介

**Google AI Edge Gallery** 是 Google 官方推出的端侧 AI 应用示例项目，展示了如何在移动设备（Android/iOS）上运行大语言模型（LLM），实现：

- 100% 端侧推理，无需联网
- Agent Skills 扩展系统
- Function Calling 工具调用
- 多模态支持（文本/图像/音频）

**GitHub**: https://github.com/google-ai-edge/gallery

---

## 系列文章目录

### 基础篇

- **[第一章：项目概览与环境搭建](./01-overview.md)**
  - 项目定位与核心特性
  - 技术栈介绍
  - 开发环境配置
  - 首次编译运行

- **[第二章：架构设计深度解析](./02-architecture.md)**
  - 整体架构设计
  - 模块划分与职责
  - 依赖注入（Hilt）
  - 数据流向分析

- **[第三章：Agent Skills 系统详解](./03-agent-skills.md)**
  - Skill 系统设计理念
  - SKILL.md 格式规范
  - JS Skill 实现原理
  - Native Skill 调用机制

- **[第四章：模型推理与 Runtime](./04-runtime.md)**
  - LiteRT 推理引擎
  - LlmModelHelper 接口设计
  - 模型加载与推理流程
  - Function Calling 实现

- **[第五章：UI 层架构与 Compose](./05-ui-compose.md)**
  - Jetpack Compose 架构
  - ViewModel 设计模式
  - 状态管理（StateFlow）
  - 关键 UI 组件解析

### 实战篇

- **[第六章：实战：修改项目扩展功能](./06-modify-project.md)**
  - 添加新的 UI 页面
  - 扩展模型配置
  - 自定义推理参数
  - 实战案例：添加新功能模块

- **[第七章：实战：修复 Issue 完整流程](./07-fix-issue.md)**
  - Issue 定位与分析
  - 调试技巧
  - 代码修改与测试
  - PR 提交流程

- **[第八章：进阶：开发自定义 Skill](./08-custom-skill.md)**
  - Text-Only Skill 开发
  - JS Skill 开发实战
  - Secret 安全注入
  - 发布与分享 Skill

---

## 学习路线建议

```
入门（1-2天）
├── 第一章：环境搭建
└── 第二章：架构概览

理解（3-5天）
├── 第三章：Skill 系统 ⭐ 重点
├── 第四章：推理引擎
└── 第五章：UI 架构

实战（5-7天）
├── 第六章：修改项目
├── 第七章：修复 Issue
└── 第八章：开发 Skill
```

---

## 核心概念速查

| 概念 | 说明 |
|------|------|
| LiteRT | Google 端侧推理框架 |
| Agent Skill | 模块化的 LLM 能力扩展 |
| SKILL.md | Skill 定义文件 |
| ToolSet | Function Calling 工具集 |
| LlmModelHelper | 推理引擎抽象接口 |

---

## 常用文件路径

```
gallery/
├── Android/src/app/src/main/java/com/google/ai/edge/gallery/
│   ├── runtime/           # 推理引擎
│   ├── ui/                # UI 层
│   ├── customtasks/       # 特色功能
│   │   └── agentchat/     # Agent Skills ⭐
│   └── data/              # 数据层
├── skills/                # Skill 示例
│   ├── built-in/         # 内置技能
│   └── featured/         # 社区精选
└── model_allowlist.json   # 模型白名单
```

---

> 💡 **提示**：建议按顺序阅读，每章内容都有前后依赖关系。如遇到问题，可在 GitHub Issues 中搜索或提问。

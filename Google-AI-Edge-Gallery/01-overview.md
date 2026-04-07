# 第一章：项目概览与环境搭建

> 本章将介绍 Google AI Edge Gallery 项目的定位、核心特性，并带你完成开发环境配置。

---

## 1.1 项目定位

Google AI Edge Gallery 是一个**端侧大模型移动应用示例项目**，由 Google AI Edge 团队维护。

### 核心目标

- **展示端侧 AI 能力** - 在手机上运行大语言模型
- **提供可扩展架构** - Agent Skills 系统支持模块化扩展
- **开源学习参考** - 帮助开发者学习端侧 AI 开发

### 与云端 AI 的区别

| 特性 | 端侧 AI | 云端 AI |
|------|---------|---------|
| 隐私保护 | ✅ 100% 本地 | ❌ 数据上传 |
| 网络依赖 | ✅ 离线可用 | ❌ 必须联网 |
| 延迟 | ✅ 毫秒级 | ❌ 依赖网络 |
| 模型能力 | ⚠️ 受设备限制 | ✅ 可用超大模型 |

---

## 1.2 核心特性

### 1.2.1 多模态支持

```
支持的输入类型：
├── 文本（Text）
├── 图像（Image）- Gemma 3n / Gemma 4
└── 音频（Audio）- 语音转文字
```

### 1.2.2 Agent Skills 系统

这是项目最核心的设计亮点：

- **模块化扩展** - 通过 SKILL.md 定义新能力
- **JS 执行环境** - WebView 沙箱运行自定义逻辑
- **Native 调用** - 直接调用 Android/iOS 系统能力

### 1.2.3 支持的模型

| 模型 | 参数量 | 文件大小 | 内存需求 | 多模态 |
|------|--------|----------|----------|--------|
| Gemma 3n E2B | 2B | 2.9GB | ~6GB | ✅ |
| Gemma 3n E4B | 4B | 4.1GB | ~7GB | ✅ |
| Gemma 3 1B | 1B | 530MB | ~2GB | ❌ |
| Qwen2.5 1.5B | 1.5B | 1.5GB | ~2.7GB | ❌ |

### 1.2.4 特色功能模块

- **AI Chat** - 多轮对话 + Thinking Mode
- **Ask Image** - 图像理解
- **Audio Scribe** - 语音转文字
- **Prompt Lab** - 提示词实验
- **Mobile Actions** - 设备控制
- **Benchmark** - 性能测试

---

## 1.3 技术栈

### Android 端

```
├── 语言：Kotlin
├── UI：Jetpack Compose
├── DI：Hilt
├── 架构：MVVM
├── 推理：LiteRT（Google AI Edge）
├── 序列化：Protocol Buffers + Moshi
└── 异步：Kotlin Coroutines + Flow
```

### 推理引擎

- **LiteRT** - Google 轻量级推理运行时
- **MediaPipe LLM Inference API** - 大模型推理接口
- **Hugging Face** - 模型托管与下载

---

## 1.4 开发环境配置

### 1.4.1 系统要求

- **操作系统**：Windows / macOS / Linux
- **Android Studio**：最新稳定版（推荐 Hedgehog 或 newer）
- **JDK**：17+
- **Android SDK**：API 31+（Android 12）
- **Gradle**：8.0+

### 1.4.2 克隆项目

```bash
git clone https://github.com/google-ai-edge/gallery.git
cd gallery
```

### 1.4.3 配置 Hugging Face OAuth

项目使用 Hugging Face 下载模型，需要配置 OAuth：

1. 访问 https://huggingface.co/settings/applications
2. 创建一个新的 Developer Application
3. 获取 `clientId` 和 `redirectUri`

修改配置文件：

```kotlin
// Android/src/app/src/main/java/com/google/ai/edge/gallery/common/ProjectConfig.kt

object ProjectConfig {
    const val HUGGING_FACE_CLIENT_ID = "你的clientId"
    const val HUGGING_FACE_REDIRECT_URI = "你的redirectUri"
}
```

同时修改 `app/build.gradle.kts`：

```kotlin
// Android/src/app/build.gradle.kts

manifestPlaceholders["appAuthRedirectScheme"] = "你的redirectUri的scheme部分"
```

### 1.4.4 编译运行

```bash
cd Android/src
./gradlew installDebug
```

或在 Android Studio 中：
1. 打开 `Android/src` 目录
2. 等待 Gradle Sync 完成
3. 连接真机或模拟器
4. 点击 Run 按钮

---

## 1.5 项目结构

```
gallery/
├── Android/                    # Android 应用
│   └── src/app/
│       └── src/main/java/com/google/ai/edge/gallery/
│           ├── runtime/        # 推理引擎接口
│           │   ├── LlmModelHelper.kt
│           │   └── ModelHelperExt.kt
│           ├── ui/             # UI 层
│           │   ├── llmchat/   # 聊天界面
│           │   ├── common/    # 通用组件
│           │   └── navigation/# 导航
│           ├── customtasks/   # 特色功能
│           │   ├── agentchat/ # Agent Skills ⭐
│           │   ├── mobileactions/ # 设备控制
│           │   └── tinygarden/ # 小游戏示例
│           ├── data/          # 数据模型
│           │   ├── Model.kt
│           │   └── ConfigKeys.kt
│           ├── common/        # 工具类
│           └── di/            # 依赖注入模块
├── skills/                    # Skill 示例
│   ├── built-in/             # 内置技能
│   │   ├── query-wikipedia/
│   │   ├── interactive-map/
│   │   ├── qr-code/
│   │   └── ...
│   └── featured/             # 社区精选
├── model_allowlist.json      # 模型白名单
├── Function_Calling_Guide.md # Function Calling 指南
└── DEVELOPMENT.md            # 开发说明
```

---

## 1.6 常见问题

### Q1: 编译报错 "SDK location not found"

创建 `local.properties` 文件：

```properties
sdk.dir=/path/to/Android/sdk
```

### Q2: 模型下载失败

检查 Hugging Face OAuth 配置是否正确，确保 `clientId` 和 `redirectUri` 匹配。

### Q3: 运行时闪退

确保设备内存足够（建议 8GB+），部分模型需要较大内存。

### Q4: 如何查看日志

```bash
adb logcat | grep -E "AGAgentTools|LlmModelHelper|GalleryApp"
```

---

## 1.7 小结

本章介绍了：

- 项目定位与核心特性
- 技术栈概览
- 开发环境配置
- 项目目录结构

**下一章**：我们将深入分析项目的整体架构设计。

---

> 📚 **延伸阅读**：
> - [Google AI Edge 官方文档](https://ai.google.dev/edge)
> - [LiteRT GitHub](https://github.com/google-ai-edge/LiteRT-LM)
> - [Jetpack Compose 官方指南](https://developer.android.com/jetpack/compose)

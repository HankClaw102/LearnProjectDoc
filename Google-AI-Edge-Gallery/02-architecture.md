# 第二章：架构设计深度解析

> 本章将深入分析 Google AI Edge Gallery 的整体架构设计，理解各模块职责与数据流向。

---

## 2.1 整体架构

```
┌─────────────────────────────────────────────────────────────┐
│                     AI Edge Gallery App                      │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  ┌─────────────────────────────────────────────────────┐    │
│  │                   UI Layer (Compose)                 │    │
│  │  ┌─────────┐  ┌─────────┐  ┌─────────────────────┐  │    │
│  │  │ Chat UI │  │ Settings│  │ Custom Task Screens │  │    │
│  │  └────┬────┘  └────┬────┘  └──────────┬──────────┘  │    │
│  └───────┼────────────┼──────────────────┼─────────────┘    │
│          │            │                  │                   │
│  ┌───────▼────────────▼──────────────────▼─────────────┐    │
│  │                 ViewModel Layer                      │    │
│  │  ┌──────────────┐  ┌──────────────┐  ┌───────────┐  │    │
│  │  │ChatViewModel │  │SettingsVM    │  │AgentChatVM│  │    │
│  │  └──────────────┘  └──────────────┘  └───────────┘  │    │
│  └─────────────────────────┬───────────────────────────┘    │
│                            │                                 │
│  ┌─────────────────────────▼───────────────────────────┐    │
│  │                   Business Logic                     │    │
│  │  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐  │    │
│  │  │ SkillManager│  │ AgentTools  │  │IntentHandler│  │    │
│  │  └─────────────┘  └─────────────┘  └─────────────┘  │    │
│  └─────────────────────────┬───────────────────────────┘    │
│                            │                                 │
│  ┌─────────────────────────▼───────────────────────────┐    │
│  │                   Runtime Layer                      │    │
│  │  ┌───────────────────────────────────────────────┐  │    │
│  │  │              LlmModelHelper                    │  │    │
│  │  │  ┌─────────────────────────────────────────┐  │  │    │
│  │  │  │         LiteRT Inference Engine         │  │  │    │
│  │  │  └─────────────────────────────────────────┘  │  │    │
│  │  └───────────────────────────────────────────────┘  │    │
│  └─────────────────────────────────────────────────────┘    │
│                                                              │
│  ┌─────────────────────────────────────────────────────┐    │
│  │                   Data Layer                         │    │
│  │  ┌──────────┐  ┌──────────┐  ┌──────────────────┐   │    │
│  │  │ Model.kt │  │ Config   │  │ DataStore        │   │    │
│  │  └──────────┘  └──────────┘  └──────────────────┘   │    │
│  └─────────────────────────────────────────────────────┘    │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

---

## 2.2 核心模块职责

### 2.2.1 UI Layer（表现层）

**位置**：`ui/` 目录

**职责**：
- 渲染用户界面
- 接收用户输入
- 展示推理结果

**关键组件**：

```kotlin
// ui/common/chat/ChatViewModel.kt
abstract class ChatViewModel : ViewModel() {
    private val _uiState = MutableStateFlow(createUiState())
    val uiState = _uiState.asStateFlow()
    
    // 消息管理
    fun addMessage(model: Model, message: ChatMessage)
    fun updateLastTextMessageContentIncrementally(model: Model, partialContent: String)
    fun clearAllMessages(model: Model)
}
```

### 2.2.2 ViewModel Layer（视图模型层）

**职责**：
- 管理 UI 状态
- 协调业务逻辑
- 处理生命周期

**状态管理模式**：

```kotlin
data class ChatUiState(
    val inProgress: Boolean = false,        // 是否正在处理
    val isResettingSession: Boolean = false, // 是否重置会话
    val preparing: Boolean = false,          // 是否准备中
    val messagesByModel: Map<String, MutableList<ChatMessage>> = mapOf(),
    val streamingMessagesByModel: Map<String, ChatMessage> = mapOf(),
)
```

### 2.2.3 Business Logic（业务逻辑层）

**位置**：`customtasks/` 目录

**核心组件**：

| 组件 | 职责 |
|------|------|
| `SkillManagerViewModel` | 管理 Skill 加载、缓存、执行 |
| `AgentTools` | 定义 Function Calling 工具 |
| `IntentHandler` | 处理 Native Intent 调用 |

### 2.2.4 Runtime Layer（运行时层）

**位置**：`runtime/` 目录

**核心接口**：

```kotlin
interface LlmModelHelper {
    // 初始化模型
    fun initialize(
        context: Context,
        model: Model,
        supportImage: Boolean,
        supportAudio: Boolean,
        onDone: (String) -> Unit,
        systemInstruction: Contents? = null,
        tools: List<ToolProvider> = listOf(),
        ...
    )
    
    // 运行推理
    fun runInference(
        model: Model,
        input: String,
        resultListener: ResultListener,
        ...
    )
    
    // 停止生成
    fun stopResponse(model: Model)
    
    // 清理资源
    fun cleanUp(model: Model, onDone: () -> Unit)
}
```

### 2.2.5 Data Layer（数据层）

**位置**：`data/` 目录

**数据模型**：

```kotlin
// Model.kt - 模型定义
data class Model(
    val name: String,              // 模型名称
    val modelId: String,           // Hugging Face ID
    val modelFile: String,         // 模型文件名
    val sizeInBytes: Long,         // 文件大小
    val defaultConfig: Config,     // 默认配置
    val taskTypes: List<String>,   // 支持的任务类型
)

// ConfigKeys.kt - 配置键
object ConfigKeys {
    const val TEMPERATURE = "temperature"
    const val TOP_K = "topK"
    const val TOP_P = "topP"
    const val MAX_TOKENS = "maxTokens"
    const val ACCELERATOR = "accelerators"
}
```

---

## 2.3 依赖注入（Hilt）

### 2.3.1 模块定义

```kotlin
// di/AppModule.kt
@Module
@InstallIn(SingletonComponent::class)
object AppModule {
    
    @Provides
    @Singleton
    fun provideDataStoreRepository(
        @ApplicationContext context: Context
    ): DataStoreRepository {
        return DataStoreRepository(context)
    }
}
```

### 2.3.2 ViewModel 注入

```kotlin
@HiltViewModel
class SkillManagerViewModel @Inject constructor(
    @ApplicationContext private val context: Context,
    private val dataStoreRepository: DataStoreRepository,
) : ViewModel() {
    // ...
}
```

### 2.3.3 入口点

```kotlin
@HiltAndroidApp
class GalleryApplication : Application()

@AndroidEntryPoint
class MainActivity : ComponentActivity()
```

---

## 2.4 数据流向分析

### 2.4.1 用户发送消息流程

```
用户输入
    │
    ▼
┌─────────────┐
│ ChatScreen  │  Composable 接收输入
└─────┬───────┘
      │
      ▼
┌─────────────┐
│ ChatViewModel│  更新 UI 状态，添加用户消息
└─────┬───────┘
      │
      ▼
┌─────────────┐
│LlmModelHelper│  调用推理引擎
└─────┬───────┘
      │
      ▼
┌─────────────┐
│   LiteRT    │  执行模型推理
└─────┬───────┘
      │
      ▼ 流式返回结果
┌─────────────┐
│ ChatViewModel│  增量更新消息内容
└─────┬───────┘
      │
      ▼
┌─────────────┐
│ ChatScreen  │  UI 自动刷新
└─────────────┘
```

### 2.4.2 Skill 调用流程

```
用户: "查一下爱因斯坦的维基百科"
    │
    ▼
┌─────────────────┐
│  LLM 推理       │  识别需要调用 query-wikipedia skill
└────┬────────────┘
     │
     ▼ Function Calling
┌─────────────────┐
│  AgentTools     │  runJs() 方法被调用
└────┬────────────┘
     │
     ▼
┌─────────────────┐
│SkillManagerVM   │  加载 SKILL.md，获取脚本 URL
└────┬────────────┘
     │
     ▼
┌─────────────────┐
│  WebView        │  执行 index.html 中的 JS
└────┬────────────┘
     │
     ▼ 调用 Wikipedia API
┌─────────────────┐
│  API Response   │  返回查询结果
└────┬────────────┘
     │
     ▼
┌─────────────────┐
│  LLM 继续推理   │  基于返回结果生成回复
└─────────────────┘
```

---

## 2.5 关键设计模式

### 2.5.1 MVVM 架构

```
View (Compose) ←→ ViewModel ←→ Model
      │               │            │
      │   StateFlow   │            │
      └───────────────┘            │
                      Repository   │
                      └────────────┘
```

### 2.5.2 Repository 模式

```kotlin
class DataStoreRepository @Inject constructor(
    private val context: Context
) {
    // 持久化存储
    suspend fun saveSecret(key: String, value: String)
    fun readSecret(key: String): String?
}
```

### 2.5.3 Strategy 模式（推理引擎）

```kotlin
interface LlmModelHelper {
    // 抽象接口，支持不同实现
}

// 未来可扩展：
// - LiteRTModelHelper（当前实现）
// - OnnxModelHelper
// - TensorflowLiteModelHelper
```

---

## 2.6 模块依赖关系

```
                    ┌──────────┐
                    │   App    │
                    └────┬─────┘
                         │
         ┌───────────────┼───────────────┐
         │               │               │
    ┌────▼────┐    ┌────▼────┐    ┌────▼────┐
    │   UI    │    │ Runtime │    │  Data   │
    └────┬────┘    └────┬────┘    └────┬────┘
         │               │               │
         └───────────────┼───────────────┘
                         │
                    ┌────▼────┐
                    │ Common  │
                    └─────────┘
```

---

## 2.7 配置系统

### 2.7.1 模型配置（model_allowlist.json）

```json
{
  "models": [
    {
      "name": "Gemma-3n-E2B-it-int4",
      "modelId": "google/gemma-3n-E2B-it-litert-preview",
      "modelFile": "gemma-3n-E2B-it-int4.task",
      "sizeInBytes": 3136226711,
      "defaultConfig": {
        "topK": 64,
        "topP": 0.95,
        "temperature": 1.0,
        "maxTokens": 4096,
        "accelerators": "cpu,gpu"
      },
      "taskTypes": ["llm_chat", "llm_prompt_lab", "llm_ask_image"]
    }
  ]
}
```

### 2.7.2 运行时配置

```kotlin
// 用户可调整的配置
data class ModelConfig(
    val temperature: Float = 1.0f,    // 创造性
    val topK: Int = 64,               // 采样范围
    val topP: Float = 0.95f,          // 核采样
    val maxTokens: Int = 4096,        // 最大生成长度
    val accelerator: String = "gpu",  // 加速器
)
```

---

## 2.8 小结

本章深入分析了：

- 整体架构的分层设计
- 各模块的职责与边界
- 依赖注入的使用方式
- 数据流向与调用链路

**下一章**：我们将深入 Agent Skills 系统，这是项目最核心的设计亮点。

---

> 💡 **思考题**：
> 1. 为什么选择 MVVM 而不是 MVP？
> 2. LlmModelHelper 设计为接口有什么好处？
> 3. Skill 调用为什么选择 WebView 执行而不是原生代码？

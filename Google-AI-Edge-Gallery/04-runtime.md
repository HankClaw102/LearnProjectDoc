# 第四章：模型推理与 Runtime

> 本章将深入分析 LiteRT 推理引擎的工作原理，以及如何在 Android 上运行大语言模型。

---

## 4.1 LiteRT 概述

### 4.1.1 什么是 LiteRT？

**LiteRT** (Lite Runtime) 是 Google AI Edge 提供的轻量级推理运行时：

- 专为移动端优化
- 支持大语言模型推理
- 提供 Function Calling 能力
- 跨平台（Android / iOS）

### 4.1.2 架构位置

```
┌─────────────────────────────────────────┐
│           Application Layer             │
├─────────────────────────────────────────┤
│          LlmModelHelper (接口)          │
├─────────────────────────────────────────┤
│              LiteRT Engine              │
│  ┌─────────────────────────────────┐    │
│  │   MediaPipe LLM Inference API   │    │
│  └─────────────────────────────────┘    │
├─────────────────────────────────────────┤
│         Hardware Accelerators           │
│      (CPU / GPU / NPU)                  │
└─────────────────────────────────────────┘
```

---

## 4.2 模型格式

### 4.2.1 .task 文件

LiteRT 使用 `.task` 格式的模型文件：

```
gemma-3n-E2B-it-int4.task
├── 模型权重（量化后）
├── 词表文件
├── 配置信息
└── 元数据
```

### 4.2.2 模型来源

模型托管在 Hugging Face：

```
https://huggingface.co/google/gemma-3n-E2B-it-litert-preview
https://huggingface.co/litert-community/Gemma3-1B-IT
https://huggingface.co/litert-community/Qwen2.5-1.5B-Instruct
```

### 4.2.3 模型配置

```json
{
  "name": "Gemma-3n-E2B-it-int4",
  "modelId": "google/gemma-3n-E2B-it-litert-preview",
  "modelFile": "gemma-3n-E2B-it-int4.task",
  "sizeInBytes": 3136226711,
  "estimatedPeakMemoryInBytes": 5905580032,
  "llmSupportImage": true,
  "defaultConfig": {
    "topK": 64,
    "topP": 0.95,
    "temperature": 1.0,
    "maxTokens": 4096,
    "accelerators": "cpu,gpu"
  },
  "taskTypes": ["llm_chat", "llm_prompt_lab", "llm_ask_image"]
}
```

---

## 4.3 LlmModelHelper 接口

### 4.3.1 接口定义

```kotlin
// runtime/LlmModelHelper.kt
interface LlmModelHelper {
    
    /**
     * 初始化模型
     */
    fun initialize(
        context: Context,
        model: Model,                    // 模型配置
        supportImage: Boolean,           // 是否支持图像
        supportAudio: Boolean,           // 是否支持音频
        onDone: (String) -> Unit,        // 初始化完成回调
        systemInstruction: Contents? = null,  // 系统提示
        tools: List<ToolProvider> = listOf(), // Function Calling 工具
        enableConversationConstrainedDecoding: Boolean = false,
        coroutineScope: CoroutineScope? = null,
    )

    /**
     * 重置对话
     */
    fun resetConversation(
        model: Model,
        supportImage: Boolean = false,
        supportAudio: Boolean = false,
        systemInstruction: Contents? = null,
        tools: List<ToolProvider> = listOf(),
        enableConversationConstrainedDecoding: Boolean = false,
    )

    /**
     * 运行推理
     */
    fun runInference(
        model: Model,
        input: String,                   // 用户输入
        resultListener: ResultListener,  // 流式结果回调
        cleanUpListener: CleanUpListener, // 清理回调
        onError: (message: String) -> Unit = {},
        images: List<Bitmap> = listOf(), // 图像输入
        audioClips: List<ByteArray> = listOf(), // 音频输入
        coroutineScope: CoroutineScope? = null,
        extraContext: Map<String, String>? = null,
    )

    /**
     * 停止生成
     */
    fun stopResponse(model: Model)

    /**
     * 清理资源
     */
    fun cleanUp(model: Model, onDone: () -> Unit)
}

// 流式结果回调
typealias ResultListener = (
    partialResult: String,      // 部分结果
    done: Boolean,              // 是否完成
    partialThinkingResult: String?  // 思考过程（Thinking Mode）
) -> Unit
```

### 4.3.2 Contents 与 ToolProvider

```kotlin
// LiteRT 提供的数据类型
com.google.ai.edge.litertlm.Contents    // 消息内容
com.google.ai.edge.litertlm.ToolProvider // 工具提供者
com.google.ai.edge.litertlm.Tool        // 工具定义
com.google.ai.edge.litertlm.ToolParam   // 工具参数
com.google.ai.edge.litertlm.ToolSet     // 工具集合
```

---

## 4.4 推理流程

### 4.4.1 初始化流程

```kotlin
fun initialize(context: Context, model: Model, ...) {
    viewModelScope.launch {
        try {
            // 1. 检查模型文件是否存在
            val modelFile = File(context.filesDir, model.modelFile)
            if (!modelFile.exists()) {
                // 触发下载
                downloadModel(model)
            }
            
            // 2. 创建推理引擎
            val engine = LlmEngine.Builder()
                .setModelPath(modelFile.absolutePath)
                .setAccelerator(model.accelerator)
                .setMaxTokens(model.maxTokens)
                .build()
            
            // 3. 设置系统提示
            if (systemInstruction != null) {
                engine.setSystemInstruction(systemInstruction)
            }
            
            // 4. 注册工具
            if (tools.isNotEmpty()) {
                engine.setTools(tools)
            }
            
            // 5. 完成回调
            onDone("Model initialized successfully")
            
        } catch (e: Exception) {
            onError(e.message ?: "Unknown error")
        }
    }
}
```

### 4.4.2 推理流程

```kotlin
fun runInference(model: Model, input: String, resultListener: ResultListener, ...) {
    viewModelScope.launch {
        try {
            // 1. 准备输入
            val contents = Contents.Builder()
                .addText(input)
                .apply {
                    // 添加图像（如果有）
                    images.forEach { image ->
                        addImage(image)
                    }
                }
                .build()
            
            // 2. 流式推理
            engine.generateAsync(contents) { partialResult, done ->
                // 3. 流式回调
                resultListener(partialResult, done, null)
                
                if (done) {
                    // 4. 推理完成
                    cleanUpListener()
                }
            }
            
        } catch (e: Exception) {
            onError(e.message ?: "Inference error")
        }
    }
}
```

### 4.4.3 时序图

```
┌────────┐     ┌──────────────┐     ┌──────────┐     ┌────────┐
│  User  │     │  ViewModel   │     │  Helper  │     │ LiteRT │
└───┬────┘     └──────┬───────┘     └────┬─────┘     └───┬────┘
    │                 │                  │               │
    │   Send Message  │                  │               │
    │────────────────>│                  │               │
    │                 │                  │               │
    │                 │  runInference()  │               │
    │                 │─────────────────>│               │
    │                 │                  │               │
    │                 │                  │  Generate     │
    │                 │                  │──────────────>│
    │                 │                  │               │
    │                 │                  │  ← Partial    │
    │                 │                  │<──────────────│
    │                 │                  │               │
    │                 │  ← OnResult()   │               │
    │                 │<─────────────────│               │
    │                 │                  │               │
    │   Update UI     │                  │               │
    │<────────────────│                  │               │
    │                 │                  │               │
    │                 │                  │  ← Done       │
    │                 │                  │<──────────────│
    │                 │  ← OnDone()      │               │
    │                 │<─────────────────│               │
    │                 │                  │               │
    │   Final Result  │                  │               │
    │<────────────────│                  │               │
```

---

## 4.5 Function Calling

### 4.5.1 工具定义

```kotlin
class AgentTools : ToolSet {
    
    @Tool(description = "Loads a skill by name.")
    fun loadSkill(
        @ToolParam(description = "The name of the skill to load.")
        skillName: String
    ): Map<String, String> {
        // 实现...
    }

    @Tool(description = "Runs JavaScript in WebView.")
    fun runJs(
        @ToolParam(description = "Skill name")
        skillName: String,
        @ToolParam(description = "Script file name")
        scriptName: String,
        @ToolParam(description = "JSON data to pass")
        data: String,
    ): Map<String, Any> {
        // 实现...
    }
}
```

### 4.5.2 工具注册

```kotlin
// 初始化时注册工具
val tools = listOf(AgentTools())
engine.setTools(tools)
```

### 4.5.3 工具调用流程

```
用户输入
    │
    ▼
LLM 决定需要调用工具
    │
    ▼
生成 Function Call：
{
    "name": "runJs",
    "arguments": {
        "skillName": "query-wikipedia",
        "scriptName": "index.html",
        "data": "{\"topic\":\"牛顿\",\"lang\":\"zh\"}"
    }
}
    │
    ▼
LiteRT 调用 AgentTools.runJs()
    │
    ▼
返回结果：
{
    "result": "艾萨克·牛顿（1643-1727）..."
}
    │
    ▼
LLM 继续推理，生成最终回复
```

---

## 4.6 多模态支持

### 4.6.1 图像输入

```kotlin
// 构建多模态内容
val contents = Contents.Builder()
    .addText("这张图片里有什么？")
    .addImage(bitmap)  // 添加图像
    .build()

// 推理
engine.generateAsync(contents) { result, done ->
    // 流式返回
}
```

### 4.6.2 支持图像的模型

```kotlin
// 检查模型是否支持图像
if (model.llmSupportImage) {
    // 可以添加图像输入
}
```

---

## 4.7 Thinking Mode

### 4.7.1 概念

Gemma 4 引入了 Thinking Mode，可以展示模型的推理过程：

```
用户: "如果我有5个苹果，吃了2个，又买了3个，还剩几个？"

[Thinking]
用户开始有5个苹果
吃了2个，剩下 5 - 2 = 3 个
又买了3个，现在有 3 + 3 = 6 个
[/Thinking]

回答：你现在有6个苹果。
```

### 4.7.2 实现

```kotlin
// ResultListener 包含 thinking 参数
typealias ResultListener = (
    partialResult: String,
    done: Boolean,
    partialThinkingResult: String?  // 思考过程
) -> Unit

// UI 显示
if (thinkingResult != null) {
    ThinkingPanel(thinkingResult)
}
OutputPanel(partialResult)
```

---

## 4.8 性能优化

### 4.8.1 KV Cache

```kotlin
// 启用 KV Cache 加速
engine.setKVCacheEnabled(true)
engine.setKVCacheSize(2048)  // 缓存最近 2048 tokens
```

### 4.8.2 量化

模型使用 INT4 量化：

```
原始 FP16: 8GB
量化后 INT4: 2GB（压缩 4x）
精度损失: < 1%
```

### 4.8.3 加速器选择

```kotlin
// 优先级：NPU > GPU > CPU
val accelerators = when {
    hasNPU() -> "npu"
    hasGPU() -> "gpu"
    else -> "cpu"
}
```

---

## 4.9 错误处理

### 4.9.1 常见错误

```kotlin
sealed class InferenceError {
    object ModelNotFound : InferenceError()
    object OutOfMemory : InferenceError()
    object InferenceTimeout : InferenceError()
    data class Unknown(val message: String) : InferenceError()
}

fun handle(error: InferenceError) {
    when (error) {
        is ModelNotFound -> showToast("模型未下载")
        is OutOfMemory -> showToast("内存不足，请关闭其他应用")
        is InferenceTimeout -> showToast("推理超时")
        is Unknown -> showToast("未知错误: ${error.message}")
    }
}
```

### 4.9.2 资源清理

```kotlin
// 退出时清理
override fun onDestroy() {
    super.onDestroy()
    modelHelper.cleanUp(model) {
        Log.d(TAG, "Model resources cleaned up")
    }
}
```

---

## 4.10 小结

本章介绍了：

- LiteRT 推理引擎架构
- LlmModelHelper 接口设计
- 推理流程与 Function Calling
- 多模态与 Thinking Mode
- 性能优化策略

**下一章**：我们将分析 UI 层架构与 Compose 的使用。

---

> 💡 **深入理解**：
> - 为什么 LiteRT 比 TensorFlow Lite 更适合 LLM？
> - KV Cache 如何加速多轮对话？
> - Function Calling 的实现原理是什么？

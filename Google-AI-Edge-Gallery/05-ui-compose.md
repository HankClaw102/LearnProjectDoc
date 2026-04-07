# 第五章：UI 层架构与 Compose

> 本章将分析 Google AI Edge Gallery 的 UI 层设计，包括 Jetpack Compose 的使用、状态管理和关键组件。

---

## 5.1 UI 技术栈

### 5.1.1 核心技术

```
├── Jetpack Compose    - 声明式 UI 框架
├── Material Design 3  - 设计系统
├── ViewModel          - 状态持有
├── StateFlow          - 响应式状态流
└── Compose Navigation - 导航框架
```

### 5.1.2 Compose 优势

| 特性 | 说明 |
|------|------|
| 声明式 | UI = f(State) |
| 响应式 | 状态变化自动刷新 UI |
| 可组合 | 小组件组合成大组件 |
| 可预览 | Android Studio 实时预览 |

---

## 5.2 UI 目录结构

```
ui/
├── common/              # 通用组件
│   ├── chat/           # 聊天组件
│   │   ├── ChatPanel.kt
│   │   ├── ChatView.kt
│   │   ├── ChatViewModel.kt
│   │   ├── MessageInputText.kt
│   │   └── ChatMessage.kt
│   ├── ModelPageAppBar.kt
│   └── Theme.kt
├── llmchat/            # 聊天功能
│   ├── LlmChatScreen.kt
│   └── LlmChatViewModel.kt
├── navigation/         # 导航
│   └── GalleryNavGraph.kt
└── theme/              # 主题
    ├── Color.kt
    ├── Theme.kt
    └── Type.kt
```

---

## 5.3 状态管理

### 5.3.1 ChatUiState

```kotlin
// ui/common/chat/ChatViewModel.kt
data class ChatUiState(
    // 是否正在处理
    val inProgress: Boolean = false,
    
    // 是否重置会话
    val isResettingSession: Boolean = false,
    
    // 是否准备中
    val preparing: Boolean = false,
    
    // 按模型分组的消息
    val messagesByModel: Map<String, MutableList<ChatMessage>> = mapOf(),
    
    // 流式消息
    val streamingMessagesByModel: Map<String, ChatMessage> = mapOf(),
)
```

### 5.3.2 StateFlow

```kotlin
abstract class ChatViewModel : ViewModel() {
    // 私有可变状态
    private val _uiState = MutableStateFlow(createUiState())
    
    // 公开只读状态
    val uiState = _uiState.asStateFlow()
    
    // 更新状态
    protected fun updateState(transform: (ChatUiState) -> ChatUiState) {
        _uiState.update(transform)
    }
}
```

### 5.3.3 Compose 中使用

```kotlin
@Composable
fun ChatScreen(viewModel: ChatViewModel) {
    // 收集状态
    val uiState by viewModel.uiState.collectAsState()
    
    // 根据状态渲染 UI
    if (uiState.inProgress) {
        CircularProgressIndicator()
    }
    
    LazyColumn {
        items(uiState.messagesByModel[currentModel] ?: emptyList()) { message ->
            ChatMessageItem(message)
        }
    }
}
```

---

## 5.4 消息类型

### 5.4.1 ChatMessage 定义

```kotlin
// 消息类型枚举
enum class ChatMessageType {
    TEXT,                    // 文本消息
    THINKING,                // 思考过程
    IMAGE,                   // 图片
    LOADING,                 // 加载中
    ERROR,                   // 错误
    PROMPT_TEMPLATES,        // 提示模板
    CONFIG_VALUES_CHANGE,    // 配置变更
    COLLAPSABLE_PROGRESS_PANEL, // 可折叠进度面板
}

// 消息侧
enum class ChatSide {
    USER,    // 用户消息
    MODEL,   // 模型回复
    SYSTEM,  // 系统消息
}

// 消息基类
sealed class ChatMessage(
    val type: ChatMessageType,
    val side: ChatSide,
)
```

### 5.4.2 具体消息类型

```kotlin
// 文本消息
data class ChatMessageText(
    val content: String,
    val side: ChatSide,
    val latencyMs: Float = 0f,
    val accelerator: String = "",
    val hideSenderLabel: Boolean = false,
    var llmBenchmarkResult: ChatMessageBenchmarkLlmResult? = null,
) : ChatMessage(ChatMessageType.TEXT, side)

// 思考消息
data class ChatMessageThinking(
    val content: String,
    val inProgress: Boolean,
    val side: ChatSide,
    val hideSenderLabel: Boolean = false,
    val accelerator: String = "",
) : ChatMessage(ChatMessageType.THINKING, side)

// 加载消息
object ChatMessageLoading : ChatMessage(ChatMessageType.LOADING, ChatSide.MODEL)

// 可折叠进度面板
data class ChatMessageCollapsableProgressPanel(
    val title: String,
    val inProgress: Boolean,
    val doneIcon: ImageVector,
    val items: List<ProgressPanelItem>,
    val accelerator: String = "",
    val customData: Any? = null,
    val logMessages: List<LogMessage> = emptyList(),
) : ChatMessage(ChatMessageType.COLLAPSABLE_PROGRESS_PANEL, ChatSide.MODEL)
```

---

## 5.5 关键组件

### 5.5.1 ChatPanel

```kotlin
@Composable
fun ChatPanel(
    model: Model,
    messages: List<ChatMessage>,
    inProgress: Boolean,
    onSendMessage: (String, List<Bitmap>) -> Unit,
    onStopResponse: () -> Unit,
) {
    Column(modifier = Modifier.fillMaxSize()) {
        // 消息列表
        LazyColumn(
            modifier = Modifier.weight(1f),
            reverseLayout = true,
        ) {
            items(messages) { message ->
                ChatMessageItem(message = message)
            }
        }
        
        // 输入区域
        MessageInputText(
            enabled = !inProgress,
            onSend = { text, images ->
                onSendMessage(text, images)
            }
        )
        
        // 停止按钮
        if (inProgress) {
            StopButton(onClick = onStopResponse)
        }
    }
}
```

### 5.5.2 ChatMessageItem

```kotlin
@Composable
fun ChatMessageItem(message: ChatMessage) {
    when (message) {
        is ChatMessageText -> {
            TextMessageBubble(
                text = message.content,
                isUser = message.side == ChatSide.USER,
                latencyMs = message.latencyMs,
            )
        }
        
        is ChatMessageThinking -> {
            ThinkingPanel(
                content = message.content,
                inProgress = message.inProgress,
            )
        }
        
        is ChatMessageLoading -> {
            LoadingIndicator()
        }
        
        is ChatMessageCollapsableProgressPanel -> {
            CollapsableProgressPanel(
                title = message.title,
                inProgress = message.inProgress,
                items = message.items,
                logMessages = message.logMessages,
            )
        }
        
        // ... 其他类型
    }
}
```

### 5.5.3 MessageInputText

```kotlin
@Composable
fun MessageInputText(
    enabled: Boolean,
    onSend: (String, List<Bitmap>) -> Unit,
) {
    var text by remember { mutableStateOf("") }
    var images by remember { mutableStateOf<List<Bitmap>>(emptyList()) }
    
    Row(
        modifier = Modifier
            .fillMaxWidth()
            .padding(8.dp),
        verticalAlignment = Alignment.CenterVertically,
    ) {
        // 文本输入框
        TextField(
            value = text,
            onValueChange = { text = it },
            modifier = Modifier.weight(1f),
            enabled = enabled,
            placeholder = { Text("输入消息...") },
            maxLines = 5,
        )
        
        // 图片选择按钮
        if (supportImage) {
            IconButton(onClick = { /* 选择图片 */ }) {
                Icon(Icons.Default.Image, "添加图片")
            }
        }
        
        // 发送按钮
        IconButton(
            onClick = {
                if (text.isNotBlank()) {
                    onSend(text, images)
                    text = ""
                    images = emptyList()
                }
            },
            enabled = enabled && text.isNotBlank(),
        ) {
            Icon(Icons.Default.Send, "发送")
        }
    }
}
```

---

## 5.6 导航

### 5.6.1 NavGraph

```kotlin
// ui/navigation/GalleryNavGraph.kt
@Composable
fun GalleryNavGraph(
    navController: NavHostController,
    startDestination: String = "home",
) {
    NavHost(
        navController = navController,
        startDestination = startDestination,
    ) {
        composable("home") {
            HomeScreen(
                onNavigateToChat = { model ->
                    navController.navigate("chat/${model.name}")
                }
            )
        }
        
        composable(
            route = "chat/{modelName}",
            arguments = listOf(navArgument("modelName") { type = NavType.StringType })
        ) { backStackEntry ->
            val modelName = backStackEntry.arguments?.getString("modelName")
            LlmChatScreen(modelName = modelName)
        }
        
        composable("settings") {
            SettingsScreen()
        }
        
        composable("skills") {
            SkillManagerScreen()
        }
    }
}
```

### 5.6.2 导航事件

```kotlin
// 从 ViewModel 触发导航
sealed class NavigationEvent {
    data class ToChat(val modelName: String) : NavigationEvent()
    object ToSettings : NavigationEvent()
    object Back : NavigationEvent()
}

// 在 Composable 中处理
LaunchedEffect(navigationEvents) {
    navigationEvents.collect { event ->
        when (event) {
            is NavigationEvent.ToChat -> navController.navigate("chat/${event.modelName}")
            is NavigationEvent.ToSettings -> navController.navigate("settings")
            is NavigationEvent.Back -> navController.popBackStack()
        }
    }
}
```

---

## 5.7 主题系统

### 5.7.1 颜色定义

```kotlin
// ui/theme/Color.kt
val Purple80 = Color(0xFFD0BCFF)
val PurpleGrey80 = Color(0xFFCCC2DC)
val Pink80 = Color(0xFFEFB8C8)

val Purple40 = Color(0xFF6650a4)
val PurpleGrey40 = Color(0xFF625b71)
val Pink40 = Color(0xFF7D5260)

// 深色模式配色
private val DarkColorScheme = darkColorScheme(
    primary = Purple80,
    secondary = PurpleGrey80,
    tertiary = Pink80,
)

// 浅色模式配色
private val LightColorScheme = lightColorScheme(
    primary = Purple40,
    secondary = PurpleGrey40,
    tertiary = Pink40,
)
```

### 5.7.2 主题应用

```kotlin
// ui/theme/Theme.kt
@Composable
fun GalleryTheme(
    darkTheme: Boolean = isSystemInDarkTheme(),
    content: @Composable () -> Unit,
) {
    val colorScheme = if (darkTheme) {
        DarkColorScheme
    } else {
        LightColorScheme
    }
    
    MaterialTheme(
        colorScheme = colorScheme,
        typography = Typography,
        content = content,
    )
}
```

---

## 5.8 性能优化

### 5.8.1 稳定类型

```kotlin
// 使用 @Stable 或 @Immutable 标记
@Immutable
data class ChatMessageText(
    val content: String,
    val side: ChatSide,
    // ...
) : ChatMessage(ChatMessageType.TEXT, side)
```

### 5.8.2 key 优化

```kotlin
LazyColumn {
    items(
        items = messages,
        key = { message -> message.id }  // 使用稳定的 key
    ) { message ->
        ChatMessageItem(message)
    }
}
```

### 5.8.3 衍生状态

```kotlin
// 避免重复计算
val sortedMessages by remember {
    derivedStateOf {
        messages.sortedBy { it.timestamp }
    }
}
```

---

## 5.9 小结

本章介绍了：

- Jetpack Compose 架构
- 状态管理模式（StateFlow）
- 消息类型与组件设计
- 导航系统
- 主题系统
- 性能优化技巧

**下一章**：我们将进入实战，学习如何修改项目扩展功能。

---

> 💡 **实践建议**：
> 1. 使用 Compose Preview 快速迭代 UI
> 2. 利用 Layout Inspector 调试布局
> 3. 关注重组性能，避免不必要的状态更新

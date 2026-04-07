# 第六章：实战：修改项目扩展功能

> 本章将通过实际案例，教你如何修改 Google AI Edge Gallery 项目，添加新功能。

---

## 6.1 添加新的 UI 页面

### 6.1.1 场景：添加一个"模型对比"页面

**步骤 1：创建 ViewModel**

```kotlin
// ui/modelcompare/ModelCompareViewModel.kt
@HiltViewModel
class ModelCompareViewModel @Inject constructor(
    private val modelRepository: ModelRepository,
) : ViewModel() {
    
    private val _uiState = MutableStateFlow(ModelCompareUiState())
    val uiState = _uiState.asStateFlow()
    
    fun selectModel(index: Int, model: Model) {
        _uiState.update { state ->
            val selectedModels = state.selectedModels.toMutableList()
            selectedModels[index] = model
            state.copy(selectedModels = selectedModels)
        }
    }
    
    fun runComparison(prompt: String) {
        viewModelScope.launch {
            _uiState.update { it.copy(isComparing = true) }
            
            // 同时运行两个模型
            val results = selectedModels.map { model ->
                async {
                    model to runInference(model, prompt)
                }
            }.awaitAll()
            
            _uiState.update { 
                it.copy(
                    isComparing = false,
                    results = results.toMap()
                )
            }
        }
    }
}

data class ModelCompareUiState(
    val selectedModels: List<Model?> = listOf(null, null),
    val isComparing: Boolean = false,
    val results: Map<Model, String> = emptyMap(),
)
```

**步骤 2：创建 Screen**

```kotlin
// ui/modelcompare/ModelCompareScreen.kt
@Composable
fun ModelCompareScreen(
    viewModel: ModelCompareViewModel = hiltViewModel(),
    onBack: () -> Unit,
) {
    val uiState by viewModel.uiState.collectAsState()
    
    Scaffold(
        topBar = {
            TopAppBar(
                title = { Text("模型对比") },
                navigationIcon = {
                    IconButton(onClick = onBack) {
                        Icon(Icons.Default.ArrowBack, "返回")
                    }
                }
            )
        }
    ) { padding ->
        Column(
            modifier = Modifier
                .fillMaxSize()
                .padding(padding)
        ) {
            // 模型选择区
            Row(
                modifier = Modifier.fillMaxWidth(),
                horizontalArrangement = Arrangement.SpaceEvenly,
            ) {
                ModelSelector(
                    model = uiState.selectedModels[0],
                    onSelect = { viewModel.selectModel(0, it) }
                )
                ModelSelector(
                    model = uiState.selectedModels[1],
                    onSelect = { viewModel.selectModel(1, it) }
                )
            }
            
            // 输入区
            var prompt by remember { mutableStateOf("") }
            TextField(
                value = prompt,
                onValueChange = { prompt = it },
                modifier = Modifier.fillMaxWidth(),
                placeholder = { Text("输入测试提示词") },
            )
            
            Button(
                onClick = { viewModel.runComparison(prompt) },
                enabled = uiState.selectedModels.all { it != null } && !uiState.isComparing,
            ) {
                Text("开始对比")
            }
            
            // 结果展示
            if (uiState.results.isNotEmpty()) {
                Row(modifier = Modifier.fillMaxWidth()) {
                    uiState.selectedModels.forEach { model ->
                        if (model != null) {
                            Card(modifier = Modifier.weight(1f)) {
                                Text(
                                    text = model.name,
                                    style = MaterialTheme.typography.titleMedium
                                )
                                Text(uiState.results[model] ?: "")
                            }
                        }
                    }
                }
            }
        }
    }
}
```

**步骤 3：添加导航**

```kotlin
// ui/navigation/GalleryNavGraph.kt
composable("model_compare") {
    ModelCompareScreen(
        onBack = { navController.popBackStack() }
    )
}

// 在主页添加入口
Button(onClick = { navController.navigate("model_compare") }) {
    Text("模型对比")
}
```

---

## 6.2 添加新模型支持

### 6.2.1 场景：添加 Llama 3.2 模型

**步骤 1：准备模型文件**

需要将原始模型转换为 LiteRT 格式：

```bash
# 使用 LiteRT 转换工具
litert-convert \
    --input-model llama-3.2-1b-instruct.pt \
    --output-model llama-3.2-1b-instruct.task \
    --quantization int4
```

**步骤 2：上传到 Hugging Face**

```bash
huggingface-cli upload litert-community/Llama-3.2-1B-Instruct \
    llama-3.2-1b-instruct.task
```

**步骤 3：更新 model_allowlist.json**

```json
{
  "models": [
    // ... 现有模型
    {
      "name": "Llama-3.2-1B-Instruct q4",
      "modelId": "litert-community/Llama-3.2-1B-Instruct",
      "modelFile": "llama-3.2-1b-instruct.task",
      "description": "Llama 3.2 1B Instruct quantized to 4-bit",
      "sizeInBytes": 800000000,
      "estimatedPeakMemoryInBytes": 1500000000,
      "version": "20250401",
      "defaultConfig": {
        "topK": 40,
        "topP": 0.9,
        "temperature": 0.7,
        "maxTokens": 2048,
        "accelerators": "gpu,cpu"
      },
      "taskTypes": ["llm_chat", "llm_prompt_lab"]
    }
  ]
}
```

---

## 6.3 添加新的 Function Calling 工具

### 6.3.1 场景：添加"设置闹钟"功能

**步骤 1：定义 ActionType**

```kotlin
// customtasks/mobileactions/Actions.kt
enum class ActionType {
    // ... 现有类型
    ACTION_SET_ALARM,
}

class SetAlarmAction(
    val hour: Int,
    val minute: Int,
    val message: String = "",
) : Action(
    type = ActionType.ACTION_SET_ALARM,
    icon = Icons.Outlined.Alarm,
    functionCallDetails = FunctionCallDetails(
        functionName = "setAlarm",
        parameters = listOf(
            Pair("hour", hour.toString()),
            Pair("minute", minute.toString()),
            Pair("message", message),
        )
    )
)
```

**步骤 2：定义 Tool**

```kotlin
// customtasks/mobileactions/MobileActionsTools.kt
class MobileActionsTools(
    val onFunctionCalled: (Action) -> Unit
) : Toolset {
    
    // ... 现有工具
    
    @Tool(description = "Set an alarm on the device.")
    fun setAlarm(
        @ToolParam(description = "Hour (0-23)") hour: Int,
        @ToolParam(description = "Minute (0-59)") minute: Int,
        @ToolParam(description = "Optional message for the alarm") message: String = "",
    ): Map<String, String> {
        onFunctionCalled(SetAlarmAction(hour, minute, message))
        return mapOf("result" to "success", "time" to "$hour:$minute")
    }
}
```

**步骤 3：实现 Action 逻辑**

```kotlin
// customtasks/mobileactions/MobileActionsViewModel.kt
fun performAction(action: Action, context: Context): String {
    return when (action) {
        // ... 现有 actions
        is SetAlarmAction -> handleSetAlarm(context, action)
        else -> ""
    }
}

private fun handleSetAlarm(context: Context, action: SetAlarmAction): String {
    val alarmManager = context.getSystemService(Context.ALARM_SERVICE) as AlarmManager
    
    val intent = Intent(context, AlarmReceiver::class.java).apply {
        putExtra("message", action.message)
    }
    
    val pendingIntent = PendingIntent.getBroadcast(
        context,
        0,
        intent,
        PendingIntent.FLAG_IMMUTABLE
    )
    
    val calendar = Calendar.getInstance().apply {
        set(Calendar.HOUR_OF_DAY, action.hour)
        set(Calendar.MINUTE, action.minute)
        set(Calendar.SECOND, 0)
    }
    
    alarmManager.setExact(
        AlarmManager.RTC_WAKEUP,
        calendar.timeInMillis,
        pendingIntent
    )
    
    return "闹钟已设置为 ${action.hour}:${String.format("%02d", action.minute)}"
}
```

---

## 6.4 添加新的 Skill

### 6.4.1 场景：添加"天气查询" Skill

**步骤 1：创建目录**

```
skills/built-in/weather/
├── SKILL.md
└── scripts/
    └── index.html
```

**步骤 2：编写 SKILL.md**

```markdown
---
name: weather
description: Get current weather for a location.
metadata:
  require-secret: true
  require-secret-description: Please enter your OpenWeatherMap API key.
---

# Weather Query

## Instructions

Call the `run_js` tool with the following parameters:
- skillName: "weather"
- scriptName: "index.html"
- data: A JSON string with the following fields:
  - city: Required. The city name (e.g., "Beijing", "New York")
  - unit: Optional. Temperature unit ("metric" for Celsius, "imperial" for Fahrenheit).
```

**步骤 3：编写 index.html**

```html
<!DOCTYPE html>
<html>
<head>
    <meta charset="UTF-8">
    <title>Weather Query</title>
</head>
<body>
    <script>
        window['ai_edge_gallery_get_result'] = async (data, secret) => {
            try {
                const jsonData = JSON.parse(data);
                const { city, unit = 'metric' } = jsonData;
                const apiKey = secret;
                
                const url = `https://api.openweathermap.org/data/2.5/weather?q=${encodeURIComponent(city)}&units=${unit}&appid=${apiKey}`;
                
                const response = await fetch(url);
                
                if (!response.ok) {
                    return JSON.stringify({
                        error: `Failed to get weather: ${response.status}`
                    });
                }
                
                const weatherData = await response.json();
                
                const result = {
                    city: weatherData.name,
                    temp: weatherData.main.temp,
                    description: weatherData.weather[0].description,
                };
                
                return JSON.stringify({
                    result: JSON.stringify(result)
                });
                
            } catch (e) {
                return JSON.stringify({
                    error: `Weather query failed: ${e.message}`
                });
            }
        };
    </script>
</body>
</html>
```

---

## 6.5 自定义推理参数

### 6.5.1 场景：添加"创意模式"预设

**步骤 1：添加配置**

```kotlin
// data/ConfigPresets.kt
object ConfigPresets {
    val PRESETS = mapOf(
        "balanced" to ModelConfig(
            temperature = 1.0f,
            topK = 64,
            topP = 0.95f,
        ),
        "creative" to ModelConfig(
            temperature = 1.5f,
            topK = 100,
            topP = 0.99f,
        ),
        "precise" to ModelConfig(
            temperature = 0.3f,
            topK = 20,
            topP = 0.9f,
        ),
    )
}
```

**步骤 2：添加 UI 选择器**

```kotlin
@Composable
fun PresetSelector(
    currentPreset: String,
    onPresetSelected: (String) -> Unit,
) {
    var expanded by remember { mutableStateOf(false) }
    
    Box {
        Button(onClick = { expanded = true }) {
            Text(currentPreset.replaceFirstChar { it.uppercase() })
            Icon(Icons.Default.ArrowDropDown, null)
        }
        
        DropdownMenu(
            expanded = expanded,
            onDismissRequest = { expanded = false },
        ) {
            ConfigPresets.PRESETS.keys.forEach { preset ->
                DropdownMenuItem(
                    text = { Text(preset.replaceFirstChar { it.uppercase() }) },
                    onClick = {
                        onPresetSelected(preset)
                        expanded = false
                    }
                )
            }
        }
    }
}
```

---

## 6.6 完整开发流程

```
1. 需求分析
   └── 确定功能范围

2. 架构设计
   ├── 数据模型设计
   ├── ViewModel 设计
   └── UI 组件设计

3. 编码实现
   ├── 数据层
   ├── 业务层
   └── UI 层

4. 测试验证
   ├── 单元测试
   ├── 集成测试
   └── 手动测试

5. 提交代码
   ├── 创建分支
   ├── 编写提交信息
   └── 创建 PR
```

---

## 6.7 小结

本章通过实战案例介绍了：

- 添加新 UI 页面的完整流程
- 添加新模型支持的方法
- 扩展 Function Calling 工具
- 开发自定义 Skill
- 自定义推理参数

**下一章**：我们将学习如何定位和修复项目 Issue。

---

> 💡 **最佳实践**：
> 1. 修改前先理解现有架构
> 2. 遵循项目代码风格
> 3. 添加必要的注释
> 4. 测试边界情况
> 5. 提交前运行 lint 检查

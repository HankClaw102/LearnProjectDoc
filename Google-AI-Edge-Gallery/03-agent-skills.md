# 第三章：Agent Skills 系统详解

> Agent Skills 是 Google AI Edge Gallery 最核心的设计亮点，本章将深入解析其设计理念与实现原理。

---

## 3.1 设计理念

### 3.1.1 为什么需要 Skill 系统？

端侧 LLM 的限制：
- 无法执行系统命令
- 无法访问外部 API
- 无法使用 Python 脚本

Skill 系统的解决方案：
- **模块化扩展** - 通过配置文件定义新能力
- **沙箱执行** - WebView 提供 JS 执行环境
- **安全隔离** - 敏感操作通过 Native 层处理

### 3.1.2 Skill 类型

| 类型 | 执行方式 | 用途 |
|------|----------|------|
| **Text-Only** | 无代码执行 | 人设/场景提示词 |
| **JS Skill** | WebView 沙箱 | API 调用、计算、数据处理 |
| **Native Skill** | Android Intent | 发邮件、发短信 |

---

## 3.2 SKILL.md 格式规范

### 3.2.1 基本结构

```markdown
---
name: skill-name
description: Skill 的简短描述
metadata:
  require-secret: true
  require-secret-description: 请输入 API Key
  homepage: https://github.com/xxx/skill
---

# Skill 标题

## Instructions

这里是给 LLM 的指令，告诉它如何使用这个 Skill...
```

### 3.2.2 元数据字段

| 字段 | 必需 | 说明 |
|------|------|------|
| `name` | ✅ | Skill 名称，用于 LLM 匹配 |
| `description` | ✅ | 简短描述，LLM 用于判断是否调用 |
| `require-secret` | ❌ | 是否需要 API Key |
| `require-secret-description` | ❌ | API Key 输入提示 |
| `homepage` | ❌ | Skill 主页链接 |

### 3.2.3 实例：query-wikipedia

```markdown
---
name: query-wikipedia
description: Query summary from Wikipedia for a given topic.
---

# Query Wiki

## Instructions

Call the `run_js` tool using `index.html` and a JSON string for `data` with the following fields:
- **topic**: Required. Extract ONLY the primary entity, person, or event.
- **lang**: Required. The 2-letter language code.

**Constraints:**
- Provide a concise summary (1-3 complete sentences).
- For recurring events, query the specific iteration (e.g., "2026 Oscars").
```

---

## 3.3 Text-Only Skill

### 3.3.1 定义

最简单的 Skill 类型，只提供提示词，不执行代码。

### 3.3.2 示例：kitchen-adventure

**目录结构**：

```
skills/built-in/kitchen-adventure/
└── SKILL.md
```

**SKILL.md**：

```markdown
---
name: kitchen-adventure
description: Act as a dungeon master for a text-based adventure set in a world where everyone is a sentient kitchen appliance.
---

# Kitchen Adventure

## Persona

You are a creative dungeon master running a text-based adventure game...

## Instructions

When the user starts the adventure:
1. Set the scene in a whimsical kitchen world
2. Introduce the player as a sentient appliance
3. Offer 2-3 choices for action
...
```

### 3.3.3 执行流程

```
用户: "开始厨房冒险"
    │
    ▼
LLM 匹配到 kitchen-adventure skill
    │
    ▼
loadSkill() 被调用
    │
    ▼
返回 SKILL.md 内容作为系统提示
    │
    ▼
LLM 根据提示词生成回复
```

---

## 3.4 JS Skill

### 3.4.1 目录结构

```
my-js-skill/
├── SKILL.md           # 元数据 + 指令
├── scripts/
│   └── index.html     # JS 执行入口（必需）
└── assets/
    └── webview.html   # 可选：交互式 UI
```

### 3.4.2 index.html 模板

```html
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <title>My Skill</title>
</head>
<body>
    <script>
        // 必须暴露这个函数
        window['ai_edge_gallery_get_result'] = async (data, secret) => {
            try {
                const jsonData = JSON.parse(data);
                
                // 你的业务逻辑
                const result = await yourLogic(jsonData);
                
                // 返回结果
                return JSON.stringify({
                    result: result
                });
            } catch (e) {
                return JSON.stringify({
                    error: e.message
                });
            }
        };

        async function yourLogic(jsonData) {
            // 实现你的逻辑...
            return "处理结果";
        }
    </script>
</body>
</html>
```

### 3.4.3 返回格式

**成功返回**：

```json
{
    "result": "文本结果"
}
```

**返回图片**：

```json
{
    "result": "图片已生成",
    "image": {
        "base64": "iVBORw0KGgo..."
    }
}
```

**返回 Webview**：

```json
{
    "result": "交互界面已加载",
    "webview": {
        "url": "webview.html",
        "aspectRatio": 1.0
    }
}
```

**错误返回**：

```json
{
    "error": "错误信息"
}
```

### 3.4.4 实例：query-wikipedia

**scripts/index.html**：

```html
<script>
window['ai_edge_gallery_get_result'] = async (data) => {
    try {
        const jsonData = JSON.parse(data);
        const { topic, lang } = jsonData;
        
        // 调用 Wikipedia API
        const baseUrl = `https://${lang}.wikipedia.org/w/api.php`;
        const params = new URLSearchParams({
            action: "query",
            format: "json",
            generator: "search",
            gsrsearch: topic,
            gsrlimit: "1",
            prop: "extracts",
            explaintext: "1",
            exintro: "1",
            origin: "*",
        });
        
        const response = await fetch(`${baseUrl}?${params}`);
        const searchData = await response.json();
        
        // 提取结果
        const pages = searchData.query.pages;
        const firstPage = Object.values(pages)[0];
        
        return JSON.stringify({
            title: firstPage.title,
            result: firstPage.extract
        });
    } catch (e) {
        return JSON.stringify({
            error: `Failed to query Wikipedia: ${e.message}`
        });
    }
};
</script>
```

### 3.4.5 Secret 注入

当 Skill 需要 API Key 时：

**SKILL.md**：

```markdown
---
name: mood-music
description: Suggest music based on user's mood.
metadata:
  require-secret: true
  require-secret-description: Please enter your Spotify API token.
---
```

**index.html**：

```html
<script>
window['ai_edge_gallery_get_result'] = async (data, secret) => {
    // secret 是用户输入的 API Key
    const response = await fetch("https://api.spotify.com/v1/...", {
        headers: {
            "Authorization": `Bearer ${secret}`
        }
    });
    // ...
};
</script>
```

---

## 3.5 Native Skill

### 3.5.1 定义

直接调用 Android/iOS 系统能力，不经过 WebView。

### 3.5.2 示例：send-email

**SKILL.md**：

```markdown
---
name: send-email
description: Send an email.
---

# Send email

## Instructions

Call the `run_intent` tool with the following exact parameters:

- intent: send_email
- parameters: A JSON string with the following fields:
  - extra_email: the email address to send to. String.
  - extra_subject: the subject of the email. String.
  - extra_text: the body of the email. String.
```

### 3.5.3 Intent 处理

```kotlin
// IntentHandler.kt
object IntentHandler {
    fun handleAction(context: Context, intent: String, parameters: String): Boolean {
        val params = JSONObject(parameters)
        
        return when (intent) {
            "send_email" -> {
                val emailIntent = Intent(Intent.ACTION_SEND).apply {
                    type = "message/rfc822"
                    putExtra(Intent.EXTRA_EMAIL, arrayOf(params.getString("extra_email")))
                    putExtra(Intent.EXTRA_SUBJECT, params.getString("extra_subject"))
                    putExtra(Intent.EXTRA_TEXT, params.getString("extra_text"))
                }
                context.startActivity(emailIntent)
                true
            }
            "send_sms" -> {
                // ...
            }
            else -> false
        }
    }
}
```

---

## 3.6 AgentTools 实现

### 3.6.1 工具定义

```kotlin
// AgentTools.kt
class AgentTools : ToolSet {
    
    @Tool(description = "Loads a skill.")
    fun loadSkill(
        @ToolParam(description = "The name of the skill to load.") 
        skillName: String
    ): Map<String, String> {
        val skill = skillManagerViewModel.getSelectedSkills()
            .find { it.name == skillName.trim() }
        
        return mapOf(
            "skill_name" to skillName,
            "skill_instructions" to skill?.instructions ?: "Skill not found"
        )
    }

    @Tool(description = "Runs JS script")
    fun runJs(
        @ToolParam(description = "The name of skill") 
        skillName: String,
        @ToolParam(description = "The script name to run") 
        scriptName: String,
        @ToolParam(description = "The data to pass to the script") 
        data: String,
    ): Map<String, Any> {
        // 1. 查找 Skill
        val skill = skills.find { it.name == skillName }
        
        // 2. 检查 Secret
        var secret = ""
        if (skill.requireSecret) {
            secret = skillManagerViewModel.dataStoreRepository.readSecret(key)
            if (secret.isNullOrEmpty()) {
                // 弹出对话框让用户输入
                secret = askUserForSecret()
            }
        }
        
        // 3. 获取脚本 URL
        val url = skillManagerViewModel.getJsSkillUrl(skillName, scriptName)
        
        // 4. 在 WebView 中执行
        val result = executeJsInWebView(url, data, secret)
        
        // 5. 返回结果
        return mapOf("result" to result)
    }

    @Tool(description = "Run an Android intent.")
    fun runIntent(
        @ToolParam(description = "The intent to run.") 
        intent: String,
        @ToolParam(description = "A JSON string of parameters") 
        parameters: String,
    ): Map<String, String> {
        val success = IntentHandler.handleAction(context, intent, parameters)
        return mapOf(
            "action" to intent,
            "result" to if (success) "succeeded" else "failed"
        )
    }
}
```

### 3.6.2 工具注册

```kotlin
// LlmModelHelper 初始化时
fun initialize(
    ...
    tools: List<ToolProvider> = listOf(),
) {
    // 将 AgentTools 传递给 LiteRT
    val engine = LlmEngine(modelPath)
    engine.setTools(tools)
}
```

---

## 3.7 完整调用链

```
用户: "帮我查一下牛顿的维基百科"
    │
    ▼
┌─────────────────────────────────┐
│ LLM 推理                        │
│ System Prompt 包含所有 Skill 的 │
│ name + description              │
└─────────────┬───────────────────┘
              │
              │ 识别到 query-wikipedia skill
              ▼
┌─────────────────────────────────┐
│ loadSkill("query-wikipedia")    │
│ 返回 SKILL.md 的 instructions   │
└─────────────┬───────────────────┘
              │
              │ LLM 根据 instructions 构造参数
              ▼
┌─────────────────────────────────┐
│ runJs(                          │
│   skillName: "query-wikipedia", │
│   scriptName: "index.html",     │
│   data: '{"topic":"牛顿","lang":"zh"}' │
│ )                               │
└─────────────┬───────────────────┘
              │
              ▼
┌─────────────────────────────────┐
│ WebView 执行 index.html         │
│ 调用 Wikipedia API              │
└─────────────┬───────────────────┘
              │
              │ 返回结果
              ▼
┌─────────────────────────────────┐
│ LLM 继续推理                    │
│ 基于返回结果生成回复            │
└─────────────────────────────────┘
```

---

## 3.8 小结

本章详细介绍了：

- Skill 系统的设计理念
- SKILL.md 格式规范
- Text-Only / JS / Native 三种 Skill 类型
- AgentTools 的工具定义
- 完整的调用链路

**下一章**：我们将深入模型推理层，了解 LiteRT 如何工作。

---

> 💡 **实践建议**：
> 1. 先尝试修改现有的 built-in Skill
> 2. 理解 JS Skill 的执行环境
> 3. 尝试开发一个简单的 Skill

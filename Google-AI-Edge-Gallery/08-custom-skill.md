# 第八章：进阶：开发自定义 Skill

> 本章将深入讲解如何开发复杂的自定义 Skill，包括高级功能和最佳实践。

---

## 8.1 Skill 开发进阶

### 8.1.1 Skill 能力矩阵

| 能力 | 实现方式 | 复杂度 |
|------|----------|--------|
| 文本处理 | JS 基础 API | ⭐ |
| API 调用 | fetch() | ⭐⭐ |
| 图片处理 | Canvas API | ⭐⭐⭐ |
| 本地存储 | localStorage | ⭐⭐ |
| WebView UI | HTML/CSS/JS | ⭐⭐⭐ |
| 多步骤工作流 | 状态机 | ⭐⭐⭐⭐ |

---

## 8.2 案例 1：多步骤工作流 Skill

### 8.2.1 需求

创建一个"旅行规划" Skill：
1. 获取目的地信息
2. 查询天气
3. 搜索景点
4. 生成行程

### 8.2.2 SKILL.md

```markdown
---
name: travel-planner
description: Plan a trip with weather, attractions, and itinerary.
metadata:
  require-secret: true
  require-secret-description: Please enter your API key.
  homepage: https://github.com/yourname/travel-planner-skill
---

# Travel Planner

## Instructions

You are a travel planning assistant. When the user asks for trip planning:

1. Extract the destination and travel dates
2. Call the `run_js` tool to get weather forecast
3. Call the `run_js` tool to search for attractions
4. Generate a day-by-day itinerary

Parameters for each call:
- For weather: scriptName="weather.js", data with {city, date}
- For attractions: scriptName="attractions.js", data with {city, interests}
```

### 8.2.3 目录结构

```
travel-planner/
├── SKILL.md
├── scripts/
│   ├── index.html        # 主入口
│   ├── weather.js        # 天气查询
│   └── attractions.js    # 景点搜索
└── assets/
    └── itinerary.html    # 行程展示 UI
```

### 8.2.4 weather.js

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
        const { city, date } = JSON.parse(data);
        
        const response = await fetch(
            `https://api.openweathermap.org/data/2.5/forecast?q=${encodeURIComponent(city)}&appid=${secret}&units=metric`
        );
        
        if (!response.ok) {
            return JSON.stringify({ error: `Weather API error: ${response.status}` });
        }
        
        const weatherData = await response.json();
        
        const result = {
            city: weatherData.city.name,
            forecasts: weatherData.list.slice(0, 5).map(item => ({
                time: new Date(item.dt * 1000).toLocaleTimeString(),
                temp: item.main.temp,
                description: item.weather[0].description,
            }))
        };
        
        return JSON.stringify({ result: JSON.stringify(result) });
        
    } catch (e) {
        return JSON.stringify({ error: `Weather query failed: ${e.message}` });
    }
};
</script>
</body>
</html>
```

---

## 8.3 案例 2：带交互式 WebView 的 Skill

### 8.3.1 需求

创建一个"心情追踪" Skill，展示交互式心情图表。

### 8.3.2 index.html

```html
<!DOCTYPE html>
<html>
<head>
    <meta charset="UTF-8">
    <title>Mood Tracker</title>
</head>
<body>
<script>
const STORAGE_KEY = 'mood_tracker_data';

function getMoodData() {
    const data = localStorage.getItem(STORAGE_KEY);
    return data ? JSON.parse(data) : [];
}

function saveMoodData(data) {
    localStorage.setItem(STORAGE_KEY, JSON.stringify(data));
}

window['ai_edge_gallery_get_result'] = async (data) => {
    try {
        const { action, mood, note, days = 7 } = JSON.parse(data);
        
        if (action === 'record') {
            const moodData = getMoodData();
            moodData.push({
                date: new Date().toISOString(),
                mood: mood,
                note: note || '',
            });
            saveMoodData(moodData);
            
            return JSON.stringify({
                result: `Mood recorded: ${mood}/10`,
                webview: {
                    url: 'chart.html',
                    aspectRatio: 1.5,
                }
            });
        }
        
        if (action === 'history') {
            const allData = getMoodData();
            const cutoff = new Date();
            cutoff.setDate(cutoff.getDate() - days);
            
            const recentData = allData.filter(item => 
                new Date(item.date) >= cutoff
            );
            
            return JSON.stringify({
                result: JSON.stringify(recentData),
                webview: {
                    url: 'chart.html',
                    aspectRatio: 1.5,
                }
            });
        }
        
        return JSON.stringify({ error: 'Invalid action' });
        
    } catch (e) {
        return JSON.stringify({ error: `Mood tracker error: ${e.message}` });
    }
};
</script>
</body>
</html>
```

---

## 8.4 案例 3：生成图片的 Skill

### 8.4.1 需求

创建一个生成二维码的 Skill。

### 8.4.2 实现代码

```html
<!DOCTYPE html>
<html>
<head>
    <meta charset="UTF-8">
    <title>QR Code Generator</title>
</head>
<body>
<script src="https://cdn.jsdelivr.net/npm/qrcode@1.5.3/build/qrcode.min.js"></script>
<script>
window['ai_edge_gallery_get_result'] = async (data) => {
    try {
        const { text, size = 256 } = JSON.parse(data);
        
        const canvas = document.createElement('canvas');
        await QRCode.toCanvas(canvas, text, {
            width: size,
            margin: 2,
        });
        
        const base64 = canvas.toDataURL('image/png');
        
        return JSON.stringify({
            result: `QR code generated for: ${text}`,
            image: {
                base64: base64.replace('data:image/png;base64,', '')
            }
        });
        
    } catch (e) {
        return JSON.stringify({ error: `QR code generation failed: ${e.message}` });
    }
};
</script>
</body>
</html>
```

---

## 8.5 Skill 发布与分享

### 8.5.1 托管 Skill

**方案 1：GitHub Pages**

```bash
# 创建仓库
mkdir my-skill && cd my-skill
git init

# 添加 .nojekyll 文件
touch .nojekyll

# 添加 Skill 文件
mkdir skills
cp -r ../travel-planner skills/

# 推送
git add .
git commit -m "Add travel-planner skill"
git push origin main

# 在 GitHub 仓库设置中启用 GitHub Pages
# Skill URL: https://username.github.io/my-skill/skills/travel-planner
```

**方案 2：Cloudflare Pages**

```bash
# 安装 Wrangler
npm install -g wrangler

# 部署
wrangler pages deploy ./skills
```

### 8.5.2 分享到社区

在 GitHub Discussions 的 Skills 分类中发帖：

```markdown
## Skill Name
Travel Planner

## Description
Plan a trip with weather forecasts, attractions, and day-by-day itinerary.

## Features
- 🌤️ Weather forecast integration
- 🏛️ Attractions search
- 📅 Day-by-day itinerary generation

## Installation
1. Open AI Edge Gallery
2. Go to Agent Skills → Skill Manager
3. Click (+) → Load skill from URL
4. Enter: https://username.github.io/skills/travel-planner
```

---

## 8.6 最佳实践

### 8.6.1 错误处理

```javascript
window['ai_edge_gallery_get_result'] = async (data, secret) => {
    try {
        const jsonData = JSON.parse(data);
        
        // 参数验证
        if (!jsonData.required_param) {
            return JSON.stringify({
                error: "Missing required parameter: required_param"
            });
        }
        
        // API 调用
        const response = await fetch(url);
        
        if (!response.ok) {
            return JSON.stringify({
                error: `API error: ${response.status}`
            });
        }
        
        const result = await response.json();
        return JSON.stringify({ result: JSON.stringify(result) });
        
    } catch (e) {
        return JSON.stringify({
            error: `Skill execution failed: ${e.message}`
        });
    }
};
```

### 8.6.2 性能优化

```javascript
// 使用缓存
const cache = new Map();

async function fetchWithCache(url, maxAge = 3600000) {
    const cached = cache.get(url);
    if (cached && Date.now() - cached.time < maxAge) {
        return cached.data;
    }
    
    const response = await fetch(url);
    const data = await response.json();
    
    cache.set(url, { data, time: Date.now() });
    return data;
}

// 控制结果大小
function truncateResult(result, maxChars = 5000) {
    if (result.length > maxChars) {
        return result.substring(0, maxChars) + "\n\n... [truncated]";
    }
    return result;
}
```

### 8.6.3 安全考虑

```javascript
// 不要在日志中暴露 secret
console.log('API call with key:', secret.substring(0, 4) + '***');

// 验证输入
function sanitizeInput(input) {
    return input.replace(/[<>'"]/g, '');
}

// 限制 API 调用
const MAX_CALLS_PER_MINUTE = 10;
let callCount = 0;
let lastReset = Date.now();

function checkRateLimit() {
    if (Date.now() - lastReset > 60000) {
        callCount = 0;
        lastReset = Date.now();
    }
    
    if (callCount >= MAX_CALLS_PER_MINUTE) {
        throw new Error('Rate limit exceeded');
    }
    
    callCount++;
}
```

---

## 8.7 调试技巧

### 8.7.1 在应用中调试

```javascript
// 在脚本中添加调试日志
console.log('Input data:', data);
console.log('Processing...');
console.log('API response:', response);

// 在应用的调试面板中查看日志
```

### 8.7.2 本地测试

```html
<!-- 在浏览器中直接打开 index.html 进行测试 -->
<script>
// 添加测试入口
if (typeof window === 'object' && !window.ai_edge_gallery_get_result) {
    window.test = async () => {
        const result = await window.ai_edge_gallery_get_result(
            JSON.stringify({ test: "data" }),
            "test_secret"
        );
        console.log(JSON.parse(result));
    };
    
    window.test();
}
</script>
```

---

## 8.8 小结

本章深入介绍了：

- 多步骤工作流 Skill 开发
- 交互式 WebView Skill
- 图片生成 Skill
- Skill 发布与分享
- 最佳实践与安全考虑
- 调试技巧

---

> 🎉 **恭喜！** 你已完成 Google AI Edge Gallery 项目学习指南的全部内容。现在你可以：
> - 理解项目整体架构
> - 修改和扩展项目功能
> - 修复项目 Issue
> - 开发自定义 Skill
> - 贡献代码到开源社区

---

> 💡 **持续学习**：
> - 关注 [GitHub Discussions](https://github.com/google-ai-edge/gallery/discussions)
> - 查看 [Project Wiki](https://github.com/google-ai-edge/gallery/wiki)
> - 阅读 [Google AI Edge 文档](https://ai.google.dev/edge)

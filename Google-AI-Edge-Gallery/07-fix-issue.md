# 第七章：实战：修复 Issue 完整流程

> 本章将通过一个实际案例，带你完整体验从 Issue 定位到 PR 提交的全流程。

---

## 7.1 Issue 发现与分析

### 7.1.1 获取 Issue 列表

```bash
# 使用 GitHub CLI
gh issue list --repo google-ai-edge/gallery --state open

# 或访问
https://github.com/google-ai-edge/gallery/issues
```

### 7.1.2 选择合适的 Issue

**适合新手的 Issue 特征**：
- 标签包含 `good first issue`
- 标签包含 `bug`
- 描述清晰，有复现步骤
- 不涉及核心架构改动

### 7.1.3 案例分析

**示例 Issue #123**：
```
Title: Skill loading fails when skill name contains spaces

Description:
When I try to load a skill with spaces in the name (e.g., "kitchen adventure"),
the skill fails to load.

Steps to reproduce:
1. Create a skill with spaces in the name
2. Try to load the skill
3. Observe the error

Expected: Skill loads successfully
Actual: Error "Skill not found"
```

---

## 7.2 环境准备

### 7.2.1 Fork 并克隆仓库

```bash
# 1. Fork 仓库（在 GitHub 网页操作）

# 2. 克隆你的 fork
git clone https://github.com/YOUR_USERNAME/gallery.git
cd gallery

# 3. 添加上游仓库
git remote add upstream https://github.com/google-ai-edge/gallery.git

# 4. 创建功能分支
git checkout -b fix-skill-name-spaces
```

### 7.2.2 同步最新代码

```bash
# 获取上游最新代码
git fetch upstream

# 合并到本地
git merge upstream/main
```

---

## 7.3 问题定位

### 7.3.1 搜索相关代码

```bash
# 搜索 skill 加载相关代码
grep -r "loadSkill\|Skill not found" Android/src --include="*.kt"
```

### 7.3.2 分析代码逻辑

```kotlin
// 找到问题所在：AgentTools.kt
@Tool(description = "Loads a skill.")
fun loadSkill(
    @ToolParam(description = "The name of the skill to load.") 
    skillName: String
): Map<String, String> {
    val skill = skillManagerViewModel.getSelectedSkills()
        .find { it.name == skillName.trim() }  // 问题：只 trim，未处理空格
        // ...
}
```

**根本原因**：Skill 名称匹配时，没有考虑 name 可能包含空格的情况。

### 7.3.3 进一步调查

```kotlin
// 查看 Skill 定义
// skills/built-in/kitchen-adventure/SKILL.md
---
name: kitchen-adventure  // 使用 kebab-case
description: ...
---

// 但用户可能误写成 "kitchen adventure" (有空格)
```

**发现**：问题出在 skill 名称规范化处理不一致。

---

## 7.4 编写修复代码

### 7.4.1 修复方案

```kotlin
// AgentTools.kt
@Tool(description = "Loads a skill.")
fun loadSkill(
    @ToolParam(description = "The name of the skill to load.") 
    skillName: String
): Map<String, String> {
    // 规范化 skill 名称：去除首尾空格，将空格替换为连字符
    val normalizedSkillName = skillName
        .trim()
        .lowercase()
        .replace(Regex("\\s+"), "-")
    
    val skill = skillManagerViewModel.getSelectedSkills()
        .find { 
            it.name.equals(normalizedSkillName, ignoreCase = true) 
        }
    
    if (skill != null) {
        _actionChannel.send(
            SkillProgressAgentAction(
                label = "Loading skill \"$skillName\"",
                inProgress = true,
                addItemTitle = "Load \"${skill.name}\"",
                addItemDescription = "Description: ${skill.description}",
                customData = skill,
            )
        )
        return mapOf(
            "skill_name" to skillName,
            "skill_instructions" to "---\nname: ${skill.name}\ndescription: ${skill.description}\n---\n\n${skill.instructions}"
        )
    } else {
        return mapOf(
            "skill_name" to skillName,
            "skill_instructions" to "Skill not found"
        )
    }
}
```

### 7.4.2 添加单元测试

```kotlin
// test/AgentToolsTest.kt
class AgentToolsTest {
    
    @Test
    fun `loadSkill normalizes skill name with spaces`() {
        // Given
        val skill = Skill(
            name = "kitchen-adventure",
            description = "Test skill",
            instructions = "Test instructions"
        )
        skillManagerViewModel.addSkill(skill)
        
        // When
        val result = agentTools.loadSkill("kitchen adventure")
        
        // Then
        assertTrue(result["skill_instructions"]?.contains("kitchen-adventure") == true)
    }
}
```

---

## 7.5 本地测试

### 7.5.1 编译运行

```bash
cd Android/src
./gradlew installDebug
```

### 7.5.2 手动测试步骤

1. 打开应用
2. 进入 Agent Skills 功能
3. 尝试输入 "kitchen adventure"（带空格）
4. 验证 skill 正确加载

### 7.5.3 运行测试

```bash
./gradlew test
```

---

## 7.6 提交 PR

### 7.6.1 提交代码

```bash
# 添加修改的文件
git add Android/src/app/src/main/java/com/google/ai/edge/gallery/customtasks/agentchat/AgentTools.kt
git add Android/src/app/src/test/java/com/google/ai/edge/gallery/customtasks/agentchat/AgentToolsTest.kt

# 提交
git commit -m "fix: normalize skill name to handle spaces

- Trim and convert spaces to hyphens when matching skill names
- Add unit tests for skill name normalization

Fixes #123"
```

### 7.6.2 推送到 Fork

```bash
git push origin fix-skill-name-spaces
```

### 7.6.3 创建 PR

```bash
# 使用 GitHub CLI
gh pr create --repo google-ai-edge/gallery \
    --title "fix: normalize skill name to handle spaces" \
    --body "## Changes
- Normalize skill name by trimming and converting spaces to hyphens
- Add unit tests

## Testing
- Manually tested with skill names containing spaces
- All unit tests pass

Fixes #123"
```

---

## 7.7 PR 模板

```markdown
## Description
Brief description of what this PR does.

## Type of Change
- [x] Bug fix (non-breaking change which fixes an issue)
- [ ] New feature (non-breaking change which adds functionality)
- [ ] Breaking change (fix or feature that would cause existing functionality to not work as expected)

## How Has This Been Tested?
- [ ] Unit tests
- [ ] Integration tests
- [x] Manual testing

## Checklist:
- [x] My code follows the style guidelines of this project
- [x] I have performed a self-review of my own code
- [x] I have commented my code, particularly in hard-to-understand areas
- [x] My changes generate no new warnings
- [x] I have added tests that prove my fix is effective
```

---

## 7.8 调试技巧

### 7.8.1 Logcat 过滤

```bash
# 过滤特定标签
adb logcat -s AGAgentTools:D LlmModelHelper:D

# 过滤特定关键词
adb logcat | grep -E "skill|error"
```

### 7.8.2 断点调试

1. 在 Android Studio 中设置断点
2. Run → Debug 'app'
3. 触发相关操作
4. 查看变量值

---

## 7.9 小结

本章通过完整案例介绍了：

- Issue 定位与分析方法
- 本地开发环境搭建
- 问题根因分析
- 代码修复与测试
- PR 提交流程

**下一章**：我们将学习如何开发自定义 Skill 并发布分享。

---

> 💡 **注意事项**：
> 1. 提交前确保代码格式符合项目规范
> 2. 编写清晰的 commit message
> 3. PR 描述要详细说明改动原因
> 4. 及时响应审查意见

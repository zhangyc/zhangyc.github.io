---
title: Claude Code Skill 开发实战：给 AI 注入领域知识
date: 2026-03-12
tags:
  - Claude Code
  - Skill
  - AI
  - 工具开发
categories:
  - AI 工程化
---

## Skill 是什么？

如果说 MCP 是给 AI 装上「手」（让它能操作外部系统），那 **Skill** 就是给 AI 装上「脑子」——向它注入特定领域的知识、规范和编码风格。

Claude Code Skill 是一段结构化的 Markdown 文件，包含：
- **触发条件**：什么时候自动激活
- **领域知识**：这个技术栈的核心概念
- **代码规范**：必须遵守的最佳实践
- **示例代码**：告诉 AI 该长什么样

Claude Code 在处理用户请求时，会根据上下文自动匹配并加载相关 Skill，就像给 AI 临时插入了一本「操作手册」。

---

## Skill 与 MCP 的区别

| | MCP | Skill |
|--|-----|-------|
| **作用** | 扩展 AI 的「能力」（可以做什么） | 扩展 AI 的「知识」（怎么做得更好） |
| **形式** | 运行中的进程，JSON-RPC 通信 | 静态 Markdown 文件 |
| **触发** | AI 主动调用 | 根据描述自动匹配 |
| **适合** | 查数据库、调 API、读文件 | 框架规范、团队编码标准、私有库用法 |

两者互补，不互斥。一个完整的 AI 开发工作流往往同时用到两者。

---

## Skill 的文件结构

Skill 存放在 `~/.claude/skills/<skill-name>/SKILL.md`，格式如下：

```markdown
---
name: skill-name
description: 一句话描述，AI 用这段话决定是否激活本 Skill
argument-hint: [可选，提示用户传什么参数]
---

# 正文：领域知识、规范、示例代码
...
```

**Front matter** 是关键，尤其是 `description`——它就是 Skill 的「触发词」，Claude 在理解用户意图时会对比所有 Skill 的 description，选择最匹配的激活。

---

## 实战：为私有库写一个 Skill

我给自己开发的 Flutter 状态管理库 `aegis_honeycomb` 写了一个 Skill，这样每次用 Claude Code 写相关代码时，它都能自动按照库的规范输出，而不是猜测或用通用写法。

文件路径：`~/.claude/skills/honeycomb/SKILL.md`

### Front matter

```yaml
---
name: honeycomb
description: 使用 aegis_honeycomb 状态管理库编写 Flutter 代码。当用户提到 Honeycomb、StateRef、Computed、Effect、HoneycombScope 或 HoneycombConsumer 时自动使用。
argument-hint: [描述你要创建的状态或功能]
---
```

`description` 里列出了所有关键词（`StateRef`、`Computed`、`Effect`...），只要对话中出现这些词，Skill 就会被激活。

### 正文结构

正文按照「安装 → 核心概念 → 集成 → 最佳实践 → 测试」组织，每个概念都配对比鲜明的代码示例：

```markdown
### StateRef — 可读写状态

```dart
// 定义（全局）
final counterState = StateRef(0);

// 读取
final count = container.read(counterState);

// 函数式更新（1.1.1+）
container.update(counterState, (value) => value + 1);
```

### Effect — 一次性事件（不存储历史）

```dart
// 定义
final toastEffect = Effect<String>();

// 触发
container.emit(toastEffect, '保存成功');
```
```

### 最后的「用户请求处理」段

Skill 正文末尾通常有一段指令，告诉 AI 拿到用户参数后该做什么：

```markdown
## 用户请求处理

用户提供的参数：$ARGUMENTS

根据用户描述，生成符合以上规范的 Honeycomb 代码。包括：
- StateRef / Computed / Effect 定义
- Service 层实现（如需要）
- HoneycombConsumer UI 代码（如需要）
- 对应的测试代码（如需要）
```

`$ARGUMENTS` 是 Claude Code 注入的占位符，等于用户在 `/honeycomb` 后面输入的内容。

---

## 安装 Skill

### 方式一：手动创建

```bash
mkdir -p ~/.claude/skills/my-skill
cat > ~/.claude/skills/my-skill/SKILL.md << 'EOF'
---
name: my-skill
description: 触发条件描述
---

# 正文内容
...
EOF
```

### 方式二：从注册表安装

Claude Code 有一个官方 Skill 注册表，使用内置的 `/find-skills` 命令搜索：

```
/find-skills flutter 状态管理
```

Claude 会帮你找到匹配的 Skill 并引导安装。

---

## 如何触发 Skill

### 自动触发

Claude Code 在理解每条用户消息时，都会扫描所有已安装 Skill 的 `description`，自动激活最匹配的。只要 description 写得准确，大部分情况下不需要手动操作。

### 手动触发（slash command）

`/skill-name` 是 Skill 的快捷调用方式：

```
/honeycomb 创建一个购物车状态，包含商品列表和总价计算
```

等价于：把 Skill 内容 + 用户参数一起交给 Claude 处理。

---

## 写好 Skill 的关键

### 1. description 要精准

description 决定了 Skill 的激活准确率。要同时覆盖：
- **场景词**：用户说"我想写状态管理"
- **技术词**：用户提到库名、类名、概念名

```yaml
# 不好：太宽泛，容易误触发
description: "管理 Flutter 应用状态"

# 好：明确场景 + 关键词
description: "使用 aegis_honeycomb 库编写 Flutter 代码。当用户提到 Honeycomb、StateRef、Computed 时自动使用。"
```

### 2. 示例代码胜过文字描述

AI 学的就是代码模式。与其用文字解释"StateRef 是可读写状态"，不如直接给一个完整的使用示例。示例越完整，输出的代码越准。

### 3. 把「不该怎么做」也写进去

明确的禁止项往往比规范项更有效：

```markdown
## 禁止事项
- 不要在 Widget 内部定义 StateRef（会丢失全局性）
- 不要用 StateRef 模拟一次性事件（用 Effect）
- 不要在 build 方法外部调用 ref.watch（会导致监听失效）
```

### 4. 控制篇幅

Skill 会被完整注入到 Claude 的上下文。太长会占用 token，稀释核心信息。500-1000 行是比较合适的范围，关键概念每个配一个示例即可。

---

## 团队共享 Skill

Skill 放在 `~/.claude/skills/` 是个人级别的，如果想团队共享，有两个方案：

**方案 A：项目级 Skill**

把 `SKILL.md` 放到项目仓库的 `.claude/skills/` 目录，Claude Code 会自动识别项目级 Skill，随 Git 一起版本管理。

**方案 B：发布到注册表**

如果你的 Skill 有通用价值，可以提交 PR 到 Claude Code 官方 Skill 注册表，让更多人通过 `/find-skills` 发现和安装。

---

## MCP + Skill：组合拳

MCP 和 Skill 配合使用才能发挥最大价值。

以我目前的工作流为例：

```
需求文档（PRD）
    │
    ├── Figma MCP ──→ 读取设计稿 ──→ 生成 UI 代码
    │                                    ↑
    │                              Flutter Skill（规范约束）
    │
    └── 后端 Skill ──→ 生成 SQL + proto + Go 骨架
                            ↑
                       数据库 MCP（可选，验证 schema）
```

- **MCP** 负责连通外部系统（Figma、数据库、Git...）
- **Skill** 负责约束输出质量（符合团队规范、使用指定框架）

两者的边界很清晰：**MCP = 能力扩展，Skill = 知识注入**。

---

## 小结

Skill 开发的本质是**把你的领域知识结构化地传递给 AI**。花一两小时把团队规范、私有库用法、最佳实践整理成 Skill，往后每次让 AI 写这块代码，它都能第一次就输出符合规范的结果，而不是反复纠正。

这是一种一次投入、长期收益的工作方式。

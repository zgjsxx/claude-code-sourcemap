# Claude Code 提示词（System Prompt）体系

> 基于 `restored-src/src` 中还原的 `@anthropic-ai/claude-code@2.1.88` 源码分析

## 概述

Claude Code 的系统提示词不是一段固定文本，而是由 **两层架构** 动态组装：

1. **内容生成层**（`constants/prompts.ts`）— 定义各段提示词的具体内容
2. **优先级选择层**（`utils/systemPrompt.ts`）— 根据运行模式决定用哪段提示词

最终输出是一个 `string[]` 数组，数组中的每个元素作为一条独立的系统消息发送给 Anthropic API。

---

## 优先级选择层

`utils/systemPrompt.ts` 中的 `buildEffectiveSystemPrompt()` 决定最终生效的提示词：

```
优先级 0: overrideSystemPrompt  → 完全替换所有内容（如 loop 模式）
优先级 1: coordinatorSystemPrompt → 协调模式提示词
优先级 2: agentSystemPrompt → 自定义 Agent 提示词
          - proactive 模式: 追加到默认提示词后面
          - 其他模式: 替换默认提示词
优先级 3: customSystemPrompt → --system-prompt CLI 参数
优先级 4: defaultSystemPrompt → 默认提示词（最常用）
          + appendSystemPrompt 始终追加在末尾（override 时除外）
```

绝大多数用户会走 **优先级 4（默认提示词）**。

---

## 默认提示词结构

`constants/prompts.ts` 中的 `getSystemPrompt()` 函数（行 444）组装默认提示词，分为静态区和动态区两部分：

### 静态区（可缓存）

静态区的内容在会话期间不会变化，可以被 Anthropic API 的 **prompt caching** 缓存，减少重复 token 计费。

| 函数 | 行号 | 主题 | 核心内容 |
|---|---|---|---|
| `getSimpleIntroSection()` | 175 | 身份定义 | "You are an interactive agent that helps users with software engineering tasks." |
| `getSimpleSystemSection()` | 186 | 系统规则 | 工具需要权限确认；被拒绝后不要重试相同调用；注意 `<system-reminder>` 标签；注意 prompt 注入；对话会被自动压缩 |
| `getSimpleDoingTasksSection()` | 199 | 行为准则 | 不要过度工程化；不要添加不必要的抽象/注释/错误处理；先读代码再改代码；不要创建不必要的文件；注意安全漏洞（OWASP Top 10） |
| `getActionsSection()` | 255 | 危险操作 | 危险操作（删除、force push、发送消息）必须先确认用户；不要用破坏性操作绕过障碍 |
| `getUsingYourToolsSection()` | 269 | 工具使用 | 优先用专用工具（Read/Edit/Write/Glob/Grep）而非 Bash；独立工具调用可以并行执行；有依赖关系的必须串行 |
| `getSimpleToneAndStyleSection()` | 430 | 语气风格 | 简洁；不用 emoji；引用代码用 `file_path:line_number` 格式 |
| `getOutputEfficiencySection()` | 403 | 输出效率 | 直截了当；不要废话；一句话能说完不要用三句 |

### 缓存边界标记

`SYSTEM_PROMPT_DYNAMIC_BOUNDARY`（行 114）是一条特殊分隔线，告诉 Anthropic API：前面的内容可以缓存，后面的内容是动态的。当启用 `shouldUseGlobalCacheScope()` 时插入。

### 动态区（每轮可能变化）

动态区通过 `systemPromptSections.ts` 的注册机制管理，有两种类型：

- **`systemPromptSection(name, compute)`** — 缓存型，计算一次后在 `/clear` 或 `/compact` 前不再重算
- **`DANGEROUS_uncachedSystemPromptSection(name, compute, reason)`** — 不缓存型，每轮都重新计算（如 MCP 指令，因为服务器可能随时连接/断开）

| 区段名 | 来源 | 内容 |
|---|---|---|
| `session_guidance` | `getSessionSpecificGuidanceSection()` | 会话特定指引：Agent 工具使用建议、Skill 调用方式、搜索工具建议 |
| `memory` | `loadMemoryPrompt()` | 用户记忆内容（CLAUDE.md、`~/.claude/projects/*/memory/` 等） |
| `ant_model_override` | `getAntModelOverrideSection()` | Anthropic 内部模型覆盖指令（仅 `USER_TYPE === 'ant'`） |
| `env_info_simple` | `computeSimpleEnvInfo()` | 运行环境信息：OS、Shell、CWD、Git 仓库状态、工作目录 |
| `language` | `getLanguageSection()` | 语言设置（如用户配置了中文回复） |
| `output_style` | `getOutputStyleSection()` | 输出风格配置 |
| `mcp_instructions` | `getMcpInstructionsSection()` | MCP 服务器提供的工具使用说明（**不缓存**，每轮重算） |
| `scratchpad` | `getScratchpadInstructions()` | 临时笔记（scratchpad）的使用指令 |
| `frc` | `getFunctionResultClearingSection()` | 工具结果清理规则（上下文压缩时如何处理工具结果） |
| `summarize_tool_results` | 常量字符串 | "把工具结果中的重要信息写下来，因为原始结果后续可能被清除" |
| `brief` | `getBriefSection()` | Brief 模式指令（KAIROS 特性） |

---

## 提示词关键内容详解

### 身份定义（Intro）

```
You are an interactive agent that helps users with software engineering tasks.
Use the instructions below and the tools available to you to assist the user.
```

核心安全指令：
- **绝不为用户生成或猜测 URL**，除非确认是用于编程帮助
- 安全测试必须在授权范围内（渗透测试、CTF、安全研究）

### 行为准则（Doing Tasks）

这部分是最长的提示词段，核心原则：

1. **最小化原则**：不要添加超出任务要求的特性、重构、注释
2. **先理解再修改**：不要对没读过的代码提修改建议
3. **优先编辑而非新建**：尽量改现有文件，减少文件膨胀
4. **安全第一**：注意 OWASP Top 10 漏洞，写完不安全代码立即修复
5. **失败诊断**：方法失败时先诊断原因，不要盲目重试或立即放弃
6. **诚实报告**：测试失败就报告失败，不要隐瞒或简化失败结果

Anthropic 内部版本（`USER_TYPE === 'ant'`）有额外指令：
- 默认不写注释，只在 WHY 不明显时才写
- 完成任务前要实际验证（跑测试、执行脚本），不能只声称完成
- 如果发现用户请求基于误解，要主动指出

### 工具使用（Using Your Tools）

核心规则：
- **不要用 Bash 替代专用工具**：Read → 不用 cat/head/tail；Edit → 不用 sed/awk；Write → 不用 echo/cat heredoc；Glob → 不用 find；Grep → 不用 grep/rg
- **独立调用可以并行**：多个无依赖的工具调用应在一条消息中同时发出
- **有依赖必须串行**：如果一个调用的结果决定下一个调用的参数，必须串行执行

### 危险操作确认（Executing Actions with Care）

需要用户确认的操作类型：
- 破坏性操作：删除文件/分支、rm -rf、覆盖未提交变更
- 难以撤销的操作：force push、git reset --hard、修改 CI/CD
- 对他人可见的操作：push 代码、创建/关闭 PR、发消息（Slack/邮件/GitHub）
- 上传到第三方：内容可能被缓存或索引

核心原则："measure twice, cut once"（三思而后行）。

---

## 特殊模式的提示词

### 简化模式（--bare / CLAUDE_CODE_SIMPLE）

```javascript
// 只返回极简提示词
`You are Claude Code, Anthropic's official CLI for Claude.\n\nCWD: ${getCwd()}\nDate: ${getSessionStartDate()}`
```

跳过所有复杂指令，适合自动化场景。

### 主动模式（Proactive / KAIROS）

```javascript
// 极简自主代理提示词
`You are an autonomous agent. Use the available tools to do useful work.`
```

加上记忆、环境信息、语言、MCP 指令、scratchpad、proactive 专属区段。

### 协调模式（Coordinator）

使用 `coordinator/coordinatorMode.ts` 中的 `getCoordinatorSystemPrompt()`，用于多 Agent 协调场景。

### Agent 模式

自定义 Agent 的提示词来自 `agentDefinition.getSystemPrompt()`，在 proactive 模式下追加到默认提示词，否则替换默认提示词。

---

## 工具自身的提示词

每个工具通过 `tool.prompt()` 方法生成一段使用说明，描述该工具的功能、参数和约束。这些说明会被注入到系统提示词中，让模型知道如何正确使用每个工具。

工具的 `description()` 方法则生成工具在 API 请求中的描述字段（`tools[].description`），这是模型在选择工具时看到的简短说明。

---

## 相关源码文件

| 文件 | 作用 |
|---|---|
| `constants/prompts.ts` | 默认提示词内容定义和组装 |
| `constants/systemPromptSections.ts` | 动态区段的注册机制（缓存/不缓存） |
| `utils/systemPrompt.ts` | 提示词优先级选择和最终组装 |
| `utils/systemPromptType.ts` | `SystemPrompt` 品牌类型定义 |
| `constants/cyberRiskInstruction.ts` | 安全测试相关指令 |
| `constants/outputStyles.ts` | 输出风格配置 |
| `memdir/memdir.ts` | 用户记忆加载 |
| `services/mcp/client.ts` | MCP 服务器连接和指令获取 |
| `Tool.ts` | 工具基类（含 `prompt()` 和 `description()` 方法） |
| 各 `tools/XxxTool/prompt.ts` | 各工具的具体提示词内容 |
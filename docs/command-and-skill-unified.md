# 命令与 Skill 的统一体系

## 核心事实：没有"命令 vs skill"的二分法

在 Claude Code 源码中，用户通过 `/xxx` 触发的所有操作——无论是 `/clear`、`/cost` 这类"命令"，还是 `/init`、`/review`、`/ppt-master` 这类"skill"——都属于同一个 `Command` 类型，放在同一个扁平列表中，走同一条查找路径。

用户输入 `/ppt-master` 和 `/clear` 经历完全相同的流程：`parseSlashCommand` → `findCommand` → 按类型执行。代码中不存在"命令"和"skill"的两套独立系统。

---

## Command 的三种执行方式

真正的区分发生在 `findCommand` 返回对象之后，取决于 `Command.type` 字段（`src/types/command.ts:205-207`）：

```
Command = CommandBase & (PromptCommand | LocalCommand | LocalJSXCommand)
```

| type | 典型例子 | SkillTool 可调用？ | 执行方式 |
|------|---------|------------------|---------|
| `prompt` | `/init`, `/review`, `/commit`, `/ppt-master` | 可以 | 注入 prompt 到对话或 fork 子 agent |
| `local` | `/clear`, `/cost`, `/status` | 不可以 | 执行本地 JS 函数，不走 Claude |
| `local-jsx` | `/config`, `/help`, `/model`, `/keybindings` | 不可以 | 渲染 React UI 组件 |

**所谓 "skill" 就是 `type: 'prompt'` 的 Command。** 所谓 "命令" 就是 `type: 'local'` 或 `type: 'local-jsx'` 的 Command。这只是执行方式的不同，不是两套独立的体系。

---

## 统一的 Command 列表

`getCommands()` (`src/commands.ts:476`) 将所有来源合并为一个扁平列表：

```
getCommands(cwd) =
  bundledSkills           ← 内置 skill（/init, /review 等）
  + builtinPluginSkills   ← 内置插件 skill
  + skillDirCommands      ← .claude/skills/ 目录的用户自定义 skill
  + workflowCommands      ← workflow 脚本命令
  + pluginCommands        ← 外部插件提供的 skill
  + COMMANDS()            ← 代码中硬编码的内置命令（/clear, /cost 等）
  + dynamicSkills         ← 运行时动态发现的 skill
```

查找时不区分来源，`findCommand()` 在整个列表中搜索：

```typescript
export function findCommand(commandName, commands) {
  return commands.find(_ =>
    _.name === commandName ||
    getCommandName(_) === commandName ||
    _.aliases?.includes(commandName)
  )
}
```

---

## 来源标记：loadedFrom

`loadedFrom` 字段标记 Command 的来源，这才是 "skill" 语义的边界：

| loadedFrom | 来源 | 说明 |
|-----------|------|------|
| `bundled` | 内置 skill | 随产品发布的专业 skill |
| `skills` | `.claude/skills/` | 用户自定义的 SKILL.md |
| `plugin` | 外部插件 | 第三方插件提供的 skill |
| `mcp` | MCP 服务器 | MCP 协议提供的 prompt-type skill |
| `commands_DEPRECATED` | `.claude/commands/` | 旧版命令目录（已废弃） |
| 无标记 | 硬编码 | `/clear`、`/cost` 这类纯 UI 操作 |

---

## SkillTool 的过滤

`getSkillToolCommands()` (`src/commands.ts:563-581`) 为 SkillTool 列出模型可调用的 skill 时，只包含 `type === 'prompt'` 且 `source !== 'builtin'` 且满足描述条件的 Command。这确保 `/clear`、`/cost` 这类纯 UI 操作不会出现在 SkillTool 的 skill 列表中——模型不会尝试调用它们。

`validateInput()` (`src/tools/SkillTool/SkillTool.ts:354-429`) 进一步拒绝非 `prompt` 类型的调用，返回 `Skill xxx is not a prompt-based skill`。

---

## 两条触发路径

用户手动输入 `/xxx` 和模型通过 SkillTool 调用走两条独立的入口路径，但最终都落到同一套核心逻辑：

### 手动输入路径

```
用户输入 "/ppt-master 帮我生成PPT"
  → processUserInput.ts（检测 '/' 开头）
  → parseSlashCommand()（拆出 commandName + args）
  → findCommand()（在统一 Command 列表中查找）
  → getMessagesForSlashCommand()
    → context === 'fork' ? executeForkedSlashCommand() : getPromptForCommand()
```

### 模型调用路径

```
模型输出 tool_use {name: "Skill", input: {skill: "ppt-master", args: "..."}}
  → toolExecution.ts → SkillTool.call()
  → findCommand()（同一个统一 Command 列表）
  → context === 'fork' ? executeForkedSkill() : processPromptSlashCommand()
```

两条路径最终都调用同一个 `getPromptForCommand()` 或同一个 `runAgent()` 子 agent 执行逻辑。

---

## 关键源码文件

| 文件 | 职责 |
|------|------|
| `src/types/command.ts` | Command/PromptCommand/LocalCommand/LocalJSXCommand 类型定义 |
| `src/commands.ts` | getCommands、findCommand、getSkillToolCommands（统一列表与过滤） |
| `src/skills/loadSkillsDir.ts` | 从 .claude/skills/ 加载 SKILL.md 为 PromptCommand |
| `src/tools/SkillTool/SkillTool.ts` | SkillTool 工具定义（模型调用入口） |
| `src/utils/processUserInput/processSlashCommand.tsx` | 手动 /xxx 输入处理 |
| `src/utils/slashCommandParsing.ts` | 拆分 /commandName + args |
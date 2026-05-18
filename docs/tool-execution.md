# Claude Code 工具调用执行机制详解

本文档基于 `@anthropic-ai/claude-code@2.1.88` 的还原源码，深入剖析工具调用的完整执行流程。

---

## 1. 总览：工具调用的生命周期

一次工具调用从模型发出 `tool_use` 响应到最终返回 `tool_result`，经历以下阶段：

```
模型响应 → 工具分发 → Schema校验 → 语义校验 → PreToolUse Hooks → 权限决策 → 执行 → PostToolUse Hooks → 结果处理 → 返回模型
```

---

## 2. 工具定义体系

### 2.1 Tool 类型（`src/Tool.ts`）

所有工具必须满足 `Tool<Input, Output, P>` 类型契约，核心字段分为以下几类：

**身份与 Schema：**
- `name` — 工具唯一标识符
- `aliases` — 向后兼容的别名（如 `KillShell` → `TaskStop`）
- `inputSchema` — Zod schema，用于输入类型校验
- `inputJSONSchema` — MCP 工具使用的原始 JSON Schema
- `shouldDefer` / `alwaysLoad` — 控制是否延迟加载（ToolSearch 机制）

**执行核心：**
- `call(args, context, canUseTool, parentMessage, onProgress)` — 实际执行方法，返回 `Promise<ToolResult<Output>>`
- `description(input, options)` — 给模型看的简短描述
- `prompt(options)` — 注入 system prompt 的完整模板

**权限与校验：**
- `validateInput(input, context)` — 语义层面的输入校验（区别于 Schema 类型校验）
- `checkPermissions(input, context)` — 工具自身权限逻辑
- `isConcurrencySafe(input)` — 是否可并行执行（默认 `false`）
- `isReadOnly(input)` — 是否只读操作（默认 `false`）
- `isDestructive(input)` — 是否不可逆操作（默认 `false`）
- `requiresUserInteraction()` — 即使 Hook 自动批准，仍需用户交互

### 2.2 buildTool 与默认值

`buildTool(def)` 函数为 `ToolDef` 填充安全默认值：

| 字段 | 默认值 | 说明 |
|------|--------|------|
| `isEnabled` | `() => true` | 工具始终可用 |
| `isConcurrencySafe` | `() => false` | 默认不可并行 |
| `isReadOnly` | `() => false` | 默认视为写操作 |
| `isDestructive` | `() => false` | 默认视为可逆 |
| `checkPermissions` | `{ behavior: 'allow' }` | 委托给通用权限系统 |
| `userFacingName` | `() => def.name` | 显示名等于工具名 |

每个工具通过 `buildTool({...}) satisfies ToolDef` 导出，保证类型完整性又免除样板代码。

---

## 3. 工具注册与加载

### 3.1 getAllBaseTools（`src/tools.ts`）

这是工具注册的源始函数，返回所有内置工具实例的扁平数组。核心工具始终包含：

- AgentTool, BashTool, FileReadTool, FileEditTool, FileWriteTool
- GlobTool, GrepTool, NotebookEditTool
- WebFetchTool, WebSearchTool
- TodoWriteTool, SkillTool, AskUserQuestionTool
- EnterPlanModeTool, ExitPlanModeV2Tool
- TaskOutputTool, TaskStopTool, MonitorTool 等

条件工具根据环境变量和 Feature Gate 动态包含：

- PowerShellTool — 仅 Windows
- REPLTool, ConfigTool — 仅内部用户（`USER_TYPE === 'ant'`）
- CronCreate/Delete/List — 仅 AGENT_TRIGGERS 模式
- LSPTool — 仅启用 LSP feature flag
- EnterWorktree/ExitWorktree — 仅启用 worktree 模式
- TaskCreate/Get/Update/List — 仅 TodoV2 模式

### 3.2 getTools — 权限过滤

`getTools(permissionContext)` 在 `getAllBaseTools()` 之上应用权限过滤：

1. `CLAUDE_CODE_SIMPLE` 模式：只保留 Bash、Read、Edit
2. `filterToolsByDenyRules` — 根据 deny 规则移除工具（模型根本看不到）
3. REPL 模式隐藏内部包装工具
4. 调用每个工具的 `isEnabled()` 做最终判定

### 3.3 assembleToolPool — 合并内置 + MCP

`assembleToolPool(permissionContext, mcpTools)` 将内置工具和 MCP 工具合并：

1. 获取内置工具 via `getTools()`
2. 过滤 MCP 工具的 deny 规则
3. 每个分区按字母排序（保证 prompt cache 稳定性）
4. 同名去重：内置工具优先，MCP 工具次之

---

## 4. API Schema 生成

### 4.1 toolToAPISchema（`src/utils/api.ts`）

将 `Tool` 对象转换为 Anthropic API 的 `BetaToolUnion`：

1. 查询 session-scoped `toolSchemaCache` 缓存（防止 mid-session GrowthBook 翻转导致 prompt cache 失效）
2. Zod → JSON Schema 转换（`zodToJsonSchema`），或 MCP 工具直接使用 `inputJSONSchema`
3. 调用 `tool.prompt()` 生成描述文本
4. 可选添加 `strict: true`（结构化输出模式）
5. 延迟工具添加 `defer_loading: true`
6. 添加 `cache_control` 标记（prompt cache 断点）
7. 添加 `eager_input_streaming`（细粒度工具流式传输）

### 4.2 ToolSearch 延迟加载

当 MCP 工具定义超过 context window 的 10% 时，启用延迟加载：

- 延迟工具发送 `defer_loading: true`，模型仅看到工具名和简短描述
- 模型必须先调用 `ToolSearchTool`（`query: "select:toolName"`）发现工具 schema
- 发现后才能正常调用该工具

---

## 5. 查询循环与工具分发

### 5.1 主循环（`src/query.ts`）

查询循环是整个系统的入口：

1. 创建 `StreamingToolExecutor`（流式模式）或使用批量 `runTools()` 路径
2. 流式接收 API 响应，收集 `assistantMessages` 和 `toolUseBlocks`
3. `tool_use` 块在流式过程中立即提交给 `streamingToolExecutor.addTool()`
4. 已完成的工具结果通过 `getCompletedResults()` 即时回传
5. 流式结束后，剩余结果通过 `getRemainingResults()` 收集

### 5.2 批量执行路径（`src/services/tools/toolOrchestration.ts`）

`runTools()` 是非流式批量执行路径：

```
partitionToolCalls(toolUseMessages, context)
  → 连续的 concurrency-safe 工具 → 并行批次
  → 非安全工具 → 串行批次
```

- 并行批次：`runToolsConcurrently()` 使用 `all()` 并发执行，上限 `CLAUDE_CODE_MAX_TOOL_USE_CONCURRENCY`（默认 10）
- 串行批次：`runToolsSerially()` 一个接一个执行，追踪 context 修改
- 每批次结束后应用 context modifier

### 5.3 流式执行器（`src/services/tools/StreamingToolExecutor.ts`）

`StreamingToolExecutor` 管理 `tool_use` 块到达时的并发执行：

```typescript
type ToolStatus = 'queued' | 'executing' | 'completed' | 'yielded'
```

核心方法：

- **addTool(block, message)** — 添加工具到追踪队列，判断 `isConcurrencySafe`，并发允许时立即开始执行
- **canExecuteTool()** — 检查并发规则：safe 工具可与其他 safe 工具并行；非 safe 工具需要独占访问
- **executeTool()** — 调用 `runToolUse()`，收集结果，处理错误级联（Bash 错误通过 `siblingAbortController` 终止兄弟进程）
- **getCompletedResults()** — 按序 yield 结果，并行结果缓冲到前序工具完成后才释放
- **getRemainingResults()** — 异步生成器等待所有剩余工具
- **discard()** — 流式 fallback 时调用：排队工具不启动，执行中工具收到合成错误

---

## 6. 核心执行管线（`src/services/tools/toolExecution.ts`）

这是单个工具调用的完整管线，入口为 `runToolUse()`。

### 6.1 runToolUse — 入口

```typescript
async function* runToolUse(toolUse, assistantMessage, canUseTool, toolUseContext)
```

1. 通过 `findToolByName()` 在可用工具中查找
2. 若未找到，检查是否是废弃别名（alias fallback）
3. 工具不存在 → yield 错误 `"No such tool available"`
4. 已中止 → yield 取消消息
5. 正常 → 委托给 `checkPermissionsAndCallTool()`

### 6.2 checkPermissionsAndCallTool — 十步管线

#### 步骤 1：Schema 类型校验

```typescript
const parsedInput = tool.inputSchema.safeParse(input)
```

使用 Zod 校验输入类型。模型有时生成无效输入（类型错误），Zod 拒绝并返回 `formatZodValidationError()` 格式的错误。

特殊处理：如果工具 schema 因延迟加载未发送给模型，追加提示要求先用 ToolSearch 加载 schema。

#### 步骤 2：语义校验

```typescript
const isValidCall = await tool.validateInput?.(parsedInput.data, toolUseContext)
```

每个工具可有自定义的语义校验逻辑（如文件必须先读取才能编辑）。

#### 步骤 3：输入预处理

- 移除防御性字段（如 Bash 的 `_simulatedSedEdit`——只应由权限系统注入）
- 调用 `backfillObservableInput()` 补充遗留/派生字段（如 SendMessageTool 添加字段、文件工具展开路径）
- 预处理后的 `processedInput` 供 hooks 和权限系统使用，原始 `callInput` 供 `tool.call()` 使用

#### 步骤 4：PreToolUse Hooks

```typescript
for await (const result of runPreToolUseHooks(toolUseContext, tool, processedInput, ...))
```

Hook 是用户在 `settings.json` 中配置的 shell 命令。每个 PreToolUse hook 结果可以是：

| 结果类型 | 含义 |
|----------|------|
| `message` | 进度或附件消息 |
| `hookPermissionResult` | 允许/拒绝/询问的权限决策 |
| `hookUpdatedInput` | 修改输入但不做权限决策 |
| `preventContinuation` | 阻止后续执行 |
| `additionalContext` | 为模型提供额外上下文 |
| `stop` | 终止信号 |

#### 步骤 5：权限决策

```typescript
const resolved = await resolveHookPermissionDecision(hookPermissionResult, tool, processedInput, ...)
```

`resolveHookPermissionDecision()` 整合 Hook 决策与规则权限：

| Hook 决策 | 规则状态 | 最终行为 |
|-----------|---------|---------|
| `allow` | 无 deny/ask 规则 | 允许（跳过交互弹窗） |
| `allow` | 有 deny 规则 | 拒绝（规则优先于 Hook） |
| `allow` | 有 ask 规则 | 询问（仍需交互确认） |
| `deny` | 任何 | 拒绝（Hook deny 是终局） |
| `ask` / 无决策 | — | 进入正常权限流程 |

#### 步骤 6：交互权限确认

如果权限决策为 `ask`，`canUseTool()` 触发交互式权限弹窗：

- `default` 模式 → 弹窗询问用户
- `dontAsk` 模式 → 自动拒绝
- `auto` 模式 → AI 分类器自动决策（Bash 命令使用 transcript classifier，文件操作使用 yolo classifier）
- `plan` 模式 → 计划模式下拒绝写操作
- `bypassPermissions` 模式 → 全部允许

权限拒绝后返回错误消息，并触发 PermissionDenied hooks（用于 auto-mode 分类器的拒绝记录）。

#### 步骤 7：工具执行

```typescript
tool.call(callInput, context, canUseTool, assistantMessage, onProgress)
```

权限通过后，调用工具自身的 `call()` 方法——每个工具有自己的执行逻辑。

#### 步骤 8：结果映射与日志

```typescript
tool.mapToolResultToToolResultBlockParam(result.data, toolUseID)
```

将工具输出转换为 API `tool_result` 格式。同时记录耗时、文件扩展名等分析数据。

#### 步骤 9：PostToolUse Hooks

```typescript
for await (const result of runPostToolUseHooks(toolUseContext, tool, toolUseID, ...))
```

PostToolUse hooks 在工具执行完成后运行：

- MCP 工具：hooks 可通过 `updatedMCPToolOutput` 修改输出
- 可添加额外上下文、阻止后续执行、报告错误
- 工具执行失败时运行 `runPostToolUseFailureHooks()`

#### 步骤 10：结果持久化

```typescript
addToolResult() → processToolResultBlock() / processPreMappedToolResultBlock()
```

大型结果超过 `maxResultSizeChars`（默认 50K）时持久化到磁盘（`tool-results/` 子目录），模型收到 `<persisted-output>` 预览及文件路径。

每条消息有总预算限制 `MAX_TOOL_RESULTS_PER_MESSAGE_CHARS`（200K），防止 N 个并行工具共同产出过多输出。

---

## 7. 并发控制详解

### 7.1 并发规则

工具并发遵循以下规则：

- **Safe 工具可并行**：所有正在执行的工具都是 safe 时，新 safe 工具可立即开始
- **非 Safe 工具需独占**：任何正在执行的工具存在时，非 safe 工具必须等待
- **非 Safe 执行时阻塞一切**：非 safe 工具执行期间，所有新工具排队等待

### 7.2 错误级联

当 Bash 工具执行出错时：

- `siblingAbortController` 触发，终止同一批次的其他 Bash 子进程
- 这是 `toolUseContext.abortController` 的子控制器——只终止兄弟进程，不终止整个查询回合

### 7.3 中断行为

`tool.interruptBehavior()` 定义用户提交新消息时工具的反应：

- `'cancel'` — 取消当前工具执行
- `'block'` — 阻止用户提交直到工具完成

---

## 8. 权限系统详解

### 8.1 权限层级

权限决策由以下层级构成，从上到下优先级递增：

```
工具自身 checkPermissions() → PreToolUse Hooks → 规则权限(settings.json) → 交互权限弹窗 → AI 分类器(auto mode)
```

### 8.2 规则权限（`src/utils/permissions/permissions.ts`）

`hasPermissionsToUseTool()` 检查四类规则：

| 规则类型 | 来源 | 说明 |
|---------|------|------|
| allow | settings.json | 允许特定工具/命令，跳过交互 |
| deny | settings.json | 拒绝特定工具/命令 |
| ask | settings.json | 强制交互确认 |
| bypassPermissions | settings.json | 全部允许（危险） |

规则来源层次：CLI → policy → local → project → user

### 8.3 Bash 分类器

在 `auto` 权限模式下，Bash 命令使用分类器自动决策：

- `startSpeculativeClassifierCheck()` 在 PreToolUse hooks 执行期间并行启动
- 分类器检查命令是否安全可自动允许
- 仅在权限决策为 `ask` 且分类器仍在运行时显示 "分类器运行中" 状态

---

## 9. Hook 系统

### 9.1 Hook 执行机制（`src/utils/hooks.ts`）

Hook 是 `settings.json` 中配置的 shell 命令：

```json
{
  "hooks": {
    "PreToolUse": [{ "matcher": "Bash", "command": "my-check.sh" }],
    "PostToolUse": [{ "matcher": "Edit", "command": "my-lint.sh" }]
  }
}
```

执行流程：
1. 加载 hook 配置
2. 为每个匹配的 hook 生成 shell 进程
3. 通过 stdin 传入 JSON（工具名、输入、上下文）
4. 解析 JSON stdout 输出
5. 管理超时和取消

### 9.2 Hook 权限交互

关键原则：**Hook `allow` 不绕过 deny 规则**。`resolveHookPermissionDecision()` 确保：

- Hook 只能在无冲突规则时自动允许
- deny 规则始终优先
- ask 规则始终要求交互，即使 Hook 说允许

---

## 10. MCP 工具机制

### 10.1 MCP 工具创建（`src/services/mcp/client.ts`）

`fetchToolsForClient()` 为每个 MCP 服务器创建工具实例：

1. 调用 MCP 服务器 `tools/list`
2. 对每个工具，基于 `MCPTool` 模板 spread 并覆盖：
   - `name` — 添加 `mcp__server__tool` 前缀
   - `description()` / `prompt()` — 来自服务器定义
   - `inputJSONSchema` — 来自服务器 schema
   - `isConcurrencySafe()` / `isReadOnly()` / `isDestructive()` — 来自 annotations
   - `checkPermissions()` — 返回 "passthrough"（委托给通用系统）
   - `call()` — 调用 MCP 服务器 `tools/call`

### 10.2 MCP 工具权限

MCP 工具的 `checkPermissions()` 返回 `{ behavior: 'passthrough' }`，将权限决策完全委托给规则权限和交互弹窗。

---

## 11. 工具结果存储

### 11.1 大结果持久化（`src/utils/toolResultStorage.ts`）

当工具结果超过阈值时：

```
结果大小 > getPersistenceThreshold()
  → 写入 tool-results/ 子目录
  → 模型收到 <persisted-output> 预览 + 文件路径
```

阈值取 `min(工具声明的 maxResultSizeChars, 50K)` 或 GrowthBook override。

### 11.2 每消息预算

`MAX_TOOL_RESULTS_PER_MESSAGE_CHARS = 200K` 限制单轮所有工具结果的总量，防止并行工具集体产出过多输出。

---

## 12. 关键源码文件索引

| 文件路径 | 职责 |
|---------|------|
| `src/Tool.ts` | Tool 类型定义、buildTool、findToolByName |
| `src/tools.ts` | getAllBaseTools、getTools、assembleToolPool |
| `src/query.ts` | 主查询循环，StreamingToolExecutor 入口 |
| `src/services/tools/toolExecution.ts` | runToolUse、checkPermissionsAndCallTool（十步管线） |
| `src/services/tools/toolOrchestration.ts` | runTools（批量并发/串行执行） |
| `src/services/tools/StreamingToolExecutor.ts` | 流式并发执行器 |
| `src/services/tools/toolHooks.ts` | PreToolUse/PostToolUse Hook 运行、resolveHookPermissionDecision |
| `src/utils/api.ts` | toolToAPISchema（Zod → JSON Schema → API 格式） |
| `src/utils/permissions/permissions.ts` | hasPermissionsToUseTool（规则权限检查） |
| `src/hooks/useCanUseTool.tsx` | canUseTool（交互权限确认） |
| `src/utils/toolResultStorage.ts` | 大结果持久化与预算管理 |
| `src/utils/toolSchemaCache.ts` | Session-scoped schema 缓存 |
| `src/utils/toolSearch.ts` | ToolSearch 延迟加载机制 |
| `src/utils/hooks.ts` | Hook shell 进程执行基础设施 |
| `src/services/mcp/client.ts` | MCP 工具创建与 `tools/call` |

---

## 13. 流程图

```
┌─────────────────────────────────────────────────────────────────┐
│                     模型 API 响应                                │
│              (包含 tool_use blocks)                              │
└─────────────────────┬───────────────────────────────────────────┘
                      │
                      ▼
┌─────────────────────────────────────────────────────────────────┐
│              查询循环 (query.ts)                                 │
│   StreamingToolExecutor.addTool() 或 runTools()                  │
└─────────────────────┬───────────────────────────────────────────┘
                      │
                      ▼
┌─────────────────────────────────────────────────────────────────┐
│              并发调度                                            │
│   partitionToolCalls → safe=并行 / non-safe=串行                │
│   StreamingToolExecutor → 按到达顺序和并发规则调度               │
└─────────────────────┬───────────────────────────────────────────┘
                      │
                      ▼
┌─────────────────────────────────────────────────────────────────┐
│         单工具执行管线 (toolExecution.ts)                        │
│                                                                 │
│  ① Zod Schema 校验                                              │
│  ② 语义校验 (validateInput)                                     │
│  ③ 输入预处理 (strip/backfill)                                  │
│  ④ PreToolUse Hooks                                             │
│  ⑤ 权限决策 (resolveHookPermissionDecision)                     │
│  ⑥ 交互权限确认 (canUseTool)                                    │
│  ⑦ 工具执行 (tool.call)                                         │
│  ⑧ 结果映射 (mapToolResultToToolResultBlockParam)               │
│  ⑨ PostToolUse Hooks                                            │
│  ⑩ 结果持久化 (processToolResultBlock)                          │
└─────────────────────┬───────────────────────────────────────────┘
                      │
                      ▼
┌─────────────────────────────────────────────────────────────────┐
│              返回模型                                            │
│   tool_result blocks → 下一轮 API 调用                           │
└─────────────────────────────────────────────────────────────────┘
```
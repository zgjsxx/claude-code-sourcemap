# Claude Code 源码学习路线

> 基于 `restored-src/src` 中还原的 `@anthropic-ai/claude-code@2.1.88` TypeScript 源码

## 项目架构概览

Claude Code 是一个基于 **React + Ink**（终端 UI 框架）构建的 CLI 应用，核心架构分五层：

| 层 | 目录 | 作用 |
|---|---|---|
| 入口 | `main.tsx` | CLI 命令解析、启动 REPL |
| 核心 | `query.ts`, `QueryEngine.ts` | 与 Claude API 交互的主循环（对话、工具调用、上下文压缩） |
| 工具 | `tools/` (40 个) | 每个 Tool 是一个独立模块（Bash、FileEdit、Grep、MCP、Agent 等） |
| 服务 | `services/` (20 个) | API 通信、MCP、压缩、OAuth、分析、策略限制等后台服务 |
| UI | `components/`, `screens/` | 终端界面渲染（消息、权限对话框、Diff 展示等） |

---

## 学习路线（由浅入深）

### 第 1 步：入口和启动流程

- **`main.tsx`** — 程序入口，看 CLI 参数如何解析、初始化流程（性能打点、MDM、Keychain 预取）
- **`replLauncher.tsx`** — REPL 启动逻辑
- **`bootstrap/state.ts`** — 启动状态管理

这是理解"程序怎么跑起来"的起点。

### 第 2 步：对话核心循环

- **`query.ts`** — 最核心的文件，对话循环：发请求 → 收响应 → 处理工具调用 → 再发请求
- **`QueryEngine.ts`** — 查询引擎的抽象
- **`services/compact/`** — 上下文压缩（AutoCompact / ReactiveCompact），控制上下文窗口不溢出

这是理解"Claude Code 如何与 AI 交互"的关键。

### 第 3 步：工具系统（最值得深入的部分）

每个 Tool 都在 `tools/XxxTool/` 下，结构统一。先看 **`Tool.ts`**（工具基类定义），再看具体实现。

重点看以下几个：

| 优先级 | Tool | 为什么重要 |
|---|---|---|
| 高 | `BashTool` | 最核心的工具，理解沙箱、权限、Shell 执行机制 |
| 高 | `FileEditTool` / `FileReadTool` / `FileWriteTool` | 文件操作三件套，理解 Claude 如何编辑代码 |
| 高 | `AgentTool` | 多 Agent 协调（子 Agent 派发、隔离、通信） |
| 高 | `MCPTool` | MCP 协议集成，理解外部工具如何接入 |
| 中 | `GrepTool` / `GlobTool` | 搜索工具，理解权限控制模式 |
| 中 | `TaskXxxTool` | 任务管理系列（Create/Update/List/Stop） |
| 低 | 其他 | 按兴趣选择（WebFetch、NotebookEdit、Voice 等） |

### 第 4 步：权限与安全

- **`utils/permissions/`** — PermissionMode、permissionRuleParser
- **`hooks/toolPermission/`** — 工具权限检查钩子
- **`components/permissions/`** — 权限确认 UI
- **`services/policyLimits/`** — 策略限制（企业管控）

理解 Claude Code 的安全模型：哪些操作需要用户确认、沙箱如何工作。

### 第 5 步：服务层（按兴趣选读）

- **`services/api/`** — API 通信（重试、流式响应、Bootstrap）
- **`services/mcp/`** — MCP 服务生命周期管理
- **`services/SessionMemory/`** — 会话记忆系统
- **`services/extractMemories/`** — 从对话中提取记忆
- **`services/analytics/`** — GrowthBook 特性开关 + 遥测

### 第 6 步：高级功能

- **`coordinator/`** — 多 Agent 协调模式（Headless 模式）
- **`assistant/`** — KAIROS 主动助手模式
- **`skills/`** — 技能系统（slash commands）
- **`plugins/`** — 插件系统
- **`vim/`** — Vim 键绑定模式
- **`remote/`** — 远程会话

---

## 阅读技巧

1. **从 `main.tsx` → `query.ts` → `Tool.ts` 开始**，这三个文件是骨架
2. 注意 `feature('XXX')` — 这是 GrowthBook 特性开关，控制功能的启用/关闭
3. `process.env.USER_TYPE === 'ant'` 标记的是 Anthropic 内部专属功能
4. 不要试图一次读完 1884 个文件，按上面的路线分步深入

---

## 源码还原方法

本项目源码通过 npm 包 `@anthropic-ai/claude-code` 内附的 source map (`cli.js.map`) 还原：

1. 下载 npm 包并解压，得到 `package/cli.js.map`
2. source map 的 `sources` 字段包含原始文件路径，`sourcesContent` 字段包含原始 TypeScript 源码
3. 运行 `node extract-sources.js` 将源码提取到 `restored-src/` 目录

详见 [extract-sources.js](../extract-sources.js) 和 [README.md](../README.md)。
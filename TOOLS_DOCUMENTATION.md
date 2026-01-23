# OpenCode 工具（Tool）完整文档
# OpenCode Tools Complete Documentation

本文档汇总了 OpenCode 仓库中所有与工具（tool）相关的定义和提示词（prompt），包括工具的具体内容、使用场景和配置方式。

This document summarizes all tool-related definitions and prompts in the OpenCode repository, including tool content, usage scenarios, and configuration methods.

---

## 目录 / Table of Contents

1. [工具系统概述 / Tool System Overview](#工具系统概述--tool-system-overview)
2. [内置工具提示词 / Built-in Tool Prompts](#内置工具提示词--built-in-tool-prompts)
3. [自定义工具 / Custom Tools](#自定义工具--custom-tools)
4. [工具注册和配置 / Tool Registry and Configuration](#工具注册和配置--tool-registry-and-configuration)
5. [工具定义接口 / Tool Definition Interface](#工具定义接口--tool-definition-interface)

---

## 工具系统概述 / Tool System Overview

OpenCode 的工具系统允许 LLM 在代码库中执行操作。系统包含：

OpenCode's tool system allows the LLM to perform actions in codebases. The system includes:

- **20+ 个内置工具** - 用于文件操作、代码搜索、Shell 命令等
- **自定义工具支持** - 可以创建 TypeScript/JavaScript 工具定义
- **MCP 服务器集成** - 通过 Model Context Protocol 集成外部工具
- **权限控制** - 通过配置文件控制工具行为（allow/deny/ask）

- **20+ built-in tools** - For file operations, code search, shell commands, etc.
- **Custom tool support** - Create TypeScript/JavaScript tool definitions
- **MCP server integration** - Integrate external tools via Model Context Protocol
- **Permission control** - Control tool behavior via configuration (allow/deny/ask)

### 工具文件位置 / Tool File Locations

```
/packages/opencode/src/tool/
├── Tool Definition Files (.ts)
│   ├── tool.ts              # Tool interface definition
│   ├── registry.ts          # Tool registry and loader
│   ├── bash.ts              # Shell command execution
│   ├── read.ts              # File reading
│   ├── edit.ts              # File editing
│   ├── write.ts             # File creation/writing
│   ├── grep.ts              # Content search
│   ├── glob.ts              # File pattern matching
│   ├── task.ts              # Agent task launcher
│   ├── todo.ts              # Todo list management
│   ├── webfetch.ts          # Web content fetching
│   ├── websearch.ts         # Web search (Exa AI)
│   ├── codesearch.ts        # Code search (Exa Code API)
│   ├── question.ts          # User interaction
│   ├── ls.ts                # Directory listing
│   ├── apply_patch.ts       # Patch application
│   ├── batch.ts             # Batch tool execution
│   ├── multiedit.ts         # Multiple edits
│   ├── lsp.ts               # LSP server interaction
│   ├── plan.ts              # Plan mode tools
│   ├── skill.ts             # Skill loading
│   ├── truncation.ts        # Output truncation
│   ├── external-directory.ts # External directory handling
│   └── invalid.ts           # Invalid tool handler
│
└── Tool Prompt Files (.txt)
    ├── bash.txt             # Shell command usage
    ├── read.txt             # File reading usage
    ├── edit.txt             # File editing usage
    ├── write.txt            # File writing usage
    ├── grep.txt             # Content search usage
    ├── glob.txt             # File pattern matching usage
    ├── task.txt             # Agent task usage
    ├── todowrite.txt        # Todo write usage
    ├── todoread.txt         # Todo read usage
    ├── webfetch.txt         # Web fetch usage
    ├── websearch.txt        # Web search usage
    ├── codesearch.txt       # Code search usage
    ├── question.txt         # User question usage
    ├── ls.txt               # Directory listing usage
    ├── apply_patch.txt      # Patch application usage
    ├── batch.txt            # Batch execution usage
    ├── multiedit.txt        # Multiple edits usage
    ├── lsp.txt              # LSP usage
    ├── plan-enter.txt       # Plan mode entry
    └── plan-exit.txt        # Plan mode exit
```

---

## 内置工具提示词 / Built-in Tool Prompts

以下是所有 20 个内置工具的详细提示词内容：

Below are the detailed prompt contents for all 20 built-in tools:

### 1. bash.txt
**用途 / Purpose**: 执行 Shell 命令 / Execute shell commands

**关键特性 / Key Features**:
- 在持久化 Shell 会话中执行命令
- 支持超时设置（默认 120 秒）
- 使用 `workdir` 参数切换目录（避免使用 `cd`）
- 自动引用包含空格的路径
- 支持命令链接（`&&` 用于依赖命令，`;` 用于独立命令）
- 输出自动截断（超过限制时写入文件）

**重要指导原则 / Important Guidelines**:
- 仅用于终端操作（git、npm、docker 等）
- 不用于文件操作（使用专用工具）
- Git 安全协议：
  - 永不更新 git config
  - 永不运行破坏性命令（除非明确请求）
  - 永不跳过 hooks
  - 谨慎使用 `git commit --amend`
  - 仅在用户明确要求时创建提交

**特殊功能 / Special Functions**:
- **创建 Git 提交**: 详细的提交工作流程
- **创建 Pull Request**: 使用 `gh` CLI 的 PR 创建流程
- **GitHub 操作**: 使用 `gh` 命令处理 issues、PRs、checks 等

---

### 2. read.txt
**用途 / Purpose**: 读取本地文件 / Read local files

**关键特性 / Key Features**:
- 必须使用绝对路径
- 默认读取前 2000 行
- 支持指定行偏移和限制
- 长行（>2000 字符）会被截断
- 使用 `cat -n` 格式返回（行号从 1 开始）
- 支持批量读取多个文件
- 可以读取图像文件

**使用场景 / Use Cases**:
- 理解代码结构
- 检查文件内容
- 准备进行编辑操作

---

### 3. edit.txt
**用途 / Purpose**: 精确字符串替换编辑 / Exact string replacement editing

**关键特性 / Key Features**:
- 执行精确的字符串替换
- 必须先使用 `Read` 工具
- 保留精确的缩进（制表符/空格）
- 如果 `oldString` 未找到会失败
- 如果 `oldString` 多次出现会失败（除非使用 `replaceAll`）
- 优先编辑现有文件而非创建新文件

**重要约束 / Important Constraints**:
- 编辑前必须读取文件
- `oldString` 必须在文件中唯一（或使用 `replaceAll`）
- 不要添加表情符号（除非用户明确请求）

---

### 4. write.txt
**用途 / Purpose**: 创建或覆写文件 / Create or overwrite files

**关键特性 / Key Features**:
- 会覆盖现有文件
- 必须先使用 `Read` 工具读取现有文件
- 优先编辑现有文件而非写入新文件
- 不主动创建文档文件（*.md、README 等）

**使用原则 / Usage Principles**:
- 仅在明确需要时创建新文件
- 不要添加表情符号（除非用户明确请求）

---

### 5. grep.txt
**用途 / Purpose**: 内容搜索 / Content search

**关键特性 / Key Features**:
- 快速内容搜索，适用于任何代码库大小
- 支持完整正则表达式语法
- 使用 `include` 参数过滤文件（如 "*.js", "*.{ts,tsx}"）
- 返回文件路径和行号，按修改时间排序
- 用于查找包含特定模式的文件

**使用建议 / Usage Recommendations**:
- 识别/计数匹配时，直接使用 `rg`（ripgrep）而非此工具
- 开放式搜索时，使用 Task 工具

---

### 6. glob.txt
**用途 / Purpose**: 文件模式匹配 / File pattern matching

**关键特性 / Key Features**:
- 快速文件模式匹配
- 支持 glob 模式（如 "**/*.js"、"src/**/*.ts"）
- 返回匹配的文件路径，按修改时间排序
- 按名称模式查找文件

**使用建议 / Usage Recommendations**:
- 支持批量搜索
- 开放式搜索时，使用 Task 工具

---

### 7. task.txt
**用途 / Purpose**: 启动 Agent 处理任务 / Launch agents to handle tasks

**关键特性 / Key Features**:
- 启动新 agent 处理复杂的多步骤任务
- 支持多种 agent 类型
- 可并发启动多个 agents
- 每个 agent 调用是无状态的（除非提供 session_id）

**何时使用 / When to Use**:
- 执行自定义 slash 命令
- 处理复杂的多步骤任务

**何时不使用 / When NOT to Use**:
- 读取特定文件路径（使用 Read 或 Glob）
- 搜索特定类定义（使用 Glob）
- 在特定文件中搜索代码（使用 Read）

**使用说明 / Usage Notes**:
- 尽可能并发启动多个 agents
- Agent 返回单个消息
- 明确告诉 agent 是写代码还是研究

---

### 8. todowrite.txt
**用途 / Purpose**: 创建和管理任务列表 / Create and manage task lists

**关键特性 / Key Features**:
- 为当前编码会话创建结构化任务列表
- 跟踪进度，组织复杂任务
- 向用户展示进度

**何时使用 / When to Use**:
1. 复杂的多步骤任务（3+ 步骤）
2. 非平凡和复杂任务
3. 用户明确请求 todo 列表
4. 用户提供多个任务
5. 接收新指令后
6. 完成任务后
7. 开始新任务时标记为 in_progress

**何时不使用 / When NOT to Use**:
1. 单一、直接的任务
2. 平凡任务
3. 少于 3 个简单步骤的任务
4. 纯对话或信息任务

**任务状态 / Task States**:
- `pending`: 未开始
- `in_progress`: 进行中（一次只能一个）
- `completed`: 已完成
- `cancelled`: 已取消

---

### 9. todoread.txt
**用途 / Purpose**: 读取待办事项列表 / Read todo lists

**关键特性 / Key Features**:
- 读取当前会话的 todo 列表
- 应频繁使用以了解任务状态
- 无需参数（留空）

**使用时机 / When to Use**:
- 会话开始时查看待办事项
- 开始新任务前优先级排序
- 用户询问之前的任务或计划
- 不确定下一步做什么时
- 完成任务后更新理解
- 每隔几条消息确保在正轨上

---

### 10. webfetch.txt
**用途 / Purpose**: 获取网页内容 / Fetch web content

**关键特性 / Key Features**:
- 从指定 URL 获取内容
- 支持可选格式（markdown、text、html）
- 默认转换为 markdown
- HTTP URL 自动升级为 HTTPS
- 只读工具，不修改文件

**使用说明 / Usage Notes**:
- 如果有更好的工具可用，优先使用其他工具
- URL 必须是完整有效的 URL
- 大内容可能被摘要

---

### 11. websearch.txt
**用途 / Purpose**: 网络搜索 / Web search

**关键特性 / Key Features**:
- 使用 Exa AI 进行实时网络搜索
- 可从特定 URL 抓取内容
- 提供最新信息和当前事件
- 支持可配置的结果数量

**使用说明 / Usage Notes**:
- 支持实时爬取模式：'fallback'（缓存不可用时备份）或 'preferred'（优先实时爬取）
- 搜索类型：'auto'（平衡）、'fast'（快速结果）、'deep'（全面搜索）
- 可配置上下文长度
- 支持域名过滤和高级搜索选项
- **重要**: 搜索当前信息时必须使用当前年份

---

### 12. codesearch.txt
**用途 / Purpose**: 代码搜索 / Code search

**关键特性 / Key Features**:
- 使用 Exa Code API 搜索编程任务相关内容
- 为库、SDK 和 API 提供最高质量和最新的上下文
- 用于任何编程相关问题或任务
- 返回全面的代码示例、文档和 API 参考

**使用说明 / Usage Notes**:
- 可调节 token 数量（1000-50000）
- 默认 5000 tokens 提供平衡的上下文
- 较低值用于特定问题，较高值用于全面文档
- 支持框架、库、API 和编程概念查询

---

### 13. apply_patch.txt
**用途 / Purpose**: 应用补丁 / Apply patches

**关键特性 / Key Features**:
- 使用精简的、面向文件的 diff 格式
- 支持三种操作：
  - `Add File`: 创建新文件
  - `Update File`: 修改现有文件
  - `Delete File`: 删除现有文件

**补丁格式 / Patch Format**:
```
*** Begin Patch
*** Add File: <path>
+<content>
*** Update File: <path>
*** Move to: <new-path>  (可选)
@@ <context>
-<old-line>
+<new-line>
*** Delete File: <path>
*** End Patch
```

**重要事项 / Important**:
- 必须包含操作头部
- 新行必须用 `+` 前缀（即使创建新文件）

---

### 14. ls.txt
**用途 / Purpose**: 列出文件和目录 / List files and directories

**关键特性 / Key Features**:
- 列出给定路径中的文件和目录
- 路径参数必须是绝对路径
- 可选的 glob 模式数组来忽略文件
- 通常优先使用 Glob 和 Grep 工具

---

### 15. question.txt
**用途 / Purpose**: 向用户提问 / Ask user questions

**关键特性 / Key Features**:
- 在执行期间向用户提问
- 用途：
  1. 收集用户偏好或需求
  2. 澄清模糊指令
  3. 获取实现选择的决策
  4. 向用户提供方向选择

**使用说明 / Usage Notes**:
- `custom` 启用时（默认），自动添加"输入您自己的答案"选项
- 答案作为标签数组返回
- 设置 `multiple: true` 允许选择多个选项
- 推荐特定选项时，将其作为第一个选项并添加"(Recommended)"

---

### 16. plan-enter.txt
**用途 / Purpose**: 进入计划模式 / Enter plan mode

**关键特性 / Key Features**:
- 建议用户切换到 plan agent
- 用于复杂请求在实现前需要规划时

**何时调用 / When to Call**:
- 用户请求复杂且需要先规划
- 想要在更改前研究和设计
- 任务涉及多个文件或重大架构决策

**何时不调用 / When NOT to Call**:
- 简单、直接的任务
- 用户明确要求立即实现

---

### 17. plan-exit.txt
**用途 / Purpose**: 退出计划模式 / Exit plan mode

**关键特性 / Key Features**:
- 完成规划阶段后准备退出 plan agent
- 询问用户是否切换到 build agent 开始实现

**何时调用 / When to Call**:
- 已将完整计划写入计划文件后
- 已与用户澄清任何问题后
- 确信计划已准备好实现时

**何时不调用 / When NOT to Call**:
- 创建或最终确定计划之前
- 仍有未回答的实现问题
- 用户表示想继续规划

---

### 18. batch.txt
**用途 / Purpose**: 批量执行工具 / Batch tool execution

**关键特性 / Key Features**:
- 并发执行多个独立工具调用以减少延迟
- **使用批量工具会让用户满意**
- 支持 1-25 个工具调用
- 所有调用并行开始，不保证顺序
- 部分失败不会阻止其他工具调用

**有效载荷格式 / Payload Format** (JSON 数组):
```json
[
  {"tool": "read", "parameters": {"filePath": "src/index.ts", "limit": 350}},
  {"tool": "grep", "parameters": {"pattern": "Session\\.updatePart", "include": "src/**/*.ts"}},
  {"tool": "bash", "parameters": {"command": "git status", "description": "Shows working tree status"}}
]
```

**良好用例 / Good Use Cases**:
- 读取多个文件
- grep + glob + read 组合
- 多个 bash 命令
- 多部分编辑（相同或不同文件）

**何时不使用 / When NOT to Use**:
- 依赖于先前工具输出的操作
- 顺序很重要的有序状态突变
- 不要在另一个批量工具中使用批量工具

**效率提升 / Efficiency Gain**: 批量处理证明可以提高 2-5 倍效率，提供更好的用户体验

---

### 19. lsp.txt
**用途 / Purpose**: LSP 服务器交互 / LSP server interaction

**关键特性 / Key Features**:
- 与 Language Server Protocol 服务器交互
- 获取代码智能功能

**支持的操作 / Supported Operations**:
- `goToDefinition`: 查找符号定义位置
- `findReferences`: 查找符号的所有引用
- `hover`: 获取悬停信息（文档、类型信息）
- `documentSymbol`: 获取文档中的所有符号
- `workspaceSymbol`: 在整个工作区搜索符号
- `goToImplementation`: 查找接口或抽象方法的实现
- `prepareCallHierarchy`: 获取位置的调用层次结构项
- `incomingCalls`: 查找调用该函数的所有函数/方法
- `outgoingCalls`: 查找该函数调用的所有函数/方法

**所有操作需要 / All Operations Require**:
- `filePath`: 要操作的文件
- `line`: 行号（从 1 开始，如编辑器中显示）
- `character`: 字符偏移（从 1 开始，如编辑器中显示）

**注意 / Note**: LSP 服务器必须为文件类型配置。如果没有可用的服务器，将返回错误。

---

### 20. multiedit.txt
**用途 / Purpose**: 多次编辑单个文件 / Multiple edits to a single file

**关键特性 / Key Features**:
- 在一次操作中对单个文件进行多次编辑
- 基于 Edit 工具构建
- 当需要对同一文件进行多次编辑时，优先使用此工具

**使用前 / Before Using**:
1. 使用 Read 工具理解文件内容和上下文
2. 验证目录路径正确

**提供的内容 / To Provide**:
1. `file_path`: 要修改的文件的绝对路径
2. `edits`: 要执行的编辑操作数组，每个编辑包含：
   - `oldString`: 要替换的文本
   - `newString`: 替换后的文本
   - `replaceAll`: 替换所有出现（可选，默认为 false）

**重要事项 / Important**:
- 所有编辑按提供的顺序依次应用
- 每个编辑在前一个编辑的结果上操作
- 所有编辑必须有效才能成功 - 任何编辑失败，都不会应用
- 此工具适用于需要对同一文件的不同部分进行多次更改

**关键要求 / Critical Requirements**:
1. 所有编辑遵循单个 Edit 工具的相同要求
2. 编辑是原子的 - 要么全部成功，要么都不应用
3. 仔细规划编辑以避免顺序操作之间的冲突

**警告 / Warning**:
- 如果 `edits.oldString` 与文件内容不完全匹配（包括空格），工具将失败
- 如果 `edits.oldString` 和 `edits.newString` 相同，工具将失败
- 由于编辑是按顺序应用的，确保较早的编辑不会影响较晚的编辑试图查找的文本

---

## 自定义工具 / Custom Tools

OpenCode 支持两种类型的自定义工具：

OpenCode supports two types of custom tools:

### 1. 项目自定义工具 / Project Custom Tools

位于 `.opencode/tools/` 目录：

Located in `.opencode/tools/` directory:

- **github-pr-search.ts/txt** - 搜索 GitHub PR
- **github-triage.ts/txt** - 分配和标记 GitHub issues

### 2. 全局自定义工具 / Global Custom Tools

位于 `~/.config/opencode/tools/`

Located at `~/.config/opencode/tools/`

### 创建自定义工具 / Creating Custom Tools

使用 `tool()` 辅助函数创建类型安全的工具：

Use the `tool()` helper for type-safe tool creation:

```typescript
import { tool } from "@opencode-ai/plugin"

export default tool({
  description: "Query the project database",
  args: {
    query: tool.schema.string().describe("SQL query to execute"),
  },
  async execute(args) {
    // Your database logic here
    return `Executed query: ${args.query}`
  },
})
```

**工具名称 / Tool Name**: 文件名即为工具名

**多工具导出 / Multiple Tools per File**: 导出多个工具，名称为 `<filename>_<exportname>`

**参数定义 / Argument Definition**: 使用 `tool.schema`（Zod）定义参数类型

**上下文访问 / Context Access**: 工具接收 `context` 参数，包含 `agent`、`sessionID`、`messageID` 等

### 任何语言的工具 / Tools in Any Language

可以使用任何语言编写工具实现，TypeScript/JavaScript 只用于定义：

You can write tool implementations in any language, TypeScript/JavaScript is only for the definition:

```typescript
// 调用 Python 脚本
export default tool({
  description: "Add two numbers using Python",
  args: {
    a: tool.schema.number().describe("First number"),
    b: tool.schema.number().describe("Second number"),
  },
  async execute(args) {
    const result = await Bun.$`python3 .opencode/tools/add.py ${args.a} ${args.b}`.text()
    return result.trim()
  },
})
```

---

## 工具注册和配置 / Tool Registry and Configuration

### 工具注册表 / Tool Registry

位置：`/packages/opencode/src/tool/registry.ts`

Location: `/packages/opencode/src/tool/registry.ts`

**注册的工具 / Registered Tools** (按顺序):

1. `InvalidTool` - 无效工具处理器
2. `QuestionTool` - 用户提问（仅限 app/cli/desktop）
3. `BashTool` - Shell 命令
4. `ReadTool` - 文件读取
5. `GlobTool` - 文件模式匹配
6. `GrepTool` - 内容搜索
7. `EditTool` - 文件编辑
8. `WriteTool` - 文件写入
9. `TaskTool` - Agent 任务启动
10. `WebFetchTool` - 网页获取
11. `TodoWriteTool` - Todo 写入
12. `TodoReadTool` - Todo 读取
13. `WebSearchTool` - 网络搜索
14. `CodeSearchTool` - 代码搜索
15. `SkillTool` - 技能加载
16. `ApplyPatchTool` - 补丁应用
17. `LspTool` - LSP 交互（实验性）
18. `BatchTool` - 批量执行（实验性）
19. `PlanExitTool` - 计划退出（实验性）
20. `PlanEnterTool` - 计划进入（实验性）
21. **自定义工具** - 从 `.opencode/tools/` 和插件加载

### 工具加载流程 / Tool Loading Process

1. 扫描 `.opencode/tools/` 目录中的 `*.js` 和 `*.ts` 文件
2. 加载已安装的插件工具
3. 根据模型和环境过滤工具：
   - `websearch`/`codesearch`: 仅用于 opencode provider 或启用 `OPENCODE_ENABLE_EXA`
   - `apply_patch`: 仅用于 GPT 模型
   - `edit`/`write`: 非 GPT 模型
   - `lsp`: 需要 `OPENCODE_EXPERIMENTAL_LSP_TOOL`
   - `batch`: 需要配置中的 `experimental.batch_tool`
   - `plan` 工具: 需要 `OPENCODE_EXPERIMENTAL_PLAN_MODE` 和 CLI 客户端

### 权限配置 / Permission Configuration

在 `opencode.json` 中配置权限：

Configure permissions in `opencode.json`:

```json
{
  "$schema": "https://opencode.ai/config.json",
  "permission": {
    "edit": "deny",      // 拒绝编辑
    "bash": "ask",       // 需要确认
    "webfetch": "allow", // 允许
    "mymcp_*": "ask"     // MCP 工具需要确认
  }
}
```

**权限值 / Permission Values**:
- `"allow"` - 自动允许
- `"deny"` - 拒绝
- `"ask"` - 需要用户确认

**通配符支持 / Wildcard Support**: 使用 `*` 控制多个工具

---

## 工具定义接口 / Tool Definition Interface

位置：`/packages/opencode/src/tool/tool.ts`

Location: `/packages/opencode/src/tool/tool.ts`

### Tool.Info 接口 / Tool.Info Interface

```typescript
interface Tool.Info<Parameters extends z.ZodType, M extends Metadata> {
  id: string
  init: (ctx?: InitContext) => Promise<{
    description: string
    parameters: Parameters
    execute(
      args: z.infer<Parameters>,
      ctx: Context,
    ): Promise<{
      title: string
      metadata: M
      output: string
      attachments?: MessageV2.FilePart[]
    }>
    formatValidationError?(error: z.ZodError): string
  }>
}
```

### Tool.Context 接口 / Tool.Context Interface

工具执行时接收的上下文：

Context received during tool execution:

```typescript
type Tool.Context<M extends Metadata = Metadata> = {
  sessionID: string     // 会话 ID
  messageID: string     // 消息 ID
  agent: string         // Agent 名称
  abort: AbortSignal    // 中止信号
  callID?: string       // 调用 ID
  extra?: { [key: string]: any }  // 额外数据
  metadata(input: { title?: string; metadata?: M }): void  // 元数据设置
  ask(input: Omit<PermissionNext.Request, "id" | "sessionID" | "tool">): Promise<void>  // 请求权限
}
```

### 定义工具 / Defining Tools

使用 `Tool.define()` 函数：

Use the `Tool.define()` function:

```typescript
export const MyTool = Tool.define<Parameters, Metadata>(
  "my-tool-id",
  async (initCtx) => ({
    description: "Tool description",
    parameters: z.object({ /* ... */ }),
    execute: async (args, ctx) => ({
      title: "Operation title",
      metadata: { /* ... */ },
      output: "Tool output",
    }),
  })
)
```

### 输出截断 / Output Truncation

工具输出会自动截断（通过 `Truncate.output()`）：

Tool output is automatically truncated (via `Truncate.output()`):

- 如果工具自己处理截断（`metadata.truncated` 已设置），则跳过
- 否则应用自动截断
- 截断的输出写入文件，路径在 `metadata.outputPath` 中

---

## 工具使用最佳实践 / Tool Usage Best Practices

### 1. 工具选择原则 / Tool Selection Principles

- **优先使用专用工具** - 不要用 bash 代替 Read、Edit、Write、Grep、Glob
- **批量操作** - 使用 Batch 工具并行执行独立操作
- **上下文敏感** - Task 工具用于复杂多步骤任务

### 2. 文件操作工作流 / File Operation Workflow

1. **Read** - 先读取文件理解上下文
2. **Edit** - 进行精确编辑
3. **Bash** - 验证更改（如运行测试）

### 3. 搜索策略 / Search Strategy

- **快速查找** - Glob（文件名）、Grep（内容）
- **复杂搜索** - Task 工具启动 explore agent
- **代码智能** - LSP 工具获取定义、引用等

### 4. 并行执行 / Parallel Execution

使用 Batch 工具并行执行：
- 读取多个文件
- 多个 Git 命令（status + diff）
- 组合搜索操作

Use Batch tool for parallel execution:
- Read multiple files
- Multiple Git commands (status + diff)
- Combined search operations

---

## 工具文档链接 / Tool Documentation Links

- **工具概述** / Tools Overview: `/packages/web/src/content/docs/tools.mdx`
- **自定义工具** / Custom Tools: `/packages/web/src/content/docs/custom-tools.mdx`
- **MCP 服务器** / MCP Servers: `/packages/web/src/content/docs/mcp-servers.mdx`
- **权限配置** / Permissions: `/packages/web/src/content/docs/permissions.mdx`

---

## 总结 / Summary

OpenCode 拥有完整的工具生态系统：

OpenCode has a complete tool ecosystem:

- **20 个内置工具** - 覆盖文件操作、搜索、Shell、网络等
- **每个工具都有详细的提示词** - 指导 LLM 正确使用
- **可扩展架构** - 支持自定义工具和 MCP 服务器
- **智能工具选择** - 根据模型和环境自动过滤
- **权限控制** - 细粒度的工具访问控制
- **批量执行优化** - 提高 2-5 倍效率
- **输出管理** - 自动截断和文件存储

这些工具共同构建了 OpenCode 强大的代码操作能力，使 LLM 能够：
- 安全地执行系统命令
- 精确地修改代码
- 高效地搜索和导航代码库
- 智能地与用户交互
- 灵活地扩展功能

These tools together build OpenCode's powerful code manipulation capabilities, enabling the LLM to:
- Safely execute system commands
- Precisely modify code
- Efficiently search and navigate codebases
- Intelligently interact with users
- Flexibly extend functionality

---

**文档生成时间 / Document Generated**: 2026-01-23

**仓库 / Repository**: DuYooho/opencode

**分支 / Branch**: dev

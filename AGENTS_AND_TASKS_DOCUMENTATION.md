# OpenCode Agent 和 Task 系统完整文档
# OpenCode Agents and Task System Complete Documentation

本文档汇总了 OpenCode 仓库中所有与 Agent（代理）、Sub-Agent（子代理）和 Task（任务）相关的定义、提示词和使用方式。

This document summarizes all agent, sub-agent, and task-related definitions, prompts, and usage methods in the OpenCode repository.

---

## 目录 / Table of Contents

1. [系统概述 / System Overview](#系统概述--system-overview)
2. [Agent 类型 / Agent Types](#agent-类型--agent-types)
3. [内置 Agent / Built-in Agents](#内置-agent--built-in-agents)
4. [Task 工具 / Task Tool](#task-工具--task-tool)
5. [Agent 配置 / Agent Configuration](#agent-配置--agent-configuration)
6. [Agent 定义接口 / Agent Definition Interface](#agent-定义接口--agent-definition-interface)
7. [创建自定义 Agent / Creating Custom Agents](#创建自定义-agent--creating-custom-agents)
8. [使用场景和最佳实践 / Use Cases and Best Practices](#使用场景和最佳实践--use-cases-and-best-practices)

---

## 系统概述 / System Overview

OpenCode 的 Agent 系统是一个灵活的多代理架构，允许为特定任务和工作流配置专门的 AI 助手。

OpenCode's Agent system is a flexible multi-agent architecture that allows configuring specialized AI assistants for specific tasks and workflows.

### 核心概念 / Core Concepts

- **Primary Agents（主代理）** - 用户直接交互的主要助手
- **Subagents（子代理）** - Primary agents 可以调用的专门助手
- **Task Tool（任务工具）** - 用于启动 subagents 执行特定任务的工具
- **Agent Modes（代理模式）** - 控制代理如何被使用（primary、subagent、all）
- **Permissions（权限）** - 细粒度控制代理可以使用的工具和操作

### 文件位置 / File Locations

```
/packages/opencode/src/agent/
├── agent.ts              # Agent 核心逻辑和定义
├── generate.txt          # Agent 生成器提示词
└── prompt/
    ├── explore.txt       # Explore subagent 提示词
    ├── summary.txt       # Summary agent 提示词
    ├── title.txt         # Title generator 提示词
    └── compaction.txt    # Compaction agent 提示词

/packages/opencode/src/tool/
├── task.ts               # Task 工具实现
└── task.txt              # Task 工具提示词

/.opencode/agent/         # 项目级自定义 agents
└── docs.md               # 文档编写 agent 示例
└── triage.md             # Issue 分类 agent 示例
```

---

## Agent 类型 / Agent Types

OpenCode 中有两种类型的 agents：

There are two types of agents in OpenCode:

### 1. Primary Agents（主代理）

**定义 / Definition**:
- 用户直接交互的主要助手
- 处理主要会话
- 可以使用 **Tab** 键或配置的 `switch_agent` 快捷键在它们之间切换

**特点 / Features**:
- 完整的会话上下文
- 工具访问通过权限配置
- 可以调用 subagents 处理特定任务
- 模式：`mode: "primary"`

**内置 Primary Agents / Built-in Primary Agents**:
- **Build** - 默认代理，所有工具启用
- **Plan** - 受限代理，用于规划和分析

### 2. Subagents（子代理）

**定义 / Definition**:
- Primary agents 可以调用的专门助手
- 用户可以通过 **@** 提及手动调用
- 通过 Task 工具启动

**特点 / Features**:
- 专注于特定任务
- 创建子会话（child sessions）
- 受限的工具访问（根据配置）
- 模式：`mode: "subagent"`

**内置 Subagents / Built-in Subagents**:
- **General** - 通用代理，全工具访问
- **Explore** - 快速只读代理，用于探索代码库

### 3. All Mode（全模式）

- `mode: "all"` - 可以作为 primary agent 或 subagent 使用
- 默认模式（如果未指定）

---

## 内置 Agent / Built-in Agents

OpenCode 自带 6 个内置 agents（2 个 primary，2 个 subagent，2 个隐藏）：

OpenCode comes with 6 built-in agents (2 primary, 2 subagent, 2 hidden):

### 1. Build Agent

**模式 / Mode**: `primary`

**用途 / Purpose**: 默认的开发工作代理，所有工具启用

**权限 / Permissions**:
```typescript
{
  "*": "allow",
  "doom_loop": "ask",
  "question": "allow",
  "plan_enter": "allow",
  "external_directory": { "*": "ask" },
  "read": {
    "*": "allow",
    "*.env": "ask",
    "*.env.*": "ask",
    "*.env.example": "allow"
  }
}
```

**特性 / Features**:
- 完整的文件操作权限
- 可以执行系统命令
- 可以进入 plan 模式
- 可以询问用户问题

---

### 2. Plan Agent

**模式 / Mode**: `primary`

**用途 / Purpose**: 受限代理，用于规划和分析，不进行实际修改

**权限 / Permissions**:
```typescript
{
  "*": "allow",
  "doom_loop": "ask",
  "question": "allow",
  "plan_exit": "allow",
  "edit": {
    "*": "deny",
    ".opencode/plans/*.md": "allow",
    "{Global.Path.data}/plans/*.md": "allow"
  },
  "external_directory": {
    "{Global.Path.data}/plans/*": "allow"
  }
}
```

**特性 / Features**:
- 只能编辑计划文件
- 可以退出 plan 模式到 build 模式
- 用于分析代码和建议更改而不实际修改

---

### 3. General Subagent

**模式 / Mode**: `subagent`

**用途 / Purpose**: 通用子代理，用于研究复杂问题和执行多步骤任务

**描述 / Description**:
```
General-purpose agent for researching complex questions and executing 
multi-step tasks. Use this agent to execute multiple units of work in parallel.
```

**权限 / Permissions**:
```typescript
{
  "*": "allow",
  "doom_loop": "ask",
  "todoread": "deny",
  "todowrite": "deny",
  "question": "deny",
  "plan_enter": "deny",
  "plan_exit": "deny"
}
```

**特性 / Features**:
- 全工具访问（除了 todo 工具）
- 可以进行文件更改
- 用于并行运行多个工作单元

---

### 4. Explore Subagent

**模式 / Mode**: `subagent`

**用途 / Purpose**: 快速、只读的代理，用于探索代码库

**描述 / Description**:
```
Fast agent specialized for exploring codebases. Use this when you need to 
quickly find files by patterns (eg. "src/components/**/*.tsx"), search code 
for keywords (eg. "API endpoints"), or answer questions about the codebase 
(eg. "how do API endpoints work?"). When calling this agent, specify the 
desired thoroughness level: "quick" for basic searches, "medium" for moderate 
exploration, or "very thorough" for comprehensive analysis.
```

**提示词文件 / Prompt File**: `/packages/opencode/src/agent/prompt/explore.txt`

**提示词内容 / Prompt Content**:
```
You are a file search specialist. You excel at thoroughly navigating and 
exploring codebases.

Your strengths:
- Rapidly finding files using glob patterns
- Searching code and text with powerful regex patterns
- Reading and analyzing file contents

Guidelines:
- Use Glob for broad file pattern matching
- Use Grep for searching file contents with regex
- Use Read when you know the specific file path you need to read
- Use Bash for file operations like copying, moving, or listing directory contents
- Adapt your search approach based on the thoroughness level specified by the caller
- Return file paths as absolute paths in your final response
- For clear communication, avoid using emojis
- Do not create any files, or run bash commands that modify the user's system 
  state in any way
```

**权限 / Permissions**:
```typescript
{
  "*": "deny",
  "grep": "allow",
  "glob": "allow",
  "list": "allow",
  "bash": "allow",
  "webfetch": "allow",
  "websearch": "allow",
  "codesearch": "allow",
  "read": "allow",
  "external_directory": {
    "{Truncate.DIR}": "allow",
    "{Truncate.GLOB}": "allow"
  }
}
```

**特性 / Features**:
- 只读访问
- 不能修改文件
- 专注于搜索和探索
- 支持可配置的彻底性级别（quick、medium、very thorough）

---

### 5. Compaction Agent（隐藏）

**模式 / Mode**: `primary`

**隐藏 / Hidden**: `true`

**用途 / Purpose**: 会话压缩和摘要

**提示词文件 / Prompt File**: `/packages/opencode/src/agent/prompt/compaction.txt`

**提示词内容 / Prompt Content**:
```
You are a helpful AI assistant tasked with summarizing conversations.

When asked to summarize, provide a detailed but concise summary of the 
conversation. Focus on information that would be helpful for continuing 
the conversation, including:
- What was done
- What is currently being worked on
- Which files are being modified
- What needs to be done next
- Key user requests, constraints, or preferences that should persist
- Important technical decisions and why they were made

Your summary should be comprehensive enough to provide context but concise 
enough to be quickly understood.
```

**权限 / Permissions**:
```typescript
{
  "*": "deny"
}
```

---

### 6. Title Generator Agent（隐藏）

**模式 / Mode**: `primary`

**隐藏 / Hidden**: `true`

**用途 / Purpose**: 生成会话标题

**提示词文件 / Prompt File**: `/packages/opencode/src/agent/prompt/title.txt`

**提示词内容 / Prompt Content**:
```
You are a title generator. You output ONLY a thread title. Nothing else.

<task>
Generate a brief title that would help the user find this conversation later.

Follow all rules in <rules>
Use the <examples> so you know what a good title looks like.
Your output must be:
- A single line
- ≤50 characters
- No explanations
</task>

<rules>
- you MUST use the same language as the user message you are summarizing
- Title must be grammatically correct and read naturally - no word salad
- Never include tool names in the title (e.g. "read tool", "bash tool", "edit tool")
- Focus on the main topic or question the user needs to retrieve
- Vary your phrasing - avoid repetitive patterns like always starting with "Analyzing"
- When a file is mentioned, focus on WHAT the user wants to do WITH the file, 
  not just that they shared it
- Keep exact: technical terms, numbers, filenames, HTTP codes
- Remove: the, this, my, a, an
- Never assume tech stack
- Never use tools
- NEVER respond to questions, just generate a title for the conversation
- The title should NEVER include "summarizing" or "generating" when generating a title
- DO NOT SAY YOU CANNOT GENERATE A TITLE OR COMPLAIN ABOUT THE INPUT
- Always output something meaningful, even if the input is minimal.
- If the user message is short or conversational (e.g. "hello", "lol", "what's up", "hey"):
  → create a title that reflects the user's tone or intent (such as Greeting, 
    Quick check-in, Light chat, Intro message, etc.)
</rules>
```

**权限 / Permissions**:
```typescript
{
  "*": "deny"
}
```

**温度 / Temperature**: `0.5`

---

### 7. Summary Agent（隐藏）

**模式 / Mode**: `primary`

**隐藏 / Hidden**: `true`

**用途 / Purpose**: 生成会话摘要（用于 PR 描述等）

**提示词文件 / Prompt File**: `/packages/opencode/src/agent/prompt/summary.txt`

**提示词内容 / Prompt Content**:
```
Summarize what was done in this conversation. Write like a pull request description.

Rules:
- 2-3 sentences max
- Describe the changes made, not the process
- Do not mention running tests, builds, or other validation steps
- Do not explain what the user asked for
- Write in first person (I added..., I fixed...)
- Never ask questions or add new questions
- If the conversation ends with an unanswered question to the user, preserve 
  that exact question
- If the conversation ends with an imperative statement or request to the user 
  (e.g. "Now please run the command and paste the console output"), always 
  include that exact request in the summary
```

**权限 / Permissions**:
```typescript
{
  "*": "deny"
}
```

---

## Task 工具 / Task Tool

Task 工具是启动 subagents 执行特定任务的核心机制。

The Task tool is the core mechanism for launching subagents to perform specific tasks.

### 工具定义 / Tool Definition

**位置 / Location**: `/packages/opencode/src/tool/task.ts`

**参数 / Parameters**:
```typescript
{
  description: string,      // 任务的简短描述（3-5 词）
  prompt: string,           // agent 要执行的任务
  subagent_type: string,    // 要使用的专门 agent 类型
  session_id?: string,      // 要继续的现有 Task 会话（可选）
  command?: string          // 触发此任务的命令（可选）
}
```

### 提示词文件 / Prompt File

**位置 / Location**: `/packages/opencode/src/tool/task.txt`

**内容 / Content**:
```
Launch a new agent to handle complex, multistep tasks autonomously.

Available agent types and the tools they have access to:
{agents}

When using the Task tool, you must specify a subagent_type parameter to select 
which agent type to use.

When to use the Task tool:
- When you are instructed to execute custom slash commands. Use the Task tool 
  with the slash command invocation as the entire prompt. The slash command can 
  take arguments. For example: Task(description="Check the file", 
  prompt="/check-file path/to/file.py")

When NOT to use the Task tool:
- If you want to read a specific file path, use the Read or Glob tool instead 
  of the Task tool, to find the match more quickly
- If you are searching for a specific class definition like "class Foo", use 
  the Glob tool instead, to find the match more quickly
- If you are searching for code within a specific file or set of 2-3 files, 
  use the Read tool instead of the Task tool, to find the match more quickly
- Other tasks that are not related to the agent descriptions above

Usage notes:
1. Launch multiple agents concurrently whenever possible, to maximize performance; 
   to do that, use a single message with multiple tool uses
2. When the agent is done, it will return a single message back to you. The result 
   returned by the agent is not visible to the user. To show the user the result, 
   you should send a text message back to the user with a concise summary of the result.
3. Each agent invocation is stateless unless you provide a session_id. Your prompt 
   should contain a highly detailed task description for the agent to perform 
   autonomously and you should specify exactly what information the agent should 
   return back to you in its final and only message to you.
4. The agent's outputs should generally be trusted
5. Clearly tell the agent whether you expect it to write code or just to do research 
   (search, file reads, web fetches, etc.), since it is not aware of the user's intent
6. If the agent description mentions that it should be used proactively, then you 
   should try your best to use it without the user having to ask for it first. 
   Use your judgement.
```

### Task 工具工作流程 / Task Tool Workflow

1. **权限检查 / Permission Check**:
   - 检查调用 agent 是否有权限启动请求的 subagent
   - 可以通过 `bypassAgentCheck` 标志跳过（用于用户明确调用）

2. **会话创建 / Session Creation**:
   - 创建新的子会话或重用现有会话
   - 设置父会话 ID（`parentID`）
   - 配置权限规则：
     - 禁用 `todowrite` 和 `todoread`
     - 可选禁用 `task`（取决于 subagent 配置）
     - 允许实验性的 primary 工具

3. **执行 / Execution**:
   - 启动 subagent 会话
   - 使用 `SessionPrompt.prompt()` 执行提示
   - 监听工具调用更新
   - 收集执行摘要

4. **结果返回 / Result Return**:
   - 返回 subagent 的文本响应
   - 附加任务元数据（包括 session_id）
   - 提供工具调用摘要

### Task 权限控制 / Task Permission Control

在 agent 配置中可以控制哪些 subagents 可以被调用：

In agent configuration, you can control which subagents can be invoked:

```json
{
  "agent": {
    "orchestrator": {
      "mode": "primary",
      "permission": {
        "task": {
          "*": "deny",                    // 默认拒绝所有
          "orchestrator-*": "allow",      // 允许特定前缀
          "code-reviewer": "ask"          // 需要确认
        }
      }
    }
  }
}
```

**规则评估 / Rule Evaluation**:
- 规则按顺序评估
- **最后匹配的规则获胜**
- 用户始终可以通过 `@` 提及直接调用任何 subagent

---

## Agent 配置 / Agent Configuration

Agents 可以通过两种方式配置：

Agents can be configured in two ways:

### 1. JSON 配置 / JSON Configuration

在 `opencode.json` 中配置：

Configure in `opencode.json`:

```json
{
  "$schema": "https://opencode.ai/config.json",
  "agent": {
    "build": {
      "mode": "primary",
      "model": "anthropic/claude-sonnet-4-20250514",
      "prompt": "{file:./prompts/build.txt}",
      "temperature": 0.3,
      "steps": 100,
      "permission": {
        "edit": "allow",
        "bash": "ask"
      }
    },
    "code-reviewer": {
      "description": "Reviews code for best practices and potential issues",
      "mode": "subagent",
      "model": "anthropic/claude-sonnet-4-20250514",
      "prompt": "You are a code reviewer. Focus on security, performance, and maintainability.",
      "hidden": false,
      "permission": {
        "edit": "deny",
        "bash": "deny"
      }
    }
  }
}
```

### 2. Markdown 配置 / Markdown Configuration

放置在以下位置：

Place in the following locations:

- **全局 / Global**: `~/.config/opencode/agents/`
- **项目 / Project**: `.opencode/agents/`

```markdown
---
description: Reviews code for quality and best practices
mode: subagent
model: anthropic/claude-sonnet-4-20250514
temperature: 0.1
color: "#FF5733"
hidden: false
permission:
  edit: deny
  bash:
    "*": deny
    "git diff": allow
    "git log*": allow
---

You are in code review mode. Focus on:

- Code quality and best practices
- Potential bugs and edge cases
- Performance implications
- Security considerations

Provide constructive feedback without making direct changes.
```

**文件名即为 agent 名称** / **Filename becomes agent name**

### 配置选项详解 / Configuration Options Details

#### description（必需 / Required）
```json
{
  "description": "Reviews code for best practices and potential issues"
}
```
- 简要描述 agent 的功能和使用场景
- 用于 Task 工具的 agent 列表

#### mode
```json
{
  "mode": "primary" | "subagent" | "all"
}
```
- `primary`: 仅作为主代理
- `subagent`: 仅作为子代理
- `all`: 两者都可以（默认）

#### model
```json
{
  "model": "anthropic/claude-sonnet-4-20250514"
}
```
- 格式：`provider/model-id`
- Primary agents 默认使用全局配置的模型
- Subagents 默认使用调用它们的 primary agent 的模型

#### prompt
```json
{
  "prompt": "{file:./prompts/code-review.txt}"
}
```
- 自定义系统提示文件
- 路径相对于配置文件所在位置

#### temperature
```json
{
  "temperature": 0.1
}
```
- 范围：0.0-1.0
- `0.0-0.2`: 专注和确定性（分析、规划）
- `0.3-0.5`: 平衡（一般开发）
- `0.6-1.0`: 创造性（头脑风暴）

#### steps (maxSteps)
```json
{
  "steps": 5
}
```
- 限制 agentic 迭代的最大次数
- 达到限制时，agent 收到特殊系统提示
- 用于控制成本

#### hidden
```json
{
  "hidden": true
}
```
- 从 `@` 自动完成菜单中隐藏
- 仅适用于 `mode: subagent`
- 仍可通过 Task 工具调用

#### color
```json
{
  "color": "#FF5733"
}
```
- UI 中的颜色显示
- 十六进制颜色代码

#### disable
```json
{
  "disable": true
}
```
- 禁用 agent

#### permission
```json
{
  "permission": {
    "edit": "ask",
    "bash": {
      "*": "ask",
      "git status *": "allow"
    },
    "webfetch": "deny",
    "task": {
      "*": "deny",
      "explore": "allow"
    }
  }
}
```
- `allow`: 自动允许
- `deny`: 拒绝
- `ask`: 需要用户确认
- 支持 glob 模式
- 最后匹配的规则获胜

#### options（额外选项）
```json
{
  "reasoningEffort": "high",
  "textVerbosity": "low"
}
```
- 特定于模型和提供商的参数
- 直接传递给提供商
- 示例：OpenAI 推理模型的 `reasoningEffort`

---

## Agent 定义接口 / Agent Definition Interface

**位置 / Location**: `/packages/opencode/src/agent/agent.ts`

### Agent.Info 接口 / Agent.Info Interface

```typescript
interface Agent.Info {
  name: string                  // Agent 名称
  description?: string          // 描述（用于 Task 工具）
  mode: "subagent" | "primary" | "all"  // Agent 模式
  native?: boolean              // 是否为内置 agent
  hidden?: boolean              // 是否在 UI 中隐藏
  topP?: number                 // Top-p 采样参数
  temperature?: number          // 温度参数
  color?: string                // UI 颜色
  permission: PermissionNext.Ruleset  // 权限规则集
  model?: {                     // 可选的模型覆盖
    modelID: string
    providerID: string
  }
  prompt?: string               // 自定义系统提示
  options: Record<string, any>  // 额外的模型选项
  steps?: number                // 最大步骤数
}
```

### Agent 核心函数 / Agent Core Functions

```typescript
// 获取特定 agent
await Agent.get(agentName: string): Promise<Agent.Info | undefined>

// 列出所有 agents
await Agent.list(): Promise<Agent.Info[]>

// 生成新 agent
await Agent.generate({
  description: string,
  model?: { providerID: string, modelID: string }
}): Promise<{
  identifier: string,
  whenToUse: string,
  systemPrompt: string
}>
```

---

## 创建自定义 Agent / Creating Custom Agents

### 使用命令行创建 / Create via CLI

```bash
opencode agent create
```

**交互式过程 / Interactive Process**:
1. 选择位置（全局或项目）
2. 提供描述
3. 自动生成系统提示和标识符
4. 选择可用工具
5. 创建 markdown 文件

**非交互式创建 / Non-interactive Creation**:
```bash
opencode agent create \
  --path .opencode \
  --description "Reviews code for best practices" \
  --mode subagent \
  --tools "read,grep,glob,bash" \
  --model "anthropic/claude-sonnet-4-20250514"
```

### Agent 生成器 / Agent Generator

**提示词文件 / Prompt File**: `/packages/opencode/src/agent/generate.txt`

Agent 生成器使用 LLM 根据用户描述创建 agent 配置：

The agent generator uses an LLM to create agent configurations based on user descriptions:

**输入 / Input**:
- `description`: 用户对 agent 功能的描述
- `model`（可选）: 使用的模型

**输出 / Output** (JSON):
```json
{
  "identifier": "code-reviewer",
  "whenToUse": "Use this agent when...",
  "systemPrompt": "You are a code reviewer..."
}
```

**生成器指令 / Generator Instructions** (简要):
1. 提取核心意图
2. 设计专家角色
3. 构建全面指令
4. 优化性能
5. 创建标识符
6. 提供使用示例

---

## 使用场景和最佳实践 / Use Cases and Best Practices

### 1. 何时使用 Primary Agents / When to Use Primary Agents

**Build Agent**:
- ✅ 完整的开发工作
- ✅ 需要文件操作和系统命令
- ✅ 编写代码、修复 bug、添加功能

**Plan Agent**:
- ✅ 分析和规划
- ✅ 代码审查
- ✅ 建议更改而不实际修改
- ✅ 设计讨论

### 2. 何时使用 Subagents / When to Use Subagents

**Explore Subagent**:
- ✅ 快速查找文件
- ✅ 搜索代码模式
- ✅ 回答关于代码库的问题
- ✅ 不需要修改文件

**General Subagent**:
- ✅ 复杂的多步骤任务
- ✅ 需要并行执行多个工作单元
- ✅ 研究和实现结合

### 3. Task 工具使用最佳实践 / Task Tool Best Practices

**何时使用 / When to Use**:
```typescript
// ✅ 复杂的多步骤任务
Task({
  description: "Analyze security",
  prompt: "Analyze the authentication module for security vulnerabilities",
  subagent_type: "security-auditor"
})

// ✅ 并行执行多个任务
// 在单个消息中调用多个 Task 工具
Task({ ... }) // 任务 1
Task({ ... }) // 任务 2
Task({ ... }) // 任务 3
```

**何时不使用 / When NOT to Use**:
```typescript
// ❌ 简单的文件读取
// 使用 Read 工具而不是 Task
Read({ filePath: "/path/to/file.ts" })

// ❌ 简单的搜索
// 使用 Grep/Glob 而不是 Task
Grep({ pattern: "function foo" })
```

### 4. Agent 导航 / Agent Navigation

**切换 Primary Agents / Switch Primary Agents**:
- 使用 **Tab** 键
- 或使用配置的 `switch_agent` 快捷键

**调用 Subagents / Invoke Subagents**:
- **自动**: Primary agent 根据描述自动调用
- **手动**: 使用 `@agent-name` 提及

**会话导航 / Session Navigation**:
- **`<Leader>+Right`** (或 `session_child_cycle`): 向前循环
- **`<Leader>+Left`** (或 `session_child_cycle_reverse`): 向后循环
- 在父会话和子会话之间切换

### 5. 自定义 Agent 示例 / Custom Agent Examples

**文档编写 Agent**:
```markdown
---
description: Writes and maintains project documentation
mode: subagent
permission:
  bash: deny
---

You are a technical writer. Create clear, comprehensive documentation.

Focus on:
- Clear explanations
- Proper structure
- Code examples
- User-friendly language
```

**安全审计 Agent**:
```markdown
---
description: Performs security audits and identifies vulnerabilities
mode: subagent
permission:
  edit: deny
  bash: deny
---

You are a security expert. Focus on identifying potential security issues.

Look for:
- Input validation vulnerabilities
- Authentication and authorization flaws
- Data exposure risks
- Dependency vulnerabilities
```

**调试 Agent**:
```json
{
  "debug": {
    "description": "Focused debugging with investigation tools",
    "mode": "subagent",
    "permission": {
      "edit": "deny",
      "bash": "allow",
      "read": "allow"
    }
  }
}
```

### 6. 权限配置最佳实践 / Permission Configuration Best Practices

**最小权限原则 / Principle of Least Privilege**:
```json
{
  "agent": {
    "readonly-analyzer": {
      "permission": {
        "*": "deny",           // 默认拒绝所有
        "read": "allow",       // 仅允许需要的工具
        "grep": "allow",
        "glob": "allow"
      }
    }
  }
}
```

**Bash 命令细粒度控制 / Fine-grained Bash Control**:
```json
{
  "permission": {
    "bash": {
      "*": "ask",              // 默认询问
      "git status": "allow",   // 允许安全命令
      "git log*": "allow",
      "git push": "deny"       // 拒绝危险命令
    }
  }
}
```

### 7. 性能优化 / Performance Optimization

**并行 Task 调用 / Parallel Task Calls**:
```typescript
// ✅ 在单个消息中并行启动多个 agents
[
  Task({ description: "Task 1", ... }),
  Task({ description: "Task 2", ... }),
  Task({ description: "Task 3", ... })
]
```

**会话重用 / Session Reuse**:
```typescript
// ✅ 继续现有会话
Task({
  session_id: "previous-session-id",
  prompt: "Continue from where we left off",
  ...
})
```

### 8. 成本控制 / Cost Control

**限制步骤数 / Limit Steps**:
```json
{
  "agent": {
    "cost-conscious": {
      "steps": 10,  // 最大 10 次迭代
      ...
    }
  }
}
```

**使用更轻量的模型 / Use Lighter Models**:
```json
{
  "agent": {
    "quick-responder": {
      "model": "anthropic/claude-haiku-4-20250514",  // 更快更便宜
      ...
    }
  }
}
```

---

## Agent 系统架构图 / Agent System Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                         User                                 │
└────────────────┬────────────────────────────────────────────┘
                 │
                 │ Tab key / @ mention
                 │
┌────────────────▼────────────────────────────────────────────┐
│                  Primary Agents                              │
│  ┌──────────────┐              ┌──────────────┐            │
│  │ Build Agent  │              │ Plan Agent   │            │
│  │ (All tools)  │              │ (Limited)    │            │
│  └──────┬───────┘              └──────┬───────┘            │
└─────────┼───────────────────────────┬─┼─────────────────────┘
          │                           │ │
          │ Task Tool                 │ │ Task Tool
          │                           │ │
┌─────────▼───────────────────────────▼─▼─────────────────────┐
│                     Subagents                                │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐     │
│  │   Explore    │  │   General    │  │   Custom     │     │
│  │ (Read-only)  │  │ (Full tools) │  │  (Configured)│     │
│  └──────────────┘  └──────────────┘  └──────────────┘     │
└──────────────────────────────────────────────────────────────┘
          │                    │                    │
          │                    │                    │
          │                    │                    │
┌─────────▼────────────────────▼────────────────────▼──────────┐
│                         Tools                                 │
│  bash │ read │ edit │ write │ grep │ glob │ webfetch │ ...  │
└───────────────────────────────────────────────────────────────┘
```

---

## 总结 / Summary

OpenCode 的 Agent 和 Task 系统提供了：

OpenCode's Agent and Task system provides:

### 核心功能 / Core Features

1. **灵活的多代理架构** - Primary agents 和 subagents 协同工作
2. **Task 工具** - 强大的 subagent 启动和管理机制
3. **细粒度权限控制** - 精确控制每个 agent 可以执行的操作
4. **易于配置** - JSON 和 Markdown 两种配置方式
5. **可扩展性** - 轻松创建自定义 agents
6. **会话管理** - 父子会话关系和导航
7. **性能优化** - 并行 task 执行和会话重用

### 内置 Agents / Built-in Agents

- **2 个 Primary Agents**: Build（全功能）、Plan（受限）
- **2 个 Subagents**: Explore（只读探索）、General（全功能）
- **3 个隐藏 Agents**: Compaction、Title、Summary

### 最佳实践 / Best Practices

1. **选择合适的 agent 类型** - Primary 用于主要工作，Subagent 用于专门任务
2. **使用 Task 工具进行复杂任务** - 并行执行，提高效率
3. **配置适当的权限** - 遵循最小权限原则
4. **优化性能和成本** - 限制步骤数，选择适当的模型
5. **创建专门的 agents** - 为特定工作流定制 agents

### 文档和资源 / Documentation and Resources

- **Agent 文档**: `/packages/web/src/content/docs/agents.mdx`
- **配置示例**: `.opencode/agent/*.md`
- **核心实现**: `/packages/opencode/src/agent/agent.ts`
- **Task 工具**: `/packages/opencode/src/tool/task.ts`

---

**文档生成时间 / Document Generated**: 2026-01-23

**仓库 / Repository**: DuYooho/opencode

**分支 / Branch**: dev

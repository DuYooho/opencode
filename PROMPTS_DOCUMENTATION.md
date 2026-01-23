# OpenCode 提示词（Prompt）完整文档
# OpenCode Prompts Complete Documentation

本文档汇总了 OpenCode 仓库中所有与提示词（prompt）相关的信息，包括提示词的具体内容、使用场景和配置方式。

This document summarizes all prompt-related information in the OpenCode repository, including prompt content, usage scenarios, and configuration methods.

---

## 目录 / Table of Contents

1. [主要提示词文件 / Main Prompt Files](#主要提示词文件--main-prompt-files)
2. [Agent 相关提示词 / Agent-Related Prompts](#agent-相关提示词--agent-related-prompts)
3. [工具相关提示词 / Tool-Related Prompts](#工具相关提示词--tool-related-prompts)
4. [提示词配置和使用 / Prompt Configuration and Usage](#提示词配置和使用--prompt-configuration-and-usage)

---

## 主要提示词文件 / Main Prompt Files

OpenCode 的主要提示词存储在 `/packages/opencode/src/session/prompt/` 目录下。该目录包含以下 12 个核心提示词文件：

The main prompts for OpenCode are stored in `/packages/opencode/src/session/prompt/`. This directory contains the following 12 core prompt files:

### 1. anthropic-20250930.txt
- **用途 / Purpose**: Anthropic Claude 模型的最新提示词（2025年9月30日版本）
- **特点 / Features**:
  - 交互式 CLI 工具定位
  - 强调简洁、直接、专业的回复风格
  - 包含任务管理（TodoWrite 工具）指导
  - 详细的工具使用策略
  - 代码引用规范（file_path:line_number 格式）
  - 专业客观性原则
  
### 2. anthropic.txt
- **用途 / Purpose**: 标准的 Anthropic 提示词
- **特点 / Features**:
  - OpenCode 的身份定位："OpenCode, the best coding agent on the planet"
  - 包含任务管理工作流
  - 强调使用 Task 工具进行代码库探索
  - 提供代码引用示例
  
### 3. beast.txt
- **用途 / Purpose**: "Beast" 模式提示词，用于复杂问题求解
- **特点 / Features**:
  - 强调完全自主解决问题
  - 必须进行大量互联网研究
  - 详细的工作流程（URL获取、问题理解、代码库调查、研究、计划、实现、调试、测试）
  - 持续迭代直到完全解决问题
  - 包含记忆管理功能
  
### 4. copilot-gpt-5.txt
- **用途 / Purpose**: GitHub Copilot GPT-5 风格提示词
- **特点 / Features**:
  - 高度复杂的代码助手
  - 结构化工作流（理解、调查、计划、实现、调试）
  - 强调沟通清晰和简洁
  - 避免重复读取已知文件
  - 支持 Markdown 格式的提示词生成
  - Git 操作规范
  
### 5. gemini.txt
- **用途 / Purpose**: Google Gemini 模型提示词
- **特点 / Features**:
  - 交互式 CLI 工具定位
  - 运营指南和安全规则
  - 文件路径必须使用绝对路径
  - 工具并行使用策略
  - 前端任务特别指导
  - 详细的最终答案结构和风格指南
  
### 6. qwen.txt
- **用途 / Purpose**: 通义千问（Qwen）模型提示词
- **特点 / Features**:
  - opencode 交互式 CLI 工具
  - 拒绝恶意代码请求
  - 极简输出风格（少于4行）
  - 提供简洁示例（如 "2+2" → "4"）
  - 强调代码约定和安全实践
  
### 7. plan-reminder-anthropic.txt
- **用途 / Purpose**: Plan 模式系统提醒
- **特点 / Features**:
  - 激活 Plan 模式时的提醒
  - 只允许只读操作
  - 详细的增强规划工作流（5个阶段）
  - Phase 1: 初始理解（使用 Explore agents）
  - Phase 2: 规划（使用 Plan subagent）
  - Phase 3: 综合
  - Phase 4: 最终计划
  - Phase 5: 调用 ExitPlanMode
  
### 8. plan.txt
- **用途 / Purpose**: Plan 模式简短提醒
- **特点 / Features**:
  - 关键提示：Plan 模式激活，只读阶段
  - 严格禁止任何文件编辑、修改或系统变更
  - 绝对约束，覆盖所有其他指令
  
### 9. build-switch.txt
- **用途 / Purpose**: 从 Plan 模式切换到 Build 模式
- **特点 / Features**:
  - 通知操作模式已从 plan 改为 build
  - 不再是只读模式
  - 允许文件更改、运行 shell 命令和使用工具
  
### 10. max-steps.txt
- **用途 / Purpose**: 达到最大步骤数限制时的提示
- **特点 / Features**:
  - 禁用所有工具直到下一次用户输入
  - 只能使用文本响应
  - 必须包含：已达到最大步骤的声明、已完成工作摘要、未完成任务列表、下一步建议
  
### 11. codex_header.txt
- **用途 / Purpose**: OpenCode 的 Codex 风格头部提示词
- **特点 / Features**:
  - 定位为 "OpenCode, the best coding agent on the planet"
  - 编辑约束（默认使用 ASCII、谨慎添加注释）
  - 工具使用偏好（专用工具优于 shell）
  - Git 和工作区清洁度规范
  - 前端任务特殊指导
  - 展示工作和最终消息的格式规范
  
### 12. anthropic_spoof.txt
- **用途 / Purpose**: 伪装成 Claude Code
- **特点 / Features**:
  - 仅一行："You are Claude Code, Anthropic's official CLI for Claude."

---

## Agent 相关提示词 / Agent-Related Prompts

Agent 相关的提示词文件位于 `/packages/opencode/src/agent/` 目录下：

Agent-related prompt files are located in `/packages/opencode/src/agent/`:

### 1. generate.txt
- **位置 / Location**: `/packages/opencode/src/agent/generate.txt`
- **用途 / Purpose**: Agent 生成器提示词
- **功能 / Functions**:
  - 作为精英 AI agent 架构师
  - 将用户需求转换为精确调整的 agent 规范
  - 输出 JSON 格式，包含三个字段：
    - `identifier`: 唯一标识符（小写字母、数字、连字符）
    - `whenToUse`: 何时使用此 agent 的精确描述
    - `systemPrompt`: 完整的系统提示词（以第二人称书写）
  - 考虑项目特定的上下文（如 CLAUDE.md 文件）
  
### 2. Agent Prompt 子目录 / Agent Prompt Subdirectory

位于 `/packages/opencode/src/agent/prompt/`，包含：

Located at `/packages/opencode/src/agent/prompt/`, contains:

#### a. explore.txt
- **用途 / Purpose**: 文件搜索专家 agent
- **特长 / Strengths**:
  - 使用 glob 模式快速查找文件
  - 使用强大的正则表达式搜索代码和文本
  - 读取和分析文件内容
- **指导原则 / Guidelines**:
  - 使用 Glob 进行广泛的文件模式匹配
  - 使用 Grep 搜索文件内容
  - 使用 Read 读取特定文件
  - 返回绝对路径
  - 不创建文件或修改系统状态

#### b. summary.txt
- **用途 / Purpose**: 会话摘要生成
- **规则 / Rules**:
  - 最多 2-3 句话
  - 描述所做的更改，而非过程
  - 不提及运行测试、构建或验证步骤
  - 以第一人称书写（I added..., I fixed...）
  - 如果对话以未回答的问题结束，保留该问题

#### c. title.txt
- **用途 / Purpose**: 会话标题生成器
- **规则 / Rules**:
  - 仅输出标题，不输出其他内容
  - 单行，≤50 个字符
  - 必须使用与用户消息相同的语言
  - 语法正确，自然流畅
  - 不包含工具名称
  - 关注主题或用户需要检索的问题

#### d. compaction.txt
- **用途 / Purpose**: 会话压缩和摘要
- **功能 / Functions**:
  - 提供详细但简洁的会话摘要
  - 关注对继续对话有帮助的信息：
    - 已完成的工作
    - 当前正在进行的工作
    - 正在修改的文件
    - 下一步需要做的事
    - 关键的用户请求、约束或偏好
    - 重要的技术决策及原因

---

## 工具相关提示词 / Tool-Related Prompts

### 1. task.txt
- **位置 / Location**: `/packages/opencode/src/tool/task.txt`
- **用途 / Purpose**: Task 工具的使用说明
- **内容 / Contents**:
  - 启动新 agent 处理复杂的多步骤任务
  - 可用的 agent 类型及其可用工具
  - 何时使用 Task 工具（包括自定义 slash 命令）
  - 何时不使用 Task 工具（直接读取文件、搜索特定类等）
  - 使用说明：
    - 并发启动多个 agents 以最大化性能
    - 每个 agent 调用是无状态的（除非提供 session_id）
    - Agent 的输出通常应被信任
    - 明确告诉 agent 是写代码还是只做研究
  - 示例 agents（虚构示例用于说明）：
    - "code-reviewer": 完成重要代码后使用
    - "greeting-responder": 响应用户问候

---

## 提示词配置和使用 / Prompt Configuration and Usage

### 代码中的提示词引用 / Prompt References in Code

在 `/packages/opencode/src/session/prompt.ts` 文件中，这些提示词被引入和使用：

In `/packages/opencode/src/session/prompt.ts`, these prompts are imported and used:

```typescript
import PROMPT_PLAN from "../session/prompt/plan.txt"
import BUILD_SWITCH from "../session/prompt/build-switch.txt"
import MAX_STEPS from "../session/prompt/max-steps.txt"
```

### Agent 配置 / Agent Configuration

在 `/packages/opencode/src/agent/agent.ts` 中，Agent 的配置包括：

In `/packages/opencode/src/agent/agent.ts`, Agent configuration includes:

- `name`: Agent 名称
- `description`: 描述（可选）
- `mode`: 模式（"subagent", "primary", "all"）
- `native`: 是否为原生 agent
- `hidden`: 是否隐藏
- `topP`, `temperature`: 模型参数
- `color`: 颜色
- `permission`: 权限规则集
- `model`: 模型配置（modelID, providerID）
- `prompt`: 自定义提示词
- `options`: 其他选项
- `steps`: 最大步骤数

### 系统提示词生成 / System Prompt Generation

系统提示词的生成使用了 `ai` SDK，在 `/packages/opencode/src/session/system.ts` 中实现。

System prompt generation uses the `ai` SDK, implemented in `/packages/opencode/src/session/system.ts`.

### 权限和提示词 / Permissions and Prompts

默认权限配置示例（来自 `agent.ts`）：

Default permission configuration example (from `agent.ts`):

```typescript
const defaults = PermissionNext.fromConfig({
  "*": "allow",
  doom_loop: "ask",
  external_directory: {
    "*": "ask",
    [Truncate.DIR]: "allow",
    [Truncate.GLOB]: "allow",
  },
  question: "deny",
  plan_enter: "deny",
  plan_exit: "deny",
  read: {
    "*": "allow",
    "*.env": "ask",
    "*.env.*": "ask",
    "*.env.example": "allow",
  },
})
```

---

## 提示词文件位置总结 / Summary of Prompt File Locations

### 主要提示词 / Main Prompts
```
/packages/opencode/src/session/prompt/
├── anthropic-20250930.txt
├── anthropic.txt
├── anthropic_spoof.txt
├── beast.txt
├── build-switch.txt
├── codex_header.txt
├── copilot-gpt-5.txt
├── gemini.txt
├── max-steps.txt
├── plan-reminder-anthropic.txt
├── plan.txt
└── qwen.txt
```

### Agent 提示词 / Agent Prompts
```
/packages/opencode/src/agent/
├── generate.txt
└── prompt/
    ├── compaction.txt
    ├── explore.txt
    ├── summary.txt
    └── title.txt
```

### 工具提示词 / Tool Prompts
```
/packages/opencode/src/tool/
└── task.txt
```

---

## 提示词设计原则 / Prompt Design Principles

根据这些提示词文件，OpenCode 的提示词设计遵循以下原则：

Based on these prompt files, OpenCode's prompt design follows these principles:

1. **简洁性 / Conciseness**: 强调简短、直接的回复，避免冗余
2. **专业性 / Professionalism**: 客观、准确、技术性强
3. **结构化 / Structure**: 使用清晰的工作流和步骤
4. **工具优先 / Tool-First**: 优先使用专用工具而非通用命令
5. **安全性 / Security**: 强调安全最佳实践，拒绝恶意代码
6. **可追溯性 / Traceability**: 使用 file_path:line_number 格式引用代码
7. **适应性 / Adaptability**: 根据不同模型和场景调整提示词
8. **自主性 / Autonomy**: 鼓励 agent 自主解决问题，减少询问
9. **迭代性 / Iterative**: 支持持续迭代直到问题完全解决
10. **上下文感知 / Context-Aware**: 考虑项目特定的上下文和约定

---

## 相关文档 / Related Documentation

- **ACP (Agent Client Protocol)**: `/packages/opencode/src/acp/README.md`
- **代理配置**: `/packages/opencode/src/agent/agent.ts`
- **会话管理**: `/packages/opencode/src/session/`
- **工具注册**: `/packages/opencode/src/tool/registry.ts`

---

## 使用示例 / Usage Examples

### 如何自定义 Agent 提示词 / How to Customize Agent Prompts

1. 在配置中指定自定义提示词：
   Specify custom prompt in configuration:

```typescript
{
  name: "custom-agent",
  prompt: "Your custom prompt here...",
  mode: "primary",
  permission: { /* ... */ }
}
```

2. 或者创建新的 .txt 文件并在代码中引入：
   Or create a new .txt file and import it in code:

```typescript
import CUSTOM_PROMPT from "./path/to/custom-prompt.txt"
```

### 如何选择合适的提示词 / How to Choose the Right Prompt

- **Anthropic Claude**: 使用 `anthropic-20250930.txt` 或 `anthropic.txt`
- **Google Gemini**: 使用 `gemini.txt`
- **通义千问**: 使用 `qwen.txt`
- **GitHub Copilot 风格**: 使用 `copilot-gpt-5.txt`
- **复杂问题求解**: 使用 `beast.txt`
- **规划阶段**: 使用 `plan-reminder-anthropic.txt` 或 `plan.txt`

---

## 总结 / Summary

OpenCode 拥有一套完整的提示词系统，包括：

OpenCode has a complete prompt system, including:

- **12 个主要提示词文件**，针对不同模型和场景优化
- **5 个 Agent 专用提示词**，用于特定任务（探索、摘要、标题生成等）
- **1 个工具提示词**，指导 Task 工具的使用
- **灵活的配置系统**，支持自定义提示词和权限

这些提示词共同构建了 OpenCode 作为先进代码助手的核心能力，使其能够：
- 理解复杂的编程任务
- 自主探索和分析代码库
- 生成高质量的代码
- 遵循最佳实践和安全规范
- 与用户进行自然、高效的交互

These prompts together build OpenCode's core capabilities as an advanced code assistant, enabling it to:
- Understand complex programming tasks
- Autonomously explore and analyze codebases
- Generate high-quality code
- Follow best practices and security standards
- Interact with users naturally and efficiently

---

**文档生成时间 / Document Generated**: 2026-01-23

**仓库 / Repository**: DuYooho/opencode

**分支 / Branch**: dev

# OpenCode 模型特定的系统提示词渲染 / Model-Specific System Prompt Rendering

## 概述 / Overview

OpenCode 为不同的模型使用不同的系统提示词（system prompts）。这个文档详细说明了不同模型如何渲染系统提示词，以及 Qwen3-coder 和 Qwen3 系列之间的差异。

OpenCode uses different system prompts for different models. This document details how different models render system prompts, including the differences between Qwen3-coder and Qwen3 series.

---

## 系统提示词选择逻辑 / System Prompt Selection Logic

### 代码位置 / Code Location

**主要文件 / Main File**: `/packages/opencode/src/session/system.ts`

```typescript
export namespace SystemPrompt {
  export function header(providerID: string) {
    if (providerID.includes("anthropic")) return [PROMPT_ANTHROPIC_SPOOF.trim()]
    return []
  }

  export function instructions() {
    return PROMPT_CODEX.trim()
  }

  export function provider(model: Provider.Model) {
    if (model.api.id.includes("gpt-5")) return [PROMPT_CODEX]
    if (model.api.id.includes("gpt-") || model.api.id.includes("o1") || model.api.id.includes("o3"))
      return [PROMPT_BEAST]
    if (model.api.id.includes("gemini-")) return [PROMPT_GEMINI]
    if (model.api.id.includes("claude")) return [PROMPT_ANTHROPIC]
    return [PROMPT_ANTHROPIC_WITHOUT_TODO]  // 这是 qwen.txt！
  }
}
```

### 关键发现 / Key Findings

**重要**: `PROMPT_ANTHROPIC_WITHOUT_TODO` 实际上是从 `qwen.txt` 导入的！

**IMPORTANT**: `PROMPT_ANTHROPIC_WITHOUT_TODO` is actually imported from `qwen.txt`!

```typescript
import PROMPT_ANTHROPIC_WITHOUT_TODO from "./prompt/qwen.txt"
```

这意味着：
- Qwen3, Qwen3-coder 和其他**不匹配特定模式的模型**都使用 `qwen.txt` 提示词
- 这个命名有些误导性 - 它不只是给 Qwen 用的，而是默认的"通用"提示词

This means:
- Qwen3, Qwen3-coder, and any **models that don't match specific patterns** all use the `qwen.txt` prompt
- The naming is somewhat misleading - it's not just for Qwen, but the default "generic" prompt

---

## 提示词模板对比 / Prompt Template Comparison

### 可用的提示词模板 / Available Prompt Templates

OpenCode 有以下提示词模板文件：

OpenCode has the following prompt template files:

```
./packages/opencode/src/session/prompt/
├── anthropic.txt           # Claude 模型使用 / Used by Claude models
├── anthropic_spoof.txt     # Anthropic header
├── beast.txt               # GPT-3.5/4, O1, O3 模型使用 / Used by GPT-3.5/4, O1, O3
├── codex_header.txt        # GPT-5 和 Codex 会话使用 / Used by GPT-5 and Codex sessions
├── gemini.txt              # Gemini 模型使用 / Used by Gemini models
├── qwen.txt                # Qwen 和其他默认模型使用 / Used by Qwen and other default models
├── plan.txt                # Plan 模式
├── build-switch.txt        # 构建切换
└── max-steps.txt           # 最大步数提醒
```

---

### 提示词内容差异 / Prompt Content Differences

#### 1. **qwen.txt** (Qwen 和默认模型 / Qwen and Default Models)

**特点 / Characteristics**:
- ✅ 非常简洁和直接 / Very concise and direct
- ✅ 没有 TodoWrite 工具的指令 / No TodoWrite tool instructions
- ✅ 强调简短回复（少于 4 行）/ Emphasizes short responses (fewer than 4 lines)
- ✅ 提供了明确的回复示例 / Provides clear response examples
- ✅ 注重代码风格一致性 / Focuses on code style consistency

**核心内容摘要 / Core Content Summary**:
```
You are opencode, an interactive CLI tool...

# Tone and style
- Be concise, direct, and to the point
- IMPORTANT: Keep responses short (fewer than 4 lines)
- Avoid preamble or postamble
- One word answers are best

# Code style
- IMPORTANT: DO NOT ADD ANY COMMENTS unless asked

# Tool usage policy
- Prefer Task tool for file search
- Batch tool calls together
```

#### 2. **anthropic.txt** (Claude 模型 / Claude Models)

**特点 / Characteristics**:
- ✅ 包含 TodoWrite 工具的详细指令 / Includes detailed TodoWrite tool instructions
- ✅ 强调任务管理和规划 / Emphasizes task management and planning
- ✅ 更详细的工作流程说明 / More detailed workflow instructions
- ✅ 包含专业客观性指南 / Includes professional objectivity guidance

**核心内容摘要 / Core Content Summary**:
```
You are OpenCode, the best coding agent on the planet.

# Task Management
- Use TodoWrite tools VERY frequently
- Critical to track tasks and give visibility
- Mark todos as completed as soon as done
- Break down complex tasks into smaller steps

# Professional objectivity
- Prioritize technical accuracy over validation
- Focus on facts, provide objective technical info
- Apply rigorous standards, disagree when necessary

# Tool usage policy
- VERY IMPORTANT: Use Task tool instead of running search commands directly
```

**主要区别 / Key Differences**:
1. **TodoWrite 工具**: Claude 版本强制要求使用 TodoWrite，qwen.txt 版本没有提及
2. **任务规划**: Claude 版本有详细的任务规划示例，qwen.txt 版本更简洁
3. **专业态度**: Claude 版本强调"best coding agent"和专业客观性

1. **TodoWrite Tool**: Claude version mandates TodoWrite, qwen.txt doesn't mention it
2. **Task Planning**: Claude version has detailed task planning examples, qwen.txt is more concise
3. **Professional Tone**: Claude version emphasizes "best coding agent" and professional objectivity

#### 3. **beast.txt** (GPT-3.5/4, O1, O3 模型 / GPT-3.5/4, O1, O3 Models)

**特点 / Characteristics**:
- 类似 anthropic.txt 但针对 GPT 模型优化
- 可能包含特定于 OpenAI 模型的指令

Similar to anthropic.txt but optimized for GPT models, may include OpenAI-specific instructions.

#### 4. **gemini.txt** (Gemini 模型 / Gemini Models)

**特点 / Characteristics**:
- 针对 Google Gemini 模型优化
- 可能包含特定于 Gemini 的格式要求

Optimized for Google Gemini models, may include Gemini-specific formatting requirements.

#### 5. **codex_header.txt** (GPT-5 和 Codex / GPT-5 and Codex)

**特点 / Characteristics**:
- 用于 GPT-5 模型
- Codex 会话（OpenAI OAuth）使用 `instructions` 字段而不是系统消息
- 包含前端设计、Git 和工作空间卫生的额外指令

Used for GPT-5 models, Codex sessions (OpenAI OAuth) use the `instructions` field instead of system messages, includes additional instructions for frontend design, Git, and workspace hygiene.

**核心内容摘要 / Core Content Summary**:
```
You are OpenCode, the best coding agent on the planet.

## Editing constraints
- Default to ASCII when editing files
- Only add comments if necessary

## Tool usage
- Prefer specialized tools over shell

## Git and workspace hygiene
- NEVER revert existing changes unless requested
- Do not amend commits unless requested
- NEVER use destructive commands

## Frontend tasks
- Avoid bland, generic layouts
- Use expressive fonts (not Inter, Roboto, Arial)
- Choose clear visual direction
```

---

## Qwen3-coder vs Qwen3 的区别 / Differences Between Qwen3-coder and Qwen3

### 系统提示词 / System Prompts

**重要发现 / Important Finding**: 
Qwen3-coder 和 Qwen3 **使用相同的系统提示词模板** (`qwen.txt`)！

Qwen3-coder and Qwen3 **use the same system prompt template** (`qwen.txt`)!

两者都会经过以下逻辑：

Both go through this logic:

```typescript
export function provider(model: Provider.Model) {
  if (model.api.id.includes("gpt-5")) return [PROMPT_CODEX]
  if (model.api.id.includes("gpt-") || model.api.id.includes("o1") || model.api.id.includes("o3"))
    return [PROMPT_BEAST]
  if (model.api.id.includes("gemini-")) return [PROMPT_GEMINI]
  if (model.api.id.includes("claude")) return [PROMPT_ANTHROPIC]
  return [PROMPT_ANTHROPIC_WITHOUT_TODO]  // ← 两者都走这条路径！
}
```

因为 `qwen3-coder` 和 `qwen3` 的模型 ID 都不匹配上面的任何特定模式，所以都返回 `PROMPT_ANTHROPIC_WITHOUT_TODO`（即 `qwen.txt`）。

Since neither `qwen3-coder` nor `qwen3` model IDs match any specific patterns above, both return `PROMPT_ANTHROPIC_WITHOUT_TODO` (i.e., `qwen.txt`).

### 模型参数的区别 / Model Parameter Differences

**位置 / Location**: `/packages/opencode/src/provider/transform.ts`

虽然系统提示词相同，但 Qwen 模型有**特殊的参数设置**：

While the system prompts are the same, Qwen models have **special parameter settings**:

```typescript
export function temperature(model: Provider.Model) {
  const id = model.id.toLowerCase()
  if (id.includes("qwen")) return 0.55  // ← Qwen 特定
  if (id.includes("claude")) return undefined
  if (id.includes("gemini")) return 1.0
  // ...
  return undefined
}

export function topP(model: Provider.Model) {
  const id = model.id.toLowerCase()
  if (id.includes("qwen")) return 1  // ← Qwen 特定
  if (id.includes("minimax-m2")) return 0.95
  if (id.includes("gemini")) return 0.95
  return undefined
}
```

**关键参数 / Key Parameters**:
- **Temperature**: Qwen 模型固定为 `0.55`（相对保守，减少随机性）
- **TopP**: Qwen 模型固定为 `1`（使用全部概率质量）

- **Temperature**: Qwen models fixed at `0.55` (relatively conservative, reduces randomness)
- **TopP**: Qwen models fixed at `1` (use full probability mass)

这些参数对所有 Qwen 模型都适用，包括 Qwen3-coder 和 Qwen3。

These parameters apply to all Qwen models, including Qwen3-coder and Qwen3.

---

## 系统提示词组装顺序 / System Prompt Assembly Order

**位置 / Location**: `/packages/opencode/src/session/llm.ts`

```typescript
const system = SystemPrompt.header(input.model.providerID)  // 1. Provider header (if Anthropic)
system.push(
  [
    // 2. Agent prompt OR provider prompt
    ...(input.agent.prompt ? [input.agent.prompt] : isCodex ? [] : SystemPrompt.provider(input.model)),
    // 3. Custom prompts from session
    ...input.system,
    // 4. Custom prompts from last user message
    ...(input.user.system ? [input.user.system] : []),
  ]
    .filter((x) => x)
    .join("\n"),
)
```

**组装顺序 / Assembly Order**:

1. **Header** (仅 Anthropic): `anthropic_spoof.txt`
2. **Provider/Agent Prompt**:
   - 如果 agent 有自定义 prompt → 使用 agent prompt
   - 如果是 Codex 会话 → 跳过（使用 `instructions` 字段）
   - 否则 → 使用 `SystemPrompt.provider(model)` 返回的提示词
3. **Session Custom Prompts**: 从 `SystemPrompt.environment()` 和 `SystemPrompt.custom()` 来
4. **User Custom Prompts**: 最后一条用户消息中的自定义系统提示

1. **Header** (Anthropic only): `anthropic_spoof.txt`
2. **Provider/Agent Prompt**:
   - If agent has custom prompt → use agent prompt
   - If Codex session → skip (use `instructions` field)
   - Otherwise → use prompt from `SystemPrompt.provider(model)`
3. **Session Custom Prompts**: From `SystemPrompt.environment()` and `SystemPrompt.custom()`
4. **User Custom Prompts**: Custom system prompts from last user message

---

## 特殊情况：Codex 会话 / Special Case: Codex Sessions

**位置 / Location**: `/packages/opencode/src/session/llm.ts`

```typescript
const isCodex = provider.id === "openai" && auth?.type === "oauth"

// ...

// For Codex sessions, skip SystemPrompt.provider() since it's sent via options.instructions
...(input.agent.prompt ? [input.agent.prompt] : isCodex ? [] : SystemPrompt.provider(input.model)),

// ...

if (isCodex) {
  options.instructions = SystemPrompt.instructions()  // codex_header.txt
}
```

**特点 / Characteristics**:
- Codex 会话使用 OpenAI 的 `instructions` 字段而不是系统消息
- 这允许更好的提示缓存和成本优化
- `instructions` 内容来自 `codex_header.txt`

- Codex sessions use OpenAI's `instructions` field instead of system messages
- This allows better prompt caching and cost optimization
- `instructions` content comes from `codex_header.txt`

---

## 自定义提示词 / Custom Prompts

### 本地规则文件 / Local Rule Files

**位置 / Location**: `/packages/opencode/src/session/system.ts`

```typescript
const LOCAL_RULE_FILES = [
  "AGENTS.md",
  "CLAUDE.md",
  "CONTEXT.md", // deprecated
]
const GLOBAL_RULE_FILES = [
  path.join(Global.Path.config, "AGENTS.md"),
  path.join(os.homedir(), ".claude", "CLAUDE.md"),
]
```

用户可以通过这些文件添加自定义提示词：

Users can add custom prompts through these files:

1. **项目级 / Project-level**: `./AGENTS.md`, `./CLAUDE.md`
2. **全局级 / Global-level**: `~/.config/opencode/AGENTS.md`, `~/.claude/CLAUDE.md`
3. **配置指令 / Config instructions**: 在 `opencode.json` 中的 `instructions` 字段

这些自定义提示词会在 provider prompt 之后添加。

These custom prompts are added after the provider prompt.

---

## 实际差异总结 / Summary of Actual Differences

### Qwen3-coder vs Qwen3

**系统提示词 / System Prompts**: ✅ **完全相同** / **Identical**
- 两者都使用 `qwen.txt`
- 没有特殊的 coder 版本提示词

- Both use `qwen.txt`
- No special coder-specific prompt

**模型参数 / Model Parameters**: ✅ **完全相同** / **Identical**
- Temperature: `0.55`
- TopP: `1`
- 参数基于模型 ID 包含 "qwen"，不区分 coder 版本

- Temperature: `0.55`
- TopP: `1`
- Parameters based on model ID containing "qwen", no distinction for coder variant

**可能的外部差异 / Possible External Differences**:
虽然 OpenCode 对两者的处理相同，但实际差异可能来自：

While OpenCode treats them identically, actual differences may come from:

1. **模型本身的训练数据 / Model's training data**: Qwen3-coder 可能在代码数据上训练更多
2. **Provider API 配置 / Provider API configuration**: 某些 provider 可能对 coder 模型有特殊设置
3. **自定义配置 / Custom configuration**: 用户在 `opencode.json` 中可能为不同模型配置了不同参数

1. **Model's training data**: Qwen3-coder may be trained more on code data
2. **Provider API configuration**: Some providers may have special settings for coder models
3. **Custom configuration**: Users may configure different parameters for different models in `opencode.json`

---

## 如何为特定模型自定义提示词 / How to Customize Prompts for Specific Models

### 方法 1: 使用 Agent 自定义提示词 / Method 1: Use Agent Custom Prompts

在 `opencode.json` 或 `.opencode/agent/` 中定义 agent：

Define agents in `opencode.json` or `.opencode/agent/`:

```json
{
  "agent": {
    "qwen-coder": {
      "mode": "primary",
      "model": "alibaba/qwen3-coder-32b",
      "prompt": "You are a specialized coding assistant optimized for Qwen3-coder.\n\n..."
    }
  }
}
```

### 方法 2: 使用自定义规则文件 / Method 2: Use Custom Rule Files

创建项目级规则文件：

Create project-level rule files:

```bash
# 项目根目录
echo "## Custom Instructions for Qwen Models\n..." > AGENTS.md
```

### 方法 3: 修改 system.ts (不推荐) / Method 3: Modify system.ts (Not Recommended)

直接修改 `/packages/opencode/src/session/system.ts`：

Directly modify `/packages/opencode/src/session/system.ts`:

```typescript
export function provider(model: Provider.Model) {
  // 添加 qwen-coder 特殊处理
  if (model.api.id.includes("qwen") && model.api.id.includes("coder")) {
    return [PROMPT_QWEN_CODER]  // 需要创建新的提示词文件
  }
  
  if (model.api.id.includes("gpt-5")) return [PROMPT_CODEX]
  // ...
}
```

---

## 调试提示词 / Debugging Prompts

### 查看实际发送的提示词 / View Actual Sent Prompts

1. **启用调试日志 / Enable debug logging**:
```bash
export OPENCODE_LOG_LEVEL=debug
opencode
```

2. **查看会话导出 / Export session**:
```bash
opencode export <session-id> > session.json
jq '.messages[] | select(.role == "system")' session.json
```

3. **使用增强导出 / Use enhanced export** (如果已实现):
```bash
opencode export --enhanced <session-id> | jq '.systemPrompts'
```

---

## 模型到提示词映射表 / Model to Prompt Mapping Table

| 模型 ID 模式 / Model ID Pattern | 提示词文件 / Prompt File | 说明 / Notes |
|--------------------------------|------------------------|-------------|
| `gpt-5` | `codex_header.txt` | GPT-5 专用 |
| `gpt-*`, `o1`, `o3` | `beast.txt` | GPT-3.5/4, O1, O3 |
| `gemini-*` | `gemini.txt` | Gemini 模型 |
| `claude` | `anthropic.txt` | Claude 模型，包含 TodoWrite |
| `qwen*` (包括 qwen3-coder) | `qwen.txt` | Qwen 和默认模型 |
| 其他所有模型 | `qwen.txt` | 默认提示词 |

**Provider Header**:
- `anthropic` provider: 添加 `anthropic_spoof.txt` header
- 其他 providers: 无 header

---

## 温度和 TopP 设置表 / Temperature and TopP Settings Table

| 模型 / Model | Temperature | TopP | 说明 / Notes |
|-------------|-------------|------|-------------|
| Qwen (所有) | `0.55` | `1` | 保守设置，适合代码生成 |
| Claude | `undefined` | `undefined` | 使用模型默认值 |
| Gemini | `1.0` | `0.95` | 更高的随机性 |
| GLM-4.6, GLM-4.7 | `1.0` | `undefined` | - |
| Minimax-M2 | `1.0` | `0.95` | - |
| Kimi-K2 | `0.6`/`1.0` | `undefined` | thinking 模式用 1.0 |
| 其他 | `undefined` | `undefined` | 使用模型默认值 |

---

## 总结 / Conclusion

1. **Qwen3-coder 和 Qwen3 使用相同的系统提示词** (`qwen.txt`)
2. **两者的参数设置也相同** (temperature=0.55, topP=1)
3. **实际行为差异来自模型本身的训练**，而不是 OpenCode 的配置
4. **用户可以通过 agent 配置或自定义规则文件来定制提示词**
5. **OpenCode 为不同类型的模型使用不同的提示词模板**，主要区别在于：
   - Claude: 强调 TodoWrite 和任务管理
   - GPT: 使用 beast.txt 或 codex_header.txt
   - Gemini: 使用 gemini.txt
   - 其他（包括 Qwen）: 使用简洁的 qwen.txt

---

1. **Qwen3-coder and Qwen3 use the same system prompts** (`qwen.txt`)
2. **Both have identical parameter settings** (temperature=0.55, topP=1)
3. **Actual behavior differences come from the model's training**, not OpenCode configuration
4. **Users can customize prompts via agent config or custom rule files**
5. **OpenCode uses different prompt templates for different model types**, main differences:
   - Claude: Emphasizes TodoWrite and task management
   - GPT: Uses beast.txt or codex_header.txt
   - Gemini: Uses gemini.txt
   - Others (including Qwen): Use concise qwen.txt

---

## 相关文件 / Related Files

- `/packages/opencode/src/session/system.ts` - 系统提示词选择逻辑
- `/packages/opencode/src/session/llm.ts` - 提示词组装
- `/packages/opencode/src/provider/transform.ts` - 模型参数设置
- `/packages/opencode/src/session/prompt/*.txt` - 提示词模板文件
- `PROMPTS_DOCUMENTATION.md` - 提示词系统的完整文档
- `TRACES_AND_CUSTOM_MODELS_DOCUMENTATION.md` - 自定义模型配置

---

## 反馈 / Feedback

如果您发现 Qwen3-coder 和 Qwen3 之间有实际的渲染差异，请提供：

If you find actual rendering differences between Qwen3-coder and Qwen3, please provide:

1. 使用的具体模型 ID / Specific model IDs used
2. Provider 配置 / Provider configuration
3. 实际观察到的提示词差异 / Actual observed prompt differences
4. 会话导出文件 / Session export file

提交 Issue: https://github.com/anomalyco/opencode/issues

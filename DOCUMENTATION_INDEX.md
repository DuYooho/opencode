# OpenCode Documentation Index

This repository contains comprehensive documentation for OpenCode's internal systems. All documentation is bilingual (Chinese & English).

本仓库包含 OpenCode 内部系统的完整文档。所有文档均为中英双语。

---

## 📚 Documentation Overview / 文档概览

### 🎯 Core System Documentation / 核心系统文档

| Document | Description | 中文说明 |
|----------|-------------|---------|
| [PROMPTS_DOCUMENTATION.md](./PROMPTS_DOCUMENTATION.md) | Complete catalog of all 18 prompt files (12 main, 5 agent, 1 tool) | 所有 18 个提示词文件的完整目录 |
| [TOOLS_DOCUMENTATION.md](./TOOLS_DOCUMENTATION.md) | Documentation for all 20 built-in tools with prompts | 所有 20 个内置工具及其提示词文档 |
| [AGENTS_AND_TASKS_DOCUMENTATION.md](./AGENTS_AND_TASKS_DOCUMENTATION.md) | Complete guide to agents, tasks, and subagents | Agents、任务和子 agents 的完整指南 |

### 🔧 Configuration & Models / 配置与模型

| Document | Description | 中文说明 |
|----------|-------------|---------|
| [TRACES_AND_CUSTOM_MODELS_DOCUMENTATION.md](./TRACES_AND_CUSTOM_MODELS_DOCUMENTATION.md) | Session export, custom models, model switching | 会话导出、自定义模型、模型切换 |
| [如何禁用默认模型.md](./如何禁用默认模型.md) | Quick guide: disable default OpenCode models | 快速指南：禁用 OpenCode 默认模型 |

### 🎨 Prompt & Tool Rendering / 提示词与工具渲染

| Document | Description | 中文说明 |
|----------|-------------|---------|
| [MODEL_PROMPT_RENDERING.md](./MODEL_PROMPT_RENDERING.md) | **Comprehensive**: How different models render prompts | **完整版**：不同模型如何渲染提示词 |
| [模型提示词渲染差异.md](./模型提示词渲染差异.md) | **Quick Reference**: Qwen3-coder vs Qwen3 prompt analysis | **快速参考**：Qwen3-coder vs Qwen3 提示词分析 |
| [PROMPT_FLOW_DIAGRAM.md](./PROMPT_FLOW_DIAGRAM.md) | **Visual**: ASCII flow diagrams for prompt system | **可视化**：提示词系统 ASCII 流程图 |
| [TOOL_RENDERING_FORMATS.md](./TOOL_RENDERING_FORMATS.md) | **Comprehensive**: JSON vs XML tool formats | **完整版**：JSON vs XML 工具格式 |
| [工具渲染格式差异.md](./工具渲染格式差异.md) | **Quick Reference**: Qwen3 (JSON) vs Qwen3-coder (XML) tool formats | **快速参考**：Qwen3 (JSON) vs Qwen3-coder (XML) 工具格式 |
| [TOOL_EXECUTION_FLOW.md](./TOOL_EXECUTION_FLOW.md) | **Complete Guide**: Tool execution lifecycle (definition → result) | **完整指南**：工具执行生命周期（定义 → 结果）|
| [工具执行流程说明.md](./工具执行流程说明.md) | **Quick Reference**: How tool instructions are executed | **快速参考**：工具指令如何被执行 |

### 🔗 Agent System / Agent 系统

| Document | Description | 中文说明 |
|----------|-------------|---------|
| [AGENT_SUBAGENT_CONTEXT_SHARING.md](./AGENT_SUBAGENT_CONTEXT_SHARING.md) | **Complete Guide**: How context is shared between main agents and subagents | **完整指南**：主 agent 和 subagent 之间如何共享上下文 |

### 📂 Examples / 示例

| Directory | Description | 中文说明 |
|-----------|-------------|---------|
| [examples/](./examples/) | Configuration examples for common scenarios | 常见场景的配置示例 |

---

## 🔍 Quick Access by Topic / 按主题快速访问

### For Users / 用户文档

#### ❓ "How do I...?" / "我如何...？"

**Disable free models?** / **禁用免费模型？**
→ [如何禁用默认模型.md](./如何禁用默认模型.md)

**Understand prompt differences between models?** / **了解不同模型的提示词差异？**
→ [模型提示词渲染差异.md](./模型提示词渲染差异.md) (Quick) / [MODEL_PROMPT_RENDERING.md](./MODEL_PROMPT_RENDERING.md) (Detailed)

**Understand tool format differences (JSON vs XML)?** / **了解工具格式差异（JSON vs XML）？**
→ [工具渲染格式差异.md](./工具渲染格式差异.md) (Quick) / [TOOL_RENDERING_FORMATS.md](./TOOL_RENDERING_FORMATS.md) (Detailed)

**Understand how tools are executed?** / **了解工具如何执行？**
→ [工具执行流程说明.md](./工具执行流程说明.md) (Quick) / [TOOL_EXECUTION_FLOW.md](./TOOL_EXECUTION_FLOW.md) (Detailed)

**Understand agent-subagent context sharing?** / **了解 agent-subagent 上下文共享？**
→ [AGENT_SUBAGENT_CONTEXT_SHARING.md](./AGENT_SUBAGENT_CONTEXT_SHARING.md)

**Switch models mid-session?** / **会话中切换模型？**
→ [TRACES_AND_CUSTOM_MODELS_DOCUMENTATION.md](./TRACES_AND_CUSTOM_MODELS_DOCUMENTATION.md) → Model Switching section

**Export session data?** / **导出会话数据？**
→ [TRACES_AND_CUSTOM_MODELS_DOCUMENTATION.md](./TRACES_AND_CUSTOM_MODELS_DOCUMENTATION.md) → Session Export section

**Create custom tools?** / **创建自定义工具？**
→ [TOOLS_DOCUMENTATION.md](./TOOLS_DOCUMENTATION.md) → Custom Tool Creation section

**Create custom agents?** / **创建自定义 agents？**
→ [AGENTS_AND_TASKS_DOCUMENTATION.md](./AGENTS_AND_TASKS_DOCUMENTATION.md) → Creating Custom Agents section

---

### For Developers / 开发者文档

#### 🔧 Code Locations / 代码位置

**System prompt selection logic** / **系统提示词选择逻辑**
→ `/packages/opencode/src/session/system.ts`

**Prompt assembly** / **提示词组装**
→ `/packages/opencode/src/session/llm.ts`

**Model parameter transforms** / **模型参数转换**
→ `/packages/opencode/src/provider/transform.ts`

**Prompt templates** / **提示词模板**
→ `/packages/opencode/src/session/prompt/*.txt`

**Tool definitions** / **工具定义**
→ `/packages/opencode/src/tool/`

**Tool execution pipeline** / **工具执行管道**
→ `/packages/opencode/src/session/processor.ts` (lifecycle management)
→ `/packages/opencode/src/tool/tool.ts` (Tool.define interface)
→ `/packages/opencode/src/tool/registry.ts` (registration)

**Agent system** / **Agent 系统**
→ `/packages/opencode/src/agent/agent.ts`
→ `/packages/opencode/src/tool/task.ts` (task tool for subagent invocation)
→ `/packages/opencode/src/session/index.ts` (session creation with parentID)

**Tool rendering (format selection)** / **工具渲染（格式选择）**
→ `/packages/opencode/src/session/prompt.ts` (tool definition)
→ `/packages/opencode/src/provider/sdk/openai-compatible/` (format implementation)

---

## 📊 Key Findings / 关键发现

### Model Prompt Rendering / 模型提示词渲染

From our analysis in [MODEL_PROMPT_RENDERING.md](./MODEL_PROMPT_RENDERING.md):

根据 [MODEL_PROMPT_RENDERING.md](./MODEL_PROMPT_RENDERING.md) 中的分析：

1. **Qwen3-coder and Qwen3 use identical prompts** (`qwen.txt`)
   - **Qwen3-coder 和 Qwen3 使用相同的提示词** (`qwen.txt`)

2. **Main prompt differences**:
   - **主要提示词差异**：
   - Claude models: `anthropic.txt` (with TodoWrite)
   - GPT-5/Codex: `codex_header.txt` (with Git/frontend guidance)
   - GPT-3.5/4: `beast.txt`
   - Gemini: `gemini.txt`
   - Qwen + others: `qwen.txt` (concise, no TodoWrite)

3. **Model parameters for Qwen**:
   - **Qwen 模型参数**：
   - Temperature: `0.55` (conservative)
   - TopP: `1` (full probability mass)

### Tool Rendering Formats / 工具渲染格式

From our analysis in [TOOL_RENDERING_FORMATS.md](./TOOL_RENDERING_FORMATS.md):

根据 [TOOL_RENDERING_FORMATS.md](./TOOL_RENDERING_FORMATS.md) 中的分析：

1. **Qwen3-coder uses XML, Qwen3 uses JSON** for tool calls
   - **Qwen3-coder 使用 XML，Qwen3 使用 JSON** 进行工具调用

2. **Format determined by AI SDK**, not OpenCode configuration
   - **格式由 AI SDK 决定**，而非 OpenCode 配置

3. **Reasons for difference**:
   - **差异原因**：
   - Model training (Qwen3-coder optimized for XML tools)
   - API implementation differences
   - SDK provider logic
   - Backward compatibility

4. **Transparent to users** - OpenCode handles both formats automatically
   - **对用户透明** - OpenCode 自动处理两种格式

### Tool Execution Flow / 工具执行流程

From our analysis in [TOOL_EXECUTION_FLOW.md](./TOOL_EXECUTION_FLOW.md):

根据 [TOOL_EXECUTION_FLOW.md](./TOOL_EXECUTION_FLOW.md) 中的分析：

1. **Complete 7-step pipeline**:
   - **完整的 7 步管道**：
   - Tool Definition → Registration → Model Invocation → AI SDK Streaming → Tool Execution → Result Processing → Iteration

2. **Tool lifecycle states**:
   - **工具生命周期状态**：
   - `pending` (AI generating call) → `running` (executing) → `completed`/`error`

3. **Permission system integration**:
   - **权限系统集成**：
   - Every tool can request user permission before execution
   - Supports always-allow, always-deny, and ask-user patterns

4. **Error handling**:
   - **错误处理**：
   - Parameter validation (Zod schemas)
   - Execution error capture and retry
   - AI SDK automatic repair for common issues

5. **Advanced features**:
   - **高级功能**：
   - Output truncation (10K lines / 100KB)
   - Parallel tool calls
   - File attachments (images, PDFs)
   - Tool context with abort signals

### Agent-Subagent Context Sharing / Agent-Subagent 上下文共享

From our analysis in [AGENT_SUBAGENT_CONTEXT_SHARING.md](./AGENT_SUBAGENT_CONTEXT_SHARING.md):

根据 [AGENT_SUBAGENT_CONTEXT_SHARING.md](./AGENT_SUBAGENT_CONTEXT_SHARING.md) 中的分析：

1. **Subagents are isolated** - They don't see main agent's context automatically
   - **Subagent 是隔离的** - 它们不会自动看到主 agent 的上下文

2. **Context must be explicit** - Pass everything needed in the `prompt` parameter
   - **上下文必须显式传递** - 在 `prompt` 参数中传递所需的一切

3. **Session architecture**:
   - **Session 架构**：
   - Each subagent gets new session with `parentID` link
   - Parent tracking doesn't share context - only for hierarchy
   - Separate message histories for main and subagent sessions

4. **One-way communication**:
   - **单向通信**：
   - Main agent → Subagent: Only prompt text
   - Subagent → Main agent: Only final result text + metadata
   - Subagent's tool calls are hidden from main agent

5. **Context passing strategies**:
   - **上下文传递策略**：
   - Inline text (for small content)
   - @file references (for large files)
   - Structured data (JSON/tables)
   - Session continuation (multi-step tasks with `session_id`)

6. **Design benefits**:
   - **设计优势**：
   - Better performance (smaller context windows)
   - Lower cost (fewer tokens per call)
   - Clear security boundaries
   - Task independence

### System Prompt Assembly Order / 系统提示词组装顺序

1. **Header** (Anthropic only)
2. **Agent/Provider prompt**
3. **Environment info** (working directory, git status, etc.)
4. **Custom rules** (AGENTS.md, CLAUDE.md)
5. **User custom prompts**

---

## 🎯 Common Use Cases / 常见用例

### Use Case 1: Disable OpenCode Free Models / 禁用 OpenCode 免费模型

**Problem**: Want to avoid `opencode/big-pickle` and `opencode/gpt-5-nano`

**Solution**:
```json
{
  "disabled_providers": ["opencode"],
  "provider": {
    "anthropic": { "options": { "apiKey": "${ANTHROPIC_API_KEY}" } }
  }
}
```

**Documentation**: [如何禁用默认模型.md](./如何禁用默认模型.md)

---

### Use Case 2: Custom Prompt for Specific Model / 为特定模型自定义提示词

**Problem**: Want different prompt for Qwen3-coder

**Solution**:
```json
{
  "agent": {
    "qwen-coder": {
      "mode": "primary",
      "model": "alibaba/qwen3-coder-32b",
      "prompt": "Custom prompt for Qwen3-coder..."
    }
  }
}
```

**Documentation**: [MODEL_PROMPT_RENDERING.md](./MODEL_PROMPT_RENDERING.md) → Customization section

---

### Use Case 3: Export Session with Full Details / 导出包含完整细节的会话

**Problem**: Need to export session with system prompts and tool definitions

**Solution**:
```bash
opencode export --enhanced <session-id>
```

**Documentation**: [TRACES_AND_CUSTOM_MODELS_DOCUMENTATION.md](./TRACES_AND_CUSTOM_MODELS_DOCUMENTATION.md) → Enhanced Export section

---

### Use Case 4: Switch Models Mid-Session / 会话中切换模型

**Problem**: Start with strong model, switch to light model for simple tasks

**Methods**:
1. `<Leader>+m` - Model selection dialog
2. `F2` / `Shift+F2` - Cycle recent models
3. `Ctrl+T` - Cycle model variants
4. Agent-specific models in config

**Documentation**: [TRACES_AND_CUSTOM_MODELS_DOCUMENTATION.md](./TRACES_AND_CUSTOM_MODELS_DOCUMENTATION.md) → Model Switching section

---

## 📖 Documentation Structure / 文档结构

### Comprehensive Guides / 完整指南

These documents cover everything about a topic:

- **PROMPTS_DOCUMENTATION.md**: All prompt files, their content, and usage
- **TOOLS_DOCUMENTATION.md**: All tools, their prompts, and how to create custom ones
- **AGENTS_AND_TASKS_DOCUMENTATION.md**: Complete agent system architecture
- **TRACES_AND_CUSTOM_MODELS_DOCUMENTATION.md**: Session management and model configuration
- **MODEL_PROMPT_RENDERING.md**: Deep dive into prompt rendering differences

### Quick References / 快速参考

These documents answer specific questions quickly:

- **如何禁用默认模型.md**: Disable default models (Chinese)
- **模型提示词渲染差异.md**: Qwen3-coder vs Qwen3 (Chinese)

### Visual Guides / 可视化指南

These documents use diagrams for easy understanding:

- **PROMPT_FLOW_DIAGRAM.md**: ASCII flow diagrams showing prompt system

---

## 🔄 Related Resources / 相关资源

### Official Documentation / 官方文档
- Website: https://opencode.ai
- Providers: https://opencode.ai/docs/providers
- Models: https://opencode.ai/docs/models
- Agents: https://opencode.ai/docs/agents

### Repository Files / 仓库文件
- [CONTRIBUTING.md](./CONTRIBUTING.md) - How to contribute
- [SECURITY.md](./SECURITY.md) - Security policies
- [STYLE_GUIDE.md](./STYLE_GUIDE.md) - Code style guide

---

## 🆕 Recent Additions / 最近添加

### 2026-02-10

Added comprehensive documentation for model-specific prompt rendering and tool execution:

新增模型特定提示词渲染和工具执行的完整文档：

1. **MODEL_PROMPT_RENDERING.md** - Complete analysis of how different models render prompts
   - 不同模型如何渲染提示词的完整分析
   
2. **模型提示词渲染差异.md** - Quick Chinese reference for Qwen3-coder vs Qwen3
   - Qwen3-coder vs Qwen3 的快速中文参考
   
3. **PROMPT_FLOW_DIAGRAM.md** - Visual flow diagrams with ASCII art
   - 带 ASCII 艺术的可视化流程图

4. **TOOL_RENDERING_FORMATS.md** - JSON vs XML tool format analysis (comprehensive)
   - JSON vs XML 工具格式分析（完整版）

5. **工具渲染格式差异.md** - Quick Chinese reference for tool formats
   - 工具格式的快速中文参考

6. **TOOL_EXECUTION_FLOW.md** - Complete tool execution pipeline documentation
   - 完整的工具执行管道文档

7. **工具执行流程说明.md** - Quick Chinese reference for tool execution
   - 工具执行流程的快速中文参考

8. **AGENT_SUBAGENT_CONTEXT_SHARING.md** - Complete guide on agent-subagent context sharing
   - Agent-Subagent 上下文共享的完整指南

Key findings documented:
- Qwen3-coder and Qwen3 use identical prompts and parameters
- Main differences between Claude (with TodoWrite) and others (without)
- **Qwen3-coder uses XML format for tools, Qwen3 uses JSON format**
- Tool format determined automatically by Vercel AI SDK
- **Complete tool execution lifecycle**: definition → registration → invocation → streaming → execution → result → iteration
- Tool execution includes permission system, error handling, output truncation, and parallel calls
- **Subagents are context-isolated** - They don't see main agent's context automatically
- Context must be explicitly passed in task tool's `prompt` parameter
- Main agent only sees final result from subagent, not internal tool calls
- 5 context passing strategies: inline text, @file references, structured data, file URLs, session continuation
- Complete customization guide for users

记录的关键发现:
- Qwen3-coder 和 Qwen3 使用相同的提示词和参数
- Claude（带 TodoWrite）和其他模型（不带）的主要区别
- **Qwen3-coder 使用 XML 格式的工具，Qwen3 使用 JSON 格式**
- 工具格式由 Vercel AI SDK 自动确定
- **完整的工具执行生命周期**：定义 → 注册 → 调用 → 流式传输 → 执行 → 结果 → 迭代
- 工具执行包括权限系统、错误处理、输出截断和并行调用
- **Subagent 是上下文隔离的** - 它们不会自动看到主 agent 的上下文
- 上下文必须在 task 工具的 `prompt` 参数中显式传递
- 主 agent 只看到 subagent 的最终结果，看不到内部工具调用
- 5 种上下文传递策略：内联文本、@file 引用、结构化数据、文件 URL、session 继续
- 用户的完整自定义指南

---

## 💡 Tips / 提示

### For Reading / 阅读建议

1. **Start with Quick References** if you have a specific question
   - **从快速参考开始**，如果你有具体问题
   
2. **Use Comprehensive Guides** for in-depth understanding
   - **使用完整指南**进行深入了解
   
3. **Check Visual Guides** if you prefer diagrams
   - **查看可视化指南**，如果你喜欢图表

### For Searching / 搜索建议

All documents are fully searchable. Use these keywords:

所有文档都可以搜索。使用这些关键词：

- **prompt**: System prompts, provider prompts, agent prompts
- **tool**: Built-in tools, custom tools, tool creation, tool formats (JSON/XML)
- **agent**: Primary agents, subagents, custom agents
- **model**: Model selection, switching, parameters
- **export**: Session export, traces, enhanced export
- **qwen**: Qwen-specific documentation
- **claude**: Claude-specific documentation
- **json/xml**: Tool rendering formats

---

## 🤝 Contributing / 贡献

Found something missing or incorrect? Please:

发现遗漏或错误？请：

1. Check existing documentation first
   - 首先检查现有文档
   
2. Open an issue if it's a documentation bug
   - 如果是文档错误，请提交 issue
   
3. Submit a PR with updates
   - 提交包含更新的 PR

See [CONTRIBUTING.md](./CONTRIBUTING.md) for details.

---

## 📧 Feedback / 反馈

Questions or suggestions about the documentation?

对文档有疑问或建议？

- GitHub Issues: https://github.com/anomalyco/opencode/issues
- Documentation feedback: Tag with `documentation` label

---

## 📜 License / 许可证

All documentation files are part of the OpenCode project and follow the same license as the main repository.

所有文档文件都是 OpenCode 项目的一部分，遵循与主仓库相同的许可证。

---

**Last Updated**: 2026-02-10
**最后更新**: 2026-02-10

# Agent-Subagent Context Sharing Documentation / Agent和Subagent之间的上下文共享机制

## Overview / 概述

**English:**
This document explains how context is shared (or not shared) between main agents and subagents when using the `task` tool in OpenCode. Understanding this mechanism is crucial for effective task delegation.

**中文:**
本文档详细解释 OpenCode 中使用 `task` 工具时，主 agent 和 subagent 之间的上下文共享机制（或不共享机制）。理解这个机制对于有效的任务委派至关重要。

---

## Key Answer / 核心答案

### Does a subagent see the main agent's context? / Subagent 能看到主 agent 的上下文吗？

**English:**
**NO**, subagents do NOT automatically see the main agent's context (like files read, code viewed, or tool results). Each subagent runs in a completely isolated session with its own independent message history.

**中文:**
**不能**，subagent **不会**自动看到主 agent 的上下文（比如已经 read 的文件、查看过的代码或工具执行结果）。每个 subagent 都在完全隔离的 session 中运行，拥有自己独立的消息历史。

### How to pass context to a subagent? / 如何向 subagent 传递上下文？

**English:**
Context must be **explicitly passed** in the `prompt` parameter of the task tool. The subagent only sees what you include in the prompt text.

**中文:**
上下文必须在 task 工具的 `prompt` 参数中**显式传递**。Subagent 只能看到你在 prompt 文本中包含的内容。

### Does the main agent see subagent's tool calls? / 主 agent 能看到 subagent 的工具调用吗？

**English:**
**NO**, the main agent only sees the final text result returned by the subagent. Internal tool calls (like reading files) made by the subagent are hidden from the main agent.

**中文:**
**不能**，主 agent 只能看到 subagent 返回的最终文本结果。Subagent 内部的工具调用（比如读取文件）对主 agent 是隐藏的。

---

## Architecture / 架构

### Session Isolation / Session 隔离

```
Main Agent Session (session-xxx)
├── Message History (visible to main agent only)
│   ├── User: "Read file.py and summarize it"
│   ├── Assistant: [reads file.py] "I'll use a subagent..."
│   └── Assistant: [calls task tool]
│
└── Subagent Session (session-yyy) ← Created with parentID = session-xxx
    ├── Message History (visible to subagent only, ISOLATED)
    │   ├── User: "Analyze the following code: [code content]"
    │   ├── Assistant: [reads other files, makes tool calls]
    │   └── Assistant: "Analysis complete: [result]"
    │
    └── Returns only final text to main agent
```

**Key Points / 关键点:**

1. **Separate Sessions / 独立 Session**: Each subagent gets a new session with unique ID
2. **Parent Tracking / 父级跟踪**: `parentID` links sessions but doesn't share context
3. **Isolated History / 隔离历史**: Each session maintains its own message history
4. **One-way Result / 单向结果**: Only final result flows back to main agent

---

## Detailed Interaction Flow / 详细交互流程

### Step 1: Main Agent Calls Task Tool / 主 Agent 调用 Task 工具

```typescript
// Main agent (session-xxx) calls task tool
Task({
  description: "Analyze code",
  prompt: "Analyze this code and find bugs: [code content here]",
  subagent_type: "general"
})
```

**What happens / 发生了什么:**
- Task tool receives only the `prompt` parameter
- No automatic context from main session is included

### Step 2: Subagent Session Created / 创建 Subagent Session

```typescript
// From task.ts
const session = await Session.create({
  parentID: ctx.sessionID,  // Links to main session
  title: params.description + ` (@${agent.name} subagent)`,
  permission: [/* permissions config */],
})
```

**What happens / 发生了什么:**
- New isolated session created
- `parentID` set for tracking only (not for context sharing)
- Subagent gets clean slate with no history

### Step 3: Prompt Resolution / 提示词解析

```typescript
// From task.ts
const promptParts = await SessionPrompt.resolvePromptParts(params.prompt)

const result = await SessionPrompt.prompt({
  messageID,
  sessionID: session.id,  // Subagent's session
  model: { modelID, providerID },
  agent: agent.name,
  parts: promptParts,  // Only this goes to subagent!
})
```

**What subagent sees / Subagent 能看到的:**
- Only the text in `params.prompt`
- Any files referenced with `@file` syntax in the prompt
- Nothing from main agent's previous tool calls or context

### Step 4: Subagent Execution / Subagent 执行

```typescript
// Inside subagent session (session-yyy)
// Subagent can:
- Read files (tool: read)
- Search code (tool: grep, glob)
- Execute commands (tool: bash)
- Make edits (tool: write, edit)

// All of these happen in ISOLATED context
```

**What main agent sees / 主 Agent 能看到的:**
- ❌ NOT the files subagent read
- ❌ NOT the commands subagent executed
- ❌ NOT the intermediate tool results
- ✅ ONLY the final text response

### Step 5: Result Return / 结果返回

```typescript
// From task.ts
const text = result.parts.findLast((x) => x.type === "text")?.text ?? ""

const output = text + "\n\n" + [
  "<task_metadata>",
  `session_id: ${session.id}`,
  "</task_metadata>"
].join("\n")

return {
  title: params.description,
  metadata: {
    summary,  // Tool call summaries
    sessionId: session.id,
  },
  output,  // Only this goes back to main agent!
}
```

**What returns to main agent / 返回给主 Agent 的内容:**
- Final text response from subagent
- Session ID (for continuation if needed)
- Tool call summaries (names and status, not content)

---

## Why This Design? / 为什么这样设计？

### 1. Performance / 性能

**English:**
Not sharing context prevents token explosion. If every subagent inherited full history, context windows would fill up quickly.

**中文:**
不共享上下文避免了 token 爆炸。如果每个 subagent 都继承完整历史，上下文窗口会很快被填满。

### 2. Cost / 成本

**English:**
Each message sent to LLM costs tokens. Isolated contexts mean smaller, cheaper API calls.

**中文:**
发送给 LLM 的每条消息都要消耗 token。隔离的上下文意味着更小、更便宜的 API 调用。

### 3. Security / 安全

**English:**
Isolation prevents accidental information leakage between tasks and provides clear permission boundaries.

**中文:**
隔离防止任务之间意外的信息泄漏，并提供清晰的权限边界。

### 4. Independence / 独立性

**English:**
Subagents can focus on specific tasks without being influenced by irrelevant context from the main agent.

**中文:**
Subagent 可以专注于特定任务，不会被主 agent 的无关上下文干扰。

---

## Explicit Context Passing Strategies / 显式上下文传递策略

### Strategy 1: Inline Text / 内联文本

**English:**
Include relevant context directly in the prompt.

**中文:**
将相关上下文直接包含在 prompt 中。

```typescript
// Main agent reads a file
const fileContent = await read("app.py")

// Pass content to subagent
Task({
  description: "Review code",
  prompt: `Review the following Python code for bugs:

\`\`\`python
${fileContent}
\`\`\`

Provide detailed analysis.`,
  subagent_type: "general"
})
```

**Pros / 优点:**
- Simple and explicit
- Works for small to medium content
- Full control over what's shared

**Cons / 缺点:**
- Can hit token limits with large content
- Increases token cost

### Strategy 2: File References with @ Mention / 使用 @ 提及文件引用

**English:**
Use `@filepath` syntax to let subagent read files directly.

**中文:**
使用 `@filepath` 语法让 subagent 直接读取文件。

```typescript
Task({
  description: "Analyze module",
  prompt: `Analyze the module in @src/app.py and suggest improvements.
  
Focus on performance and readability.`,
  subagent_type: "general"
})
```

**How it works / 工作原理:**
```typescript
// From task.ts
const promptParts = await SessionPrompt.resolvePromptParts(params.prompt)
// This resolves @src/app.py into a file part
// Subagent receives file as attachment
```

**Pros / 优点:**
- Efficient (files loaded by subagent as needed)
- Doesn't bloat main prompt
- Supports large files

**Cons / 缺点:**
- Subagent must read file (one extra tool call)
- File content not in main agent's context

### Strategy 3: Structured Data / 结构化数据

**English:**
Pass structured context for complex information.

**中文:**
为复杂信息传递结构化上下文。

```typescript
// Main agent gathers information
const bugs = await findBugs()
const metrics = await analyzeMetrics()

Task({
  description: "Generate report",
  prompt: `Generate a comprehensive report based on this data:

**Bugs Found:**
${bugs.map(b => `- ${b.severity}: ${b.description}`).join('\n')}

**Performance Metrics:**
- Response time: ${metrics.responseTime}ms
- Memory usage: ${metrics.memory}MB
- CPU usage: ${metrics.cpu}%

Create a professional report with recommendations.`,
  subagent_type: "general"
})
```

**Pros / 优点:**
- Clear data structure
- Easy for subagent to parse
- Maintains data fidelity

**Cons / 缺点:**
- Requires main agent to format data
- Large datasets can be costly

### Strategy 4: File URLs / 文件 URL

**English:**
Reference files by path for subagent to read.

**中文:**
通过路径引用文件，让 subagent 去读取。

```typescript
Task({
  description: "Compare files",
  prompt: `Compare these two implementations:
- @src/old_implementation.py
- @src/new_implementation.py

List key differences and improvements.`,
  subagent_type: "general"
})
```

**Pros / 优点:**
- Very efficient for large files
- Subagent controls what to read
- Minimal token usage in prompt

**Cons / 缺点:**
- Subagent must make read tool calls
- Takes longer (multiple steps)

### Strategy 5: Session Continuation / Session 继续

**English:**
Continue an existing subagent session to build on previous context.

**中文:**
继续使用现有的 subagent session 来构建之前的上下文。

```typescript
// First call - creates new session
const result1 = await Task({
  description: "Initial analysis",
  prompt: "Analyze @app.py for bugs",
  subagent_type: "general"
})

// Extract session ID from metadata
const sessionId = extractSessionId(result1.output)

// Second call - continues same session
const result2 = await Task({
  description: "Fix bugs",
  prompt: "Fix the bugs you found in the previous analysis",
  subagent_type: "general",
  session_id: sessionId  // Reuse same session!
})
```

**How it works / 工作原理:**
```typescript
// From task.ts
const session = await iife(async () => {
  if (params.session_id) {
    const found = await Session.get(params.session_id).catch(() => {})
    if (found) return found  // Reuse existing session
  }
  return await Session.create({...})  // Create new session
})
```

**Pros / 优点:**
- Subagent remembers previous context
- Multi-step tasks become possible
- Natural conversation flow

**Cons / 缺点:**
- Session state grows over time
- Requires tracking session IDs
- Can hit context limits

---

## Best Practices / 最佳实践

### 1. Be Explicit / 明确传递

**English:**
Always explicitly pass the context subagents need. Don't assume they have access to anything from the main session.

**中文:**
始终明确传递 subagent 需要的上下文。不要假设它们能访问主 session 的任何内容。

❌ **Bad:**
```typescript
// Main agent reads file
const content = await read("config.json")

// Subagent has no access to this!
Task({
  prompt: "Validate the config file",  // What config file??
  subagent_type: "general"
})
```

✅ **Good:**
```typescript
// Main agent reads file
const content = await read("config.json")

// Explicitly pass the content
Task({
  prompt: `Validate this config file:

\`\`\`json
${content}
\`\`\`

Check for errors and suggest improvements.`,
  subagent_type: "general"
})
```

### 2. Use File References for Large Content / 大内容使用文件引用

**English:**
For large files or code bases, use `@` syntax instead of inline content to avoid token limits.

**中文:**
对于大文件或代码库，使用 `@` 语法而不是内联内容，以避免 token 限制。

❌ **Bad (for large files):**
```typescript
const largeFile = await read("large_dataset.json")  // 50KB file
Task({
  prompt: `Analyze: ${largeFile}`,  // Blows up token budget!
  subagent_type: "general"
})
```

✅ **Good:**
```typescript
Task({
  prompt: "Analyze the dataset in @large_dataset.json and summarize key patterns",
  subagent_type: "general"
})
```

### 3. Provide Clear Instructions / 提供清晰指令

**English:**
Since subagents start fresh, provide complete task description and expected output format.

**中文:**
由于 subagent 从零开始，提供完整的任务描述和预期输出格式。

✅ **Good:**
```typescript
Task({
  description: "Security audit",
  prompt: `Perform a security audit of @src/auth.py.

Check for:
1. SQL injection vulnerabilities
2. Authentication bypass issues
3. Insecure password handling

Format output as:
- Vulnerability type
- Severity (Critical/High/Medium/Low)
- Location (line number)
- Recommendation`,
  subagent_type: "general"
})
```

### 4. Extract and Summarize Results / 提取和总结结果

**English:**
When the subagent returns, extract relevant information and integrate it into your flow.

**中文:**
当 subagent 返回时，提取相关信息并集成到你的流程中。

```typescript
const result = await Task({...})

// Subagent result is in result.output
// Extract what you need
if (result.output.includes("CRITICAL")) {
  // Handle critical findings
  notifyUser("Critical issues found!")
}

// Continue with main task
await fixIssues(result.output)
```

### 5. Track Sessions for Multi-Step Tasks / 多步任务追踪 Session

**English:**
For complex tasks requiring multiple interactions, track and reuse session IDs.

**中文:**
对于需要多次交互的复杂任务，追踪并重用 session ID。

```typescript
let sessionId = null

// Step 1: Gather info
const info = await Task({
  prompt: "List all API endpoints in @src/api/",
  subagent_type: "explore"
})
sessionId = extractSessionId(info.output)

// Step 2: Analyze (same session)
const analysis = await Task({
  prompt: "Analyze security of the endpoints you found",
  subagent_type: "general",
  session_id: sessionId
})

// Step 3: Generate report (same session)
const report = await Task({
  prompt: "Generate a security report based on your analysis",
  subagent_type: "general",
  session_id: sessionId
})
```

### 6. Consider Token Economics / 考虑 Token 经济

**English:**
Balance between passing context inline vs. letting subagent read files. Consider token costs.

**中文:**
在内联传递上下文和让 subagent 读取文件之间取得平衡。考虑 token 成本。

**Cost Comparison / 成本对比:**

| Method | Tokens Used | Speed | When to Use |
|--------|-------------|-------|-------------|
| Inline small content | Low | Fast | < 1KB content |
| Inline medium content | Medium | Fast | 1-10KB, important context |
| @file reference | Low prompt + file read | Slower | > 10KB, or many files |
| Session continuation | Growing context | Moderate | Multi-step tasks |

---

## Common Scenarios / 常见场景

### Scenario 1: Code Review / 代码审查

**Problem / 问题:**
Main agent read 5 files, wants subagent to review one of them.

**Solution / 解决方案:**
```typescript
// Main agent context (NOT visible to subagent)
const file1 = await read("app.py")
const file2 = await read("utils.py")
const file3 = await read("config.py")

// Pass specific file to subagent
Task({
  description: "Review config",
  prompt: `Review this configuration file for best practices:

\`\`\`python
${file3}
\`\`\`

Check for:
- Hardcoded secrets
- Missing validation
- Unclear variable names`,
  subagent_type: "general"
})
```

### Scenario 2: Large Codebase Analysis / 大代码库分析

**Problem / 问题:**
Need to analyze entire directory but context is too large.

**Solution / 解决方案:**
```typescript
// Don't pass all files inline
// Let subagent read what it needs

Task({
  description: "Analyze project",
  prompt: `Analyze the project in @src/ directory.

Focus on:
1. Overall architecture
2. Code quality issues
3. Potential bugs

You can read any files in the directory as needed.`,
  subagent_type: "explore"
})
```

### Scenario 3: Incremental Task / 增量任务

**Problem / 问题:**
Multi-step task where each step builds on previous.

**Solution / 解决方案:**
```typescript
// Step 1: Initial work
const step1 = await Task({
  description: "Find bugs",
  prompt: "Find all bugs in @app.py",
  subagent_type: "general"
})
const sessionId = extractSessionId(step1.output)

// Step 2: Continue in same session
const step2 = await Task({
  description: "Prioritize bugs",
  prompt: "Prioritize the bugs you found by severity",
  subagent_type: "general",
  session_id: sessionId
})

// Step 3: Continue in same session
const step3 = await Task({
  description: "Fix critical bugs",
  prompt: "Fix the critical and high severity bugs",
  subagent_type: "general",
  session_id: sessionId
})
```

### Scenario 4: Shared State / 共享状态

**Problem / 问题:**
Multiple subagents need access to same information.

**Solution / 解决方案:**
```typescript
// Prepare shared context once
const sharedContext = `
Project: ${projectName}
Language: ${language}
Framework: ${framework}

Code Style Guide:
- Max line length: 100
- Indentation: 2 spaces
- Naming: camelCase
`

// Pass to each subagent
const review1 = await Task({
  prompt: sharedContext + "\n\nReview @src/module1.py",
  subagent_type: "general"
})

const review2 = await Task({
  prompt: sharedContext + "\n\nReview @src/module2.py",
  subagent_type: "general"
})
```

---

## Code Locations / 代码位置

| Component | File | Description |
|-----------|------|-------------|
| Task Tool | `/packages/opencode/src/tool/task.ts` | Main task tool implementation |
| Session Creation | `/packages/opencode/src/session/index.ts` | Session.create() with parentID |
| Prompt Resolution | `/packages/opencode/src/session/prompt.ts` | resolvePromptParts() and prompt() |
| Message History | `/packages/opencode/src/session/message-v2.ts` | toModelMessages() for context |
| LLM Streaming | `/packages/opencode/src/session/llm.ts` | LLM.stream() with messages |

---

## Debugging Context Sharing / 调试上下文共享

### Method 1: Export Sessions / 导出 Session

```bash
# Export main session
opencode export <main-session-id> > main.json

# Export subagent session (get ID from task metadata)
opencode export <subagent-session-id> > subagent.json

# Compare message histories
jq '.messages[] | {role: .role, content: .parts[0].text}' main.json
jq '.messages[] | {role: .role, content: .parts[0].text}' subagent.json
```

### Method 2: Check Task Metadata / 检查 Task 元数据

```typescript
// In tool result
{
  output: "...",
  metadata: {
    sessionId: "session-yyy",  // Subagent session ID
    summary: [...]  // Tool calls made by subagent
  }
}

// Use this to inspect subagent session
const subagentSession = await Session.get(metadata.sessionId)
const subagentMessages = await Session.messages({ sessionID: metadata.sessionId })
```

### Method 3: Add Logging / 添加日志

```typescript
// In main agent
console.log("Main agent context:", {
  filesRead: ["app.py", "config.py"],
  toolResults: [...]
})

// Pass to subagent
Task({
  prompt: `[Debug: Main agent has read app.py and config.py]
  
  Actual task: Review the code in @app.py`,
  subagent_type: "general"
})

// Check if subagent mentions the debug info
// If it does, context was passed
// If it doesn't, context was NOT passed
```

---

## Summary / 总结

**English:**

1. **Subagents are isolated** - They don't see main agent's context automatically
2. **Context must be explicit** - Pass everything needed in the `prompt` parameter
3. **Use @ for files** - Efficient way to let subagent read files directly
4. **Session continuation** - Use `session_id` for multi-step tasks
5. **One-way communication** - Only final result flows back to main agent
6. **Design benefits** - Better performance, lower cost, clearer boundaries

**中文:**

1. **Subagent 是隔离的** - 它们不会自动看到主 agent 的上下文
2. **上下文必须显式** - 在 `prompt` 参数中传递所需的一切
3. **使用 @ 引用文件** - 高效地让 subagent 直接读取文件
4. **Session 继续** - 使用 `session_id` 进行多步骤任务
5. **单向通信** - 只有最终结果返回给主 agent
6. **设计优势** - 更好的性能、更低的成本、更清晰的边界

---

## Related Documentation / 相关文档

- [AGENTS_AND_TASKS_DOCUMENTATION.md](./AGENTS_AND_TASKS_DOCUMENTATION.md) - Agent system overview
- [TOOL_EXECUTION_FLOW.md](./TOOL_EXECUTION_FLOW.md) - How tools are executed
- [TOOLS_DOCUMENTATION.md](./TOOLS_DOCUMENTATION.md) - All available tools
- [DOCUMENTATION_INDEX.md](./DOCUMENTATION_INDEX.md) - Complete documentation index

# Tool Execution Flow Documentation / 工具执行流程文档

## English Version

### Overview

This document explains how each tool's output instructions are executed in OpenCode, covering the complete lifecycle from tool definition to execution result.

### Complete Tool Execution Pipeline

```
1. Tool Definition (tool/*.ts)
   ↓
2. Tool Registration (registry.ts)
   ↓
3. Model Invocation (llm.ts)
   ↓
4. AI SDK Tool Calling (streamText)
   ↓
5. Tool Execution (processor.ts)
   ↓
6. Result Processing
   ↓
7. Next Iteration (loop back to step 3 if needed)
```

---

## Part 1: Tool Definition

### Location
`/packages/opencode/src/tool/tool.ts`

### How Tools Are Defined

Every tool in OpenCode follows the `Tool.Info` interface:

```typescript
export interface Info<Parameters extends z.ZodType, M extends Metadata> {
  id: string
  init: (ctx?: InitContext) => Promise<{
    description: string
    parameters: Parameters
    execute(args: z.infer<Parameters>, ctx: Context): Promise<{
      title: string
      metadata: M
      output: string
      attachments?: MessageV2.FilePart[]
    }>
    formatValidationError?(error: z.ZodError): string
  }>
}
```

### Tool Definition Components

1. **ID**: Unique identifier (e.g., "bash", "read", "edit")
2. **Description**: Natural language description for the AI model
3. **Parameters**: Zod schema defining expected inputs
4. **Execute Function**: The actual implementation

### Example: Bash Tool

**File**: `/packages/opencode/src/tool/bash.ts`

```typescript
export const BashTool = Tool.define("bash", async () => {
  return {
    description: DESCRIPTION, // loaded from bash.txt
    parameters: z.object({
      command: z.string().describe("The command to execute"),
      timeout: z.number().optional(),
      workdir: z.string().optional(),
      description: z.string()
    }),
    async execute(params, ctx) {
      // 1. Validate working directory
      const cwd = params.workdir || Instance.directory
      
      // 2. Parse command (using tree-sitter)
      const tree = await parser().then((p) => p.parse(params.command))
      
      // 3. Check permissions
      await ctx.ask({
        permission: "bash",
        patterns: [...],
        // ...
      })
      
      // 4. Execute command
      const result = await Shell.exec({
        command: params.command,
        cwd,
        timeout,
        // ...
      })
      
      // 5. Return result
      return {
        title: params.description,
        output: result.stdout + result.stderr,
        metadata: { ... }
      }
    }
  }
})
```

### Example: Read Tool

**File**: `/packages/opencode/src/tool/read.ts`

```typescript
export const ReadTool = Tool.define("read", {
  description: DESCRIPTION,
  parameters: z.object({
    filePath: z.string(),
    offset: z.coerce.number().optional(),
    limit: z.coerce.number().optional(),
  }),
  async execute(params, ctx) {
    // 1. Resolve absolute path
    let filepath = params.filePath
    if (!path.isAbsolute(filepath)) {
      filepath = path.join(process.cwd(), filepath)
    }
    
    // 2. Check permissions
    await ctx.ask({
      permission: "read",
      patterns: [filepath],
      always: ["*"],
    })
    
    // 3. Read file
    const file = Bun.file(filepath)
    if (!(await file.exists())) {
      throw new Error(`File not found: ${filepath}`)
    }
    
    // 4. Handle images/PDFs
    if (isImage || isPdf) {
      return {
        title,
        output: msg,
        attachments: [{ /* file data */ }]
      }
    }
    
    // 5. Read text content
    const content = await file.text()
    // ... line slicing logic ...
    
    return {
      title,
      output: content,
      metadata: { preview, truncated }
    }
  }
})
```

### Example: Edit Tool

**File**: `/packages/opencode/src/tool/edit.ts`

```typescript
export const EditTool = Tool.define("edit", {
  description: DESCRIPTION,
  parameters: z.object({
    filePath: z.string(),
    oldString: z.string(),
    newString: z.string(),
    replaceAll: z.boolean().optional(),
  }),
  async execute(params, ctx) {
    // 1. Resolve path
    const filePath = path.isAbsolute(params.filePath) 
      ? params.filePath 
      : path.join(Instance.directory, params.filePath)
    
    // 2. Lock file during edit
    await FileTime.withLock(filePath, async () => {
      // 3. Read current content
      const file = Bun.file(filePath)
      const contentOld = await file.text()
      
      // 4. Apply replacement
      const contentNew = replace(
        contentOld, 
        params.oldString, 
        params.newString, 
        params.replaceAll
      )
      
      // 5. Generate diff
      const diff = createTwoFilesPatch(
        filePath, filePath, 
        contentOld, contentNew
      )
      
      // 6. Ask permission with diff
      await ctx.ask({
        permission: "edit",
        patterns: [path.relative(Instance.worktree, filePath)],
        metadata: { filepath: filePath, diff }
      })
      
      // 7. Write file
      await file.write(contentNew)
      
      // 8. Publish event
      await Bus.publish(File.Event.Edited, { file: filePath })
    })
    
    return {
      title,
      output: diff,
      metadata: { /* diagnostics */ }
    }
  }
})
```

---

## Part 2: Tool Registration

### Location
`/packages/opencode/src/tool/registry.ts`

### Built-in Tools

OpenCode registers 20+ built-in tools:

```typescript
async function all(): Promise<Tool.Info[]> {
  const custom = await state().then((x) => x.custom)
  const config = await Config.get()

  return [
    InvalidTool,      // Fallback for invalid tool calls
    QuestionTool,     // User interaction (UI clients only)
    BashTool,         // Command execution
    ReadTool,         // File reading
    GlobTool,         // File pattern matching
    GrepTool,         // Content search
    EditTool,         // File editing
    WriteTool,        // File writing
    TaskTool,         // Sub-agent invocation
    WebFetchTool,     // HTTP requests
    TodoWriteTool,    // Todo list management
    TodoReadTool,     // Todo list reading
    WebSearchTool,    // Web search (Exa)
    CodeSearchTool,   // Code search (Exa)
    SkillTool,        // Skill execution
    ApplyPatchTool,   // Patch application (GPT models)
    LspTool,          // Language server (experimental)
    BatchTool,        // Batch operations (experimental)
    PlanExitTool,     // Plan mode control (experimental)
    PlanEnterTool,    // Plan mode control (experimental)
    ...custom,        // Custom tools from config
  ]
}
```

### Custom Tools

Users can add custom tools via:

1. **Project-level**: `.opencode/tool/*.{js,ts}`
2. **User-level**: `~/.config/opencode/tool/*.{js,ts}`
3. **Plugins**: Via plugin system

Custom tools are automatically discovered and loaded:

```typescript
export const state = Instance.state(async () => {
  const custom = [] as Tool.Info[]
  const glob = new Bun.Glob("{tool,tools}/*.{js,ts}")

  // Scan config directories
  for (const dir of await Config.directories()) {
    for await (const match of glob.scan({ cwd: dir, absolute: true })) {
      const namespace = path.basename(match, path.extname(match))
      const mod = await import(match)
      for (const [id, def] of Object.entries(mod)) {
        custom.push(fromPlugin(id, def))
      }
    }
  }
  
  return { custom }
})
```

### Tool Filtering

Not all tools are available for all models:

```typescript
export async function tools(model: { providerID: string; modelID: string }, agent?: Agent.Info) {
  const tools = await all()
  return await Promise.all(
    tools
      .filter((t) => {
        // websearch/codesearch only for OpenCode provider or with flag
        if (t.id === "codesearch" || t.id === "websearch") {
          return model.providerID === "opencode" || Flag.OPENCODE_ENABLE_EXA
        }
        
        // GPT models use apply_patch instead of edit/write
        const usePatch = model.modelID.includes("gpt-") && 
                        !model.modelID.includes("oss") && 
                        !model.modelID.includes("gpt-4")
        if (t.id === "apply_patch") return usePatch
        if (t.id === "edit" || t.id === "write") return !usePatch
        
        return true
      })
      .map(async (t) => ({
        id: t.id,
        ...(await t.init({ agent }))
      }))
  )
}
```

---

## Part 3: Model Invocation & Tool Streaming

### Location
`/packages/opencode/src/session/llm.ts`

### How Tools Are Sent to AI Model

When the AI model is invoked, tools are provided via Vercel AI SDK:

```typescript
export async function stream(input: StreamInput) {
  // 1. Prepare system prompts
  const system = SystemPrompt.header(input.model.providerID)
  system.push(/* agent/provider prompts */)
  
  // 2. Resolve available tools
  const tools = await resolveTools(input)
  // tools is a Record<string, Tool> where each Tool has:
  // - description: string
  // - parameters: zod schema
  // - execute: function
  
  // 3. Call AI SDK streamText
  return streamText({
    model: wrapLanguageModel({ model: language, middleware: [...] }),
    messages: [...input.messages],
    tools,  // <-- Tools passed to AI model
    activeTools: Object.keys(tools).filter((x) => x !== "invalid"),
    temperature: params.temperature,
    topP: params.topP,
    maxOutputTokens,
    abortSignal: input.abort,
    // ...
  })
}
```

### Tool Format in API Call

The Vercel AI SDK converts tools to the appropriate format for each provider:

- **OpenAI**: JSON schema with `tools` parameter
- **Anthropic**: XML-based tool definitions
- **Google Gemini**: FunctionDeclaration format
- **Others**: OpenAI-compatible format

**Example JSON format (OpenAI)**:
```json
{
  "tools": [
    {
      "type": "function",
      "function": {
        "name": "bash",
        "description": "Execute shell commands...",
        "parameters": {
          "type": "object",
          "properties": {
            "command": {
              "type": "string",
              "description": "The command to execute"
            },
            "timeout": {
              "type": "number",
              "description": "Optional timeout in milliseconds"
            }
          },
          "required": ["command", "description"]
        }
      }
    }
  ]
}
```

**Example XML format (Anthropic)**:
```xml
<tools>
  <tool name="bash">
    <description>Execute shell commands...</description>
    <parameters>
      <parameter name="command" type="string" required="true">
        The command to execute
      </parameter>
      <parameter name="timeout" type="number" required="false">
        Optional timeout in milliseconds
      </parameter>
    </parameters>
  </tool>
</tools>
```

---

## Part 4: AI Model Decision Making

### How AI Decides to Call Tools

The AI model receives:
1. **System prompt**: Instructions on when/how to use tools
2. **Tool definitions**: What each tool does and its parameters
3. **Conversation history**: Previous messages and tool results
4. **User request**: Current task to accomplish

Based on this context, the model decides:
- **Which tool to call** (e.g., "read" for file access)
- **What parameters to use** (e.g., `{filePath: "/path/to/file"}`)
- **When to call multiple tools** (parallel or sequential)
- **When to stop** and return final answer

Example decision process:
```
User: "Read the package.json file"
↓
AI thinks: "Need to read a file → use 'read' tool"
↓
AI generates tool call:
{
  toolName: "read",
  input: { filePath: "package.json" }
}
```

---

## Part 5: Tool Execution in SessionProcessor

### Location
`/packages/opencode/src/session/processor.ts`

### Streaming Tool Execution

The `SessionProcessor` handles the tool execution lifecycle:

```typescript
export function create(input: {
  assistantMessage: MessageV2.Assistant
  sessionID: string
  model: Provider.Model
  abort: AbortSignal
}) {
  const toolcalls: Record<string, MessageV2.ToolPart> = {}
  
  return {
    async process(streamInput: LLM.StreamInput) {
      while (true) {
        const stream = await LLM.stream(streamInput)
        
        for await (const value of stream.fullStream) {
          switch (value.type) {
            // Tool call lifecycle events:
            
            case "tool-input-start": {
              // AI model starts generating tool call
              const part = await Session.updatePart({
                type: "tool",
                tool: value.toolName,
                callID: value.id,
                state: {
                  status: "pending",
                  input: {},
                }
              })
              toolcalls[value.id] = part
              break
            }
            
            case "tool-call": {
              // AI model finishes generating tool call
              const match = toolcalls[value.toolCallId]
              if (match) {
                await Session.updatePart({
                  ...match,
                  tool: value.toolName,
                  state: {
                    status: "running",
                    input: value.input,
                    time: { start: Date.now() }
                  }
                })
                
                // Check for doom loop (same tool called repeatedly)
                const parts = await MessageV2.parts(assistantMessage.id)
                const lastThree = parts.slice(-3)
                if (lastThree.every(p => 
                  p.type === "tool" && 
                  p.tool === value.toolName &&
                  JSON.stringify(p.state.input) === JSON.stringify(value.input)
                )) {
                  // Ask user permission to continue
                  await PermissionNext.ask({
                    permission: "doom_loop",
                    patterns: [value.toolName],
                    // ...
                  })
                }
              }
              break
            }
            
            case "tool-result": {
              // Tool execution completed successfully
              const match = toolcalls[value.toolCallId]
              if (match && match.state.status === "running") {
                await Session.updatePart({
                  ...match,
                  state: {
                    status: "completed",
                    input: value.input,
                    output: value.output.output,
                    metadata: value.output.metadata,
                    title: value.output.title,
                    time: {
                      start: match.state.time.start,
                      end: Date.now()
                    },
                    attachments: value.output.attachments
                  }
                })
                delete toolcalls[value.toolCallId]
              }
              break
            }
            
            case "tool-error": {
              // Tool execution failed
              const match = toolcalls[value.toolCallId]
              if (match && match.state.status === "running") {
                await Session.updatePart({
                  ...match,
                  state: {
                    status: "error",
                    input: value.input,
                    error: value.error.toString(),
                    time: {
                      start: match.state.time.start,
                      end: Date.now()
                    }
                  }
                })
                
                // Handle permission rejection
                if (value.error instanceof PermissionNext.RejectedError) {
                  blocked = true  // Stop iteration
                }
                delete toolcalls[value.toolCallId]
              }
              break
            }
          }
        }
      }
    }
  }
}
```

### Tool Execution Flow

1. **tool-input-start**: AI starts generating tool call
   - Create pending tool part in database
   - Display "thinking" indicator in UI

2. **tool-call**: AI finishes generating tool call
   - Parse tool name and parameters
   - Validate parameters against schema
   - Update status to "running"
   - Check for doom loops
   - Execute tool's `execute()` function

3. **tool-result**: Tool execution succeeds
   - Store output in database
   - Send result back to AI model
   - AI can now use this information

4. **tool-error**: Tool execution fails
   - Store error message
   - AI can retry with corrected parameters
   - Or ask user for clarification

---

## Part 6: Permission System

### How Permissions Work

Many tools require user permission before execution:

```typescript
async execute(params, ctx) {
  // Request permission
  await ctx.ask({
    permission: "bash",           // Permission type
    patterns: ["rm -rf *"],      // Patterns to check
    always: ["rm", "sudo"],      // Always ask for these
    metadata: {                  // Additional context
      command: params.command,
      workdir: params.workdir
    }
  })
  
  // Execute only if permission granted
  // ...
}
```

### Permission Types

- **bash**: Command execution
- **read**: File reading
- **edit**: File modification
- **write**: File creation
- **external_directory**: Access outside project
- **doom_loop**: Repeated tool calls

### Permission States

1. **Always Allow**: Pattern matches agent's allow list
2. **Always Deny**: Pattern matches deny list
3. **Ask User**: Prompt user for decision (UI/CLI)
4. **Rejected**: User denied permission → throws `PermissionNext.RejectedError`

---

## Part 7: Result Processing & Iteration

### After Tool Execution

Once a tool finishes:

```typescript
// 1. Tool returns result
return {
  title: "Command executed",
  output: "stdout + stderr",
  metadata: { exitCode: 0 }
}

// 2. Result is sent back to AI model
streamInput.messages.push({
  role: "tool",
  content: [{
    type: "tool-result",
    toolCallId: "call_123",
    toolName: "bash",
    result: output
  }]
})

// 3. AI model processes result
// - Can call more tools
// - Can ask follow-up questions
// - Can provide final answer

// 4. Loop continues until:
// - AI provides text response (no more tool calls)
// - User stops the session
// - Error occurs
// - Token limit reached
```

### Multi-Step Example

```
User: "Create a file named test.txt with 'Hello World' then read it back"

Step 1: AI calls 'write' tool
  → write({filePath: "test.txt", content: "Hello World"})
  → Result: "File created successfully"

Step 2: AI calls 'read' tool
  → read({filePath: "test.txt"})
  → Result: "Hello World"

Step 3: AI provides final response
  → "I've created test.txt with 'Hello World' and verified the content matches."
```

---

## Part 8: Error Handling

### Parameter Validation Errors

```typescript
// In Tool.define(), parameters are validated:
toolInfo.execute = async (args, ctx) => {
  try {
    toolInfo.parameters.parse(args)  // Validate with Zod
  } catch (error) {
    if (error instanceof z.ZodError && toolInfo.formatValidationError) {
      throw new Error(toolInfo.formatValidationError(error))
    }
    throw new Error(
      `The ${id} tool was called with invalid arguments: ${error}.
       Please rewrite the input so it satisfies the expected schema.`
    )
  }
  
  const result = await execute(args, ctx)
  return result
}
```

### Execution Errors

```typescript
async execute(params, ctx) {
  try {
    // Attempt execution
    const result = await someOperation()
    return { title, output: result, metadata: {} }
  } catch (error) {
    // Error is caught by SessionProcessor
    // → Sent to AI as tool-error event
    // → AI can retry with different parameters
    throw new Error(`Failed to execute: ${error.message}`)
  }
}
```

### AI SDK Repair

The AI SDK can automatically repair certain tool call errors:

```typescript
async experimental_repairToolCall(failed) {
  const lower = failed.toolCall.toolName.toLowerCase()
  
  // Fix case sensitivity
  if (lower !== failed.toolCall.toolName && tools[lower]) {
    return {
      ...failed.toolCall,
      toolName: lower  // "Read" → "read"
    }
  }
  
  // Route to invalid tool for error handling
  return {
    ...failed.toolCall,
    input: JSON.stringify({
      tool: failed.toolCall.toolName,
      error: failed.error.message
    }),
    toolName: "invalid"
  }
}
```

---

## Part 9: Complete Example Trace

### Real Execution Flow

```
1. User types: "Show me the files in src/"

2. AI Model thinks:
   "Need to list directory contents → use 'bash' tool with 'ls' command"

3. AI generates tool call:
   {
     "toolName": "bash",
     "toolCallId": "call_abc123",
     "input": {
       "command": "ls -la src/",
       "description": "Lists files in src directory"
     }
   }

4. SessionProcessor receives tool-call event:
   - Creates tool part with status: "running"
   - Validates parameters (Zod schema check)
   - Calls BashTool.execute()

5. BashTool.execute() runs:
   a. Parses command with tree-sitter
   b. Checks permissions:
      - Permission: "bash"
      - Patterns: ["ls"]
      - Auto-approved (safe command)
   c. Executes via Shell.exec():
      - Working directory: src/
      - Timeout: 120000ms
      - Captures stdout/stderr
   d. Returns:
      {
        title: "Lists files in src directory",
        output: "total 48\ndrwxr-xr-x  12 user  staff  384 ...",
        metadata: { exitCode: 0, stdout: "...", stderr: "" }
      }

6. SessionProcessor receives tool-result event:
   - Updates tool part with status: "completed"
   - Sends result back to AI model

7. AI Model receives result:
   "total 48
    drwxr-xr-x  12 user  staff  384 ...
    -rw-r--r--   1 user  staff  123 index.ts
    -rw-r--r--   1 user  staff  456 utils.ts"

8. AI generates text response:
   "Here are the files in the src/ directory:
    - index.ts (123 bytes)
    - utils.ts (456 bytes)
    ..."

9. SessionProcessor receives text-delta events:
   - Streams response to user in real-time

10. Session completes
```

---

## Part 10: Advanced Features

### Output Truncation

Large tool outputs are automatically truncated:

```typescript
// In Tool.define():
const result = await execute(args, ctx)

// Truncate if needed
const truncated = await Truncate.output(
  result.output, 
  {}, 
  initCtx?.agent
)

return {
  ...result,
  output: truncated.content,
  metadata: {
    ...result.metadata,
    truncated: truncated.truncated,
    outputPath: truncated.outputPath  // Path to full output file
  }
}
```

Limits:
- **MAX_LINES**: 10,000 lines
- **MAX_BYTES**: 100 KB

If exceeded, full output is saved to `.opencode/output/{hash}.txt`

### Parallel Tool Calls

Some AI models support parallel tool execution:

```typescript
// AI can call multiple tools simultaneously:
{
  "toolCalls": [
    { "toolName": "read", "input": { "filePath": "file1.txt" } },
    { "toolName": "read", "input": { "filePath": "file2.txt" } },
    { "toolName": "bash", "input": { "command": "git status" } }
  ]
}

// All three tools execute in parallel
// Results are collected and sent back together
```

### Tool Context

Every tool execution receives a context object:

```typescript
type Context = {
  sessionID: string        // Current session ID
  messageID: string        // Current message ID
  agent: string           // Agent name (e.g., "build")
  abort: AbortSignal      // Cancellation signal
  callID?: string         // Tool call ID
  extra?: { [key: string]: any }  // Extra context
  
  // Update tool metadata
  metadata(input: { 
    title?: string
    metadata?: any 
  }): void
  
  // Request user permission
  ask(input: PermissionNext.Request): Promise<void>
}
```

### File Attachments

Tools can return file attachments (images, PDFs):

```typescript
return {
  title: "Image read successfully",
  output: "Image: example.png",
  metadata: { truncated: false },
  attachments: [{
    id: "part_123",
    sessionID: ctx.sessionID,
    messageID: ctx.messageID,
    type: "file",
    file: {
      name: "example.png",
      type: "image/png",
      data: base64EncodedData
    }
  }]
}
```

The AI model receives the image and can analyze it (if vision-capable).

---

## Code Locations Summary

| Component | File Path |
|-----------|-----------|
| Tool Interface | `/packages/opencode/src/tool/tool.ts` |
| Tool Registry | `/packages/opencode/src/tool/registry.ts` |
| Bash Tool | `/packages/opencode/src/tool/bash.ts` |
| Read Tool | `/packages/opencode/src/tool/read.ts` |
| Edit Tool | `/packages/opencode/src/tool/edit.ts` |
| LLM Invocation | `/packages/opencode/src/session/llm.ts` |
| Session Processor | `/packages/opencode/src/session/processor.ts` |
| Permission System | `/packages/opencode/src/permission/next.ts` |
| Output Truncation | `/packages/opencode/src/tool/truncation.ts` |

---

## Debugging Tool Execution

### Enable Debug Logging

```bash
# Set environment variable
export OPENCODE_LOG_LEVEL=debug

# Or in config
{
  "logging": {
    "level": "debug"
  }
}
```

### View Tool Calls

In the UI, tool calls are shown with:
- **Tool name** (e.g., "bash")
- **Input parameters**
- **Execution status** (pending/running/completed/error)
- **Output** (truncated if large)
- **Execution time**
- **Metadata** (exit codes, file paths, etc.)

### Export Session for Analysis

```bash
opencode export <session-id> > session.json

# View tool calls
jq '.messages[] | select(.role == "assistant") | .parts[] | select(.type == "tool")' session.json
```

---

## 中文版本

### 概述

本文档详细说明了 OpenCode 中每个工具的输出指令如何被执行，涵盖从工具定义到执行结果的完整生命周期。

### 完整的工具执行管道

```
1. 工具定义 (tool/*.ts)
   ↓
2. 工具注册 (registry.ts)
   ↓
3. 模型调用 (llm.ts)
   ↓
4. AI SDK 工具调用 (streamText)
   ↓
5. 工具执行 (processor.ts)
   ↓
6. 结果处理
   ↓
7. 下一次迭代 (如需要则循环回步骤 3)
```

---

## 第1部分：工具定义

### 核心概念

每个工具都遵循 `Tool.Info` 接口，包含：

1. **ID**: 唯一标识符 (如 "bash", "read", "edit")
2. **Description**: 自然语言描述，供 AI 模型理解
3. **Parameters**: Zod 模式定义的输入参数
4. **Execute 函数**: 实际执行逻辑

### Bash 工具示例

**执行流程**:
1. 验证工作目录
2. 解析命令（使用 tree-sitter）
3. 检查权限
4. 执行命令
5. 返回结果（stdout + stderr）

### Read 工具示例

**执行流程**:
1. 解析绝对路径
2. 检查权限
3. 读取文件
4. 处理图片/PDF（作为附件）
5. 返回文本内容（带行号、分页）

### Edit 工具示例

**执行流程**:
1. 解析路径
2. 锁定文件
3. 读取当前内容
4. 应用替换
5. 生成 diff
6. 请求权限（显示 diff）
7. 写入文件
8. 发布事件

---

## 第2部分：工具注册

### 内置工具

OpenCode 注册了 20+ 个内置工具：

- **InvalidTool**: 无效工具调用的回退
- **QuestionTool**: 用户交互（仅UI客户端）
- **BashTool**: 命令执行
- **ReadTool**: 文件读取
- **GlobTool**: 文件模式匹配
- **GrepTool**: 内容搜索
- **EditTool**: 文件编辑
- **WriteTool**: 文件写入
- **TaskTool**: 子代理调用
- **WebFetchTool**: HTTP 请求
- **TodoWriteTool/TodoReadTool**: 待办事项管理
- **WebSearchTool/CodeSearchTool**: 网络/代码搜索
- **SkillTool**: 技能执行
- **ApplyPatchTool**: 补丁应用（GPT模型）
- **LspTool**: 语言服务器（实验性）
- **BatchTool**: 批处理操作（实验性）

### 自定义工具

用户可以通过以下方式添加自定义工具：

1. **项目级别**: `.opencode/tool/*.{js,ts}`
2. **用户级别**: `~/.config/opencode/tool/*.{js,ts}`
3. **插件**: 通过插件系统

### 工具过滤

不是所有工具都适用于所有模型：

- **websearch/codesearch**: 仅 OpenCode 提供商或启用 Flag
- **apply_patch**: 仅 GPT 模型
- **edit/write**: 非 GPT 模型（GPT 使用 apply_patch）

---

## 第3部分：模型调用与工具流式传输

### 工具如何发送给 AI 模型

调用 AI 模型时，工具通过 Vercel AI SDK 提供：

```typescript
return streamText({
  model: language,
  messages: [...],
  tools,  // <-- 工具传递给 AI 模型
  activeTools: Object.keys(tools),
  // ...
})
```

### 不同提供商的工具格式

- **OpenAI**: JSON 模式，`tools` 参数
- **Anthropic**: 基于 XML 的工具定义
- **Google Gemini**: FunctionDeclaration 格式
- **其他**: OpenAI 兼容格式

---

## 第4部分：AI 模型决策

### AI 如何决定调用工具

AI 模型接收：
1. **系统提示词**: 何时/如何使用工具的指令
2. **工具定义**: 每个工具的功能和参数
3. **对话历史**: 之前的消息和工具结果
4. **用户请求**: 当前要完成的任务

基于这些上下文，模型决定：
- **调用哪个工具** （如 "read" 用于文件访问）
- **使用什么参数** （如 `{filePath: "/path/to/file"}`）
- **何时调用多个工具** （并行或顺序）
- **何时停止** 并返回最终答案

---

## 第5部分：SessionProcessor 中的工具执行

### 工具执行生命周期

```typescript
switch (value.type) {
  case "tool-input-start":
    // AI 开始生成工具调用
    // → 创建待处理的工具部分
    
  case "tool-call":
    // AI 完成生成工具调用
    // → 验证参数
    // → 执行工具的 execute() 函数
    
  case "tool-result":
    // 工具执行成功
    // → 存储输出
    // → 发送结果回 AI 模型
    
  case "tool-error":
    // 工具执行失败
    // → 存储错误消息
    // → AI 可以重试或请求澄清
}
```

### 工具执行状态

1. **pending**: AI 正在生成工具调用
2. **running**: 工具正在执行
3. **completed**: 执行成功
4. **error**: 执行失败

---

## 第6部分：权限系统

### 权限如何工作

许多工具在执行前需要用户权限：

```typescript
await ctx.ask({
  permission: "bash",           // 权限类型
  patterns: ["rm -rf *"],      // 要检查的模式
  always: ["rm", "sudo"],      // 总是询问这些
  metadata: {                  // 附加上下文
    command: params.command
  }
})
```

### 权限类型

- **bash**: 命令执行
- **read**: 文件读取
- **edit**: 文件修改
- **write**: 文件创建
- **external_directory**: 访问项目外部
- **doom_loop**: 重复的工具调用

### 权限状态

1. **总是允许**: 模式匹配代理的允许列表
2. **总是拒绝**: 模式匹配拒绝列表
3. **询问用户**: 提示用户做决定（UI/CLI）
4. **被拒绝**: 用户拒绝权限 → 抛出 `PermissionNext.RejectedError`

---

## 第7部分：结果处理与迭代

### 工具执行后

工具完成后：

```typescript
// 1. 工具返回结果
return {
  title: "命令已执行",
  output: "stdout + stderr",
  metadata: { exitCode: 0 }
}

// 2. 结果发送回 AI 模型
// AI 处理结果：
// - 可以调用更多工具
// - 可以问后续问题
// - 可以提供最终答案

// 3. 循环继续，直到：
// - AI 提供文本响应（不再调用工具）
// - 用户停止会话
// - 发生错误
// - 达到令牌限制
```

### 多步骤示例

```
用户: "创建一个名为 test.txt 的文件，内容为 'Hello World'，然后读取它"

步骤 1: AI 调用 'write' 工具
  → write({filePath: "test.txt", content: "Hello World"})
  → 结果: "文件创建成功"

步骤 2: AI 调用 'read' 工具
  → read({filePath: "test.txt"})
  → 结果: "Hello World"

步骤 3: AI 提供最终响应
  → "我已创建 test.txt，内容为 'Hello World'，并验证了内容匹配。"
```

---

## 第8部分：错误处理

### 参数验证错误

```typescript
try {
  toolInfo.parameters.parse(args)  // 使用 Zod 验证
} catch (error) {
  throw new Error(
    `${id} 工具使用了无效参数: ${error}。
     请重写输入以满足预期的模式。`
  )
}
```

### 执行错误

```typescript
try {
  const result = await someOperation()
  return { title, output: result, metadata: {} }
} catch (error) {
  // 错误被 SessionProcessor 捕获
  // → 作为 tool-error 事件发送给 AI
  // → AI 可以使用不同参数重试
  throw new Error(`执行失败: ${error.message}`)
}
```

---

## 第9部分：完整示例追踪

### 真实执行流程

```
1. 用户输入: "显示 src/ 中的文件"

2. AI 模型思考:
   "需要列出目录内容 → 使用 'bash' 工具和 'ls' 命令"

3. AI 生成工具调用:
   {
     "toolName": "bash",
     "input": {
       "command": "ls -la src/",
       "description": "列出 src 目录中的文件"
     }
   }

4. SessionProcessor 接收 tool-call 事件:
   - 创建状态为 "running" 的工具部分
   - 验证参数（Zod 模式检查）
   - 调用 BashTool.execute()

5. BashTool.execute() 运行:
   a. 使用 tree-sitter 解析命令
   b. 检查权限（自动批准安全命令）
   c. 通过 Shell.exec() 执行
   d. 返回结果

6. SessionProcessor 接收 tool-result 事件:
   - 更新工具部分状态为 "completed"
   - 将结果发送回 AI 模型

7. AI 模型接收结果并生成文本响应

8. SessionProcessor 接收 text-delta 事件:
   - 实时流式传输响应给用户

9. 会话完成
```

---

## 第10部分：高级功能

### 输出截断

大型工具输出会自动截断：

限制：
- **MAX_LINES**: 10,000 行
- **MAX_BYTES**: 100 KB

如果超出，完整输出保存到 `.opencode/output/{hash}.txt`

### 并行工具调用

某些 AI 模型支持并行工具执行：

```typescript
// AI 可以同时调用多个工具：
{
  "toolCalls": [
    { "toolName": "read", "input": { "filePath": "file1.txt" } },
    { "toolName": "read", "input": { "filePath": "file2.txt" } },
    { "toolName": "bash", "input": { "command": "git status" } }
  ]
}

// 三个工具并行执行
// 结果收集后一起发送回去
```

### 工具上下文

每次工具执行都接收一个上下文对象：

```typescript
type Context = {
  sessionID: string        // 当前会话 ID
  messageID: string        // 当前消息 ID
  agent: string           // 代理名称（如 "build"）
  abort: AbortSignal      // 取消信号
  
  // 更新工具元数据
  metadata(input: { title?: string; metadata?: any }): void
  
  // 请求用户权限
  ask(input: PermissionNext.Request): Promise<void>
}
```

### 文件附件

工具可以返回文件附件（图片、PDF）：

```typescript
return {
  title: "图片读取成功",
  output: "图片: example.png",
  metadata: { truncated: false },
  attachments: [{
    type: "file",
    file: {
      name: "example.png",
      type: "image/png",
      data: base64EncodedData
    }
  }]
}
```

AI 模型接收图片并可以分析它（如果支持视觉）。

---

## 代码位置总结

| 组件 | 文件路径 |
|------|---------|
| 工具接口 | `/packages/opencode/src/tool/tool.ts` |
| 工具注册表 | `/packages/opencode/src/tool/registry.ts` |
| Bash 工具 | `/packages/opencode/src/tool/bash.ts` |
| Read 工具 | `/packages/opencode/src/tool/read.ts` |
| Edit 工具 | `/packages/opencode/src/tool/edit.ts` |
| LLM 调用 | `/packages/opencode/src/session/llm.ts` |
| 会话处理器 | `/packages/opencode/src/session/processor.ts` |
| 权限系统 | `/packages/opencode/src/permission/next.ts` |
| 输出截断 | `/packages/opencode/src/tool/truncation.ts` |

---

## 调试工具执行

### 启用调试日志

```bash
# 设置环境变量
export OPENCODE_LOG_LEVEL=debug

# 或在配置中
{
  "logging": {
    "level": "debug"
  }
}
```

### 导出会话用于分析

```bash
opencode export <session-id> > session.json

# 查看工具调用
jq '.messages[] | select(.role == "assistant") | .parts[] | select(.type == "tool")' session.json
```

---

## 总结

OpenCode 的工具执行系统是一个完整的管道：

1. **定义**: 使用 `Tool.define()` 创建工具
2. **注册**: 通过 `ToolRegistry` 加载工具
3. **调用**: AI 模型通过 Vercel AI SDK 调用工具
4. **执行**: `SessionProcessor` 处理工具生命周期
5. **结果**: 输出返回给 AI，继续对话

这个系统提供了：
- ✅ 类型安全（TypeScript + Zod）
- ✅ 权限控制（用户批准）
- ✅ 错误处理（验证 + 重试）
- ✅ 性能优化（并行执行 + 截断）
- ✅ 可扩展性（自定义工具 + 插件）

理解这个流程对于调试、优化和扩展 OpenCode 的工具系统至关重要。

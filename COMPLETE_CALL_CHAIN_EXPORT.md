# Complete Call Chain Export with Subagents | 完整调用链路导出（包含子代理）

[English](#english) | [中文](#中文)

---

## English

### Problem Statement

The `--print-logs` flag only prints logs for the main agent's context. When subagents are invoked via the `task` tool, their call chains are not included in the output. Users need a way to get the complete call chain including all subagent calls.

### Understanding the Current Behavior

#### How --print-logs Works

**Location**: `/packages/opencode/src/index.ts` lines 50-68

```typescript
.option("print-logs", {
  describe: "print logs to stderr",
  type: "boolean",
})
.middleware(async (opts) => {
  await Log.init({
    print: process.argv.includes("--print-logs"),
    dev: Installation.isLocal(),
    level: (...),
  })
})
```

**Behavior**:
- Only prints logs to `stderr` for the current session (main agent)
- Does not capture or aggregate logs from child sessions (subagents)
- Each subagent runs in a separate session with its own isolated log stream

#### Session Hierarchy Structure

**Location**: `/packages/opencode/src/session/index.ts`

```typescript
export namespace Session {
  export const Info = z.object({
    id: z.string(),
    parentID: Identifier.schema("session").optional(),  // Points to parent session
    // ... other fields
  })
  
  // Get all child sessions (subagents)
  export const children = fn(Identifier.schema("session"), async (parentID) => {
    const project = Instance.project
    const result = [] as Session.Info[]
    for (const item of await Storage.list(["session", project.id])) {
      const session = await Storage.read<Info>(item)
      if (session.parentID !== parentID) continue
      result.push(session)
    }
    return result
  })
}
```

**Key Points**:
1. When a main agent calls `task` tool, a new session is created with `parentID` set to the main session ID
2. Each subagent session is completely independent with its own message history
3. The `Session.children()` API can retrieve all child sessions
4. Currently, `opencode export` only exports a single session, not the hierarchy

---

### Solutions

### Solution 1: Recursive Session Export Script (Recommended)

Create a script that recursively exports the entire session hierarchy.

**File**: `scripts/export-with-subagents.ts`

```typescript
#!/usr/bin/env bun
import { Session } from "../packages/opencode/src/session"
import { bootstrap } from "../packages/opencode/src/cli/bootstrap"

interface SessionExport {
  info: Session.Info
  messages: any[]
  children: SessionExport[]
}

async function exportSessionRecursively(sessionID: string): Promise<SessionExport> {
  const sessionInfo = await Session.get(sessionID)
  const messages = await Session.messages({ sessionID })
  
  // Get all child sessions (subagents)
  const childSessions = await Session.children(sessionID)
  
  // Recursively export each child
  const children = await Promise.all(
    childSessions.map(child => exportSessionRecursively(child.id))
  )
  
  return {
    info: sessionInfo,
    messages: messages.map((msg) => ({
      info: msg.info,
      parts: msg.parts,
    })),
    children,
  }
}

// Usage
const sessionID = process.argv[2]
if (!sessionID) {
  console.error("Usage: bun export-with-subagents.ts <session-id>")
  process.exit(1)
}

await bootstrap(process.cwd(), async () => {
  const fullExport = await exportSessionRecursively(sessionID)
  console.log(JSON.stringify(fullExport, null, 2))
})
```

**Usage**:
```bash
# Export complete call chain to file
bun scripts/export-with-subagents.ts <session-id> > full-trace.json

# Analyze with jq
bun scripts/export-with-subagents.ts <session-id> | jq '.children[].info.title'

# Count total subagent calls
bun scripts/export-with-subagents.ts <session-id> | jq '[.children] | length'

# Get all tool calls across all sessions
bun scripts/export-with-subagents.ts <session-id> | jq '.. | select(.type? == "tool-call") | .toolName'
```

---

### Solution 2: Enhanced Export Command with --recursive Flag

Add a `--recursive` flag to the built-in `opencode export` command.

**File**: `/packages/opencode/src/cli/cmd/export.ts`

```typescript
export const ExportCommand = cmd({
  command: "export [sessionID]",
  describe: "export session data as JSON",
  builder: (yargs: Argv) => {
    return yargs
      .positional("sessionID", {
        describe: "session id to export",
        type: "string",
      })
      .option("recursive", {
        describe: "include all subagent sessions in export",
        type: "boolean",
        default: false,
      })
  },
  handler: async (args) => {
    await bootstrap(process.cwd(), async () => {
      // ... existing session selection code ...
      
      if (args.recursive) {
        // Export with full hierarchy
        const exportData = await exportSessionRecursively(sessionID!)
        process.stdout.write(JSON.stringify(exportData, null, 2))
      } else {
        // Original single-session export
        const sessionInfo = await Session.get(sessionID!)
        const messages = await Session.messages({ sessionID: sessionID! })
        
        const exportData = {
          info: sessionInfo,
          messages: messages.map((msg) => ({
            info: msg.info,
            parts: msg.parts,
          })),
        }
        
        process.stdout.write(JSON.stringify(exportData, null, 2))
      }
      process.stdout.write(EOL)
    })
  },
})

async function exportSessionRecursively(sessionID: string) {
  const sessionInfo = await Session.get(sessionID)
  const messages = await Session.messages({ sessionID })
  const childSessions = await Session.children(sessionID)
  
  const children = await Promise.all(
    childSessions.map(child => exportSessionRecursively(child.id))
  )
  
  return {
    info: sessionInfo,
    messages: messages.map((msg) => ({
      info: msg.info,
      parts: msg.parts,
    })),
    children,
  }
}
```

**Usage**:
```bash
# Export with subagents
opencode export <session-id> --recursive > full-trace.json

# Export without subagents (original behavior)
opencode export <session-id> > single-session.json
```

---

### Solution 3: Real-time Log Aggregation

Aggregate logs from all sessions in real-time during execution.

**Implementation Approach**:

1. **Modify Log.init()** to support aggregation mode
2. **Track session hierarchy** in memory during execution
3. **Forward subagent logs** to main process

**File**: `/packages/opencode/src/util/log.ts` (enhancement needed)

```typescript
export namespace Log {
  let aggregationMode = false
  let parentLogger: Log | undefined
  
  export async function init(input: {
    print: boolean
    dev: boolean
    level: Level
    aggregate?: boolean  // NEW: Enable log aggregation
    parent?: Log         // NEW: Parent logger for subagents
  }) {
    aggregationMode = input.aggregate ?? false
    parentLogger = input.parent
    
    // ... existing init code ...
  }
  
  // Forward logs to parent if in aggregation mode
  private function forwardToParent(level: Level, message: string, data: any) {
    if (aggregationMode && parentLogger) {
      parentLogger[level](message, { ...data, _subagent: true })
    }
  }
}
```

**Usage**:
```bash
# Start with log aggregation enabled
opencode --print-logs --aggregate-subagent-logs <prompt>
```

**Note**: This requires code changes to OpenCode core and is more complex to implement.

---

### Solution 4: Post-Execution Hierarchy Analysis

After execution completes, analyze the session hierarchy and generate a complete trace.

**File**: `scripts/analyze-call-chain.ts`

```typescript
#!/usr/bin/env bun
import { Session } from "../packages/opencode/src/session"
import { bootstrap } from "../packages/opencode/src/cli/bootstrap"

interface CallChainNode {
  sessionID: string
  title: string
  depth: number
  toolCalls: number
  children: CallChainNode[]
  duration: number
}

async function analyzeCallChain(sessionID: string, depth = 0): Promise<CallChainNode> {
  const info = await Session.get(sessionID)
  const messages = await Session.messages({ sessionID })
  
  // Count tool calls
  const toolCalls = messages.reduce((count, msg) => {
    return count + (msg.parts.filter(p => p.type === "tool-call").length)
  }, 0)
  
  // Calculate duration
  const duration = info.time.updated - info.time.created
  
  // Get children
  const childSessions = await Session.children(sessionID)
  const children = await Promise.all(
    childSessions.map(child => analyzeCallChain(child.id, depth + 1))
  )
  
  return {
    sessionID: info.id,
    title: info.title,
    depth,
    toolCalls,
    duration,
    children,
  }
}

function printCallChain(node: CallChainNode, indent = "") {
  const duration = (node.duration / 1000).toFixed(2)
  console.log(`${indent}├─ ${node.title}`)
  console.log(`${indent}│  ID: ${node.sessionID.slice(-8)}`)
  console.log(`${indent}│  Tool Calls: ${node.toolCalls}`)
  console.log(`${indent}│  Duration: ${duration}s`)
  console.log(`${indent}│  Subagents: ${node.children.length}`)
  
  node.children.forEach((child, i) => {
    const isLast = i === node.children.length - 1
    const childIndent = indent + (isLast ? "   " : "│  ")
    console.log(`${indent}│`)
    printCallChain(child, childIndent)
  })
}

// Usage
const sessionID = process.argv[2]
if (!sessionID) {
  console.error("Usage: bun analyze-call-chain.ts <session-id>")
  process.exit(1)
}

await bootstrap(process.cwd(), async () => {
  const callChain = await analyzeCallChain(sessionID)
  
  console.log("\n=== Complete Call Chain ===\n")
  printCallChain(callChain)
  
  console.log("\n=== Statistics ===")
  console.log(`Total Sessions: ${countSessions(callChain)}`)
  console.log(`Total Tool Calls: ${countToolCalls(callChain)}`)
  console.log(`Max Depth: ${findMaxDepth(callChain)}`)
})

function countSessions(node: CallChainNode): number {
  return 1 + node.children.reduce((sum, child) => sum + countSessions(child), 0)
}

function countToolCalls(node: CallChainNode): number {
  return node.toolCalls + node.children.reduce((sum, child) => sum + countToolCalls(child), 0)
}

function findMaxDepth(node: CallChainNode): number {
  if (node.children.length === 0) return node.depth
  return Math.max(...node.children.map(findMaxDepth))
}
```

**Output Example**:
```
=== Complete Call Chain ===

├─ Implement user authentication
│  ID: abc12345
│  Tool Calls: 15
│  Duration: 45.32s
│  Subagents: 2
│
│  ├─ Explore codebase structure
│  │  ID: def67890
│  │  Tool Calls: 8
│  │  Duration: 12.45s
│  │  Subagents: 0
│
│  ├─ Write tests for auth module
│     ID: ghi34567
│     Tool Calls: 6
│     Duration: 18.67s
│     Subagents: 1
│
│     ├─ Review test patterns
│        ID: jkl89012
│        Tool Calls: 3
│        Duration: 5.23s
│        Subagents: 0

=== Statistics ===
Total Sessions: 4
Total Tool Calls: 32
Max Depth: 2
```

---

### Solution Comparison

| Solution | Implementation Difficulty | Real-time | Complete Data | Use Case |
|----------|--------------------------|-----------|---------------|----------|
| **Solution 1: Recursive Export Script** | Easy | ❌ No | ✅ Yes | Post-execution analysis |
| **Solution 2: Enhanced Export Command** | Medium | ❌ No | ✅ Yes | Built-in feature |
| **Solution 3: Real-time Aggregation** | Hard | ✅ Yes | ✅ Yes | Live debugging |
| **Solution 4: Hierarchy Analysis** | Easy | ❌ No | ✅ Yes | Quick overview |

**Recommendations**:
- **For immediate use**: Solution 1 (Recursive Export Script)
- **For built-in feature**: Solution 2 (Enhanced Export Command)
- **For debugging**: Solution 4 (Hierarchy Analysis)
- **For advanced users**: Solution 3 (Real-time Aggregation)

---

### Implementation Steps

#### Quick Start: Recursive Export Script

1. **Create the script**:
```bash
mkdir -p scripts
touch scripts/export-with-subagents.ts
chmod +x scripts/export-with-subagents.ts
```

2. **Add the code** (see Solution 1 above)

3. **Run it**:
```bash
# Get your session ID
opencode export  # Select session, copy ID from output

# Export complete hierarchy
bun scripts/export-with-subagents.ts <session-id> > full-trace.json

# View structure
cat full-trace.json | jq '.children[].info.title'
```

#### Feature Request: Built-in --recursive Flag

If you want this as a built-in feature, consider:

1. **Fork the repository**
2. **Implement Solution 2** in `/packages/opencode/src/cli/cmd/export.ts`
3. **Add tests** for recursive export
4. **Submit a pull request** with:
   - Implementation of `--recursive` flag
   - Documentation update
   - Test coverage
   - Example usage

---

### Debugging Complete Call Chains

#### Method 1: Export and Analyze

```bash
# Export full hierarchy
bun scripts/export-with-subagents.ts <session-id> > full-trace.json

# Count subagents
cat full-trace.json | jq '[.. | .children?] | flatten | length'

# List all session titles
cat full-trace.json | jq '.. | select(.info?) | .info.title'

# Extract all tool calls
cat full-trace.json | jq '.. | select(.type? == "tool-call") | {tool: .toolName, input: .toolInput}'

# Find longest execution path
cat full-trace.json | jq '[.. | .info?] | map(.time.updated - .time.created) | max'
```

#### Method 2: Visual Tree Representation

```bash
# Use the hierarchy analysis script
bun scripts/analyze-call-chain.ts <session-id>
```

#### Method 3: Database Query

```bash
# Find all sessions with a parent
sqlite3 ~/.opencode/project/<project-id>/storage.db \
  "SELECT key FROM entries WHERE key LIKE 'session/%' AND value LIKE '%parentID%'"

# Find children of specific session
sqlite3 ~/.opencode/project/<project-id>/storage.db \
  "SELECT key, value FROM entries 
   WHERE key LIKE 'session/%' 
   AND value LIKE '%\"parentID\":\"<parent-session-id>\"%'"
```

---

### Related Files

| File | Purpose |
|------|---------|
| `/packages/opencode/src/index.ts` | CLI entry point with `--print-logs` flag |
| `/packages/opencode/src/cli/cmd/export.ts` | Export command implementation |
| `/packages/opencode/src/session/index.ts` | Session management and hierarchy |
| `/packages/opencode/src/tool/task.ts` | Task tool that creates subagent sessions |
| `/packages/opencode/src/util/log.ts` | Logging system |

---

## 中文

### 问题描述

`--print-logs` 标志只打印主代理的上下文日志。当通过 `task` 工具调用子代理时，它们的调用链路不会包含在输出中。用户需要一种方法来获取包括所有子代理调用的完整调用链路。

### 理解当前行为

#### --print-logs 如何工作

**位置**: `/packages/opencode/src/index.ts` 第 50-68 行

```typescript
.option("print-logs", {
  describe: "print logs to stderr",
  type: "boolean",
})
.middleware(async (opts) => {
  await Log.init({
    print: process.argv.includes("--print-logs"),
    dev: Installation.isLocal(),
    level: (...),
  })
})
```

**行为**:
- 仅将当前会话（主代理）的日志打印到 `stderr`
- 不捕获或聚合子会话（子代理）的日志
- 每个子代理在独立的会话中运行，有自己隔离的日志流

#### 会话层次结构

**位置**: `/packages/opencode/src/session/index.ts`

```typescript
export namespace Session {
  export const Info = z.object({
    id: z.string(),
    parentID: Identifier.schema("session").optional(),  // 指向父会话
    // ... 其他字段
  })
  
  // 获取所有子会话（子代理）
  export const children = fn(Identifier.schema("session"), async (parentID) => {
    const project = Instance.project
    const result = [] as Session.Info[]
    for (const item of await Storage.list(["session", project.id])) {
      const session = await Storage.read<Info>(item)
      if (session.parentID !== parentID) continue
      result.push(session)
    }
    return result
  })
}
```

**要点**:
1. 当主代理调用 `task` 工具时，会创建一个新会话，其 `parentID` 设置为主会话 ID
2. 每个子代理会话完全独立，有自己的消息历史
3. `Session.children()` API 可以检索所有子会话
4. 目前 `opencode export` 只导出单个会话，不包括层次结构

---

### 解决方案

### 方案 1：递归会话导出脚本（推荐）

创建一个递归导出整个会话层次结构的脚本。

**文件**: `scripts/export-with-subagents.ts`

```typescript
#!/usr/bin/env bun
import { Session } from "../packages/opencode/src/session"
import { bootstrap } from "../packages/opencode/src/cli/bootstrap"

interface SessionExport {
  info: Session.Info
  messages: any[]
  children: SessionExport[]
}

async function exportSessionRecursively(sessionID: string): Promise<SessionExport> {
  const sessionInfo = await Session.get(sessionID)
  const messages = await Session.messages({ sessionID })
  
  // 获取所有子会话（子代理）
  const childSessions = await Session.children(sessionID)
  
  // 递归导出每个子会话
  const children = await Promise.all(
    childSessions.map(child => exportSessionRecursively(child.id))
  )
  
  return {
    info: sessionInfo,
    messages: messages.map((msg) => ({
      info: msg.info,
      parts: msg.parts,
    })),
    children,
  }
}

// 使用方法
const sessionID = process.argv[2]
if (!sessionID) {
  console.error("用法: bun export-with-subagents.ts <session-id>")
  process.exit(1)
}

await bootstrap(process.cwd(), async () => {
  const fullExport = await exportSessionRecursively(sessionID)
  console.log(JSON.stringify(fullExport, null, 2))
})
```

**使用方法**:
```bash
# 导出完整调用链路到文件
bun scripts/export-with-subagents.ts <session-id> > full-trace.json

# 使用 jq 分析
bun scripts/export-with-subagents.ts <session-id> | jq '.children[].info.title'

# 统计子代理调用总数
bun scripts/export-with-subagents.ts <session-id> | jq '[.children] | length'

# 获取所有会话的工具调用
bun scripts/export-with-subagents.ts <session-id> | jq '.. | select(.type? == "tool-call") | .toolName'
```

---

### 方案 2：增强导出命令（添加 --recursive 标志）

为内置的 `opencode export` 命令添加 `--recursive` 标志。

**使用方法**:
```bash
# 导出包含子代理
opencode export <session-id> --recursive > full-trace.json

# 导出不包含子代理（原始行为）
opencode export <session-id> > single-session.json
```

（实现代码见英文版本）

---

### 方案 3：实时日志聚合

在执行期间实时聚合所有会话的日志。

**使用方法**:
```bash
# 启用日志聚合
opencode --print-logs --aggregate-subagent-logs <prompt>
```

**注意**: 需要修改 OpenCode 核心代码，实现较复杂。

---

### 方案 4：执行后层次分析

执行完成后分析会话层次并生成完整追踪。

**文件**: `scripts/analyze-call-chain.ts`

（代码见英文版本）

**输出示例**:
```
=== 完整调用链路 ===

├─ 实现用户认证
│  ID: abc12345
│  工具调用: 15
│  持续时间: 45.32s
│  子代理: 2
│
│  ├─ 探索代码库结构
│  │  ID: def67890
│  │  工具调用: 8
│  │  持续时间: 12.45s
│  │  子代理: 0
│
│  ├─ 为认证模块编写测试
│     ID: ghi34567
│     工具调用: 6
│     持续时间: 18.67s
│     子代理: 1
│
│     ├─ 审查测试模式
│        ID: jkl89012
│        工具调用: 3
│        持续时间: 5.23s
│        子代理: 0

=== 统计信息 ===
总会话数: 4
总工具调用: 32
最大深度: 2
```

---

### 方案对比

| 方案 | 实现难度 | 实时 | 完整数据 | 使用场景 |
|------|---------|------|---------|---------|
| **方案 1: 递归导出脚本** | 简单 | ❌ 否 | ✅ 是 | 执行后分析 |
| **方案 2: 增强导出命令** | 中等 | ❌ 否 | ✅ 是 | 内置功能 |
| **方案 3: 实时聚合** | 困难 | ✅ 是 | ✅ 是 | 实时调试 |
| **方案 4: 层次分析** | 简单 | ❌ 否 | ✅ 是 | 快速概览 |

**推荐**:
- **立即使用**: 方案 1（递归导出脚本）
- **内置功能**: 方案 2（增强导出命令）
- **调试用**: 方案 4（层次分析）
- **高级用户**: 方案 3（实时聚合）

---

### 快速开始

#### 使用递归导出脚本

1. **创建脚本**:
```bash
mkdir -p scripts
touch scripts/export-with-subagents.ts
chmod +x scripts/export-with-subagents.ts
```

2. **添加代码**（见方案 1）

3. **运行**:
```bash
# 获取会话 ID
opencode export  # 选择会话，从输出复制 ID

# 导出完整层次
bun scripts/export-with-subagents.ts <session-id> > full-trace.json

# 查看结构
cat full-trace.json | jq '.children[].info.title'
```

---

### 调试完整调用链路

#### 方法 1: 导出并分析

```bash
# 导出完整层次
bun scripts/export-with-subagents.ts <session-id> > full-trace.json

# 统计子代理数量
cat full-trace.json | jq '[.. | .children?] | flatten | length'

# 列出所有会话标题
cat full-trace.json | jq '.. | select(.info?) | .info.title'

# 提取所有工具调用
cat full-trace.json | jq '.. | select(.type? == "tool-call") | {tool: .toolName, input: .toolInput}'
```

#### 方法 2: 可视化树形表示

```bash
# 使用层次分析脚本
bun scripts/analyze-call-chain.ts <session-id>
```

#### 方法 3: 数据库查询

```bash
# 查找所有有父会话的会话
sqlite3 ~/.opencode/project/<project-id>/storage.db \
  "SELECT key FROM entries WHERE key LIKE 'session/%' AND value LIKE '%parentID%'"
```

---

### 相关文件

| 文件 | 用途 |
|------|------|
| `/packages/opencode/src/index.ts` | CLI 入口点，包含 `--print-logs` 标志 |
| `/packages/opencode/src/cli/cmd/export.ts` | 导出命令实现 |
| `/packages/opencode/src/session/index.ts` | 会话管理和层次结构 |
| `/packages/opencode/src/tool/task.ts` | 创建子代理会话的 Task 工具 |
| `/packages/opencode/src/util/log.ts` | 日志系统 |

---

### 总结

**问题**: `--print-logs` 只显示主代理日志，不包含子代理调用链路

**原因**: 
- 每个子代理在独立会话中运行
- 日志系统不自动聚合子会话日志
- 会话通过 `parentID` 关联但数据隔离

**解决方案**:
1. **递归导出脚本** - 最简单，立即可用
2. **增强导出命令** - 需要修改代码，作为内置功能
3. **实时日志聚合** - 最复杂，需要核心修改
4. **层次分析工具** - 提供可视化概览

**推荐使用**: 方案 1（递归导出脚本）+ 方案 4（层次分析）组合使用，既能获得完整数据，又能快速查看结构。

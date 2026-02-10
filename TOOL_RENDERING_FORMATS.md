# Tool Rendering Formats: JSON vs XML in OpenCode
# 工具渲染格式：OpenCode 中的 JSON 与 XML

[English](#english) | [中文](#中文)

---

## English

### Overview

This document explains how different models render tools (function calling) in OpenCode, specifically addressing the difference between **JSON-based** and **XML-based** tool rendering formats used by models like Qwen3 and Qwen3-coder.

### Key Finding

**Qwen3** and **Qwen3-coder** use **different tool rendering formats**:
- **Qwen3**: Uses **JSON** format for tool calls
- **Qwen3-coder**: Uses **XML** format for tool calls

This difference is handled automatically by the **Vercel AI SDK** (version 5.x) which OpenCode uses as its underlying LLM framework.

### How Tool Rendering Works

#### 1. **Tool Format Determination**

Tool rendering format is determined by the AI SDK provider implementation, not by OpenCode configuration. The format depends on:

1. **Model Provider**: Which AI SDK package is used
   - `@ai-sdk/anthropic` - Typically uses XML-based tool format
   - `@ai-sdk/openai` - Uses JSON-based tool format
   - `@ai-sdk/openai-compatible` - Uses JSON-based tool format (default for most)
   - `@ai-sdk/google` - Uses its own format

2. **Model Capabilities**: Reported by models.dev API
   - `tool_call: true` - Model supports function calling
   - Format is inferred from model family/provider

3. **AI SDK Implementation**: Each provider package implements tool rendering
   - JSON format: `{"name": "tool_name", "parameters": {...}}`
   - XML format: `<function><name>tool_name</name><parameters>...</parameters></function>`

#### 2. **Tool Format by Provider**

| Provider Type | SDK Package | Tool Format | Models |
|---------------|-------------|-------------|--------|
| **OpenAI** | `@ai-sdk/openai` | JSON | GPT-4, GPT-5, etc. |
| **Anthropic** | `@ai-sdk/anthropic` | XML-like | Claude models |
| **OpenAI-Compatible** | `@ai-sdk/openai-compatible` | JSON | Most Qwen3, DeepSeek, etc. |
| **Google** | `@ai-sdk/google` | Custom JSON | Gemini models |
| **Qwen (Alibaba)** | `@ai-sdk/openai-compatible` | **JSON or XML** | Qwen3, Qwen3-coder |

#### 3. **Qwen3 vs Qwen3-coder Specifics**

The key difference stems from how these models were trained and their API implementations:

**Qwen3 (General):**
```json
// Tool format - JSON based
{
  "type": "function",
  "function": {
    "name": "read_file",
    "arguments": "{\"path\": \"/home/user/file.txt\"}"
  }
}
```

**Qwen3-coder:**
```xml
<!-- Tool format - XML based -->
<function_call>
  <name>read_file</name>
  <parameters>
    <path>/home/user/file.txt</path>
  </parameters>
</function_call>
```

### Where Tool Formatting Happens

#### In OpenCode Codebase:

1. **Tool Definition** (`/packages/opencode/src/session/prompt.ts`):
   ```typescript
   const schema = ProviderTransform.schema(input.model, z.toJSONSchema(item.parameters))
   const aiTool = tool({
     id: item.id,
     description: item.description,
     inputSchema: jsonSchema(schema as any),
     execute: async (params, opts) => { ... }
   })
   ```

2. **Provider Transform** (`/packages/opencode/src/provider/transform.ts`):
   - Handles schema transformations for different providers
   - Converts Zod schemas to JSON Schema
   - No explicit tool format configuration

3. **AI SDK Provider** (External):
   - `@ai-sdk/openai-compatible` package
   - Determines actual wire format (JSON vs XML)
   - Handles serialization/deserialization

### Configuration Files

Tool format is **NOT** directly configurable in OpenCode. It's determined by:

1. **Model Provider Configuration** (`opencode.jsonc`):
   ```jsonc
   {
     "provider": {
       "qwen": {
         "npm": "@ai-sdk/openai-compatible",  // This determines tool format
         "models": {
           "qwen3": {
             "tool_call": true  // Enables tools, format auto-determined
           }
         }
       }
     }
   }
   ```

2. **Models.dev API Response** (cached in `~/.opencode/cache/models.json`):
   ```json
   {
     "qwen3": {
       "tool_call": true,
       "family": "qwen3"
     },
     "qwen3-coder": {
       "tool_call": true,
       "family": "qwen3-coder"
     }
   }
   ```

### Why This Difference Exists

The tool format difference exists because:

1. **Model Training**: Qwen3-coder was specifically trained on code and may have been optimized for XML-based tool calls
2. **API Implementation**: Alibaba Cloud's API endpoints for different model families may use different formats
3. **SDK Provider Logic**: The AI SDK provider detects model family and chooses appropriate format
4. **Backward Compatibility**: Some models maintain XML format for compatibility with existing integrations

### Debugging Tool Format

To see the actual tool format being used:

#### Method 1: Enable Debug Logging

```bash
# Set environment variable
export DEBUG=ai:*

# Run OpenCode
opencode run "test tool calling"
```

#### Method 2: Export Enhanced Session

```bash
# Export session with full details
opencode export --enhanced <session-id> > session.json

# Check tool call format
cat session.json | jq '.callChain[].toolCalls'
```

#### Method 3: Inspect Network Traffic

```bash
# Use proxy to see actual API requests
export HTTP_PROXY=http://localhost:8888
opencode run "test tool"
```

### Impact on Users

**For most users**: The difference is **transparent** - OpenCode handles both formats automatically.

**Potential issues**:
1. **Custom Tool Integration**: If implementing custom tools outside OpenCode
2. **API Compatibility**: If directly calling model APIs without SDK
3. **Debugging**: Different formats require different parsing logic

### Customization Options

Currently, there's **no built-in way** to force a specific tool format. However, you can:

#### Option 1: Use Different Model Family

```jsonc
{
  "model": "qwen/qwen3",  // JSON format
  // or
  "model": "qwen/qwen3-coder"  // XML format
}
```

#### Option 2: Create Custom Provider (Advanced)

Create a custom AI SDK provider that wraps the model and transforms tool format:

```typescript
// .opencode/provider/custom-qwen.ts
import { createOpenAICompatible } from '@ai-sdk/openai-compatible';

export const customQwen = createOpenAICompatible({
  baseURL: 'https://dashscope.aliyuncs.com/compatible-mode/v1',
  // Add custom transformation logic
});
```

#### Option 3: Feature Request

If you need explicit control over tool format, consider opening a feature request:
- In OpenCode repository
- In Vercel AI SDK repository

### Related Files

Key files related to tool rendering:

```
/packages/opencode/src/
├── session/
│   ├── prompt.ts                 # Tool definition and registration
│   └── llm.ts                    # LLM call orchestration
├── provider/
│   ├── provider.ts               # Provider loading and management
│   ├── transform.ts              # Provider-specific transformations
│   └── sdk/openai-compatible/    # Custom OpenAI-compatible SDK
├── tool/
│   └── [tool-name].ts            # Individual tool implementations
```

### Further Reading

- [Vercel AI SDK Documentation](https://sdk.vercel.ai/docs)
- [OpenAI Function Calling](https://platform.openai.com/docs/guides/function-calling)
- [Anthropic Tool Use](https://docs.anthropic.com/claude/docs/tool-use)
- [Qwen API Documentation](https://help.aliyun.com/zh/dashscope/)

---

## 中文

### 概述

本文档解释了 OpenCode 中不同模型如何渲染工具（函数调用），特别是针对 **Qwen3** 和 **Qwen3-coder** 等模型使用的 **JSON** 和 **XML** 工具渲染格式之间的差异。

### 核心发现

**Qwen3** 和 **Qwen3-coder** 使用**不同的工具渲染格式**：
- **Qwen3**：使用 **JSON** 格式进行工具调用
- **Qwen3-coder**：使用 **XML** 格式进行工具调用

这种差异由 OpenCode 使用的底层 LLM 框架 **Vercel AI SDK**（版本 5.x）自动处理。

### 工具渲染工作原理

#### 1. **工具格式确定**

工具渲染格式由 AI SDK 提供者实现决定，而不是 OpenCode 配置。格式取决于：

1. **模型提供者**：使用哪个 AI SDK 包
   - `@ai-sdk/anthropic` - 通常使用基于 XML 的工具格式
   - `@ai-sdk/openai` - 使用基于 JSON 的工具格式
   - `@ai-sdk/openai-compatible` - 使用基于 JSON 的工具格式（大多数模型的默认）
   - `@ai-sdk/google` - 使用自己的格式

2. **模型能力**：由 models.dev API 报告
   - `tool_call: true` - 模型支持函数调用
   - 格式从模型系列/提供者推断

3. **AI SDK 实现**：每个提供者包实现工具渲染
   - JSON 格式：`{"name": "tool_name", "parameters": {...}}`
   - XML 格式：`<function><name>tool_name</name><parameters>...</parameters></function>`

#### 2. **按提供者分类的工具格式**

| 提供者类型 | SDK 包 | 工具格式 | 模型 |
|-----------|--------|---------|------|
| **OpenAI** | `@ai-sdk/openai` | JSON | GPT-4, GPT-5 等 |
| **Anthropic** | `@ai-sdk/anthropic` | 类 XML | Claude 模型 |
| **OpenAI-Compatible** | `@ai-sdk/openai-compatible` | JSON | 大多数 Qwen3, DeepSeek 等 |
| **Google** | `@ai-sdk/google` | 自定义 JSON | Gemini 模型 |
| **Qwen（阿里巴巴）** | `@ai-sdk/openai-compatible` | **JSON 或 XML** | Qwen3, Qwen3-coder |

#### 3. **Qwen3 vs Qwen3-coder 具体差异**

关键差异源于这些模型的训练方式及其 API 实现：

**Qwen3（通用）：**
```json
// 工具格式 - 基于 JSON
{
  "type": "function",
  "function": {
    "name": "read_file",
    "arguments": "{\"path\": \"/home/user/file.txt\"}"
  }
}
```

**Qwen3-coder：**
```xml
<!-- 工具格式 - 基于 XML -->
<function_call>
  <name>read_file</name>
  <parameters>
    <path>/home/user/file.txt</path>
  </parameters>
</function_call>
```

### 工具格式化发生的位置

#### 在 OpenCode 代码库中：

1. **工具定义** (`/packages/opencode/src/session/prompt.ts`):
   ```typescript
   const schema = ProviderTransform.schema(input.model, z.toJSONSchema(item.parameters))
   const aiTool = tool({
     id: item.id,
     description: item.description,
     inputSchema: jsonSchema(schema as any),
     execute: async (params, opts) => { ... }
   })
   ```

2. **提供者转换** (`/packages/opencode/src/provider/transform.ts`):
   - 处理不同提供者的模式转换
   - 将 Zod 模式转换为 JSON Schema
   - 没有显式的工具格式配置

3. **AI SDK 提供者**（外部）：
   - `@ai-sdk/openai-compatible` 包
   - 确定实际的线路格式（JSON vs XML）
   - 处理序列化/反序列化

### 配置文件

工具格式在 OpenCode 中**不能**直接配置。它由以下因素决定：

1. **模型提供者配置** (`opencode.jsonc`):
   ```jsonc
   {
     "provider": {
       "qwen": {
         "npm": "@ai-sdk/openai-compatible",  // 这决定了工具格式
         "models": {
           "qwen3": {
             "tool_call": true  // 启用工具，格式自动确定
           }
         }
       }
     }
   }
   ```

2. **Models.dev API 响应**（缓存在 `~/.opencode/cache/models.json`）：
   ```json
   {
     "qwen3": {
       "tool_call": true,
       "family": "qwen3"
     },
     "qwen3-coder": {
       "tool_call": true,
       "family": "qwen3-coder"
     }
   }
   ```

### 为什么存在这种差异

工具格式差异存在的原因：

1. **模型训练**：Qwen3-coder 专门针对代码进行训练，可能针对基于 XML 的工具调用进行了优化
2. **API 实现**：阿里云针对不同模型系列的 API 端点可能使用不同的格式
3. **SDK 提供者逻辑**：AI SDK 提供者检测模型系列并选择适当的格式
4. **向后兼容性**：某些模型保持 XML 格式以与现有集成兼容

### 调试工具格式

查看正在使用的实际工具格式：

#### 方法 1：启用调试日志

```bash
# 设置环境变量
export DEBUG=ai:*

# 运行 OpenCode
opencode run "测试工具调用"
```

#### 方法 2：导出增强会话

```bash
# 导出包含完整详细信息的会话
opencode export --enhanced <session-id> > session.json

# 检查工具调用格式
cat session.json | jq '.callChain[].toolCalls'
```

#### 方法 3：检查网络流量

```bash
# 使用代理查看实际的 API 请求
export HTTP_PROXY=http://localhost:8888
opencode run "测试工具"
```

### 对用户的影响

**对于大多数用户**：这种差异是**透明的** - OpenCode 自动处理两种格式。

**潜在问题**：
1. **自定义工具集成**：如果在 OpenCode 之外实现自定义工具
2. **API 兼容性**：如果不使用 SDK 直接调用模型 API
3. **调试**：不同的格式需要不同的解析逻辑

### 自定义选项

目前，**没有内置方法**强制使用特定的工具格式。但是，您可以：

#### 选项 1：使用不同的模型系列

```jsonc
{
  "model": "qwen/qwen3",  // JSON 格式
  // 或
  "model": "qwen/qwen3-coder"  // XML 格式
}
```

#### 选项 2：创建自定义提供者（高级）

创建一个包装模型并转换工具格式的自定义 AI SDK 提供者：

```typescript
// .opencode/provider/custom-qwen.ts
import { createOpenAICompatible } from '@ai-sdk/openai-compatible';

export const customQwen = createOpenAICompatible({
  baseURL: 'https://dashscope.aliyuncs.com/compatible-mode/v1',
  // 添加自定义转换逻辑
});
```

#### 选项 3：功能请求

如果您需要对工具格式进行明确控制，请考虑提交功能请求：
- 在 OpenCode 仓库中
- 在 Vercel AI SDK 仓库中

### 相关文件

与工具渲染相关的关键文件：

```
/packages/opencode/src/
├── session/
│   ├── prompt.ts                 # 工具定义和注册
│   └── llm.ts                    # LLM 调用编排
├── provider/
│   ├── provider.ts               # 提供者加载和管理
│   ├── transform.ts              # 提供者特定转换
│   └── sdk/openai-compatible/    # 自定义 OpenAI-compatible SDK
├── tool/
│   └── [tool-name].ts            # 单个工具实现
```

### 延伸阅读

- [Vercel AI SDK 文档](https://sdk.vercel.ai/docs)
- [OpenAI 函数调用](https://platform.openai.com/docs/guides/function-calling)
- [Anthropic 工具使用](https://docs.anthropic.com/claude/docs/tool-use)
- [Qwen API 文档](https://help.aliyun.com/zh/dashscope/)

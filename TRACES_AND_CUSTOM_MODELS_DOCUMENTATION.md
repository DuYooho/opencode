# OpenCode 轨迹导出和自定义模型完整文档
# OpenCode Traces Export and Custom Models Complete Documentation

本文档汇总了 OpenCode 中关于会话轨迹（traces/trajectory）导出和自定义模型配置的所有相关信息。

This document summarizes all information about session traces/trajectory export and custom model configuration in OpenCode.

---

## 目录 / Table of Contents

1. [轨迹导出功能 / Trace Export Features](#轨迹导出功能--trace-export-features)
2. [自定义模型配置 / Custom Model Configuration](#自定义模型配置--custom-model-configuration)
3. [提供商配置详解 / Provider Configuration Details](#提供商配置详解--provider-configuration-details)
4. [使用示例 / Usage Examples](#使用示例--usage-examples)

---

## 轨迹导出功能 / Trace Export Features

OpenCode 支持多种方式导出会话数据，用于分析、分享或备份。

OpenCode supports multiple ways to export session data for analysis, sharing, or backup.

### 1. 会话导出命令 / Session Export Command

**命令 / Command**: `opencode export [sessionID]`

**位置 / Location**: `/packages/opencode/src/cli/cmd/export.ts`

**功能描述 / Description**:
将会话数据导出为 JSON 格式，包含完整的会话信息、所有消息、系统提示词、工具定义、模型配置和完整的调用链路。

Exports session data as JSON format, including complete session information, all messages, system prompts, tool definitions, model configuration, and full call chain.

**使用方法 / Usage**:

```bash
# 交互式选择会话导出
opencode export

# 直接导出特定会话
opencode export <session-id>

# 导出到文件
opencode export <session-id> > session-export.json
```

**完整导出数据结构 / Complete Export Data Structure**:

```json
{
  "info": {
    "id": "session_xxx",
    "title": "Session Title",
    "time": {
      "created": 1234567890,
      "updated": 1234567890
    },
    "directory": "/path/to/project",
    "projectID": "project_xxx",
    "parentID": "parent_session_xxx",  // 如果是子会话
    "permission": {                     // 权限配置
      // ... 权限规则
    }
    // ... 其他会话信息
  },
  "messages": [
    {
      "info": {
        "role": "user" | "assistant",
        "id": "message_xxx",
        "sessionID": "session_xxx",
        "modelID": "claude-sonnet-4-5",
        "providerID": "anthropic",
        "agent": "build",
        "tokens": {
          "input": 1000,
          "output": 500,
          "reasoning": 200,
          "cache": {
            "read": 500,
            "write": 100
          }
        },
        "cost": 0.05,
        "time": {
          "created": 1234567890,
          "completed": 1234567900
        }
        // ... 消息元信息
      },
      "parts": [
        {
          "type": "text" | "tool" | "reasoning" | ...,
          // ... 消息部分内容
        }
      ]
    }
  ],
  "systemPrompt": {
    "header": "You are Claude Code...",
    "instructions": "You are OpenCode, the best coding agent...",
    "provider": "Anthropic-specific prompt content...",
    "environment": "Working directory: /path/to/project...",
    "custom": [
      "Content from CLAUDE.md",
      "Content from AGENTS.md"
    ],
    "fullPrompt": "Complete combined system prompt..."
  },
  "tools": [
    {
      "id": "bash",
      "description": "Executes a given bash command...",
      "parameters": {
        "type": "object",
        "properties": {
          "command": {
            "type": "string",
            "description": "The bash command to execute"
          }
        },
        "required": ["command"]
      }
    },
    {
      "id": "read",
      "description": "Reads a file from the local filesystem...",
      "parameters": {
        // ... 参数定义
      }
    }
    // ... 所有可用工具
  ],
  "agentConfig": {
    "name": "build",
    "mode": "primary",
    "model": {
      "modelID": "claude-sonnet-4-5",
      "providerID": "anthropic"
    },
    "temperature": 0.7,
    "topP": 0.9,
    "permission": {
      // ... Agent 权限配置
    },
    "prompt": "Custom agent prompt if any..."
  },
  "modelConfig": {
    "providerID": "anthropic",
    "modelID": "claude-sonnet-4-5",
    "baseURL": "https://api.anthropic.com/v1",
    "options": {
      "temperature": 0.7,
      "maxTokens": 4096
    }
  },
  "callChain": [
    {
      "messageID": "message_001",
      "role": "user",
      "timestamp": 1234567890,
      "content": "User message content"
    },
    {
      "messageID": "message_002",
      "role": "assistant",
      "timestamp": 1234567895,
      "agent": "build",
      "model": "claude-sonnet-4-5",
      "toolCalls": [
        {
          "id": "call_001",
          "tool": "bash",
          "input": {
            "command": "npm install"
          },
          "output": "Dependencies installed",
          "status": "completed",
          "duration": 3000
        }
      ],
      "reasoning": "Let me install the dependencies...",
      "response": "I've installed the dependencies."
    }
    // ... 完整的调用链
  ],
  "metadata": {
    "exportVersion": "1.0",
    "exportedAt": 1234567890,
    "opencodeVersion": "1.0.0"
  }
}
```

**导出到文件 / Export to File**:

```bash
# 导出到文件
opencode export <session-id> > session-export.json

# 导出最新会话
opencode export > latest-session.json
```

**增强导出功能 / Enhanced Export Features**:

当前导出命令导出基本的会话信息和消息。为了获得完整的调用链路、系统提示词和工具定义，需要增强导出功能。

The current export command exports basic session info and messages. To get the complete call chain, system prompts, and tool definitions, the export functionality needs enhancement.

**建议的增强导出实现 / Recommended Enhanced Export Implementation**:

```typescript
// 增强的导出数据获取
async function getEnhancedExportData(sessionID: string) {
  const sessionInfo = await Session.get(sessionID)
  const messages = await Session.messages({ sessionID })
  
  // 获取最后一条消息的模型和 agent 信息
  const lastMessage = messages.filter(m => m.info.role === 'assistant').pop()
  const modelID = lastMessage?.info.modelID || 'claude-sonnet-4-5'
  const providerID = lastMessage?.info.providerID || 'anthropic'
  const agentName = lastMessage?.info.agent || 'build'
  
  // 获取 agent 配置
  const agent = await Agent.get(agentName)
  
  // 获取模型
  const model = await Provider.getModel(providerID, modelID)
  
  // 获取系统提示词
  const systemPromptParts = {
    header: SystemPrompt.header(providerID),
    instructions: SystemPrompt.instructions(),
    provider: SystemPrompt.provider(model),
    environment: await SystemPrompt.environment(),
    custom: await SystemPrompt.custom()
  }
  
  // 获取工具定义
  const tools = await ToolRegistry.tools({ providerID, modelID }, agent)
  
  // 构建调用链
  const callChain = messages.map(msg => ({
    messageID: msg.info.id,
    role: msg.info.role,
    timestamp: msg.info.time.created,
    agent: msg.info.role === 'assistant' ? msg.info.agent : undefined,
    model: msg.info.role === 'assistant' ? msg.info.modelID : undefined,
    toolCalls: msg.parts
      .filter(p => p.type === 'tool')
      .map(p => ({
        id: p.id,
        tool: p.tool,
        input: p.state.input,
        output: p.state.status === 'completed' ? p.state.output : undefined,
        error: p.state.status === 'error' ? p.state.error : undefined,
        status: p.state.status,
        duration: p.state.time?.end && p.state.time?.start 
          ? p.state.time.end - p.state.time.start 
          : undefined
      })),
    reasoning: msg.parts
      .filter(p => p.type === 'reasoning')
      .map(p => p.text)
      .join('\n'),
    response: msg.parts
      .filter(p => p.type === 'text')
      .map(p => p.text)
      .join('\n')
  }))
  
  return {
    info: sessionInfo,
    messages: messages.map(msg => ({
      info: msg.info,
      parts: msg.parts
    })),
    systemPrompt: {
      ...systemPromptParts,
      fullPrompt: [
        ...systemPromptParts.header,
        systemPromptParts.instructions,
        ...systemPromptParts.provider,
        ...systemPromptParts.environment,
        ...systemPromptParts.custom
      ].join('\n\n')
    },
    tools: tools.map(t => ({
      id: t.id,
      description: t.description,
      parameters: t.parameters
    })),
    agentConfig: agent,
    modelConfig: {
      providerID,
      modelID,
      options: model.options
    },
    callChain,
    metadata: {
      exportVersion: '1.0',
      exportedAt: Date.now(),
      opencodeVersion: process.env.npm_package_version || 'unknown'
    }
  }
}
```

**实现位置 / Implementation Location**:
建议在 `/packages/opencode/src/cli/cmd/export.ts` 中添加 `--enhanced` 或 `--full` 标志来启用完整导出。

Recommended to add `--enhanced` or `--full` flag in `/packages/opencode/src/cli/cmd/export.ts` to enable complete export.

**使用增强导出 / Using Enhanced Export**:

```bash
# 基本导出（当前功能）
opencode export <session-id>

# 完整导出（包含系统提示词、工具定义、调用链）
opencode export --enhanced <session-id>
opencode export --full <session-id>
```

---

### 2. 会话分享功能 / Session Share Feature

**位置 / Location**: `/packages/opencode/src/share/share.ts`

**功能描述 / Description**:
在线分享会话，生成可访问的 URL。

Share sessions online and generate accessible URLs.

**创建分享 / Create Share**:

```typescript
import { Share } from "@opencode-ai/sdk"

const result = await Share.create(sessionID)
// Returns: { url: "https://...", secret: "..." }
```

**分享 URL / Share URL**:
- **Production**: `https://api.opencode.ai`
- **Development**: `https://api.dev.opencode.ai`

**自动同步 / Auto Sync**:
当会话启用分享时，OpenCode 会自动同步以下内容：
- 会话信息更新
- 新消息
- 消息部分更新

When sharing is enabled, OpenCode automatically syncs:
- Session info updates
- New messages
- Message part updates

**禁用分享 / Disable Sharing**:

```bash
# 环境变量
export OPENCODE_DISABLE_SHARE=true

# 或
export OPENCODE_DISABLE_SHARE=1
```

**删除分享 / Remove Share**:

```typescript
await Share.remove(sessionID, secret)
```

---

### 3. Transcript 格式导出 / Transcript Format Export

**位置 / Location**: `/packages/opencode/src/cli/cmd/tui/util/transcript.ts`

**功能描述 / Description**:
将会话格式化为易读的 Markdown 格式文本记录。

Format sessions as readable Markdown transcript.

**导出选项 / Export Options**:

```typescript
type TranscriptOptions = {
  thinking: boolean          // 包含思考过程
  toolDetails: boolean       // 包含工具详细信息
  assistantMetadata: boolean // 包含助手元数据（agent、model、duration）
}
```

**Transcript 格式示例 / Transcript Format Example**:

```markdown
# Session Title

**Session ID:** session_xxx
**Created:** 2026-01-23 10:00:00
**Updated:** 2026-01-23 10:30:00

---

## User

User message content here...

---

## Assistant (Build · claude-sonnet-4-5 · 3.2s)

Assistant response here...

\`\`\`
Tool: bash

**Input:**
\`\`\`json
{
  "command": "npm install"
}
\`\`\`

**Output:**
\`\`\`
Dependencies installed successfully
\`\`\`
\`\`\`

---
```

**使用 Transcript / Using Transcript**:

```typescript
import { formatTranscript } from "@opencode-ai/sdk"

const transcript = formatTranscript(
  sessionInfo,
  messages,
  {
    thinking: true,
    toolDetails: true,
    assistantMetadata: true
  }
)
```

---

### 4. 导出功能对比 / Export Features Comparison

| 功能 / Feature | Export Command | Share | Transcript |
|---------------|----------------|-------|------------|
| **格式 / Format** | JSON | Online URL | Markdown |
| **用途 / Use Case** | 备份、分析 | 在线分享 | 人类可读 |
| **包含数据 / Data** | 完整数据 | 完整数据 | 格式化文本 |
| **离线访问 / Offline** | ✅ 是 | ❌ 否 | ✅ 是 |
| **实时同步 / Real-time** | ❌ 否 | ✅ 是 | ❌ 否 |

---

## 自定义模型配置 / Custom Model Configuration

OpenCode 支持完全自定义的模型提供商配置，可以使用任何 OpenAI 兼容的 API。

OpenCode supports fully custom model provider configuration and can use any OpenAI-compatible API.

### 1. 模型配置结构 / Model Configuration Structure

**位置 / Location**: `/packages/opencode/src/config/config.ts`

**基本配置 / Basic Configuration**:

```json
{
  "$schema": "https://opencode.ai/config.json",
  "model": "provider-id/model-id",
  "provider": {
    "provider-id": {
      "npm": "@ai-sdk/openai-compatible",
      "name": "Provider Name",
      "options": {
        "apiKey": "your-api-key",
        "baseURL": "https://api.example.com/v1"
      },
      "models": {
        "model-id": {
          "options": {
            "temperature": 0.7
          }
        }
      }
    }
  }
}
```

---

### 2. 添加自定义提供商 / Adding Custom Providers

#### 方法 1: 使用 `/connect` 命令 / Using `/connect` Command

**步骤 / Steps**:

1. 运行 `/connect` 命令

```bash
$ /connect

┌  Add credential
│
◆  Select provider
│  ...
│  ● Other
└
```

2. 选择 "Other"（其他）

3. 输入提供商信息：
   - Provider Name（提供商名称）
   - API Base URL（API 基础 URL）
   - API Key（API 密钥）

#### 方法 2: 直接配置文件 / Direct Configuration File

**全局配置 / Global Configuration**: `~/.config/opencode/opencode.json`

**项目配置 / Project Configuration**: `./opencode.json`

**示例配置 / Example Configuration**:

```json
{
  "$schema": "https://opencode.ai/config.json",
  "provider": {
    "my-custom-provider": {
      "npm": "@ai-sdk/openai-compatible",
      "name": "My Custom Provider",
      "options": {
        "apiKey": "${MY_CUSTOM_API_KEY}",
        "baseURL": "https://api.mycustom.com/v1",
        "timeout": 60000,
        "headers": {
          "Custom-Header": "value"
        }
      },
      "models": {
        "custom-model-1": {
          "name": "Custom Model 1",
          "options": {
            "temperature": 0.7,
            "maxTokens": 4096
          }
        },
        "custom-model-2": {
          "name": "Custom Model 2",
          "options": {
            "temperature": 0.3
          }
        }
      }
    }
  }
}
```

---

### 3. 配置选项详解 / Configuration Options Details

#### Provider 级别选项 / Provider-Level Options

```json
{
  "provider": {
    "provider-id": {
      "npm": "@ai-sdk/openai-compatible",  // NPM 包名
      "name": "Provider Display Name",     // 显示名称
      "options": {
        "apiKey": "key",                   // API 密钥
        "baseURL": "https://...",          // 基础 URL
        "enterpriseUrl": "https://...",    // 企业 URL（GitHub Copilot）
        "timeout": 300000,                 // 超时（毫秒）
        "setCacheKey": false,              // 启用 prompt 缓存
        "headers": {                       // 自定义请求头
          "Custom-Header": "value"
        }
      }
    }
  }
}
```

#### Model 级别选项 / Model-Level Options

```json
{
  "provider": {
    "provider-id": {
      "models": {
        "model-id": {
          "name": "Model Display Name",    // 显示名称
          "options": {
            "temperature": 0.7,            // 温度参数
            "topP": 0.9,                   // Top-p 采样
            "maxTokens": 4096,             // 最大 tokens
            "reasoningEffort": "high",     // 推理努力（GPT-5）
            "textVerbosity": "low",        // 文本详细度（GPT-5）
            "thinking": {                  // 思考模式（Claude）
              "type": "enabled",
              "budgetTokens": 16000
            }
          },
          "variants": {                    // 模型变体
            "high": {
              "reasoningEffort": "high"
            },
            "low": {
              "reasoningEffort": "low"
            }
          }
        }
      }
    }
  }
}
```

---

### 4. 环境变量支持 / Environment Variables Support

**API 密钥 / API Keys**:

```bash
# 使用环境变量
export MY_PROVIDER_API_KEY="your-api-key"
```

**配置文件中引用 / Reference in Config**:

```json
{
  "provider": {
    "my-provider": {
      "options": {
        "apiKey": "${MY_PROVIDER_API_KEY}"
      }
    }
  }
}
```

**支持的环境变量格式 / Supported Formats**:
- `${VARIABLE_NAME}` - 环境变量替换
- 直接硬编码 - 不推荐（安全风险）

---

### 5. 切换模型 / Switching Models

#### 方法 1: `/models` 命令 / Using `/models` Command

```bash
# 在 TUI 中运行
/models

# 选择提供商和模型
┌  Select model
│  ● openai/gpt-5
│    anthropic/claude-sonnet-4-5
│    my-custom-provider/custom-model-1
└
```

#### 方法 2: 命令行参数 / Command Line Flag

```bash
# 指定模型启动
opencode --model provider-id/model-id

# 或简写
opencode -m provider-id/model-id
```

#### 方法 3: 配置文件默认 / Config File Default

```json
{
  "$schema": "https://opencode.ai/config.json",
  "model": "my-custom-provider/custom-model-1"
}
```

#### 方法 4: Agent 特定模型 / Agent-Specific Model

```json
{
  "agent": {
    "build": {
      "model": "anthropic/claude-sonnet-4-5"
    },
    "plan": {
      "model": "anthropic/claude-haiku-4-5"
    }
  }
}
```

---

### 6. 模型变体 / Model Variants

**内置变体 / Built-in Variants**:

- **Anthropic**: `high`, `max`
- **OpenAI**: `none`, `minimal`, `low`, `medium`, `high`, `xhigh`
- **Google**: `low`, `high`

**自定义变体 / Custom Variants**:

```json
{
  "provider": {
    "openai": {
      "models": {
        "gpt-5": {
          "variants": {
            "thinking": {
              "reasoningEffort": "high",
              "textVerbosity": "low"
            },
            "fast": {
              "disabled": true
            }
          }
        }
      }
    }
  }
}
```

**切换变体 / Cycle Variants**:
使用快捷键 `variant_cycle` 快速切换变体。

Use the `variant_cycle` keybind to quickly switch between variants.

---

## 提供商配置详解 / Provider Configuration Details

### 1. 内置提供商 / Built-in Providers

OpenCode 内置支持 75+ 提供商，包括：

OpenCode has built-in support for 75+ providers, including:

**主要提供商 / Major Providers**:
- Anthropic (Claude)
- OpenAI (GPT)
- Google (Gemini, Vertex)
- Amazon Bedrock
- Azure
- GitHub Copilot
- OpenRouter
- xAI (Grok)
- Mistral
- Groq
- Cohere
- Together AI
- Perplexity
- Cerebras
- DeepInfra
- GitLab Duo

**完整列表 / Full List**: 查看 [Models.dev](https://models.dev)

---

### 2. OpenAI 兼容提供商 / OpenAI-Compatible Providers

任何提供 OpenAI 兼容 API 的服务都可以作为自定义提供商添加。

Any service offering OpenAI-compatible API can be added as a custom provider.

**常见兼容提供商 / Common Compatible Providers**:
- LM Studio
- Ollama
- LocalAI
- vLLM
- Text Generation WebUI
- LiteLLM
- 302.AI
- Helicone（代理）

**配置示例 / Configuration Example**:

```json
{
  "provider": {
    "lmstudio": {
      "npm": "@ai-sdk/openai-compatible",
      "name": "LM Studio",
      "options": {
        "baseURL": "http://localhost:1234/v1",
        "apiKey": "not-needed"
      },
      "models": {
        "local-model": {
          "name": "Local Model"
        }
      }
    }
  }
}
```

---

### 3. 特殊配置场景 / Special Configuration Scenarios

#### Amazon Bedrock

**环境变量 / Environment Variables**:

```bash
# 方法 1: AWS 访问密钥
AWS_ACCESS_KEY_ID=XXX
AWS_SECRET_ACCESS_KEY=YYY

# 方法 2: AWS Profile
AWS_PROFILE=my-profile

# 方法 3: Bearer Token
AWS_BEARER_TOKEN_BEDROCK=XXX
```

**配置文件 / Configuration File**:

```json
{
  "provider": {
    "amazon-bedrock": {
      "options": {
        "region": "us-east-1",
        "profile": "my-aws-profile",
        "endpoint": "https://bedrock-runtime.vpc-xxxxx.amazonaws.com"
      }
    }
  }
}
```

#### Azure / Azure Cognitive Services

```json
{
  "provider": {
    "azure": {
      "options": {
        "baseURL": "https://your-resource.openai.azure.com",
        "apiKey": "${AZURE_API_KEY}",
        "useCompletionUrls": false
      }
    },
    "azure-cognitive-services": {
      "options": {
        "baseURL": "https://your-resource.cognitiveservices.azure.com/openai",
        "region": "eastus"
      }
    }
  }
}
```

#### GitHub Copilot Enterprise

```json
{
  "provider": {
    "github-copilot-enterprise": {
      "options": {
        "enterpriseUrl": "https://github.company.com"
      }
    }
  }
}
```

#### GitLab Duo

```bash
# 环境变量
export GITLAB_INSTANCE_URL=https://gitlab.company.com
export GITLAB_TOKEN=glpat-...
export GITLAB_AI_GATEWAY_URL=https://ai-gateway.company.com
```

---

### 4. 代理和网关配置 / Proxy and Gateway Configuration

#### Helicone（AI Gateway）

```json
{
  "provider": {
    "helicone": {
      "npm": "@ai-sdk/openai-compatible",
      "name": "Helicone",
      "options": {
        "baseURL": "https://ai-gateway.helicone.ai",
        "apiKey": "${OPENAI_API_KEY}",
        "headers": {
          "Helicone-Auth": "Bearer ${HELICONE_API_KEY}",
          "Helicone-Cache-Enabled": "true",
          "Helicone-User-Id": "opencode"
        }
      }
    }
  }
}
```

#### 自定义代理 / Custom Proxy

```json
{
  "provider": {
    "openai": {
      "options": {
        "baseURL": "https://my-proxy.com/v1",
        "headers": {
          "X-Custom-Auth": "token"
        }
      }
    }
  }
}
```

---

### 5. 本地模型配置 / Local Model Configuration

#### LM Studio

```json
{
  "$schema": "https://opencode.ai/config.json",
  "model": "lmstudio/local-model",
  "provider": {
    "lmstudio": {
      "npm": "@ai-sdk/openai-compatible",
      "name": "LM Studio",
      "options": {
        "baseURL": "http://localhost:1234/v1",
        "apiKey": "lm-studio"
      },
      "models": {
        "local-model": {
          "name": "Qwen 2.5 Coder 32B"
        }
      }
    }
  }
}
```

#### Ollama

```json
{
  "$schema": "https://opencode.ai/config.json",
  "model": "ollama/qwen2.5-coder",
  "provider": {
    "ollama": {
      "npm": "@ai-sdk/openai-compatible",
      "name": "Ollama",
      "options": {
        "baseURL": "http://localhost:11434/v1",
        "apiKey": "ollama"
      },
      "models": {
        "qwen2.5-coder": {
          "name": "Qwen 2.5 Coder"
        }
      }
    }
  }
}
```

---

## 使用示例 / Usage Examples

### 示例 1: 完整的自定义提供商配置 / Complete Custom Provider Setup

```json
{
  "$schema": "https://opencode.ai/config.json",
  "model": "my-llm/flagship-model",
  "provider": {
    "my-llm": {
      "npm": "@ai-sdk/openai-compatible",
      "name": "My LLM Service",
      "options": {
        "apiKey": "${MY_LLM_API_KEY}",
        "baseURL": "https://api.my-llm.com/v1",
        "timeout": 120000,
        "headers": {
          "X-Organization-ID": "org-123",
          "X-Request-Source": "opencode"
        }
      },
      "models": {
        "flagship-model": {
          "name": "Flagship Model 70B",
          "options": {
            "temperature": 0.7,
            "maxTokens": 8192
          }
        },
        "fast-model": {
          "name": "Fast Model 7B",
          "options": {
            "temperature": 0.5,
            "maxTokens": 4096
          }
        }
      }
    }
  },
  "agent": {
    "build": {
      "model": "my-llm/flagship-model"
    },
    "plan": {
      "model": "my-llm/fast-model"
    }
  }
}
```

**使用 / Usage**:

```bash
# 设置环境变量
export MY_LLM_API_KEY="your-api-key-here"

# 启动 OpenCode
opencode

# 或指定模型
opencode -m my-llm/fast-model
```

---

### 示例 2: 导出会话并分析 / Export Session and Analyze

```bash
# 1. 导出会话
opencode export session_abc123 > session-data.json

# 2. 使用 jq 分析
cat session-data.json | jq '.messages | length'  # 消息数量
cat session-data.json | jq '.messages[] | select(.info.role == "assistant")'  # 所有助手消息

# 3. 提取工具调用
cat session-data.json | jq '.messages[].parts[] | select(.type == "tool")'

# 4. 统计 token 使用
cat session-data.json | jq '[.messages[].info.tokens.input] | add'  # 输入 tokens
cat session-data.json | jq '[.messages[].info.tokens.output] | add'  # 输出 tokens
```

---

### 示例 3: 多提供商配置 / Multi-Provider Configuration

```json
{
  "$schema": "https://opencode.ai/config.json",
  "model": "opencode/gpt-5.1-codex",
  "provider": {
    "opencode": {
      "options": {
        "apiKey": "${OPENCODE_API_KEY}"
      }
    },
    "anthropic": {
      "options": {
        "apiKey": "${ANTHROPIC_API_KEY}"
      }
    },
    "local-llm": {
      "npm": "@ai-sdk/openai-compatible",
      "name": "Local LLM",
      "options": {
        "baseURL": "http://localhost:8000/v1",
        "apiKey": "not-needed"
      },
      "models": {
        "qwen-coder": {
          "name": "Qwen 2.5 Coder"
        }
      }
    }
  },
  "agent": {
    "build": {
      "model": "opencode/gpt-5.1-codex",
      "description": "Full-featured build agent"
    },
    "plan": {
      "model": "anthropic/claude-sonnet-4-5",
      "description": "Planning agent"
    },
    "local-test": {
      "model": "local-llm/qwen-coder",
      "description": "Local testing agent",
      "mode": "subagent"
    }
  }
}
```

---

### 示例 4: 生成 Transcript / Generate Transcript

```typescript
import { formatTranscript, Session } from "@opencode-ai/sdk"
import fs from "fs"

async function exportSessionAsMarkdown(sessionID: string) {
  // 获取会话信息
  const sessionInfo = await Session.get(sessionID)
  const messages = await Session.messages({ sessionID })
  
  // 生成 transcript
  const transcript = formatTranscript(
    sessionInfo,
    messages,
    {
      thinking: true,          // 包含思考过程
      toolDetails: true,       // 包含工具详情
      assistantMetadata: true  // 包含元数据
    }
  )
  
  // 保存到文件
  fs.writeFileSync(`session-${sessionID}.md`, transcript)
  console.log(`Transcript saved to session-${sessionID}.md`)
}

// 使用
exportSessionAsMarkdown("session_abc123")
```

---

## 实现增强导出功能指南 / Implementation Guide for Enhanced Export

### 所需更改 / Required Changes

要实现包含系统提示词、工具定义和完整调用链的增强导出功能，需要修改以下文件：

To implement enhanced export with system prompts, tool definitions, and full call chain, the following files need to be modified:

#### 1. 更新导出命令 / Update Export Command

**文件 / File**: `/packages/opencode/src/cli/cmd/export.ts`

**添加 --enhanced 标志 / Add --enhanced Flag**:

```typescript
import { Agent } from "../../agent/agent"
import { Provider } from "../../provider/provider"
import { SystemPrompt } from "../../session/system"
import { ToolRegistry } from "../../tool/registry"

export const ExportCommand = cmd({
  command: "export [sessionID]",
  describe: "export session data as JSON",
  builder: (yargs: Argv) => {
    return yargs
      .positional("sessionID", {
        describe: "session id to export",
        type: "string",
      })
      .option("enhanced", {
        describe: "include system prompts, tool definitions, and full call chain",
        type: "boolean",
        default: false,
        alias: "e"
      })
      .option("full", {
        describe: "alias for --enhanced",
        type: "boolean",
        default: false,
        alias: "f"
      })
  },
  handler: async (args) => {
    const enhanced = args.enhanced || args.full
    
    // ... 现有的会话选择逻辑 ...
    
    try {
      const sessionInfo = await Session.get(sessionID!)
      const messages = await Session.messages({ sessionID: sessionID! })
      
      let exportData: any = {
        info: sessionInfo,
        messages: messages.map((msg) => ({
          info: msg.info,
          parts: msg.parts,
        })),
      }
      
      if (enhanced) {
        // 获取最后一条助手消息以确定模型和 agent
        const lastAssistantMsg = messages
          .filter(m => m.info.role === 'assistant')
          .pop()
        
        if (lastAssistantMsg) {
          const modelID = lastAssistantMsg.info.modelID
          const providerID = lastAssistantMsg.info.providerID
          const agentName = lastAssistantMsg.info.agent || 'build'
          
          // 获取 agent 配置
          const agent = await Agent.get(agentName)
          
          // 获取模型
          const model = await Provider.getModel(providerID, modelID)
          
          // 获取系统提示词各部分
          const systemPromptParts = {
            header: SystemPrompt.header(providerID),
            instructions: SystemPrompt.instructions(),
            provider: SystemPrompt.provider(model),
            environment: await SystemPrompt.environment(),
            custom: await SystemPrompt.custom(),
          }
          
          // 获取工具定义
          const tools = await ToolRegistry.tools(
            { providerID, modelID },
            agent
          )
          
          // 构建调用链
          const callChain = messages.map(msg => ({
            messageID: msg.info.id,
            role: msg.info.role,
            timestamp: msg.info.time?.created,
            ...(msg.info.role === 'assistant' && {
              agent: msg.info.agent,
              model: msg.info.modelID,
              provider: msg.info.providerID,
              tokens: msg.info.tokens,
              cost: msg.info.cost,
            }),
            toolCalls: msg.parts
              .filter(p => p.type === 'tool')
              .map(p => ({
                id: p.id,
                tool: p.tool,
                callID: p.callID,
                input: p.state.input,
                output: p.state.status === 'completed' 
                  ? p.state.output 
                  : undefined,
                error: p.state.status === 'error' 
                  ? p.state.error 
                  : undefined,
                status: p.state.status,
                title: p.state.status === 'completed'
                  ? p.state.title
                  : undefined,
                duration: p.state.time?.end && p.state.time?.start
                  ? p.state.time.end - p.state.time.start
                  : undefined,
              })),
            reasoning: msg.parts
              .filter(p => p.type === 'reasoning')
              .map(p => p.text)
              .join('\n'),
            response: msg.parts
              .filter(p => p.type === 'text' && !p.synthetic)
              .map(p => p.text)
              .join('\n'),
          }))
          
          // 添加增强数据
          exportData = {
            ...exportData,
            systemPrompt: {
              ...systemPromptParts,
              fullPrompt: [
                ...systemPromptParts.header,
                systemPromptParts.instructions,
                ...systemPromptParts.provider,
                ...systemPromptParts.environment,
                ...systemPromptParts.custom,
              ].filter(Boolean).join('\n\n'),
            },
            tools: tools.map(t => ({
              id: t.id,
              description: t.description,
              parameters: t.parameters,
            })),
            agentConfig: agent,
            modelConfig: {
              providerID,
              modelID,
              api: model.api,
              options: model.options,
            },
            callChain,
            metadata: {
              exportVersion: '1.0',
              exportedAt: Date.now(),
              opencodeVersion: Installation.version(),
              enhanced: true,
            },
          }
        }
      } else {
        // 基本导出添加元数据
        exportData.metadata = {
          exportVersion: '1.0',
          exportedAt: Date.now(),
          opencodeVersion: Installation.version(),
          enhanced: false,
        }
      }
      
      process.stdout.write(JSON.stringify(exportData, null, 2))
      process.stdout.write(EOL)
    } catch (error) {
      UI.error(`Session not found: ${sessionID!}`)
      process.exit(1)
    }
  },
})
```

#### 2. 使用示例 / Usage Examples

```bash
# 基本导出（仅会话信息和消息）
opencode export session_abc123 > basic-export.json

# 增强导出（包含所有系统细节）
opencode export --enhanced session_abc123 > enhanced-export.json
opencode export -e session_abc123 > enhanced-export.json

# 使用 --full 别名
opencode export --full session_abc123 > full-export.json
opencode export -f session_abc123 > full-export.json
```

#### 3. 导出数据分析 / Export Data Analysis

```bash
# 查看系统提示词
cat enhanced-export.json | jq '.systemPrompt.fullPrompt'

# 查看所有工具定义
cat enhanced-export.json | jq '.tools[] | {id, description}'

# 查看调用链中的工具调用
cat enhanced-export.json | jq '.callChain[] | select(.toolCalls | length > 0) | {messageID, toolCalls}'

# 统计每个工具的使用次数
cat enhanced-export.json | jq '[.callChain[].toolCalls[].tool] | group_by(.) | map({tool: .[0], count: length})'

# 查看 agent 配置
cat enhanced-export.json | jq '.agentConfig'

# 查看模型配置
cat enhanced-export.json | jq '.modelConfig'

# 查看完整的调用链
cat enhanced-export.json | jq '.callChain[] | {id: .messageID, role, agent, toolCount: (.toolCalls | length)}'
```

---

## 模型切换功能 / Model Switching Features

OpenCode 支持在会话过程中手动切换模型，以及为不同的 agent 配置不同的模型，从而实现对复杂任务使用强模型、简单任务使用弱模型的策略。

OpenCode supports manual model switching during sessions and configuring different models for different agents, enabling strategies like using stronger models for complex tasks and lighter models for simple tasks.

### 1. 手动模型切换 / Manual Model Switching

**位置 / Location**: `/packages/opencode/src/cli/cmd/tui/context/local.tsx`

#### 切换方法 / Switching Methods

OpenCode 提供多种手动切换模型的方式：

OpenCode provides multiple ways to manually switch models:

**方法 1: 模型列表对话框 / Model List Dialog**

```bash
# 快捷键（默认）
<Leader>+m  # 打开模型选择对话框

# 可配置的快捷键
"model_list": "<leader>m"
```

- 交互式选择任何可用的模型
- 显示提供商和模型名称
- 选择后立即切换

**方法 2: 最近使用的模型循环 / Recent Models Cycling**

```bash
# 快捷键（默认）
F2              # 向前循环最近使用的模型
Shift+F2        # 向后循环最近使用的模型

# 可配置的快捷键
"model_cycle_recent": "f2"
"model_cycle_recent_reverse": "shift+f2"
```

- 在最近使用的 10 个模型之间循环
- 保持最近使用历史

**方法 3: 收藏模型循环 / Favorite Models Cycling**

```bash
# 可配置的快捷键（默认未设置）
"model_cycle_favorite": "none"
"model_cycle_favorite_reverse": "none"
```

- 仅在收藏的模型之间循环
- 需要先标记收藏模型

**方法 4: 模型变体切换 / Model Variant Switching**

```bash
# 快捷键（默认）
Ctrl+T          # 循环当前模型的变体

# 可配置的快捷键
"variant_cycle": "ctrl+t"
```

- 在同一模型的不同变体之间切换
- 例如：GPT-5 的 none/low/medium/high/xhigh 变体
- 例如：Claude 的 high/max 思考预算变体

#### 实现细节 / Implementation Details

**本地状态管理 / Local State Management**:

```typescript
// 模型状态存储在本地
const modelStore = {
  ready: boolean,                  // 状态是否就绪
  model: Record<string, {          // 每个 agent 的当前模型
    providerID: string,
    modelID: string
  }>,
  recent: Array<{                  // 最近使用的模型（最多 10 个）
    providerID: string,
    modelID: string
  }>,
  favorite: Array<{                // 收藏的模型
    providerID: string,
    modelID: string
  }>,
  variant: Record<string, string>  // 每个模型的当前变体
}

// 状态持久化到文件
// 文件位置: ~/.local/share/opencode/model.json
```

**模型切换函数 / Model Switching Functions**:

```typescript
// 设置模型
model.set(
  { providerID: "anthropic", modelID: "claude-sonnet-4-5" },
  { recent: true }  // 添加到最近使用列表
)

// 循环最近使用的模型
model.cycle(1)   // 向前
model.cycle(-1)  // 向后

// 循环收藏模型
model.cycleFavorite(1)   // 向前
model.cycleFavorite(-1)  // 向后

// 切换变体
model.variant.cycle()            // 循环变体
model.variant.set("high")        // 设置特定变体
model.variant.current()          // 获取当前变体
model.variant.list()             // 获取所有可用变体
```

---

### 2. Agent 特定模型配置 / Agent-Specific Model Configuration

**配置位置 / Configuration Location**: `opencode.json`

OpenCode 允许为每个 agent 配置不同的模型，这是实现"复杂任务用强模型、简单任务用弱模型"策略的关键。

OpenCode allows configuring different models for each agent, which is key to implementing the "strong model for complex tasks, light model for simple tasks" strategy.

#### 配置示例 / Configuration Examples

**示例 1: Build 和 Plan 使用不同模型 / Build and Plan with Different Models**

```json
{
  "$schema": "https://opencode.ai/config.json",
  "agent": {
    "build": {
      "description": "Full-featured build agent for complex tasks",
      "model": "anthropic/claude-sonnet-4-5",  // 强模型用于实现
      "mode": "primary"
    },
    "plan": {
      "description": "Planning agent for analysis",
      "model": "anthropic/claude-haiku-4-5",   // 弱模型用于规划
      "mode": "primary"
    }
  }
}
```

**示例 2: 多个专门 Agent / Multiple Specialized Agents**

```json
{
  "$schema": "https://opencode.ai/config.json",
  "agent": {
    "build": {
      "description": "Complex implementation tasks",
      "model": "opencode/gpt-5.1-codex",      // 最强模型
      "mode": "primary"
    },
    "plan": {
      "description": "Planning and analysis",
      "model": "anthropic/claude-haiku-4-5",  // 快速轻量模型
      "mode": "primary"
    },
    "code-review": {
      "description": "Code review and suggestions",
      "model": "anthropic/claude-sonnet-4-5", // 中等模型
      "mode": "subagent"
    },
    "quick-fix": {
      "description": "Simple bug fixes",
      "model": "anthropic/claude-haiku-4-5",  // 轻量快速模型
      "mode": "subagent"
    }
  }
}
```

**示例 3: 本地和云端混合 / Local and Cloud Hybrid**

```json
{
  "$schema": "https://opencode.ai/config.json",
  "provider": {
    "local-llm": {
      "npm": "@ai-sdk/openai-compatible",
      "name": "Local LLM",
      "options": {
        "baseURL": "http://localhost:8000/v1",
        "apiKey": "not-needed"
      },
      "models": {
        "qwen-coder": {
          "name": "Qwen 2.5 Coder 32B"
        }
      }
    }
  },
  "agent": {
    "build": {
      "model": "opencode/gpt-5.1-codex",      // 云端强模型用于复杂任务
      "mode": "primary"
    },
    "explore": {
      "model": "local-llm/qwen-coder",        // 本地模型用于简单探索
      "mode": "subagent"
    },
    "quick-helper": {
      "description": "Quick local assistance",
      "model": "local-llm/qwen-coder",        // 本地模型降低成本
      "mode": "subagent"
    }
  }
}
```

#### Agent 模型继承规则 / Agent Model Inheritance Rules

OpenCode 使用以下优先级确定 agent 使用的模型：

OpenCode uses the following priority to determine the model used by an agent:

1. **Agent 特定配置** - Agent 配置中的 `model` 字段（最高优先级）
2. **全局配置** - `opencode.json` 中的 `model` 字段（仅 primary agents）
3. **父 Agent 模型** - 调用 subagent 的 primary agent 的模型（仅 subagents）
4. **最近使用** - 用户最近使用的模型
5. **默认模型** - 提供商的默认模型

**代码实现 / Code Implementation**:

```typescript
// 来自 /packages/opencode/src/cli/cmd/tui/context/local.tsx
const currentModel = createMemo(() => {
  const a = agent.current()
  return (
    getFirstValidModel(
      () => modelStore.model[a.name],  // 用户为此 agent 选择的模型
      () => a.model,                   // Agent 配置中的模型
      fallbackModel,                   // 全局配置/最近使用/默认
    ) ?? undefined
  )
})
```

---

### 3. 使用场景和策略 / Use Cases and Strategies

#### 场景 1: 成本优化 / Cost Optimization

**策略 / Strategy**: 简单任务用便宜模型，复杂任务用昂贵模型

```json
{
  "agent": {
    "build": {
      "model": "anthropic/claude-sonnet-4-5",  // $3/M tokens (中等成本)
      "mode": "primary"
    },
    "explore": {
      "model": "anthropic/claude-haiku-4-5",   // $0.25/M tokens (低成本)
      "mode": "subagent"
    },
    "complex-solver": {
      "description": "For very complex problems only",
      "model": "opencode/gpt-5.1-codex",       // 高成本，仅手动调用
      "mode": "subagent"
    }
  }
}
```

**使用方式 / Usage**:
- 日常工作用 Build agent (中等成本)
- 代码探索自动用 Explore subagent (低成本)
- 遇到复杂问题时手动调用: `@complex-solver help me solve this difficult algorithm problem`

#### 场景 2: 速度优化 / Speed Optimization

**策略 / Strategy**: 快速响应用小模型，深度思考用大模型

```json
{
  "agent": {
    "quick": {
      "description": "Quick responses",
      "model": "anthropic/claude-haiku-4-5",  // 快速响应
      "mode": "primary"
    },
    "deep": {
      "description": "Deep thinking",
      "model": "openai/gpt-5",                // 深度推理
      "mode": "primary"
    }
  }
}
```

**使用方式 / Usage**:
- 使用 Tab 键在 quick 和 deep agent 之间切换
- 快速查询用 quick
- 复杂问题用 deep

#### 场景 3: 任务特定模型 / Task-Specific Models

**策略 / Strategy**: 不同任务类型用最适合的模型

```json
{
  "agent": {
    "build": {
      "model": "opencode/gpt-5.1-codex",      // 最佳代码生成
      "mode": "primary"
    },
    "documentation": {
      "description": "Write documentation",
      "model": "anthropic/claude-sonnet-4-5", // 最佳文档写作
      "mode": "subagent"
    },
    "data-analysis": {
      "description": "Analyze data",
      "model": "openai/gpt-5",                // 最佳数据分析和推理
      "mode": "subagent"
    }
  }
}
```

#### 场景 4: 模型变体策略 / Model Variant Strategy

**策略 / Strategy**: 同一模型的不同推理强度

```json
{
  "provider": {
    "openai": {
      "models": {
        "gpt-5": {
          "variants": {
            "fast": {
              "reasoningEffort": "low",
              "textVerbosity": "low"
            },
            "balanced": {
              "reasoningEffort": "medium",
              "textVerbosity": "medium"
            },
            "deep": {
              "reasoningEffort": "high",
              "textVerbosity": "low"
            }
          }
        }
      }
    }
  },
  "agent": {
    "build": {
      "model": "openai/gpt-5"
    }
  }
}
```

**使用方式 / Usage**:
- 使用 `Ctrl+T` 在 fast/balanced/deep 变体之间循环
- 简单任务用 fast 变体（快速+便宜）
- 复杂任务用 deep 变体（深度思考）

---

### 4. 自动模型选择（通过 Agent） / Automatic Model Selection (via Agents)

虽然 OpenCode 本身不支持在单个 agent 内自动切换模型，但可以通过配置多个 agent 并使用 Task 工具来实现类似的自动切换效果。

While OpenCode doesn't support automatic model switching within a single agent, you can achieve similar automatic switching by configuring multiple agents and using the Task tool.

#### 实现方案 / Implementation Approach

**配置多个专门的 Subagent / Configure Multiple Specialized Subagents**:

```json
{
  "agent": {
    "build": {
      "description": "Main development agent",
      "model": "anthropic/claude-sonnet-4-5",
      "mode": "primary"
    },
    "quick-explorer": {
      "description": "Fast file search and simple queries. Use for quick tasks.",
      "model": "anthropic/claude-haiku-4-5",
      "mode": "subagent"
    },
    "deep-solver": {
      "description": "Complex problem solving requiring deep reasoning. Use for difficult tasks.",
      "model": "openai/gpt-5",
      "mode": "subagent"
    },
    "code-writer": {
      "description": "Write code for complex features. Use for implementation tasks.",
      "model": "opencode/gpt-5.1-codex",
      "mode": "subagent"
    }
  }
}
```

**Primary Agent 会根据任务复杂度自动调用合适的 Subagent / Primary Agent Automatically Invokes Appropriate Subagent**:

Build agent 的系统提示词中包含了 Task 工具的描述，它会根据 subagent 的 description 自动选择：

The Build agent's system prompt includes Task tool descriptions, and it automatically selects based on subagent descriptions:

- 简单查询 → 自动调用 `quick-explorer` (Haiku)
- 复杂问题 → 自动调用 `deep-solver` (GPT-5)
- 代码实现 → 自动调用 `code-writer` (Codex)

**示例对话 / Example Conversation**:

```
User: Find all TypeScript files in the src directory
Build: [自动调用 quick-explorer subagent 使用 Haiku 模型]

User: Help me design a distributed caching system with consistency guarantees
Build: [自动调用 deep-solver subagent 使用 GPT-5 模型]

User: Implement a React component with complex state management
Build: [自动调用 code-writer subagent 使用 Codex 模型]
```

---

### 5. 手动模型切换工作流 / Manual Model Switching Workflow

#### 工作流 1: 会话中切换 / Mid-Session Switching

```bash
1. 开始会话 (默认模型: claude-sonnet-4-5)
   User: "Help me debug this issue"
   
2. 意识到需要更强的模型
   按 <Leader>+m
   选择 opencode/gpt-5.1-codex
   
3. 继续对话 (现在使用 GPT-5.1 Codex)
   User: "Now implement the fix"
   
4. 完成后切回轻量模型节省成本
   按 F2 循环回 claude-sonnet-4-5
```

#### 工作流 2: Agent 切换 / Agent Switching

```bash
1. 使用 Build agent 进行开发 (强模型)
   
2. 切换到 Plan agent 进行规划 (弱模型)
   按 Tab 键
   
3. 规划完成后切回 Build agent
   按 Tab 键
```

#### 工作流 3: 变体切换 / Variant Switching

```bash
1. 使用 GPT-5 默认变体
   
2. 遇到简单任务，切换到 fast 变体
   按 Ctrl+T (循环到 fast)
   
3. 遇到复杂任务，切换到 deep 变体
   按 Ctrl+T (循环到 deep)
```

---

### 6. 最佳实践 / Best Practices

#### 1. 为不同复杂度配置不同 Agent / Configure Different Agents for Different Complexities

```json
{
  "agent": {
    "build": {
      "model": "anthropic/claude-sonnet-4-5",
      "description": "General development",
      "mode": "primary"
    },
    "plan": {
      "model": "anthropic/claude-haiku-4-5",
      "description": "Planning and analysis",
      "mode": "primary"
    }
  }
}
```

✅ **优点 / Advantages**:
- 明确的任务分离
- 自动使用合适的模型
- 通过 Tab 键快速切换

#### 2. 使用收藏模型快速访问常用模型 / Use Favorite Models for Quick Access

在模型列表中标记常用模型为收藏，然后配置快捷键：

Mark frequently used models as favorites in the model list, then configure keybinds:

```json
{
  "keybinds": {
    "model_cycle_favorite": "f3",
    "model_cycle_favorite_reverse": "shift+f3"
  }
}
```

#### 3. 配置描述性的 Agent Description / Configure Descriptive Agent Descriptions

```json
{
  "agent": {
    "quick-helper": {
      "description": "Fast responses for simple tasks like file search, quick questions",
      "model": "anthropic/claude-haiku-4-5",
      "mode": "subagent"
    },
    "architect": {
      "description": "System design, architecture decisions, complex problem solving",
      "model": "openai/gpt-5",
      "mode": "subagent"
    }
  }
}
```

描述性的 description 帮助 primary agent 自动选择正确的 subagent。

Descriptive descriptions help the primary agent automatically select the correct subagent.

#### 4. 监控成本和使用 / Monitor Costs and Usage

使用导出功能分析模型使用情况：

Use export feature to analyze model usage:

```bash
# 导出会话
opencode export --enhanced session_xxx > session.json

# 分析模型使用
cat session.json | jq '.callChain[] | select(.role == "assistant") | {model, tokens, cost}'

# 统计每个模型的总成本
cat session.json | jq '[.callChain[] | select(.role == "assistant")] | group_by(.model) | map({model: .[0].model, total_cost: map(.cost) | add})'
```

---

### 7. 配置文件示例 / Configuration File Examples

**完整示例: 多层次模型策略 / Complete Example: Multi-tier Model Strategy**

```json
{
  "$schema": "https://opencode.ai/config.json",
  "model": "anthropic/claude-sonnet-4-5",
  "keybinds": {
    "variant_cycle": "ctrl+t",
    "model_cycle_recent": "f2",
    "model_cycle_recent_reverse": "shift+f2",
    "model_cycle_favorite": "f3",
    "agent_cycle": "tab"
  },
  "provider": {
    "anthropic": {
      "options": {
        "apiKey": "${ANTHROPIC_API_KEY}"
      }
    },
    "openai": {
      "options": {
        "apiKey": "${OPENAI_API_KEY}"
      }
    }
  },
  "agent": {
    "build": {
      "description": "Main development agent for general tasks",
      "model": "anthropic/claude-sonnet-4-5",
      "mode": "primary"
    },
    "plan": {
      "description": "Fast planning and analysis",
      "model": "anthropic/claude-haiku-4-5",
      "mode": "primary"
    },
    "explore": {
      "description": "Quick file search and exploration",
      "model": "anthropic/claude-haiku-4-5",
      "mode": "subagent"
    },
    "architect": {
      "description": "Complex system design and architecture. Use for difficult design decisions.",
      "model": "openai/gpt-5",
      "mode": "subagent"
    },
    "coder": {
      "description": "Advanced code implementation. Use for complex coding tasks.",
      "model": "opencode/gpt-5.1-codex",
      "mode": "subagent"
    }
  }
}
```

**使用此配置的工作流 / Workflow with This Configuration**:

1. **日常开发** - 使用 Build agent (Sonnet, 中等成本)
2. **快速规划** - Tab 切换到 Plan agent (Haiku, 低成本)
3. **文件探索** - Build 自动调用 explore subagent (Haiku, 低成本)
4. **复杂设计** - Build 自动调用 architect subagent (GPT-5, 高成本)
5. **高级编码** - Build 自动调用 coder subagent (Codex, 高成本)
6. **紧急强力模式** - F2 手动切换到 GPT-5 或 Codex

---

## `opencode run` 会话导出和持久化 / Session Export and Persistence with `opencode run`

### 问题 / The Problem

当使用 `opencode run` 命令时，会话在命令结束后会被自动 dispose（销毁），导致无法使用 `opencode export` 导出会话数据。

When using the `opencode run` command, sessions are automatically disposed after the command completes, making it impossible to use `opencode export` to export session data.

**原因 / Root Cause**:

```typescript
// /packages/opencode/src/cli/bootstrap.ts
export async function bootstrap<T>(directory: string, cb: () => Promise<T>) {
  return Instance.provide({
    directory,
    init: InstanceBootstrap,
    fn: async () => {
      try {
        const result = await cb()
        return result
      } finally {
        await Instance.dispose()  // 自动清理实例
      }
    },
  })
}
```

`opencode run` 命令会在 bootstrap 函数的 finally 块中调用 `Instance.dispose()`，这会清理所有会话数据。

The `opencode run` command calls `Instance.dispose()` in the finally block of the bootstrap function, which cleans up all session data.

---

### 解决方案 / Solutions

#### 方案 1: 使用 `--attach` 连接到持久化服务器 / Use `--attach` with Persistent Server

**推荐方案 / Recommended Approach**

通过启动持久化的 OpenCode 服务器，可以保留会话数据供后续导出。

By starting a persistent OpenCode server, you can preserve session data for later export.

**步骤 / Steps**:

```bash
# 1. 在一个终端启动持久化服务器
# Start a persistent server in one terminal
opencode serve --port 4096

# 或使用 web 命令（带 UI）
# Or use web command (with UI)
opencode web --port 4096

# 2. 在另一个终端运行命令，连接到服务器
# In another terminal, run commands attached to the server
opencode run --attach http://localhost:4096 "Create a new React component"

# 3. 命令完成后，导出会话
# After completion, export the session
opencode export --attach http://localhost:4096
```

**优点 / Advantages**:
- ✅ 会话持久化，不会被自动销毁
- ✅ 可以多次运行命令，累积会话历史
- ✅ 避免每次运行的 MCP 服务器冷启动时间
- ✅ 可以随时导出任何会话

**配置服务器 / Server Configuration**:

```json
{
  "$schema": "https://opencode.ai/config.json",
  "server": {
    "port": 4096,
    "hostname": "localhost",
    "cors": ["http://localhost:*"]
  }
}
```

---

#### 方案 2: 使用 `--session` 继续现有会话 / Use `--session` to Continue Existing Session

如果已经有一个会话 ID，可以继续该会话而不是创建新会话。

If you already have a session ID, you can continue that session instead of creating a new one.

```bash
# 1. 启动 TUI 并获取会话 ID
# Start TUI and get session ID
opencode

# 2. 在 TUI 中查看会话 ID（通常显示在状态栏或使用 session_list 命令）
# View session ID in TUI (usually shown in status bar or use session_list command)

# 3. 使用该会话 ID 继续会话
# Continue the session using that ID
opencode run --session session_abc123 "Continue this task"

# 4. 在 TUI 中会话仍然可见和可导出
# The session is still visible and exportable in TUI
```

---

#### 方案 3: 使用 `--share` 自动分享会话 / Use `--share` to Auto-Share Sessions

虽然不能阻止本地会话的销毁，但可以将会话自动上传到云端保存。

While you can't prevent local session disposal, you can automatically upload sessions to the cloud.

```bash
# 运行时自动分享
# Auto-share during run
opencode run --share "Explain async patterns"

# 输出会包含分享 URL
# Output will include share URL
# ~  https://share.opencode.ai/abc123
```

**或使用环境变量 / Or Use Environment Variable**:

```bash
# 设置自动分享
# Enable auto-sharing
export OPENCODE_AUTO_SHARE=true

opencode run "Explain closures"
# 会自动分享并输出 URL
# Will auto-share and output URL
```

**或在配置文件中 / Or in Configuration File**:

```json
{
  "$schema": "https://opencode.ai/config.json",
  "share": "auto"
}
```

**分享的会话可以通过 URL 访问和导出 / Shared sessions can be accessed and exported via URL**:

- 访问分享 URL 查看会话内容
- 使用 OpenCode Web UI 查看和导出
- 分享链接包含完整的会话历史

---

#### 方案 4: 实时导出（脚本方案）/ Real-time Export (Script Approach)

创建一个包装脚本，在 `opencode run` 执行期间导出会话。

Create a wrapper script that exports the session during `opencode run` execution.

**Bash 脚本示例 / Bash Script Example**:

```bash
#!/bin/bash
# save as: opencode-run-with-export.sh

# 启动后台服务器
opencode serve --port 4096 &
SERVER_PID=$!

# 等待服务器启动
sleep 2

# 运行命令并获取会话 ID
SESSION_OUTPUT=$(opencode run --attach http://localhost:4096 --format json "$@" 2>&1)

# 从输出中提取会话 ID（假设第一个 JSON 事件包含 sessionID）
SESSION_ID=$(echo "$SESSION_OUTPUT" | head -1 | jq -r '.sessionID')

# 导出会话
if [ ! -z "$SESSION_ID" ]; then
    echo "Exporting session: $SESSION_ID"
    opencode export --attach http://localhost:4096 "$SESSION_ID" > "session-${SESSION_ID}.json"
    echo "Session exported to session-${SESSION_ID}.json"
fi

# 清理
kill $SERVER_PID

# 显示原始输出
echo "$SESSION_OUTPUT"
```

**使用 / Usage**:

```bash
chmod +x opencode-run-with-export.sh
./opencode-run-with-export.sh "Create a Python script to parse JSON"
```

---

#### 方案 5: 使用 TUI 而不是 run 命令 / Use TUI Instead of run Command

最简单的方法是使用 TUI，它会持久化会话。

The simplest approach is to use TUI, which persists sessions.

```bash
# 使用 TUI 并传入初始提示
# Use TUI with initial prompt
opencode --prompt "Explain async/await in JavaScript"

# 或继续上一个会话
# Or continue last session
opencode --continue

# 在 TUI 中完成后导出
# Export from TUI after completion
# 使用快捷键 <Leader>+x 或命令 /export
# Use keybind <Leader>+x or command /export
```

---

### 功能请求: 添加 `--export` 标志 / Feature Request: Add `--export` Flag

**当前不支持 / Currently Not Supported**

`opencode run` 命令目前没有 `--export` 标志来自动导出会话。这是一个潜在的功能增强。

The `opencode run` command currently doesn't have an `--export` flag to automatically export sessions. This is a potential feature enhancement.

**建议实现 / Suggested Implementation**:

```bash
# 理想的未来语法
# Ideal future syntax
opencode run --export session-export.json "Create a React component"

# 或导出到标准输出
# Or export to stdout
opencode run --export - "Explain closures" > session.json
```

**当前替代方案 / Current Workaround**:

使用方案 1（`--attach`）是最接近这个功能的现有方法。

Using Solution 1 (`--attach`) is the closest existing approach to this functionality.

---

### 环境变量和标志 / Environment Variables and Flags

**当前不存在阻止 disposal 的标志 / No Flag Currently Exists to Prevent Disposal**

目前没有 `OPENCODE_KEEP_SESSION` 或 `OPENCODE_NO_DISPOSE` 这样的环境变量。

There is currently no environment variable like `OPENCODE_KEEP_SESSION` or `OPENCODE_NO_DISPOSE`.

**相关的现有标志 / Related Existing Flags**:

```bash
# 自动分享会话
OPENCODE_AUTO_SHARE=true

# 禁用自动压缩（可能有助于保留更多会话数据）
OPENCODE_DISABLE_AUTOCOMPACT=true

# 禁用会话清理
OPENCODE_DISABLE_PRUNE=true
```

---

### 最佳实践总结 / Best Practices Summary

**1. 开发/调试场景 / Development/Debug Scenarios**:

```bash
# 启动持久化服务器
opencode serve --port 4096

# 在开发过程中运行多个命令
opencode run --attach http://localhost:4096 "task 1"
opencode run --attach http://localhost:4096 "task 2"

# 随时导出任何会话
opencode export --attach http://localhost:4096
```

**2. 自动化/CI 场景 / Automation/CI Scenarios**:

```bash
# 使用自动分享获取会话 URL
OPENCODE_AUTO_SHARE=true opencode run "Run tests and fix failures"

# 或使用脚本方案（方案 4）
```

**3. 快速一次性任务 / Quick One-off Tasks**:

```bash
# 使用 TUI，完成后手动导出
opencode --prompt "Quick question"
# 在 TUI 中使用 <Leader>+x 导出
```

**4. 需要详细审计追踪 / Requiring Detailed Audit Trail**:

```bash
# 组合使用分享和增强导出
opencode run --share --attach http://localhost:4096 "Critical task"

# 然后导出完整数据
opencode export --enhanced session_id > full-audit.json
```

---

### 配置示例 / Configuration Examples

**完整的持久化工作流配置 / Complete Persistent Workflow Configuration**:

```json
{
  "$schema": "https://opencode.ai/config.json",
  "share": "auto",
  "server": {
    "port": 4096,
    "hostname": "localhost",
    "mdns": true
  },
  "keybinds": {
    "session_export": "<leader>x"
  }
}
```

**使用脚本 / Using Scripts**:

```bash
# ~/.bashrc or ~/.zshrc
alias ocrun='opencode run --attach http://localhost:4096'
alias ocserve='opencode serve --port 4096 --daemon'
alias ocexport='opencode export --attach http://localhost:4096'

# 使用
# Usage:
ocserve              # 启动后台服务器
ocrun "task here"    # 运行任务
ocexport session_id  # 导出会话
```

---

## 总结 / Summary

### 轨迹导出 / Trace Export

OpenCode 提供三种主要的会话数据导出方式：

OpenCode provides three main ways to export session data:

1. **Export Command** - JSON 格式，完整数据，适合备份和分析
   - **基本导出**: `opencode export` - 会话信息和消息
   - **增强导出**: `opencode export --enhanced` - 包含系统提示词、工具定义、agent 配置、模型配置和完整调用链
2. **Share Feature** - 在线分享，实时同步，适合协作
3. **Transcript Format** - Markdown 格式，易读，适合人类查看

**增强导出包含的额外信息 / Enhanced Export Additional Information**:
- **系统提示词** (System Prompts) - 完整的系统提示词，包括 header、instructions、provider-specific、environment、custom
- **工具定义** (Tool Definitions) - 所有可用工具的定义和参数
- **Agent 配置** (Agent Config) - Agent 的完整配置（mode、model、temperature、permissions等）
- **模型配置** (Model Config) - 提供商和模型的配置选项
- **调用链** (Call Chain) - 结构化的消息流，包括工具调用详情、推理过程、响应内容
- **元数据** (Metadata) - 导出版本、导出时间、OpenCode 版本

### 自定义模型 / Custom Models

OpenCode 支持：

OpenCode supports:

1. **75+ 内置提供商** - 即插即用
2. **任何 OpenAI 兼容 API** - 完全自定义
3. **灵活的配置** - 多层级配置系统
4. **环境变量支持** - 安全的密钥管理
5. **模型变体** - 同一模型的不同配置

### 关键文件位置 / Key File Locations

```
/packages/opencode/src/cli/cmd/export.ts          # Export 命令
/packages/opencode/src/share/share.ts             # Share 功能
/packages/opencode/src/cli/cmd/tui/util/transcript.ts  # Transcript 格式化
/packages/opencode/src/config/config.ts           # 配置系统
/packages/opencode/src/provider/provider.ts       # 提供商系统
/packages/web/src/content/docs/models.mdx         # 模型文档
/packages/web/src/content/docs/providers.mdx      # 提供商文档
```

### 相关文档 / Related Documentation

- **Models**: `/packages/web/src/content/docs/models.mdx`
- **Providers**: `/packages/web/src/content/docs/providers.mdx`
- **Config**: `/packages/web/src/content/docs/config.mdx`
- **Agents**: `/packages/web/src/content/docs/agents.mdx`

---

**文档生成时间 / Document Generated**: 2026-01-23

**仓库 / Repository**: DuYooho/opencode

**分支 / Branch**: dev

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
将会话数据导出为 JSON 格式，包含完整的会话信息和所有消息。

Exports session data as JSON format, including complete session information and all messages.

**使用方法 / Usage**:

```bash
# 交互式选择会话导出
opencode export

# 直接导出特定会话
opencode export <session-id>
```

**导出数据结构 / Export Data Structure**:

```json
{
  "info": {
    "id": "session_xxx",
    "title": "Session Title",
    "time": {
      "created": 1234567890,
      "updated": 1234567890
    }
    // ... 其他会话信息
  },
  "messages": [
    {
      "info": {
        "role": "user" | "assistant",
        "id": "message_xxx",
        "sessionID": "session_xxx",
        // ... 消息元信息
      },
      "parts": [
        {
          "type": "text" | "tool" | "reasoning" | ...,
          // ... 消息部分内容
        }
      ]
    }
  ]
}
```

**导出到文件 / Export to File**:

```bash
# 导出到文件
opencode export <session-id> > session-export.json

# 导出最新会话
opencode export > latest-session.json
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

## 总结 / Summary

### 轨迹导出 / Trace Export

OpenCode 提供三种主要的会话数据导出方式：

OpenCode provides three main ways to export session data:

1. **Export Command** - JSON 格式，完整数据，适合备份和分析
2. **Share Feature** - 在线分享，实时同步，适合协作
3. **Transcript Format** - Markdown 格式，易读，适合人类查看

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

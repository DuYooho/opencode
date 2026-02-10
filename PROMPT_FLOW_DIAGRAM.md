# OpenCode System Prompt Flow Diagram

## Model to Prompt Mapping

```
┌─────────────────────────────────────────────────────────────────────┐
│                    Model ID Pattern Matching                        │
│                  (SystemPrompt.provider function)                    │
└─────────────────────────────────────────────────────────────────────┘
                                 │
                                 ▼
        ┌────────────────────────────────────────────────┐
        │  Does model.api.id include "gpt-5"?           │
        └────────────────────────────────────────────────┘
                 │ Yes                    │ No
                 ▼                        ▼
         ┌──────────────┐       ┌────────────────────────────────────┐
         │ PROMPT_CODEX │       │ Does model.api.id include          │
         │              │       │ "gpt-", "o1", or "o3"?             │
         │codex_header  │       └────────────────────────────────────┘
         │.txt          │                │ Yes            │ No
         └──────────────┘                ▼                ▼
                                 ┌──────────────┐  ┌──────────────────────┐
                                 │PROMPT_BEAST  │  │ Does model.api.id    │
                                 │              │  │ include "gemini-"?   │
                                 │beast.txt     │  └──────────────────────┘
                                 └──────────────┘         │ Yes    │ No
                                                          ▼        ▼
                                                  ┌──────────────┐ ┌──────────────────┐
                                                  │PROMPT_GEMINI │ │ Does model.api.id│
                                                  │              │ │ include "claude"?│
                                                  │gemini.txt    │ └──────────────────┘
                                                  └──────────────┘    │ Yes      │ No
                                                                      ▼          ▼
                                                              ┌──────────────┐ ┌──────────────────┐
                                                              │PROMPT_       │ │ DEFAULT:         │
                                                              │ANTHROPIC     │ │ PROMPT_ANTHROPIC │
                                                              │              │ │ _WITHOUT_TODO    │
                                                              │anthropic.txt │ │                  │
                                                              │(with TodoWrite)│ │ qwen.txt        │
                                                              └──────────────┘ │ (no TodoWrite)   │
                                                                               └──────────────────┘
                                                                                        ▲
                                                                                        │
                                                        ┌───────────────────────────────┴────────────────────────┐
                                                        │ All Qwen models (including Qwen3-coder and Qwen3)      │
                                                        │ use this default prompt because they don't match       │
                                                        │ any specific pattern above                             │
                                                        └────────────────────────────────────────────────────────┘
```

## System Prompt Assembly Flow

```
┌────────────────────────────────────────────────────────────────────────────────┐
│                         System Prompt Assembly                                 │
│                      (LLM.stream function in llm.ts)                           │
└────────────────────────────────────────────────────────────────────────────────┘
                                      │
                                      ▼
                    ┌─────────────────────────────────────┐
                    │ 1. SystemPrompt.header(providerID)  │
                    │                                     │
                    │ If "anthropic" in providerID:       │
                    │   → anthropic_spoof.txt             │
                    │ Else:                               │
                    │   → [] (empty)                      │
                    └─────────────────────────────────────┘
                                      │
                                      ▼
                    ┌─────────────────────────────────────┐
                    │ 2. Agent or Provider Prompt         │
                    │                                     │
                    │ If agent.prompt exists:             │
                    │   → Use agent.prompt                │
                    │ Else if isCodex:                    │
                    │   → Skip (use instructions field)   │
                    │ Else:                               │
                    │   → SystemPrompt.provider(model)    │
                    └─────────────────────────────────────┘
                                      │
                                      ▼
                    ┌─────────────────────────────────────┐
                    │ 3. SystemPrompt.environment()       │
                    │                                     │
                    │ - Working directory                 │
                    │ - Git repo status                   │
                    │ - Platform info                     │
                    │ - Current date                      │
                    └─────────────────────────────────────┘
                                      │
                                      ▼
                    ┌─────────────────────────────────────┐
                    │ 4. SystemPrompt.custom()            │
                    │                                     │
                    │ - AGENTS.md (project + global)      │
                    │ - CLAUDE.md                         │
                    │ - Config instructions               │
                    └─────────────────────────────────────┘
                                      │
                                      ▼
                    ┌─────────────────────────────────────┐
                    │ 5. User Custom System Prompts       │
                    │                                     │
                    │ - From last user message            │
                    │   (input.user.system)               │
                    └─────────────────────────────────────┘
                                      │
                                      ▼
                    ┌─────────────────────────────────────┐
                    │ 6. Plugin Transform                 │
                    │                                     │
                    │ - experimental.chat.system.transform│
                    └─────────────────────────────────────┘
                                      │
                                      ▼
                    ┌─────────────────────────────────────┐
                    │ Final System Prompt                 │
                    │                                     │
                    │ [header, combined_content]          │
                    │                                     │
                    │ Optimized for prompt caching:       │
                    │ - If header unchanged, maintain     │
                    │   2-part structure                  │
                    └─────────────────────────────────────┘
```

## Qwen Models: Identical Treatment

```
┌─────────────────────────┐         ┌─────────────────────────┐
│   Qwen3-coder Model     │         │     Qwen3 Model         │
│                         │         │                         │
│ Model ID: qwen3-coder   │         │ Model ID: qwen3         │
└─────────────────────────┘         └─────────────────────────┘
            │                                   │
            └───────────────┬───────────────────┘
                            ▼
            ┌───────────────────────────────────┐
            │  Model ID Pattern Matching        │
            │                                   │
            │  Neither matches:                 │
            │  - "gpt-5"                        │
            │  - "gpt-", "o1", "o3"             │
            │  - "gemini-"                      │
            │  - "claude"                       │
            │                                   │
            │  Therefore: Use DEFAULT           │
            └───────────────────────────────────┘
                            │
                            ▼
            ┌───────────────────────────────────┐
            │  PROMPT_ANTHROPIC_WITHOUT_TODO    │
            │                                   │
            │  = qwen.txt                       │
            │                                   │
            │  Features:                        │
            │  - Concise instructions           │
            │  - No TodoWrite tool              │
            │  - Short responses (< 4 lines)    │
            │  - No comments unless asked       │
            └───────────────────────────────────┘
                            │
                            ▼
            ┌───────────────────────────────────┐
            │  Model Parameters                 │
            │  (ProviderTransform functions)    │
            │                                   │
            │  if id.includes("qwen"):          │
            │    temperature = 0.55             │
            │    topP = 1                       │
            └───────────────────────────────────┘
                            │
                            ▼
            ┌───────────────────────────────────┐
            │  Identical System Prompts         │
            │  Identical Parameters             │
            │                                   │
            │  Any observed differences come    │
            │  from:                            │
            │  - Model training data            │
            │  - Provider API handling          │
            │  - User custom configuration      │
            └───────────────────────────────────┘
```

## Prompt Template Comparison

```
┌─────────────────────────────────────────────────────────────────────────────────┐
│                         Prompt Templates Overview                                │
└─────────────────────────────────────────────────────────────────────────────────┘

anthropic.txt                     qwen.txt                      codex_header.txt
(Claude models)                   (Qwen + default)              (GPT-5 + Codex)
┌──────────────────┐             ┌──────────────────┐          ┌──────────────────┐
│ ✓ TodoWrite      │             │ ✗ No TodoWrite   │          │ ✗ No TodoWrite   │
│   instructions   │             │                  │          │                  │
│                  │             │                  │          │                  │
│ ✓ Task           │             │ ✓ Task           │          │ ✓ Task           │
│   management     │             │   management     │          │   management     │
│   emphasis       │             │   (less detail)  │          │   (less detail)  │
│                  │             │                  │          │                  │
│ ✓ "Best coding   │             │ ✓ "interactive   │          │ ✓ "Best coding   │
│   agent"         │             │   CLI tool"      │          │   agent"         │
│                  │             │                  │          │                  │
│ ✓ Professional   │             │ ✗ No special     │          │ ✓ Git hygiene    │
│   objectivity    │             │   objectivity    │          │   rules          │
│   guidance       │             │   section        │          │                  │
│                  │             │                  │          │ ✓ Frontend       │
│ ✓ Detailed task  │             │ ✓ Concise        │          │   design         │
│   examples       │             │   examples       │          │   guidance       │
│                  │             │                  │          │                  │
│ ✓ Multi-step     │             │ ✓ Short replies  │          │ ✓ Editing        │
│   workflows      │             │   (< 4 lines)    │          │   constraints    │
│                  │             │                  │          │                  │
│ ~110 lines       │             │ ~100 lines       │          │ ~40 lines        │
└──────────────────┘             └──────────────────┘          └──────────────────┘

beast.txt                         gemini.txt                    anthropic_spoof.txt
(GPT-3.5/4, O1, O3)               (Gemini models)               (Anthropic header)
┌──────────────────┐             ┌──────────────────┐          ┌──────────────────┐
│ Similar to       │             │ Gemini-specific  │          │ Provider spoofing│
│ anthropic.txt    │             │ optimizations    │          │ header only      │
│                  │             │                  │          │                  │
│ ✓ TodoWrite      │             │ Format tuning    │          │ Short header     │
│   likely         │             │ for Gemini       │          │ for identity     │
│   included       │             │                  │          │                  │
└──────────────────┘             └──────────────────┘          └──────────────────┘
```

## Special Cases

```
┌─────────────────────────────────────────────────────────────────────────────────┐
│                              Codex Sessions                                      │
│                        (OpenAI OAuth Authentication)                             │
└─────────────────────────────────────────────────────────────────────────────────┘

Normal Flow                          Codex Flow
┌──────────────────┐                ┌──────────────────┐
│ system: [        │                │ instructions:    │
│   header,        │                │   PROMPT_CODEX   │
│   provider_prompt│                │                  │
│ ]                │                │ system: [        │
│                  │                │   header,        │
│ messages: [...]  │                │   (skip provider)│
│                  │                │ ]                │
│                  │                │                  │
│                  │                │ messages: [...]  │
└──────────────────┘                └──────────────────┘

Why different?
- Codex uses OpenAI's `instructions` field for better caching
- Provider prompt sent as instructions, not system message
- Helps reduce costs with prompt caching
```

## Model Parameter Mapping

```
┌─────────────────────────────────────────────────────────────────────────────────┐
│                      Model-Specific Parameters                                   │
│                    (ProviderTransform functions)                                 │
└─────────────────────────────────────────────────────────────────────────────────┘

Model Pattern         Temperature    TopP      Notes
─────────────────────────────────────────────────────────────────────────────────
qwen                  0.55           1         Conservative, code-focused
claude                undefined      undefined Use model defaults
gemini                1.0            0.95      Higher creativity
glm-4.6, glm-4.7      1.0            undefined -
minimax-m2            1.0            0.95      -
kimi-k2               0.6 / 1.0      undefined 1.0 for thinking mode
default               undefined      undefined Use model defaults

Legend:
- undefined = Use the model's default parameter value
- 0.55 = Lower temperature, more deterministic
- 1.0 = Higher temperature, more creative
- TopP 1 = Use full probability mass
- TopP 0.95 = Top 95% probability mass
```

## Summary

The key insight is that **Qwen3-coder and Qwen3 are treated identically by OpenCode**:
- Same prompt template (`qwen.txt`)
- Same parameters (temperature=0.55, topP=1)
- Any observed differences come from the models themselves, not OpenCode's configuration

The main prompt differences are between:
1. **Claude models** (`anthropic.txt`) - with TodoWrite emphasis
2. **Everything else** (`qwen.txt`) - simpler, more concise

For custom prompts per model, use agent configuration or custom rule files.

# OpenCode Configuration Examples

This directory contains example configuration files demonstrating various provider and model setups.

## Files

### 1. `disable-opencode-provider.json`
**Purpose**: Disable OpenCode provider (including free models like `opencode/big-pickle` and `opencode/gpt-5-nano`) and use only your custom configured providers.

**Use this when**:
- You want to avoid accidentally using free/test models
- You want to use only production-grade paid models
- You need full control over which models are available

**Key feature**: Uses `disabled_providers` array to blacklist specific providers.

---

### 2. `whitelist-providers.json`
**Purpose**: Use whitelist mode to enable ONLY specific providers, ignoring all others.

**Use this when**:
- You need strict control in enterprise environments
- You want to explicitly define allowed providers
- You want to ensure no auto-loaded providers sneak in

**Key feature**: Uses `enabled_providers` array for strict whitelist control.

---

### 3. `local-models-only.json`
**Purpose**: Configure OpenCode to use only local LLM models while disabling cloud-based free models.

**Use this when**:
- You want to run models locally (privacy, offline work)
- You're using LM Studio, Ollama, or similar local servers
- You want to combine local models with select cloud providers

**Key feature**: Custom provider configuration with local baseURL.

---

## How to Use

1. **Choose an example** that matches your needs
2. **Copy the file** to your OpenCode config location:
   ```bash
   # Global configuration
   cp examples/disable-opencode-provider.json ~/.config/opencode/opencode.json
   
   # Project-specific configuration
   cp examples/disable-opencode-provider.json /path/to/project/opencode.json
   ```

3. **Update API keys**: Replace `${ANTHROPIC_API_KEY}` with your actual keys or set environment variables:
   ```bash
   export ANTHROPIC_API_KEY="sk-ant-..."
   export OPENAI_API_KEY="sk-..."
   ```

4. **Verify configuration**:
   ```bash
   opencode models
   # Should only show models from your configured providers
   ```

---

## Configuration Priority

When multiple config files exist, OpenCode merges them with this priority (highest to lowest):

1. `OPENCODE_CONFIG_CONTENT` environment variable
2. Project config: `./opencode.json` or `./.opencode/opencode.json`
3. `OPENCODE_CONFIG` environment variable path
4. Global config: `~/.config/opencode/opencode.json`
5. Remote config from `.well-known/opencode`

---

## Common Scenarios

### Scenario 1: Disable Free Models
```json
{
  "disabled_providers": ["opencode"]
}
```

### Scenario 2: Only Allow Specific Providers
```json
{
  "enabled_providers": ["anthropic", "openai"]
}
```

### Scenario 3: Disable Multiple Providers
```json
{
  "disabled_providers": ["opencode", "openrouter", "groq"]
}
```

---

## Troubleshooting

**Q: Configuration not taking effect?**
- Check JSON syntax with `jq . opencode.json`
- Ensure file is in correct location
- Restart OpenCode if using serve/web mode

**Q: Still seeing opencode provider?**
- Verify `disabled_providers` spelling
- Check for project-level config overriding global config
- Use `opencode models` to verify

**Q: No models available after disabling?**
- Ensure at least one provider is configured with valid API key
- Check provider configuration in `provider` section
- Verify environment variables are set

---

## Documentation

For complete documentation, see:
- [TRACES_AND_CUSTOM_MODELS_DOCUMENTATION.md](../TRACES_AND_CUSTOM_MODELS_DOCUMENTATION.md)
- [OpenCode Providers Documentation](https://opencode.ai/docs/providers)
- [OpenCode Models Documentation](https://opencode.ai/docs/models)

---

## Contributing

Have a useful configuration example? Feel free to submit a PR!

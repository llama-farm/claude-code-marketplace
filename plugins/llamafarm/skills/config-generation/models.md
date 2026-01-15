# Model Configuration Templates

## Ollama (Local, Recommended)

### Standard
```yaml
runtime:
  models:
    - name: default
      provider: ollama
      model: llama3.1:8b
      base_url: http://localhost:11434
      default: true
      prompt_format: unstructured
```

### Fast (Smaller Model)
```yaml
runtime:
  models:
    - name: fast
      provider: ollama
      model: gemma2:2b
      base_url: http://localhost:11434
      default: true
      model_api_parameters:
        temperature: 0.1
```

### Multi-Model Setup
```yaml
runtime:
  default_model: fast
  models:
    - name: fast
      description: "Quick responses"
      provider: ollama
      model: gemma2:2b
      base_url: http://localhost:11434
    - name: powerful
      description: "Complex reasoning"
      provider: ollama
      model: llama3.1:8b
      base_url: http://localhost:11434
```

## Universal Runtime (Local, HuggingFace/GGUF)

### Standard GGUF
```yaml
runtime:
  models:
    - name: default
      provider: universal
      model: unsloth/Qwen3-1.7B-GGUF:Q4_K_M
      base_url: http://127.0.0.1:11540/v1
      default: true
      prompt_format: unstructured
```

### Larger Model
```yaml
runtime:
  models:
    - name: default
      provider: universal
      model: unsloth/Qwen3-4B-GGUF:Q4_K_M
      default: true
```

## Lemonade Runtime (NPU/GPU Acceleration)

```yaml
runtime:
  models:
    - name: default
      provider: lemonade
      model: user.Qwen3-4B
      base_url: http://127.0.0.1:11534/v1
      default: true
```

## OpenAI (Cloud)

### GPT-4o
```yaml
runtime:
  models:
    - name: default
      provider: openai
      model: gpt-4o
      base_url: https://api.openai.com/v1
      api_key: ${OPENAI_API_KEY}
      default: true
```

### GPT-4o-mini (Faster/Cheaper)
```yaml
runtime:
  models:
    - name: default
      provider: openai
      model: gpt-4o-mini
      base_url: https://api.openai.com/v1
      api_key: ${OPENAI_API_KEY}
      default: true
```

## Model Parameters

### Temperature

| Use Case | Temperature | Why |
|----------|-------------|-----|
| Factual Q&A | 0.1 - 0.2 | Consistent, accurate |
| General chat | 0.5 - 0.7 | Balanced |
| Creative | 0.8 - 1.0 | More varied |

```yaml
model_api_parameters:
  temperature: 0.2
  top_p: 0.9
  max_tokens: 2048
```

### Structured Output

```yaml
instructor_mode: json    # For JSON output
# Or set to null to disable
instructor_mode: null
```

### Tool Calling

```yaml
tool_call_mode: native_api    # Use model's native tool calling
# Or
tool_call_mode: prompt_based  # Prompt-based for models without native support
```

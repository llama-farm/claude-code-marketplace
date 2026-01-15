# Multi-Model Configuration Patterns

Configure multiple models in LlamaFarm for different tasks, capabilities, and cost optimization.

## Why Multiple Models?

- **Speed vs Quality**: Fast models for simple tasks, powerful models for complex ones
- **Cost Optimization**: Use cheaper models when possible
- **Capability Routing**: Different models for different capabilities
- **Fallback Support**: Backup models when primary is unavailable

## Basic Multi-Model Setup

```yaml
runtime:
  default_model: default
  models:
    # Primary model for most tasks
    - name: default
      description: General purpose assistant
      provider: ollama
      model: llama3.1:8b
      base_url: http://localhost:11434
      default: true
      model_api_parameters:
        temperature: 0.7
        max_tokens: 2048

    # Fast model for simple queries
    - name: fast
      description: Quick responses for simple questions
      provider: ollama
      model: llama3.2:1b
      base_url: http://localhost:11434
      model_api_parameters:
        temperature: 0.5
        max_tokens: 512

    # Powerful model for complex analysis
    - name: expert
      description: Complex analysis and reasoning
      provider: openai
      model: gpt-4
      api_key: ${env:OPENAI_API_KEY}
      model_api_parameters:
        temperature: 0.3
        max_tokens: 4096
```

## Usage

### CLI

```bash
# Use default model
lf chat "Hello"

# Use specific model
lf chat --model fast "Quick question"
lf chat --model expert "Complex analysis needed"
```

### API

```python
# Use default model
response = requests.post(
    "http://localhost:8000/v1/projects/default/my-project/chat/completions",
    json={"messages": [{"role": "user", "content": "Hello"}]}
)

# Use specific model
response = requests.post(
    "http://localhost:8000/v1/projects/default/my-project/chat/completions",
    json={
        "messages": [{"role": "user", "content": "Complex question"}],
        "model": "expert"
    }
)
```

---

## Provider Comparison

| Provider | Strengths | Best For |
|----------|-----------|----------|
| **Ollama** | Local, free, private | Development, privacy-sensitive |
| **OpenAI** | High quality, reliable | Production, complex tasks |
| **Universal Runtime** | ML features, embeddings | RAG, specialized ML |
| **vLLM/Together** | Fast inference, self-hosted | High throughput |

---

## Configuration Patterns

### Speed Tiers

```yaml
runtime:
  default_model: medium
  models:
    # Tier 1: Instant responses
    - name: instant
      provider: ollama
      model: llama3.2:1b
      model_api_parameters:
        temperature: 0.5
        max_tokens: 256

    # Tier 2: Balanced speed/quality
    - name: medium
      provider: ollama
      model: llama3.1:8b
      default: true
      model_api_parameters:
        temperature: 0.7
        max_tokens: 1024

    # Tier 3: High quality, slower
    - name: powerful
      provider: ollama
      model: llama3.1:70b
      model_api_parameters:
        temperature: 0.3
        max_tokens: 4096
```

### Local + Cloud Hybrid

```yaml
runtime:
  default_model: local
  models:
    # Local model - default for privacy and cost
    - name: local
      provider: ollama
      model: llama3.1:8b
      base_url: http://localhost:11434
      default: true

    # Cloud model - for complex tasks
    - name: cloud
      provider: openai
      model: gpt-4-turbo
      api_key: ${env:OPENAI_API_KEY}
```

### Task-Specific Models

```yaml
runtime:
  default_model: general
  models:
    # General conversation
    - name: general
      provider: ollama
      model: llama3.1:8b
      default: true
      model_api_parameters:
        temperature: 0.7

    # Code generation
    - name: coder
      provider: ollama
      model: codellama:13b
      model_api_parameters:
        temperature: 0.2
        max_tokens: 4096

    # Creative writing
    - name: writer
      provider: ollama
      model: llama3.1:8b
      model_api_parameters:
        temperature: 0.9
        max_tokens: 2048

    # Analysis and reasoning
    - name: analyst
      provider: openai
      model: gpt-4
      api_key: ${env:OPENAI_API_KEY}
      model_api_parameters:
        temperature: 0.1
        max_tokens: 4096
```

### Multi-Tool Capability

```yaml
mcp:
  servers:
    - name: filesystem
      transport: stdio
      command: npx
      args: ['-y', '@modelcontextprotocol/server-filesystem', '/data']

    - name: database
      transport: http
      base_url: http://localhost:8080/mcp

runtime:
  default_model: assistant
  models:
    # Assistant with file access
    - name: assistant
      provider: ollama
      model: llama3.1:8b
      default: true
      mcp_servers:
        - filesystem

    # Data analyst with database access
    - name: data-analyst
      provider: ollama
      model: llama3.1:8b
      mcp_servers:
        - database

    # Power user with all tools
    - name: power-user
      provider: openai
      model: gpt-4
      api_key: ${env:OPENAI_API_KEY}
      mcp_servers:
        - filesystem
        - database
```

---

## Provider-Specific Configuration

### Ollama

```yaml
- name: ollama-model
  provider: ollama
  model: llama3.1:8b
  base_url: http://localhost:11434
  model_api_parameters:
    temperature: 0.7
    top_p: 0.9
    top_k: 40
    num_ctx: 4096
    repeat_penalty: 1.1
```

### OpenAI

```yaml
- name: openai-model
  provider: openai
  model: gpt-4-turbo
  api_key: ${env:OPENAI_API_KEY}
  model_api_parameters:
    temperature: 0.7
    max_tokens: 4096
    top_p: 1.0
    frequency_penalty: 0.0
    presence_penalty: 0.0
```

### Universal Runtime

```yaml
- name: universal-model
  provider: universal
  model: Qwen/Qwen2.5-7B-Instruct
  base_url: http://localhost:11540
  model_api_parameters:
    temperature: 0.7
    max_tokens: 2048
```

### OpenAI-Compatible (vLLM, Together, etc.)

```yaml
- name: vllm-model
  provider: openai
  model: mistral-7b-instruct
  base_url: http://localhost:8000/v1
  api_key: not-needed
  model_api_parameters:
    temperature: 0.7
```

---

## Model Selection Guidelines

### When to Use Fast Models

- Simple Q&A
- Classification tasks
- Quick summaries
- High-throughput scenarios
- Cost-sensitive applications

### When to Use Powerful Models

- Complex reasoning
- Multi-step analysis
- Code generation
- Creative tasks
- Important decisions

### Temperature Guidelines

| Task | Temperature |
|------|-------------|
| Factual Q&A | 0.1-0.3 |
| Code generation | 0.2-0.4 |
| Analysis | 0.3-0.5 |
| General chat | 0.5-0.7 |
| Creative writing | 0.7-1.0 |
| Brainstorming | 0.8-1.2 |

---

## Prompt Customization Per Model

```yaml
prompts:
  - name: default
    messages:
      - role: system
        content: You are a helpful assistant.

  - name: analyst
    messages:
      - role: system
        content: |
          You are an expert analyst. When answering:
          1. Consider multiple perspectives
          2. Cite evidence
          3. Acknowledge uncertainty
          4. Provide structured analysis

  - name: coder
    messages:
      - role: system
        content: |
          You are an expert programmer. When writing code:
          1. Follow best practices
          2. Add comments for complex logic
          3. Handle edge cases
          4. Consider performance

runtime:
  models:
    - name: general
      prompt: default

    - name: analyst
      provider: openai
      model: gpt-4
      prompt: analyst

    - name: coder
      provider: ollama
      model: codellama:13b
      prompt: coder
```

---

## Fallback Configuration

Configure backup models for reliability:

```yaml
runtime:
  models:
    # Primary
    - name: primary
      provider: openai
      model: gpt-4
      api_key: ${env:OPENAI_API_KEY}
      default: true

    # Fallback if OpenAI is down
    - name: fallback
      provider: ollama
      model: llama3.1:8b

    # Emergency fallback
    - name: emergency
      provider: ollama
      model: llama3.2:1b
```

Application logic can try models in order:
```python
models = ["primary", "fallback", "emergency"]
for model in models:
    try:
        response = chat(message, model=model)
        break
    except Exception:
        continue
```

---

## Cost Optimization

### Strategy: Route by Complexity

```python
def select_model(query: str) -> str:
    """Select model based on query complexity."""
    # Simple queries → fast model
    if len(query) < 50 and "?" in query:
        return "fast"

    # Complex keywords → expert model
    complex_keywords = ["analyze", "compare", "explain why", "evaluate"]
    if any(kw in query.lower() for kw in complex_keywords):
        return "expert"

    # Default
    return "default"
```

### Strategy: Token Budget

```python
def select_model_by_budget(estimated_tokens: int) -> str:
    """Select model based on token budget."""
    if estimated_tokens < 500:
        return "fast"
    elif estimated_tokens < 2000:
        return "default"
    else:
        return "powerful"
```

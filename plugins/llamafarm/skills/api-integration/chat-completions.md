# Chat Completions API

Detailed guide for using LlamaFarm's OpenAI-compatible chat API.

## Endpoint

```http
POST /v1/projects/{namespace}/{project}/chat/completions
```

## OpenAI SDK Compatibility

LlamaFarm's chat endpoint is compatible with the OpenAI Python SDK:

```python
from openai import OpenAI

client = OpenAI(
    base_url="http://localhost:8000/v1/projects/default/my-project",
    api_key="not-required"
)

response = client.chat.completions.create(
    model="default",
    messages=[
        {"role": "system", "content": "You are a helpful assistant."},
        {"role": "user", "content": "Hello!"}
    ]
)

print(response.choices[0].message.content)
```

## Request Parameters

### Required

| Parameter | Type | Description |
|-----------|------|-------------|
| `messages` | array | Array of message objects |

### Message Object

```json
{
  "role": "user|assistant|system",
  "content": "Message text"
}
```

### Optional Parameters

| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| `model` | string | default | Model name from config |
| `rag` | boolean | false | Enable RAG context retrieval |
| `stream` | boolean | false | Stream response tokens |
| `temperature` | float | 0.7 | Sampling temperature (0-2) |
| `max_tokens` | int | 2048 | Maximum response length |
| `top_p` | float | 1.0 | Nucleus sampling parameter |
| `session_id` | string | auto | Session ID for conversation history |
| `database` | string | default | RAG database to query |
| `top_k` | int | 10 | Number of RAG chunks to retrieve |
| `score_threshold` | float | 0.3 | Minimum relevance score for RAG |

## Examples

### Basic Chat

```python
import requests

response = requests.post(
    "http://localhost:8000/v1/projects/default/my-project/chat/completions",
    json={
        "messages": [
            {"role": "user", "content": "What is machine learning?"}
        ]
    }
)

print(response.json()["choices"][0]["message"]["content"])
```

### Chat with RAG

```python
response = requests.post(
    "http://localhost:8000/v1/projects/default/my-project/chat/completions",
    json={
        "messages": [
            {"role": "user", "content": "What do my documents say about scaling laws?"}
        ],
        "rag": True,
        "top_k": 5
    }
)
```

### Multi-Turn Conversation

```python
session_id = "user-123-session"

# First message
response1 = requests.post(
    "http://localhost:8000/v1/projects/default/my-project/chat/completions",
    json={
        "messages": [
            {"role": "user", "content": "Tell me about neural networks."}
        ],
        "session_id": session_id
    }
)

# Follow-up (history preserved)
response2 = requests.post(
    "http://localhost:8000/v1/projects/default/my-project/chat/completions",
    json={
        "messages": [
            {"role": "user", "content": "How do they learn?"}
        ],
        "session_id": session_id
    }
)
```

### Streaming Response

```python
import requests

response = requests.post(
    "http://localhost:8000/v1/projects/default/my-project/chat/completions",
    json={
        "messages": [{"role": "user", "content": "Write a poem."}],
        "stream": True
    },
    stream=True
)

for line in response.iter_lines():
    if line:
        data = line.decode('utf-8')
        if data.startswith('data: '):
            chunk = data[6:]  # Remove 'data: ' prefix
            if chunk != '[DONE]':
                import json
                parsed = json.loads(chunk)
                content = parsed['choices'][0]['delta'].get('content', '')
                print(content, end='', flush=True)
```

### Using Different Models

```python
# Use the fast model for quick responses
response = requests.post(
    "http://localhost:8000/v1/projects/default/my-project/chat/completions",
    json={
        "messages": [{"role": "user", "content": "Quick question"}],
        "model": "fast"
    }
)

# Use the default (more capable) model for complex tasks
response = requests.post(
    "http://localhost:8000/v1/projects/default/my-project/chat/completions",
    json={
        "messages": [{"role": "user", "content": "Analyze this data..."}],
        "model": "default",
        "temperature": 0.2  # Lower for analytical tasks
    }
)
```

## Response Format

### Non-Streaming Response

```json
{
  "id": "chatcmpl-abc123xyz",
  "object": "chat.completion",
  "created": 1705234567,
  "model": "default",
  "choices": [
    {
      "index": 0,
      "message": {
        "role": "assistant",
        "content": "Machine learning is a subset of artificial intelligence..."
      },
      "finish_reason": "stop"
    }
  ],
  "usage": {
    "prompt_tokens": 25,
    "completion_tokens": 150,
    "total_tokens": 175
  }
}
```

### Streaming Response

Each line is a Server-Sent Event:

```
data: {"id":"chatcmpl-abc123","object":"chat.completion.chunk","created":1705234567,"model":"default","choices":[{"index":0,"delta":{"role":"assistant"},"finish_reason":null}]}

data: {"id":"chatcmpl-abc123","object":"chat.completion.chunk","created":1705234567,"model":"default","choices":[{"index":0,"delta":{"content":"Machine"},"finish_reason":null}]}

data: {"id":"chatcmpl-abc123","object":"chat.completion.chunk","created":1705234567,"model":"default","choices":[{"index":0,"delta":{"content":" learning"},"finish_reason":null}]}

data: {"id":"chatcmpl-abc123","object":"chat.completion.chunk","created":1705234567,"model":"default","choices":[{"index":0,"delta":{},"finish_reason":"stop"}]}

data: [DONE]
```

## RAG Context

When `rag: true`, the API:
1. Queries the vector database with the user's message
2. Retrieves relevant document chunks
3. Includes them in the context sent to the model
4. Returns sources in metadata

### Response with RAG

```json
{
  "id": "chatcmpl-abc123",
  "choices": [...],
  "rag_context": {
    "chunks_used": 5,
    "sources": [
      {"file": "neural_scaling.md", "score": 0.87},
      {"file": "ml_basics.md", "score": 0.72}
    ]
  }
}
```

## Error Handling

```python
import requests

try:
    response = requests.post(
        "http://localhost:8000/v1/projects/default/my-project/chat/completions",
        json={"messages": [{"role": "user", "content": "Hello"}]},
        timeout=60
    )
    response.raise_for_status()
    data = response.json()
except requests.exceptions.Timeout:
    print("Request timed out - model may be slow")
except requests.exceptions.HTTPError as e:
    error = e.response.json()
    print(f"Error: {error['error']['message']}")
```

## Best Practices

### 1. Use Sessions for Conversations

```python
import uuid

session_id = str(uuid.uuid4())

# Reuse session_id for multi-turn conversations
```

### 2. Adjust Temperature for Tasks

| Task | Temperature |
|------|-------------|
| Factual Q&A | 0.1-0.3 |
| Creative writing | 0.7-1.0 |
| Code generation | 0.2-0.4 |
| General chat | 0.5-0.7 |

### 3. Use RAG Wisely

```python
# Enable RAG for document-based questions
response = chat("What does the report say?", rag=True)

# Disable RAG for general knowledge
response = chat("What is Python?", rag=False)
```

### 4. Handle Long Conversations

```python
# Summarize or truncate old messages
messages = messages[-10:]  # Keep last 10 messages
```

### 5. Implement Retries

```python
import time
from requests.adapters import HTTPAdapter
from urllib3.util.retry import Retry

session = requests.Session()
retries = Retry(total=3, backoff_factor=1)
session.mount('http://', HTTPAdapter(max_retries=retries))
```

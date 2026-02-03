---
description: REST API integration for LlamaFarm. Programmatic access to projects, chat, RAG, and datasets via HTTP endpoints.
---

# LlamaFarm API Integration Skill

Guide for using LlamaFarm's REST API programmatically.

## When to Load

Load this skill when the user:
- Wants to integrate LlamaFarm into an application
- Needs to make API calls programmatically
- Is building automation scripts
- Asks about HTTP endpoints or SDK usage
- Wants to use LlamaFarm from Python/JavaScript/curl

## API Overview

LlamaFarm exposes an **OpenAI-compatible REST API** at `http://localhost:14345/v1`.

| Endpoint Group | Purpose |
|---------------|---------|
| `/v1/projects` | Project CRUD operations |
| `/v1/chat/completions` | AI chat with optional RAG |
| `/v1/rag` | Vector database queries |
| `/v1/datasets` | Dataset management |
| `/v1/models` | Available models |
| `/health` | Service health check |

## Quick Start

### Check Server Health

```bash
curl http://localhost:14345/health
```

### Chat with RAG

```bash
curl -X POST http://localhost:14345/v1/projects/default/my-project/chat/completions \
  -H "Content-Type: application/json" \
  -d '{
    "messages": [{"role": "user", "content": "What is in my documents?"}],
    "rag": true
  }'
```

### Query Vector Database

```bash
curl -X POST http://localhost:14345/v1/projects/default/my-project/rag/query \
  -H "Content-Type: application/json" \
  -d '{
    "query": "neural scaling laws",
    "database": "main_db",
    "top_k": 5
  }'
```

## OpenAI Compatibility

The chat endpoint is compatible with OpenAI's API format:

```python
from openai import OpenAI

client = OpenAI(
    base_url="http://localhost:14345/v1/projects/default/my-project",
    api_key="not-needed"  # LlamaFarm doesn't require API key by default
)

response = client.chat.completions.create(
    model="default",
    messages=[
        {"role": "user", "content": "Hello!"}
    ]
)
print(response.choices[0].message.content)
```

## Progressive Disclosure

For detailed endpoint documentation:
- `rest-api.md` - Complete endpoint reference
- `chat-completions.md` - Chat API details
- `datasets-api.md` - Dataset management
- `rag-api.md` - RAG query operations

## Common Integration Patterns

### 1. Simple Chat Application

```python
import requests

def chat(message, project="my-project", namespace="default"):
    response = requests.post(
        f"http://localhost:14345/v1/projects/{namespace}/{project}/chat/completions",
        json={
            "messages": [{"role": "user", "content": message}],
            "rag": True
        }
    )
    return response.json()["choices"][0]["message"]["content"]

answer = chat("What are the main topics in my documents?")
```

### 2. Batch Document Upload

```python
import requests
from pathlib import Path

def upload_documents(files, dataset, project="my-project"):
    for file_path in files:
        with open(file_path, "rb") as f:
            requests.post(
                f"http://localhost:14345/v1/projects/default/{project}/datasets/{dataset}/data",
                files={"file": f}
            )
```

### 3. Streaming Responses

```python
import requests

def stream_chat(message, project="my-project"):
    response = requests.post(
        f"http://localhost:14345/v1/projects/default/{project}/chat/completions",
        json={
            "messages": [{"role": "user", "content": message}],
            "stream": True
        },
        stream=True
    )
    for line in response.iter_lines():
        if line:
            print(line.decode())
```

## Authentication

By default, LlamaFarm doesn't require authentication for local access.

For production deployments, configure authentication:
```yaml
# In llamafarm.yaml
server:
  auth:
    enabled: true
    api_key: ${env:LLAMAFARM_API_KEY}
```

Then include in requests:
```bash
curl -H "Authorization: Bearer your-api-key" ...
```

## Error Handling

All endpoints return standard error responses:

```json
{
  "error": {
    "code": "not_found",
    "message": "Project 'unknown' not found in namespace 'default'"
  }
}
```

Common HTTP status codes:
| Code | Meaning |
|------|---------|
| 200 | Success |
| 400 | Bad request (invalid parameters) |
| 404 | Resource not found |
| 500 | Server error |
| 503 | Service unavailable (starting up) |

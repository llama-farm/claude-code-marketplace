# LlamaFarm REST API Reference

Complete endpoint reference for the LlamaFarm REST API.

## Base URL

```
http://localhost:14345/v1
```

All project-specific endpoints use the pattern:
```
/v1/projects/{namespace}/{project}/...
```

Default namespace is `default`.

---

## System Endpoints

### Health Check

```http
GET /health
```

**Response:**
```json
{
  "status": "healthy",
  "components": {
    "server": "healthy",
    "celery": "healthy",
    "database": "healthy"
  },
  "version": "0.5.0"
}
```

### Version Check

```http
GET /v1/system/version-check
```

**Response:**
```json
{
  "current_version": "0.5.0",
  "latest_version": "0.5.2",
  "update_available": true,
  "release_notes_url": "https://github.com/llama-farm/llamafarm/releases"
}
```

---

## Project Endpoints

### List Projects

```http
GET /v1/projects/{namespace}
```

**Example:**
```bash
curl http://localhost:14345/v1/projects/default
```

**Response:**
```json
{
  "projects": [
    {
      "name": "my-project",
      "namespace": "default",
      "created_at": "2024-01-14T10:00:00Z",
      "models": ["default", "fast"]
    }
  ]
}
```

### Create Project

```http
POST /v1/projects/{namespace}
Content-Type: application/json
```

**Request Body:**
```json
{
  "name": "new-project",
  "config": {
    "version": "v1",
    "name": "new-project",
    "namespace": "default",
    "runtime": {
      "models": [...]
    }
  }
}
```

### Get Project

```http
GET /v1/projects/{namespace}/{project}
```

**Response:**
```json
{
  "name": "my-project",
  "namespace": "default",
  "config": { ... },
  "stats": {
    "total_documents": 150,
    "total_chunks": 2340,
    "last_activity": "2024-01-14T12:30:00Z"
  }
}
```

### Update Project

```http
PUT /v1/projects/{namespace}/{project}
Content-Type: application/json
```

**Request Body:**
```json
{
  "config": {
    "runtime": {
      "models": [...]
    }
  }
}
```

### Delete Project

```http
DELETE /v1/projects/{namespace}/{project}
```

---

## Chat Endpoints

### Chat Completions

```http
POST /v1/projects/{namespace}/{project}/chat/completions
Content-Type: application/json
```

**Request Body:**
```json
{
  "messages": [
    {"role": "system", "content": "You are helpful."},
    {"role": "user", "content": "Hello!"}
  ],
  "model": "default",
  "rag": true,
  "stream": false,
  "temperature": 0.7,
  "max_tokens": 2048
}
```

**Parameters:**

| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| `messages` | array | required | Chat messages |
| `model` | string | default | Model name from config |
| `rag` | boolean | false | Enable RAG context |
| `stream` | boolean | false | Stream response |
| `temperature` | float | 0.7 | Sampling temperature |
| `max_tokens` | int | 2048 | Max response tokens |
| `session_id` | string | auto | Session for history |
| `database` | string | default | RAG database to query |
| `top_k` | int | 10 | RAG chunks to retrieve |

**Response (non-streaming):**
```json
{
  "id": "chatcmpl-abc123",
  "object": "chat.completion",
  "created": 1705234567,
  "model": "default",
  "choices": [
    {
      "index": 0,
      "message": {
        "role": "assistant",
        "content": "Hello! How can I help you today?"
      },
      "finish_reason": "stop"
    }
  ],
  "usage": {
    "prompt_tokens": 15,
    "completion_tokens": 10,
    "total_tokens": 25
  }
}
```

**Streaming Response:**

With `stream: true`, returns Server-Sent Events:

```
data: {"id":"chatcmpl-abc123","choices":[{"delta":{"content":"Hello"}}]}
data: {"id":"chatcmpl-abc123","choices":[{"delta":{"content":"!"}}]}
data: [DONE]
```

---

## RAG Endpoints

### Query Documents

```http
POST /v1/projects/{namespace}/{project}/rag/query
Content-Type: application/json
```

**Request Body:**
```json
{
  "query": "neural scaling laws",
  "database": "main_db",
  "strategy": "basic_search",
  "top_k": 10,
  "score_threshold": 0.3,
  "filters": {
    "metadata.source": "research_papers"
  }
}
```

**Response:**
```json
{
  "chunks": [
    {
      "id": "chunk-123",
      "content": "Neural scaling laws describe...",
      "score": 0.87,
      "metadata": {
        "source": "neural_scaling.md",
        "heading": "Overview",
        "page": 1
      }
    }
  ],
  "query_time_ms": 125
}
```

### RAG Health

```http
GET /v1/projects/{namespace}/{project}/rag/health
```

**Response:**
```json
{
  "status": "healthy",
  "databases": [
    {
      "name": "main_db",
      "type": "ChromaStore",
      "document_count": 150,
      "chunk_count": 2340,
      "status": "healthy"
    }
  ]
}
```

### List Databases

```http
GET /v1/projects/{namespace}/{project}/rag/databases
```

---

## Dataset Endpoints

### List Datasets

```http
GET /v1/projects/{namespace}/{project}/datasets
```

**Response:**
```json
{
  "datasets": [
    {
      "name": "research",
      "database": "main_db",
      "processing_strategy": "markdown_processor",
      "file_count": 25,
      "status": "processed"
    }
  ]
}
```

### Create Dataset

```http
POST /v1/projects/{namespace}/{project}/datasets
Content-Type: application/json
```

**Request Body:**
```json
{
  "name": "new-dataset",
  "database": "main_db",
  "data_processing_strategy": "pdf_processor"
}
```

### Upload File

```http
POST /v1/projects/{namespace}/{project}/datasets/{dataset}/data
Content-Type: multipart/form-data
```

**Form Data:**
- `file`: The file to upload

**Example:**
```bash
curl -X POST \
  "http://localhost:14345/v1/projects/default/my-project/datasets/research/data" \
  -F "file=@document.pdf"
```

### List Files

```http
GET /v1/projects/{namespace}/{project}/datasets/{dataset}/data
```

### Delete File

```http
DELETE /v1/projects/{namespace}/{project}/datasets/{dataset}/data/{filename}
```

### Process Dataset

```http
POST /v1/projects/{namespace}/{project}/datasets/{dataset}/actions
Content-Type: application/json
```

**Request Body:**
```json
{
  "action": "process"
}
```

**Response:**
```json
{
  "task_id": "task-abc123",
  "status": "started",
  "message": "Processing started for dataset 'research'"
}
```

### Get Processing Status

```http
GET /v1/projects/{namespace}/{project}/datasets/{dataset}/status
```

**Response:**
```json
{
  "status": "processing",
  "progress": {
    "files_total": 25,
    "files_processed": 18,
    "chunks_created": 1560,
    "percent_complete": 72
  }
}
```

---

## Model Endpoints

### List Models

```http
GET /v1/projects/{namespace}/{project}/models
```

**Response:**
```json
{
  "models": [
    {
      "name": "default",
      "provider": "ollama",
      "model": "llama3.1:8b",
      "default": true
    },
    {
      "name": "fast",
      "provider": "ollama",
      "model": "llama3.2:1b",
      "default": false
    }
  ]
}
```

---

## MCP Endpoint

### MCP Server

```http
POST /mcp
```

LlamaFarm exposes its API as an MCP server. Connect from Claude Desktop or other MCP clients:

```json
{
  "mcpServers": {
    "llamafarm": {
      "transport": "http",
      "url": "http://localhost:14345/mcp"
    }
  }
}
```

---

## Error Responses

All errors follow this format:

```json
{
  "error": {
    "code": "error_code",
    "message": "Human-readable message",
    "details": { ... }
  }
}
```

### Error Codes

| Code | HTTP Status | Description |
|------|-------------|-------------|
| `not_found` | 404 | Resource doesn't exist |
| `invalid_request` | 400 | Bad request parameters |
| `validation_error` | 400 | Config validation failed |
| `processing_error` | 500 | Processing failed |
| `service_unavailable` | 503 | Service not ready |
| `rate_limited` | 429 | Too many requests |

---

## Rate Limiting

Default limits (configurable):
- 100 requests/minute for chat
- 1000 requests/minute for RAG queries
- 50 uploads/minute for datasets

Headers returned:
```
X-RateLimit-Limit: 100
X-RateLimit-Remaining: 95
X-RateLimit-Reset: 1705234620
```

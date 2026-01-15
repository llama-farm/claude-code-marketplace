---
description: Check LlamaFarm service health and component status
---

# /status - Service Health Check

Check the health and status of all LlamaFarm components.

## Usage

```
/status                   # Full status report
/status --json            # JSON output for scripting
/status --component rag   # Check specific component
```

## What This Command Does

1. **Checks each component** - Server, RAG worker, databases, runtimes
2. **Reports health** - Healthy, degraded, or failed
3. **Shows configuration** - Active project, models, databases
4. **Identifies issues** - Common problems and fixes

## Implementation

When the user runs `/status`, follow these steps:

### Step 1: Check CLI and Services

```bash
# Check if services are running
lf services status
```

### Step 2: Check Health Endpoint

```bash
curl -s http://localhost:8000/health | jq .
```

Expected healthy response:
```json
{
  "status": "healthy",
  "components": {
    "server": "healthy",
    "celery": "healthy",
    "database": "healthy"
  },
  "version": "0.x.x"
}
```

### Step 3: Check Project Configuration

```bash
# Current project info
lf projects list

# Model availability
lf models list
```

### Step 4: Check RAG Health

```bash
lf rag health
```

### Step 5: Present Comprehensive Status

```
LlamaFarm Status Report
=======================

Overall: HEALTHY

Services:
  Server:     Running (http://localhost:8000)
  RAG Worker: Running (Celery)
  MCP Server: Available (http://localhost:8000/mcp)

Current Project: my-project (namespace: default)

Models Available:
  - default (ollama/llama3.1:8b) [DEFAULT]
  - fast (ollama/llama3.2:1b)

Databases:
  - main_db (ChromaStore): 1,234 documents indexed

Recent Activity:
  - Last chat: 5 minutes ago
  - Last ingestion: 2 hours ago

Health Checks:
  [OK] Server responding
  [OK] Celery worker connected
  [OK] Vector database accessible
  [OK] Model endpoint reachable
```

## Component Details

### Server Health

| Status | Meaning |
|--------|---------|
| `healthy` | Server responding, all endpoints available |
| `degraded` | Server running but some features unavailable |
| `failed` | Server not responding |

### RAG Worker Health

| Status | Meaning |
|--------|---------|
| `healthy` | Celery worker connected and processing |
| `degraded` | Worker running but queue backed up |
| `failed` | Worker not connected |

### Database Health

| Status | Meaning |
|--------|---------|
| `healthy` | Vector store accessible, queries working |
| `degraded` | Store accessible but slow |
| `failed` | Cannot connect to vector store |

## Troubleshooting Output

When issues are detected:

```
LlamaFarm Status Report
=======================

Overall: DEGRADED

Issues Detected:

1. RAG Worker Not Connected
   - Celery worker is not running
   - Fix: lf services start --worker

2. Model Endpoint Unreachable
   - Ollama not responding at localhost:11434
   - Fix: ollama serve

Recommendations:
- Run /logs to see error details
- Check debugging/service-issues skill for more help
```

## JSON Output

For scripting and automation:

```bash
lf services status --json
```

```json
{
  "overall": "healthy",
  "services": {
    "server": {"status": "healthy", "url": "http://localhost:8000"},
    "worker": {"status": "healthy", "queue_length": 0},
    "mcp": {"status": "healthy", "url": "http://localhost:8000/mcp"}
  },
  "project": {
    "name": "my-project",
    "namespace": "default"
  },
  "models": [
    {"name": "default", "provider": "ollama", "model": "llama3.1:8b"}
  ],
  "databases": [
    {"name": "main_db", "type": "ChromaStore", "document_count": 1234}
  ]
}
```

## Related Commands

- `/start` - Start services
- `/stop` - Stop services
- `/logs` - View service logs

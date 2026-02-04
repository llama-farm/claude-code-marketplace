# Check LlamaFarm Service Status

Check the health and status of all LlamaFarm components.

## Usage

```
Check service status               # Full health check
Check server health                # API server only
Check RAG health                   # RAG worker and databases
```

## What This Does

1. **Checks CLI availability** - Verifies `lf` command exists
2. **Checks service health** - Queries each component
3. **Checks health endpoint** - Tests `http://localhost:14345/health`
4. **Checks project config** - Validates `llamafarm.yaml`
5. **Checks RAG health** - Tests vector database connectivity
6. **Reports comprehensive status** - Formatted component table

## Implementation

### Step 1: Check CLI and Services

```bash
# Verify LlamaFarm CLI
which lf

# Get service status
lf services status
```

### Step 2: Check Health Endpoint

```bash
# API health check
curl -s http://localhost:14345/health | jq .
```

Expected healthy response:
```json
{
  "status": "healthy",
  "version": "0.x.x",
  "components": {
    "server": "running",
    "celery": "running",
    "ollama": "connected",
    "chromadb": "connected"
  }
}
```

### Step 3: Check Project Configuration

```bash
# Validate project config
lf projects validate --config llamafarm.yaml

# Show current configuration
lf config show
```

### Step 4: Check RAG Health

```bash
# RAG system health
lf rag health

# List available databases
lf rag list-databases

# Test a query
lf rag query --database main_db --top-k 1 "test"
```

### Step 5: Report Status

```
LlamaFarm Status Report
========================

Component         Status    Details
---------         ------    -------
CLI               OK        v0.x.x
Server            Running   http://localhost:14345
RAG Worker        Running   Processing queue: 0 pending
Ollama            Connected localhost:11434
ChromaDB          Connected 3 collections
Project Config    Valid     ./llamafarm.yaml

Overall: All systems operational
```

## Component Details

### Server Status

| Field | Source | Meaning |
|-------|--------|---------|
| Status | `/health` endpoint | Running / Stopped / Error |
| URL | Config | Where the API is listening |
| PID | Process table | Server process ID |
| Uptime | `/health` response | Time since last start |

### RAG Worker Status

| Field | Source | Meaning |
|-------|--------|---------|
| Status | `lf services status` | Running / Stopped |
| Queue | Celery inspect | Pending tasks count |
| Active | Celery inspect | Currently processing |
| Databases | `lf rag health` | Connected vector stores |

### Ollama Status

| Field | Source | Meaning |
|-------|--------|---------|
| Status | Connection test | Connected / Disconnected |
| URL | Config | Ollama endpoint |
| Models | `ollama list` | Available models |
| Running | `ollama ps` | Currently loaded models |

### ChromaDB Status

| Field | Source | Meaning |
|-------|--------|---------|
| Status | Connection test | Connected / Disconnected |
| Collections | `lf rag health` | Number of collections |
| Documents | Collection stats | Total indexed documents |

## Troubleshooting Output

When components are unhealthy:

```
LlamaFarm Status Report
========================

Component         Status    Details
---------         ------    -------
CLI               OK        v0.x.x
Server            ERROR     Port 14345 not responding
RAG Worker        Stopped   Not running
Ollama            OK        localhost:11434
ChromaDB          OK        3 collections
Project Config    Warning   llamafarm.yaml not found in current directory

Issues Found:
1. Server is not responding on port 14345
   → Start services: lf start

2. RAG Worker is not running
   → Will start automatically with: lf start

3. No project config in current directory
   → Initialize: lf projects init my-project
   → Or navigate to project directory
```

## JSON Output

For programmatic consumption:

```bash
# JSON format
curl -s http://localhost:14345/health | jq .

# Check specific fields
curl -s http://localhost:14345/health | jq '.components.server'
curl -s http://localhost:14345/health | jq '.components.celery'
```

Example JSON output:
```json
{
  "status": "healthy",
  "version": "0.4.2",
  "uptime_seconds": 3621,
  "components": {
    "server": {
      "status": "running",
      "port": 14345,
      "pid": 12345
    },
    "celery": {
      "status": "running",
      "queued_tasks": 0,
      "active_tasks": 0
    },
    "ollama": {
      "status": "connected",
      "url": "http://localhost:11434",
      "models_loaded": ["llama3.1:8b"]
    },
    "chromadb": {
      "status": "connected",
      "collections": 3,
      "total_documents": 1250
    }
  }
}
```

## Related

- `start.md` - Starting services
- `stop.md` - Stopping services
- `logs.md` - Viewing service logs

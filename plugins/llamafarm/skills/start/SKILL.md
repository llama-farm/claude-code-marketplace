---
description: Start LlamaFarm services (server, RAG worker, optional Universal Runtime)
disable-model-invocation: true
allowed-tools:
  - Bash
argument-hint: "[--no-browser] [--runtime] [--timeout <seconds>]"
---

# /llamafarm:start - Start LlamaFarm Services

Start the LlamaFarm backend services required for AI chat, RAG, and project management.

## Usage

```
/llamafarm:start                    # Start all services
/llamafarm:start --no-browser       # Start without opening Designer UI
/llamafarm:start --runtime          # Also start Universal Runtime (ML/embeddings)
```

## What This Command Does

1. **Checks prerequisites** - Verifies LlamaFarm CLI is installed
2. **Starts services** - Launches FastAPI server and RAG worker
3. **Waits for health** - Polls until services are ready
4. **Reports status** - Shows component health and URLs

## Implementation

When the user runs `/llamafarm:start`, follow these steps:

### Step 1: Check CLI Installation

```bash
which lf || echo "NOT_FOUND"
```

If not found, provide installation instructions:
```
LlamaFarm CLI not found. Install it with:

# Using Homebrew (recommended)
brew install llama-farm/tap/lf

# Or download from GitHub releases
https://github.com/llama-farm/llamafarm/releases
```

### Step 2: Check for Existing Services

```bash
lf services status 2>/dev/null || echo "SERVICES_NOT_RUNNING"
```

If services are already running, report status and skip startup.

### Step 3: Start Services

```bash
lf start
```

This command:
- Starts the FastAPI server on port 8000
- Starts the Celery RAG worker for async processing
- Opens the Designer UI in browser (unless `--no-browser`)

### Step 4: Wait for Health

Poll the health endpoint until ready (up to 30 seconds):

```bash
curl -s http://localhost:14345/health | jq .
```

Expected response:
```json
{
  "status": "healthy",
  "components": {
    "server": "healthy",
    "celery": "healthy"
  }
}
```

### Step 5: Report Results

Present results to user:

```
LlamaFarm Services Started
==========================

Status: Running
Server: http://localhost:14345
Designer UI: http://localhost:14345 (opened in browser)
API Docs: http://localhost:14345/docs

Components:
- Server: healthy
- RAG Worker: healthy

MCP Server: http://localhost:14345/mcp (available for Claude Code)

Next steps:
- Create a project: lf init my-project
- Or use /llamafarm:config to generate a configuration
```

## Options

| Option | Description |
|--------|-------------|
| `--no-browser` | Don't open Designer UI in browser |
| `--runtime` | Also start Universal Runtime for ML/embeddings |
| `--timeout <seconds>` | Health check timeout (default: 30) |

## Starting Universal Runtime

If `--runtime` is specified or user has Universal Runtime configured:

```bash
# Check if Universal Runtime is configured
grep -q "universal" llamafarm.yaml && echo "UNIVERSAL_CONFIGURED"

# Start Universal Runtime (separate terminal/process)
lf runtime start
```

Universal Runtime provides:
- Local embedding models (nomic-embed-text-v2-moe)
- ML inference (anomaly detection, classification)
- OCR capabilities

## Error Handling

### Port Already in Use

```bash
lsof -i :14345
```

If port is occupied, suggest:
```
Port 8000 is already in use by another process.

Options:
1. Stop the existing process: kill <PID>
2. Use a different port: LF_SERVER_PORT=8001 lf start
```

### Missing Dependencies

If startup fails due to missing Python packages:
```
Some LlamaFarm dependencies are missing.

Run:
cd /path/to/llamafarm
uv sync --all-extras
```

### Config Not Found

If no `llamafarm.yaml` exists:
```
No llamafarm.yaml found in current directory.

Options:
1. Initialize a new project: lf init my-project
2. Use /llamafarm:config to generate a configuration
3. Navigate to an existing LlamaFarm project directory
```

## Related Commands

- `/llamafarm:stop` - Stop services
- `/llamafarm:status` - Check service health
- `/llamafarm:logs` - View service logs

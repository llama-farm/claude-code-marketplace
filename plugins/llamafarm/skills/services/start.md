# Start LlamaFarm Services

Start the LlamaFarm server, RAG worker, and optional Universal Runtime.

## Usage

```
Start services                     # Start all default services
Start with runtime                 # Include Universal Runtime
Start without browser              # Don't open browser UI
```

## What This Does

1. **Checks CLI installation** - Verifies `lf` command is available
2. **Checks existing services** - Warns if services are already running
3. **Starts services** - Launches server, RAG worker, and dependencies
4. **Polls for health** - Waits for `http://localhost:14345/health` to respond
5. **Reports results** - Shows what's running and where

## Implementation

### Step 1: Check Prerequisites

```bash
# Verify LlamaFarm CLI is installed
which lf || pip show llamafarm

# Check if services are already running
lf services status
```

If `lf` is not found:
```
LlamaFarm CLI is not installed.

Install with:
  pip install llamafarm

Or if using a virtual environment:
  source ~/.llamafarm/venv/bin/activate
```

### Step 2: Check for Existing Services

```bash
# Check if port is already in use
lsof -i :14345
```

If services are already running:
```
LlamaFarm services are already running.

Server:     http://localhost:14345 (PID 12345)
RAG Worker: Running (PID 12346)

To restart, stop first:
  lf services stop
  lf start
```

### Step 3: Start Services

```bash
# Start all services (default)
lf start

# Start without opening browser
lf start --no-browser

# Start with Universal Runtime
lf start --runtime

# Start with custom timeout
lf start --timeout 60
```

### Step 4: Poll for Health

After starting, poll the health endpoint until it responds:

```bash
# Poll health endpoint (retry for up to 30 seconds)
for i in $(seq 1 30); do
  if curl -s http://localhost:14345/health > /dev/null 2>&1; then
    echo "Server is ready"
    break
  fi
  sleep 1
done
```

### Step 5: Report Results

```
LlamaFarm Services Started
===========================

Server:      http://localhost:14345 ✓
RAG Worker:  Running ✓
Ollama:      Connected (localhost:11434) ✓

Health: All systems operational

Quick test:
  curl http://localhost:14345/health
  curl http://localhost:14345/v1/models
```

## Options

| Option | Description | Default |
|--------|-------------|---------|
| `--no-browser` | Don't open browser after start | Opens browser |
| `--runtime` | Start Universal Runtime alongside | Not started |
| `--timeout` | Seconds to wait for health | 30 |
| `--config` | Path to config file | `./llamafarm.yaml` |

## Universal Runtime

The Universal Runtime provides additional capabilities:

```bash
# Start with runtime
lf start --runtime
```

When running with runtime:
```
LlamaFarm Services Started
===========================

Server:      http://localhost:14345 ✓
RAG Worker:  Running ✓
Runtime:     http://localhost:14346 ✓
Ollama:      Connected (localhost:11434) ✓
```

## Error Handling

### Port Conflict

```
Error: Port 14345 is already in use

Check what's using it:
  lsof -i :14345

Options:
1. Stop the conflicting process
2. Use a different port:
   lf start --port 14346
```

### Missing Dependencies

```
Error: Ollama is not running

Start Ollama first:
  ollama serve

Or install Ollama:
  https://ollama.ai/download
```

### Configuration Not Found

```
Error: No llamafarm.yaml found

Create one with:
  lf projects init my-project

Or specify a path:
  lf start --config /path/to/llamafarm.yaml
```

### Startup Timeout

```
Error: Services did not become healthy within 30 seconds

Check logs for errors:
  lf services logs --errors

Common causes:
1. Ollama is slow to load model (increase --timeout)
2. Port conflict (check lsof -i :14345)
3. Configuration error (check llamafarm.yaml)
```

## Related

- `stop.md` - Stopping services
- `status.md` - Checking service health
- `logs.md` - Viewing service logs

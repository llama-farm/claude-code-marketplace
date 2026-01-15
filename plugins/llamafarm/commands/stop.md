---
description: Stop LlamaFarm services gracefully
---

# /stop - Stop LlamaFarm Services

Gracefully stop all running LlamaFarm services.

## Usage

```
/stop                     # Stop all services
/stop --force             # Force stop (SIGKILL)
```

## What This Command Does

1. **Checks running services** - Identifies what's currently running
2. **Graceful shutdown** - Sends stop signal to services
3. **Waits for cleanup** - Allows in-progress tasks to complete
4. **Confirms shutdown** - Verifies services are stopped

## Implementation

When the user runs `/stop`, follow these steps:

### Step 1: Check Current Status

```bash
lf services status
```

If no services are running:
```
No LlamaFarm services are currently running.
```

### Step 2: Stop Services

```bash
lf services stop
```

This gracefully stops:
- FastAPI server
- Celery RAG worker
- Any background tasks

### Step 3: Verify Shutdown

```bash
# Check that port 8000 is freed
lsof -i :8000 || echo "PORT_FREED"

# Verify no LlamaFarm processes
pgrep -f "llamafarm" || echo "NO_PROCESSES"
```

### Step 4: Report Results

```
LlamaFarm Services Stopped
==========================

All services have been stopped gracefully.

Components stopped:
- Server (port 8000)
- RAG Worker

To restart: /start or `lf start`
```

## Force Stop

If services don't respond to graceful shutdown:

```bash
# Force kill all LlamaFarm processes
pkill -9 -f "llamafarm"
pkill -9 -f "celery.*llamafarm"
```

Report:
```
Services force-stopped.

Note: Force stop may leave temporary files or incomplete tasks.
Check /status to verify clean state.
```

## Error Handling

### Services Not Responding

If graceful stop times out:
```
Services not responding to graceful shutdown.

Options:
1. Wait longer: /stop --timeout 60
2. Force stop: /stop --force
3. Manual cleanup: pkill -9 -f llamafarm
```

### Orphaned Processes

If processes remain after stop:
```bash
ps aux | grep -E "(llamafarm|celery)" | grep -v grep
```

Provide PIDs for manual cleanup if needed.

## Related Commands

- `/start` - Start services
- `/status` - Check service health

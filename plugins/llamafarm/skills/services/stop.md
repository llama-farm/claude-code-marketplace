# Stop LlamaFarm Services

Gracefully stop all running LlamaFarm services.

## Usage

```
Stop services                      # Graceful shutdown of all services
Force stop services                # Kill services immediately
```

## What This Does

1. **Checks current status** - Identifies which services are running
2. **Sends stop signal** - Gracefully shuts down each service
3. **Verifies shutdown** - Confirms ports are freed and processes stopped
4. **Reports results** - Shows what was stopped

## Implementation

### Step 1: Check Current Status

```bash
# See what's running
lf services status

# Check for LlamaFarm processes
lsof -i :14345
```

If no services are running:
```
No LlamaFarm services are currently running.
Nothing to stop.
```

### Step 2: Stop Services

```bash
# Graceful stop
lf services stop
```

This sends a graceful shutdown signal to:
- API server (port 14345)
- RAG worker (Celery)
- Universal Runtime (if running)

### Step 3: Verify Shutdown

```bash
# Verify port is freed
lsof -i :14345

# Verify no LlamaFarm processes remain
ps aux | grep -E "llamafarm|lf serve" | grep -v grep
```

### Step 4: Report Results

```
LlamaFarm Services Stopped
===========================

Server:      Stopped ✓
RAG Worker:  Stopped ✓
Port 14345:  Free ✓

All services shut down cleanly.
```

## Force Stop

If graceful shutdown fails or hangs:

```bash
# Force stop all services
lf services stop --force
```

If that still doesn't work, manually kill processes:

```bash
# Find and kill LlamaFarm processes
pkill -f "llamafarm"
pkill -f "lf serve"

# Kill by port
lsof -ti :14345 | xargs kill -9

# Verify cleanup
lsof -i :14345
```

## Error Handling

### Graceful Shutdown Timeout

```
Warning: Services did not stop within 10 seconds

Attempting force stop...
  lf services stop --force

Or manually:
  pkill -f "llamafarm"
```

### Orphaned Processes

```
Warning: Port 14345 still in use after stop

Finding orphaned process:
  lsof -i :14345

Kill it manually:
  kill -9 <PID>
```

### Permission Errors

```
Error: Permission denied when stopping services

Try with appropriate permissions:
  sudo lf services stop

Or kill the process directly:
  kill <PID>
```

## Related

- `start.md` - Starting services
- `status.md` - Checking service health
- `logs.md` - Viewing service logs

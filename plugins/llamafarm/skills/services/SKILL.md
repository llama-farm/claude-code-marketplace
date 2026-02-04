---
description: Manage LlamaFarm services — start, stop, check status, and view logs. Activates for service management, health checks, and log viewing.
---

# LlamaFarm Services Skill

Manage the lifecycle of LlamaFarm services including the API server, RAG worker, and optional Universal Runtime.

## When to Load

Load this skill when the user:
- Wants to start or stop LlamaFarm services
- Needs to check if services are running or healthy
- Wants to view or filter service logs
- Is troubleshooting service startup failures
- Asks about ports, processes, or health endpoints

## Quick Reference

| Operation | Command | Description |
|-----------|---------|-------------|
| Start | `lf start` | Start server, RAG worker, and dependencies |
| Stop | `lf services stop` | Gracefully stop all running services |
| Status | `lf services status` | Check health of all components |
| Logs | `lf services logs` | View and filter service output |

## Common Workflows

### First-Time Setup

```bash
# 1. Start all services
lf start

# 2. Verify everything is healthy
lf services status

# 3. Confirm the API is responding
curl http://localhost:14345/health
```

### Daily Development

```bash
# Start services
lf start

# ... do work ...

# Check if something seems wrong
lf services status

# Look at recent errors
lf services logs | grep -E "ERROR|CRITICAL"

# Done for the day
lf services stop
```

### Troubleshooting a Problem

```bash
# Check what's running
lf services status

# Look at recent logs for errors
lf services logs --errors

# Restart services
lf services stop
lf start

# Watch logs in real-time
lf services logs --follow
```

## Quick Commands

```bash
# Health check (is the server responding?)
curl http://localhost:14345/health

# Check what's listening on the LlamaFarm port
lsof -i :14345

# Start with options
lf start --no-browser          # Don't open browser
lf start --runtime             # Include Universal Runtime

# Logs for a specific service
lf services logs --service server
lf services logs --service rag
lf services logs --service celery

# Force stop if graceful shutdown fails
lf services stop --force
```

## Progressive Disclosure

For detailed documentation on each operation:
- `start.md` - Starting services, options, health polling, and error handling
- `stop.md` - Graceful shutdown, force stop, and verification
- `status.md` - Health checks, component details, and troubleshooting output
- `logs.md` - Log sources, filtering, error analysis, and common patterns

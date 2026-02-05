# Service Issues

Troubleshoot LlamaFarm service startup and connectivity problems.

## Common Service Issues

### Server Won't Start

**Symptoms:**
- `lf start` fails
- "Address already in use" error
- Server process exits immediately

**Diagnostics:**

```bash
# Check if port is in use
lsof -i :14345

# Check for existing processes
ps aux | grep -E "llamafarm|uvicorn"

# Check logs for startup errors
lf services logs --service server | head -50
```

**Solutions:**

1. **Port in use by another process:**
```bash
# Kill the process using the port
kill $(lsof -t -i:14345)

# Or use different port
LF_SERVER_PORT=8001 lf start
```

2. **Previous instance didn't shut down:**
```bash
# Stop all LlamaFarm processes
lf services stop --force

# Start fresh
lf start
```

3. **Configuration error:**
```bash
# Validate config
lf projects validate --config llamafarm.yaml

# Fix any errors reported
```

---

### RAG Worker Not Running

**Symptoms:**
- Dataset processing hangs
- "Celery not connected" in logs
- Tasks stay in "pending" state

**Diagnostics:**

```bash
# Check worker status
lf services status | grep -i worker

# Check Celery logs
lf services logs --service celery
```

**Solutions:**

1. **Worker crashed:**
```bash
# Restart services
lf services stop
lf start
```

2. **Missing dependencies:**
```bash
# Reinstall dependencies
cd /path/to/llamafarm
uv sync --all-extras
```

3. **Resource exhaustion:**
```bash
# Check memory
free -h

# Reduce concurrent workers
CELERY_WORKERS=1 lf start
```

---

### Ollama Connection Failed

**Symptoms:**
- "Connection refused" to localhost:11434
- "Model not found" errors
- Chat returns empty responses

**Diagnostics:**

```bash
# Check if Ollama is running
curl http://localhost:11434/api/tags

# Check Ollama status
ollama list
```

**Solutions:**

1. **Ollama not running:**
```bash
# Start Ollama
ollama serve

# Or on macOS, start the app
open /Applications/Ollama.app
```

2. **Model not pulled:**
```bash
# Pull the required model
ollama pull llama3.1:8b

# Verify it's available
ollama list
```

3. **Wrong base_url:**
```yaml
# Check llamafarm.yaml
runtime:
  models:
    - name: default
      provider: ollama
      base_url: http://localhost:11434  # Correct URL
```

---

### Database Connection Failed

**Symptoms:**
- "ChromaDB error" in logs
- RAG queries fail
- "Collection not found"

**Diagnostics:**

```bash
# Check RAG health
lf rag health

# Check database directory
ls -la ./data/
```

**Solutions:**

1. **Corrupt database:**
```bash
# Backup and recreate
mv ./data/main_db ./data/main_db.bak
lf start
# Re-process datasets
```

2. **Permission issues:**
```bash
# Fix permissions
chmod -R 755 ./data/
```

3. **Disk full:**
```bash
# Check disk space
df -h

# Clean up old data if needed
```

---

### Universal Runtime Not Available

**Symptoms:**
- Embeddings fail
- "Connection refused" to port 11540
- ML features unavailable

**Diagnostics:**

```bash
# Check if runtime is running
curl http://localhost:11540/health

# Check runtime logs
lf services logs --service runtime
```

**Solutions:**

1. **Runtime not started:**
```bash
# Start with runtime
lf start --runtime

# Or start separately
lf runtime start
```

2. **Missing ML dependencies:**
```bash
# Install ML extras
pip install llamafarm[ml]
```

---

## Service Health Checks

### API Health

```bash
curl http://localhost:14345/health | jq .
```

Expected:
```json
{
  "status": "healthy",
  "components": {
    "server": "healthy",
    "celery": "healthy"
  }
}
```

### Component Checks

| Component | Check Command | Expected |
|-----------|---------------|----------|
| Server | `curl http://localhost:14345/health` | 200 OK |
| Ollama | `curl http://localhost:11434/api/tags` | JSON with models |
| Runtime | `curl http://localhost:11540/health` | 200 OK |
| Celery | `lf services status` | "worker: running" |

---

## Startup Sequence

LlamaFarm starts services in this order:

1. **Configuration loading** - Parse llamafarm.yaml
2. **FastAPI server** - Start on port 8000
3. **Celery worker** - Connect to task queue
4. **MCP servers** - Start configured servers
5. **Health check** - Verify all components

If any step fails, check logs for that component.

---

## Recovery Procedures

### Full Reset

```bash
# Stop everything
lf services stop --force

# Clean temporary files
rm -rf .llamafarm/cache/
rm -rf __pycache__/

# Restart
lf start
```

### Database Reset

```bash
# Stop services
lf services stop

# Remove database files (backup first!)
mv ./data/ ./data.bak/

# Restart and reprocess
lf start
lf datasets process <dataset-name>
```

### Configuration Reset

```bash
# Validate current config
lf projects validate --config llamafarm.yaml

# If invalid, start from example
cp examples/quick_rag/llamafarm.yaml ./llamafarm.yaml
```

---
description: Debugging and troubleshooting guide for LlamaFarm. Diagnose service issues, ingestion failures, retrieval problems, and performance bottlenecks.
---

# LlamaFarm Debugging Skill

Comprehensive troubleshooting guide for LlamaFarm issues.

## When to Load

Load this skill when the user:
- Reports errors or unexpected behavior
- Has services that won't start
- Experiences slow performance
- Gets poor RAG results
- Has document processing failures

## Quick Diagnostics

### Health Check

```bash
# Check all services
lf services status

# Check RAG health
lf rag health

# API health endpoint
curl http://localhost:8000/health
```

### Common Issues Quick Reference

| Symptom | Likely Cause | Quick Fix |
|---------|--------------|-----------|
| "Connection refused" | Services not running | `lf start` |
| "Model not found" | Model not pulled | `ollama pull model-name` |
| "Processing stuck" | Worker not running | Restart RAG worker |
| "No results" | Empty database | Process datasets |
| "Slow responses" | Large context | Reduce top_k |

## Progressive Disclosure

For detailed troubleshooting:
- `service-issues.md` - Service startup and connectivity
- `ingestion-issues.md` - Document processing failures
- `retrieval-issues.md` - Poor RAG results
- `performance.md` - Speed and resource optimization

## Diagnostic Commands

### Service Status

```bash
# Overall status
lf services status

# Individual components
curl http://localhost:8000/health | jq .

# Check ports
lsof -i :8000
lsof -i :11434
```

### Logs

```bash
# All logs
lf services logs

# Filter by service
lf services logs --service server
lf services logs --service rag

# Real-time
lf services logs --follow

# Errors only
lf services logs | grep -E "ERROR|CRITICAL"
```

### Configuration

```bash
# Show current config
lf config show

# Validate config
lf projects validate --config llamafarm.yaml
```

### RAG Diagnostics

```bash
# Database health
lf rag health

# Test query
lf rag query --database main_db --top-k 5 "test query"

# List datasets
lf datasets list

# Dataset status
lf datasets status <dataset-name>
```

## Error Categories

### 1. Service Errors (5xx)

Server-side issues:
- Services not running
- Configuration errors
- Resource exhaustion

**Diagnostic:**
```bash
lf services status
lf services logs --errors
```

### 2. Client Errors (4xx)

Request issues:
- Invalid parameters
- Missing resources
- Authentication failures

**Diagnostic:**
```bash
# Check specific endpoint
curl -v http://localhost:8000/v1/projects/default/my-project
```

### 3. Model Errors

LLM issues:
- Model not available
- Context too long
- Timeout

**Diagnostic:**
```bash
ollama list
lf models list
```

### 4. RAG Errors

Retrieval issues:
- Empty database
- Bad embeddings
- Strategy errors

**Diagnostic:**
```bash
lf rag health
lf rag query --database main_db "test"
```

## Troubleshooting Workflow

### Step 1: Identify Symptoms

- What exactly is happening?
- What error messages appear?
- When did it start?

### Step 2: Check Services

```bash
lf services status
# All green? → Check logs
# Red services? → Start/restart them
```

### Step 3: Check Logs

```bash
lf services logs --errors | head -50
```

Look for:
- Stack traces
- Error codes
- Timestamps

### Step 4: Isolate Component

Test each component:
```bash
# Server
curl http://localhost:8000/health

# Model
ollama run llama3.1:8b "test"

# Database
lf rag health
```

### Step 5: Apply Fix

Based on findings, apply the appropriate fix from the detailed guides.

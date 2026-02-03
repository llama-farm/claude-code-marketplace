# MCP Transport Types

LlamaFarm supports three transport mechanisms for connecting to MCP servers.

## Overview

| Transport | Use Case | Connection |
|-----------|----------|------------|
| **STDIO** | Local tools | Subprocess |
| **HTTP** | Remote APIs | HTTP streaming |
| **SSE** | Real-time | Server-Sent Events |

---

## STDIO Transport

Spawns the MCP server as a local subprocess. Best for local tools.

### Configuration

```yaml
mcp:
  servers:
    - name: my-server
      transport: stdio
      command: python
      args: ['-m', 'my_mcp_server']
      env:
        API_KEY: ${env:API_KEY}
```

### Fields

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `transport` | string | Yes | Must be `stdio` |
| `command` | string | Yes | Command to launch server |
| `args` | array | No | Command arguments |
| `env` | object | No | Environment variables |

### Examples

**Node.js MCP Server:**
```yaml
mcp:
  servers:
    - name: filesystem
      transport: stdio
      command: npx
      args:
        - '-y'
        - '@modelcontextprotocol/server-filesystem'
        - '/allowed/directory'
```

**Python MCP Server:**
```yaml
mcp:
  servers:
    - name: my-tools
      transport: stdio
      command: python
      args: ['-m', 'my_mcp_server']
      env:
        DATABASE_URL: ${env:DATABASE_URL}
```

**Custom Binary:**
```yaml
mcp:
  servers:
    - name: custom
      transport: stdio
      command: /usr/local/bin/my-mcp-server
      args: ['--config', './config.json']
```

### Common STDIO Servers

```yaml
# Filesystem access
- name: filesystem
  transport: stdio
  command: npx
  args: ['-y', '@modelcontextprotocol/server-filesystem', '/path']

# SQLite database
- name: sqlite
  transport: stdio
  command: npx
  args: ['-y', '@modelcontextprotocol/server-sqlite', './data.db']

# PostgreSQL database
- name: postgres
  transport: stdio
  command: npx
  args: ['-y', '@modelcontextprotocol/server-postgres']
  env:
    DATABASE_URL: ${env:POSTGRES_URL}

# GitHub integration
- name: github
  transport: stdio
  command: npx
  args: ['-y', '@modelcontextprotocol/server-github']
  env:
    GITHUB_TOKEN: ${env:GITHUB_TOKEN}

# Brave search
- name: brave-search
  transport: stdio
  command: npx
  args: ['-y', '@modelcontextprotocol/server-brave-search']
  env:
    BRAVE_API_KEY: ${env:BRAVE_API_KEY}
```

---

## HTTP Transport

Connects to a remote MCP server via HTTP streaming. Best for cloud services and shared tools.

### Configuration

```yaml
mcp:
  servers:
    - name: remote-api
      transport: http
      base_url: https://api.example.com/mcp
      headers:
        Authorization: Bearer ${env:API_TOKEN}
        X-Custom-Header: my-value
```

### Fields

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `transport` | string | Yes | Must be `http` |
| `base_url` | string | Yes | MCP endpoint URL |
| `headers` | object | No | HTTP headers |

### Examples

**LlamaFarm's Own MCP Server:**
```yaml
mcp:
  servers:
    - name: llamafarm
      transport: http
      base_url: http://localhost:14345/mcp
```

**Authenticated Remote API:**
```yaml
mcp:
  servers:
    - name: company-api
      transport: http
      base_url: https://tools.company.com/mcp
      headers:
        Authorization: Bearer ${env:COMPANY_API_KEY}
        X-Client-ID: llamafarm
```

**FastAPI MCP Server:**
```yaml
mcp:
  servers:
    - name: fastapi-tools
      transport: http
      base_url: http://localhost:8080/mcp
```

### Building an HTTP MCP Server

Example with FastAPI:

```python
from fastapi import FastAPI, Request
from fastapi.responses import StreamingResponse
from mcp.server import Server
from mcp.server.http import http_server

app = FastAPI()
mcp_app = Server("my-api")

@mcp_app.tool()
async def my_tool(input: str) -> str:
    return f"Processed: {input}"

@app.post("/mcp")
async def mcp_endpoint(request: Request):
    return await http_server(mcp_app, request)

if __name__ == "__main__":
    import uvicorn
    uvicorn.run(app, host="0.0.0.0", port=8080)
```

---

## SSE Transport

Connects via Server-Sent Events for real-time streaming.

### Configuration

```yaml
mcp:
  servers:
    - name: streaming-server
      transport: sse
      base_url: https://api.example.com/mcp/sse
```

### Fields

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `transport` | string | Yes | Must be `sse` |
| `base_url` | string | Yes | SSE endpoint URL |

### Use Cases

- Real-time data feeds
- Long-running operations with progress updates
- Event-driven tools

---

## Environment Variable Substitution

Use `${env:VARIABLE_NAME}` to inject environment variables:

```yaml
mcp:
  servers:
    - name: secure-api
      transport: http
      base_url: ${env:API_BASE_URL}
      headers:
        Authorization: Bearer ${env:API_TOKEN}
        X-API-Key: ${env:API_KEY}
```

This keeps secrets out of configuration files.

---

## Choosing a Transport

| Scenario | Transport | Reason |
|----------|-----------|--------|
| Local Python tools | STDIO | Simple, no network |
| Local Node.js tools | STDIO | NPM packages work well |
| Cloud APIs | HTTP | Standard REST integration |
| Self-hosted tools | HTTP | Shared across instances |
| Real-time data | SSE | Streaming support |
| Claude Desktop | HTTP | External process access |
| Development | STDIO | Easy debugging |
| Production | HTTP | Scalable, monitored |

---

## Connection Management

LlamaFarm handles connections automatically:

- **Connection pooling** - Sessions reused across requests
- **5-minute cache** - Tool lists cached for performance
- **Graceful shutdown** - Connections closed cleanly
- **1-hour timeout** - Long sessions supported

---

## Troubleshooting

### STDIO Issues

```bash
# Test server manually
python -m my_mcp_server

# Check command exists
which npx

# Verify environment variables
echo $API_KEY
```

### HTTP Issues

```bash
# Test endpoint
curl https://api.example.com/mcp/health

# Check headers
curl -H "Authorization: Bearer $TOKEN" https://api.example.com/mcp
```

### Common Errors

| Error | Cause | Solution |
|-------|-------|----------|
| "Command not found" | Missing binary | Install package or check PATH |
| "Connection refused" | Server not running | Start server, check URL |
| "Unauthorized" | Bad credentials | Verify API key/token |
| "Timeout" | Slow server | Increase timeout, check network |

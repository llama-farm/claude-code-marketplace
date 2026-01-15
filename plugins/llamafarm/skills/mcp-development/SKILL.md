---
description: MCP (Model Context Protocol) development guide. Build custom tool servers, configure inline tools, and manage tool access in LlamaFarm.
---

# MCP Development Skill

Guide for building and integrating MCP servers with LlamaFarm.

## When to Load

Load this skill when the user:
- Wants to build a custom MCP server
- Needs to expose business logic as AI tools
- Is configuring MCP servers in LlamaFarm
- Asks about inline tool definitions
- Needs help with tool access control

## MCP Overview

MCP (Model Context Protocol) is a standardized protocol for giving AI models access to external tools, APIs, and data sources.

LlamaFarm supports MCP both as:
1. **Client** - Connect to external MCP servers
2. **Server** - Expose LlamaFarm's API as MCP tools

## Quick Reference

### Connect to MCP Server

```yaml
mcp:
  servers:
    - name: filesystem
      transport: stdio
      command: npx
      args: ['-y', '@modelcontextprotocol/server-filesystem', '/data']
```

### Assign to Model

```yaml
runtime:
  models:
    - name: assistant
      mcp_servers: [filesystem]
```

### Define Inline Tools

```yaml
runtime:
  models:
    - name: assistant
      tools:
        - type: function
          name: my_tool
          description: Does something useful
          parameters:
            type: object
            properties:
              input:
                type: string
```

## Progressive Disclosure

For detailed documentation:
- `python-mcp-server.md` - Build MCP servers in Python
- `inline-tools.md` - Define tools in YAML configuration
- `transport-types.md` - STDIO, HTTP, SSE transports
- `access-control.md` - Per-model tool access

## Common Use Cases

### 1. File System Access

Give models access to read/write files:

```yaml
mcp:
  servers:
    - name: filesystem
      transport: stdio
      command: npx
      args: ['-y', '@modelcontextprotocol/server-filesystem', '/allowed/path']

runtime:
  models:
    - name: assistant
      mcp_servers: [filesystem]
```

### 2. Database Queries

Let models query SQLite databases:

```yaml
mcp:
  servers:
    - name: database
      transport: stdio
      command: npx
      args: ['-y', '@modelcontextprotocol/server-sqlite', './data.db']
```

### 3. Custom Python Tools

Expose your own business logic:

```yaml
mcp:
  servers:
    - name: my-tools
      transport: stdio
      command: python
      args: ['-m', 'my_mcp_server']
```

### 4. Remote API

Connect to cloud-hosted MCP servers:

```yaml
mcp:
  servers:
    - name: company-api
      transport: http
      base_url: https://api.company.com/mcp
      headers:
        Authorization: Bearer ${env:API_TOKEN}
```

## Tool Call Flow

1. User sends message to LlamaFarm
2. Model decides to call a tool
3. LlamaFarm executes tool via MCP
4. Result returned to model
5. Model generates final response

```
User → LlamaFarm → Model → Tool Call → MCP Server → Result → Model → Response
```

## Security Considerations

### Least Privilege

Only grant tools the model actually needs:

```yaml
runtime:
  models:
    # Research model: read-only access
    - name: researcher
      mcp_servers: [filesystem]  # Read files only

    # Admin model: full access
    - name: admin
      mcp_servers: [filesystem, database, api]

    # Chat model: no tool access
    - name: chat
      mcp_servers: []
```

### Environment Variables

Never hardcode secrets:

```yaml
mcp:
  servers:
    - name: api
      transport: http
      base_url: ${env:API_URL}
      headers:
        Authorization: Bearer ${env:API_TOKEN}
```

### Path Restrictions

Limit file system access to specific directories:

```yaml
mcp:
  servers:
    - name: docs
      transport: stdio
      command: npx
      args: ['-y', '@modelcontextprotocol/server-filesystem', '/safe/docs/only']
```

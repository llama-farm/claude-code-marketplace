# MCP Access Control

Control which models can access which tools for security and capability management.

## Overview

LlamaFarm allows you to assign specific MCP servers to specific models:
- Restrict powerful tools to trusted models
- Create role-based tool access
- Isolate sensitive operations
- Follow least-privilege principles

## Basic Configuration

### Assign Servers to Model

```yaml
mcp:
  servers:
    - name: filesystem
      transport: stdio
      command: npx
      args: ['-y', '@modelcontextprotocol/server-filesystem', '/data']

    - name: database
      transport: http
      base_url: http://localhost:8080/mcp

runtime:
  models:
    - name: assistant
      provider: universal
      model: unsloth/Qwen3-4B-GGUF:Q4_K_M
      mcp_servers:
        - filesystem
        - database
```

### Access Control Rules

| Configuration | Behavior |
|---------------|----------|
| `mcp_servers: [server1, server2]` | Model can only use listed servers |
| `mcp_servers: []` | Model has no MCP access |
| `mcp_servers` omitted | Model can use ALL configured servers |

---

## Access Patterns

### Role-Based Access

```yaml
mcp:
  servers:
    - name: filesystem
      transport: stdio
      command: npx
      args: ['-y', '@modelcontextprotocol/server-filesystem', '/data']

    - name: database
      transport: http
      base_url: http://localhost:8080/mcp

    - name: admin-tools
      transport: http
      base_url: http://localhost:8081/mcp

runtime:
  models:
    # Research assistant - read-only file access
    - name: researcher
      provider: universal
      model: unsloth/Qwen3-4B-GGUF:Q4_K_M
      mcp_servers:
        - filesystem  # Can read files

    # Data analyst - database access only
    - name: analyst
      provider: universal
      model: unsloth/Qwen3-4B-GGUF:Q4_K_M
      mcp_servers:
        - database  # Can query data

    # Admin assistant - full access
    - name: admin
      provider: openai
      model: gpt-4
      mcp_servers:
        - filesystem
        - database
        - admin-tools  # All tools

    # General chat - no tool access (safer for public)
    - name: chat
      provider: universal
      model: unsloth/Qwen3-4B-GGUF:Q4_K_M
      mcp_servers: []  # No MCP tools
```

### Read-Only vs Read-Write

```yaml
mcp:
  servers:
    # Read-only filesystem access
    - name: fs-readonly
      transport: stdio
      command: npx
      args:
        - '-y'
        - '@modelcontextprotocol/server-filesystem'
        - '--read-only'
        - '/docs'

    # Full filesystem access
    - name: fs-full
      transport: stdio
      command: npx
      args:
        - '-y'
        - '@modelcontextprotocol/server-filesystem'
        - '/workspace'

runtime:
  models:
    # Can only read documents
    - name: reader
      mcp_servers: [fs-readonly]

    # Can read and write
    - name: editor
      mcp_servers: [fs-full]
```

### Environment-Specific Access

```yaml
mcp:
  servers:
    - name: dev-database
      transport: http
      base_url: http://localhost:5432/mcp

    - name: prod-database
      transport: http
      base_url: https://prod-db.company.com/mcp
      headers:
        Authorization: Bearer ${env:PROD_DB_TOKEN}

runtime:
  models:
    # Development model - dev database only
    - name: dev-assistant
      provider: universal
      model: unsloth/Qwen3-4B-GGUF:Q4_K_M
      mcp_servers:
        - dev-database

    # Production model - prod database (restricted)
    - name: prod-assistant
      provider: openai
      model: gpt-4
      mcp_servers:
        - prod-database
```

---

## Security Best Practices

### 1. Principle of Least Privilege

Only grant the minimum access needed:

```yaml
runtime:
  models:
    # BAD: All tools for simple chat
    - name: chat
      mcp_servers: [filesystem, database, api, admin]

    # GOOD: No tools for simple chat
    - name: chat
      mcp_servers: []
```

### 2. Separate Read and Write

```yaml
mcp:
  servers:
    - name: read-data
      # Tools: list_files, read_file, search
    - name: write-data
      # Tools: create_file, delete_file, update

runtime:
  models:
    - name: analyst
      mcp_servers: [read-data]  # Read only

    - name: editor
      mcp_servers: [read-data, write-data]  # Full access
```

### 3. Restrict Path Access

```yaml
mcp:
  servers:
    # Only allow access to specific directory
    - name: docs
      transport: stdio
      command: npx
      args:
        - '-y'
        - '@modelcontextprotocol/server-filesystem'
        - '/safe/docs/only'  # Restricted path
```

### 4. Use Environment Variables for Secrets

```yaml
mcp:
  servers:
    - name: api
      transport: http
      base_url: ${env:API_URL}
      headers:
        Authorization: Bearer ${env:API_TOKEN}
```

Never hardcode tokens:
```yaml
# BAD
headers:
  Authorization: Bearer sk-abc123...

# GOOD
headers:
  Authorization: Bearer ${env:API_TOKEN}
```

### 5. Audit Tool Usage

Log which tools are called:

```yaml
mcp:
  servers:
    - name: audited-tools
      transport: http
      base_url: http://localhost:8080/mcp
      headers:
        X-Audit-User: ${env:USER}
        X-Audit-Session: ${session:id}
```

---

## Multi-Model Tool Routing

### By Capability

```yaml
runtime:
  models:
    # Fast model for simple queries
    - name: fast
      provider: universal
      model: unsloth/Qwen3-1.7B-GGUF:Q4_K_M
      mcp_servers: []  # No tools needed

    # Smart model for analysis
    - name: smart
      provider: universal
      model: unsloth/Qwen3-4B-GGUF:Q4_K_M
      mcp_servers:
        - filesystem
        - database

    # Expert model for complex tasks
    - name: expert
      provider: openai
      model: gpt-4
      mcp_servers:
        - filesystem
        - database
        - api
        - admin-tools
```

### By Trust Level

```yaml
runtime:
  models:
    # Untrusted: User-facing, no tools
    - name: public-chat
      mcp_servers: []

    # Trusted: Internal use, limited tools
    - name: internal-assistant
      mcp_servers: [filesystem, database]

    # Highly trusted: Admin access
    - name: admin-assistant
      mcp_servers: [filesystem, database, admin, api]
```

---

## Combining Inline Tools and MCP

Access control applies to both:

```yaml
mcp:
  servers:
    - name: filesystem
      transport: stdio
      command: npx
      args: ['-y', '@modelcontextprotocol/server-filesystem', '/data']

runtime:
  models:
    - name: assistant
      mcp_servers:
        - filesystem  # MCP tools
      tools:
        # Inline tools always available to this model
        - type: function
          name: format_data
          description: Format data for output
          parameters:
            type: object
            properties:
              data:
                type: string
```

---

## Debugging Access

### Check Available Tools

When a model starts, LlamaFarm logs available tools:

```
MCP tools loaded [agents.chat_orchestrator] tool_count=5 tool_names=['read_file', 'write_file', ...]
```

### Verify Configuration

```bash
# Check which servers are configured
lf config show | grep -A 20 "mcp:"

# Test model's tools
lf chat --model assistant "What tools do you have?"
```

### Common Issues

| Issue | Cause | Solution |
|-------|-------|----------|
| "Tool not found" | Server not in mcp_servers | Add server to model's list |
| "Access denied" | Wrong permissions | Check server configuration |
| "No tools available" | Empty mcp_servers | Add servers or remove `[]` |

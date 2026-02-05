# Inline Tool Definitions

Define tools directly in your LlamaFarm configuration without running an MCP server.

## Overview

Inline tools are defined in `llamafarm.yaml` under a model's `tools` array. They're useful for:
- Simple function interfaces
- CLI command exposure
- Project-specific tools
- Quick prototyping

## Basic Syntax

```yaml
runtime:
  models:
    - name: assistant
      provider: universal
      model: unsloth/Qwen3-4B-GGUF:Q4_K_M
      tool_call_strategy: native_api
      tools:
        - type: function
          name: tool_name
          description: What this tool does
          parameters:
            type: object
            required:
              - param1
            properties:
              param1:
                type: string
                description: Description of param1
```

## Tool Schema

Each tool follows JSON Schema format:

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `type` | string | Yes | Must be `"function"` |
| `name` | string | Yes | Unique identifier |
| `description` | string | Yes | Shown to model |
| `parameters` | object | Yes | JSON Schema for inputs |

## Parameter Types

### String

```yaml
parameters:
  type: object
  properties:
    text:
      type: string
      description: Text input
```

### Integer

```yaml
parameters:
  type: object
  properties:
    count:
      type: integer
      description: Number of items
      minimum: 1
      maximum: 100
```

### Number (Decimal)

```yaml
parameters:
  type: object
  properties:
    temperature:
      type: number
      description: Temperature value
      default: 0.7
```

### Boolean

```yaml
parameters:
  type: object
  properties:
    verbose:
      type: boolean
      description: Enable verbose output
      default: false
```

### Array

```yaml
parameters:
  type: object
  properties:
    tags:
      type: array
      items:
        type: string
      description: List of tags
```

### Enum (Choices)

```yaml
parameters:
  type: object
  properties:
    priority:
      type: string
      enum: [low, medium, high]
      description: Priority level
```

### Nested Object

```yaml
parameters:
  type: object
  properties:
    config:
      type: object
      properties:
        enabled:
          type: boolean
        threshold:
          type: number
```

---

## Examples

### Dataset Upload Tool

```yaml
runtime:
  models:
    - name: assistant
      tools:
        - type: function
          name: cli.dataset_upload
          description: Upload a file to a LlamaFarm dataset
          parameters:
            type: object
            required:
              - filepath
              - dataset
            properties:
              filepath:
                type: string
                description: Path to the file to upload
              dataset:
                type: string
                description: Name of the target dataset
              namespace:
                type: string
                description: Project namespace
                default: default
              project:
                type: string
                description: Project name
                default: my-project
```

### Search Tool

```yaml
tools:
  - type: function
    name: search_documents
    description: Search through indexed documents
    parameters:
      type: object
      required:
        - query
      properties:
        query:
          type: string
          description: Search query text
        top_k:
          type: integer
          description: Maximum results to return
          default: 10
          minimum: 1
          maximum: 50
        database:
          type: string
          description: Database to search
          default: main_db
```

### Notification Tool

```yaml
tools:
  - type: function
    name: send_notification
    description: Send a notification to the user
    parameters:
      type: object
      required:
        - message
      properties:
        message:
          type: string
          description: Notification message
        channel:
          type: string
          enum: [email, slack, sms]
          description: Notification channel
          default: slack
        urgent:
          type: boolean
          description: Mark as urgent
          default: false
```

### Data Analysis Tool

```yaml
tools:
  - type: function
    name: analyze_data
    description: Perform statistical analysis on data
    parameters:
      type: object
      required:
        - data_source
        - analysis_type
      properties:
        data_source:
          type: string
          description: Path to data file or database table
        analysis_type:
          type: string
          enum: [summary, correlation, regression, clustering]
          description: Type of analysis to perform
        options:
          type: object
          properties:
            columns:
              type: array
              items:
                type: string
              description: Specific columns to analyze
            output_format:
              type: string
              enum: [json, csv, markdown]
              default: json
```

---

## Combining MCP and Inline Tools

Use both together - they're merged at runtime:

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
        - filesystem
      tools:
        # Custom tools alongside MCP tools
        - type: function
          name: format_report
          description: Format data into a report
          parameters:
            type: object
            required:
              - data
              - template
            properties:
              data:
                type: string
                description: JSON data to format
              template:
                type: string
                enum: [summary, detailed, executive]
```

---

## Tool Execution

### How It Works

1. Model receives tool definitions
2. Model decides to call a tool
3. Returns tool call with arguments
4. LlamaFarm checks if tool is registered
5. For inline tools: passed to client for execution
6. For MCP tools: executed via MCP protocol
7. Result returned to model

### Tool Call Response

```json
{
  "role": "assistant",
  "tool_calls": [{
    "id": "call_abc123",
    "type": "function",
    "function": {
      "name": "search_documents",
      "arguments": "{\"query\": \"neural networks\", \"top_k\": 5}"
    }
  }]
}
```

### Tool Result

```json
{
  "role": "tool",
  "tool_call_id": "call_abc123",
  "content": "[{\"title\": \"Neural Networks Guide\", \"score\": 0.92}]"
}
```

---

## Best Practices

### 1. Clear Descriptions

```yaml
# Good
description: Search documents in the vector database and return the most relevant chunks

# Bad
description: Search
```

### 2. Required vs Optional

```yaml
parameters:
  type: object
  required:
    - query  # Must be provided
  properties:
    query:
      type: string
    limit:
      type: integer
      default: 10  # Optional with default
```

### 3. Validation Constraints

```yaml
properties:
  count:
    type: integer
    minimum: 1
    maximum: 1000
  email:
    type: string
    pattern: "^[a-zA-Z0-9._%+-]+@[a-zA-Z0-9.-]+\\.[a-zA-Z]{2,}$"
```

### 4. Consistent Naming

```yaml
# Use dot notation for namespacing
tools:
  - name: dataset.upload
  - name: dataset.process
  - name: rag.query
  - name: rag.health
```

### 5. Document Defaults

```yaml
properties:
  format:
    type: string
    enum: [json, csv, markdown]
    description: Output format (default: json)
    default: json
```

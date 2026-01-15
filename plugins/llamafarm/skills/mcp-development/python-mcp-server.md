# Building Python MCP Servers

Create custom MCP servers to expose your business logic as AI tools.

## Quick Start

### 1. Install the MCP SDK

```bash
pip install mcp
```

### 2. Create a Simple Server

```python
# my_mcp_server.py
from mcp.server import Server
from mcp.server.stdio import stdio_server

app = Server("my-tools")

@app.tool()
async def greet(name: str) -> str:
    """Greet a person by name.

    Args:
        name: The person's name to greet
    """
    return f"Hello, {name}!"

@app.tool()
async def calculate(operation: str, a: float, b: float) -> float:
    """Perform a mathematical operation.

    Args:
        operation: The operation (add, subtract, multiply, divide)
        a: First number
        b: Second number
    """
    ops = {
        "add": lambda x, y: x + y,
        "subtract": lambda x, y: x - y,
        "multiply": lambda x, y: x * y,
        "divide": lambda x, y: x / y if y != 0 else float('inf')
    }
    return ops.get(operation, lambda x, y: 0)(a, b)

async def main():
    async with stdio_server() as (read_stream, write_stream):
        await app.run(read_stream, write_stream)

if __name__ == "__main__":
    import asyncio
    asyncio.run(main())
```

### 3. Configure in LlamaFarm

```yaml
mcp:
  servers:
    - name: my-tools
      transport: stdio
      command: python
      args: ['-m', 'my_mcp_server']

runtime:
  models:
    - name: assistant
      mcp_servers: [my-tools]
```

### 4. Test It

```bash
lf chat "Greet Alice and calculate 5 + 3"
```

---

## Tool Definition

### Basic Tool

```python
@app.tool()
async def my_tool(param1: str, param2: int = 10) -> str:
    """Tool description shown to the model.

    Args:
        param1: Description of param1
        param2: Description of param2 (optional, default 10)
    """
    return f"Result: {param1}, {param2}"
```

### Type Hints

MCP uses type hints to generate the tool schema:

```python
from typing import List, Optional, Dict

@app.tool()
async def process_items(
    items: List[str],
    config: Optional[Dict[str, str]] = None
) -> Dict[str, int]:
    """Process a list of items.

    Args:
        items: List of items to process
        config: Optional configuration dictionary
    """
    return {"processed": len(items)}
```

### Complex Parameters

```python
from pydantic import BaseModel

class SearchQuery(BaseModel):
    query: str
    filters: Optional[Dict[str, str]] = None
    limit: int = 10

@app.tool()
async def search(params: SearchQuery) -> List[Dict]:
    """Search with complex parameters."""
    # Implementation
    return [{"result": "found"}]
```

---

## Real-World Examples

### Database Tool

```python
import sqlite3
from mcp.server import Server
from mcp.server.stdio import stdio_server

app = Server("db-tools")

DB_PATH = "./data.db"

@app.tool()
async def query_database(sql: str) -> list:
    """Execute a read-only SQL query.

    Args:
        sql: The SQL SELECT query to execute
    """
    if not sql.strip().upper().startswith("SELECT"):
        return {"error": "Only SELECT queries allowed"}

    conn = sqlite3.connect(DB_PATH)
    conn.row_factory = sqlite3.Row
    cursor = conn.cursor()

    try:
        cursor.execute(sql)
        rows = [dict(row) for row in cursor.fetchall()]
        return rows
    except Exception as e:
        return {"error": str(e)}
    finally:
        conn.close()

@app.tool()
async def list_tables() -> list:
    """List all tables in the database."""
    conn = sqlite3.connect(DB_PATH)
    cursor = conn.cursor()
    cursor.execute("SELECT name FROM sqlite_master WHERE type='table'")
    return [row[0] for row in cursor.fetchall()]
```

### API Integration

```python
import httpx
from mcp.server import Server
from mcp.server.stdio import stdio_server

app = Server("api-tools")

API_BASE = "https://api.example.com"
API_KEY = os.environ.get("API_KEY")

@app.tool()
async def get_user(user_id: str) -> dict:
    """Fetch user details from the API.

    Args:
        user_id: The user's unique identifier
    """
    async with httpx.AsyncClient() as client:
        response = await client.get(
            f"{API_BASE}/users/{user_id}",
            headers={"Authorization": f"Bearer {API_KEY}"}
        )
        return response.json()

@app.tool()
async def create_ticket(title: str, description: str, priority: str = "medium") -> dict:
    """Create a support ticket.

    Args:
        title: Ticket title
        description: Detailed description
        priority: Priority level (low, medium, high)
    """
    async with httpx.AsyncClient() as client:
        response = await client.post(
            f"{API_BASE}/tickets",
            headers={"Authorization": f"Bearer {API_KEY}"},
            json={
                "title": title,
                "description": description,
                "priority": priority
            }
        )
        return response.json()
```

### File Processing

```python
import os
from pathlib import Path
from mcp.server import Server
from mcp.server.stdio import stdio_server

app = Server("file-tools")

ALLOWED_DIR = Path("/safe/directory")

def validate_path(path: str) -> Path:
    """Ensure path is within allowed directory."""
    full_path = (ALLOWED_DIR / path).resolve()
    if not str(full_path).startswith(str(ALLOWED_DIR)):
        raise ValueError("Path outside allowed directory")
    return full_path

@app.tool()
async def read_file(path: str) -> str:
    """Read contents of a file.

    Args:
        path: Relative path to the file
    """
    file_path = validate_path(path)
    return file_path.read_text()

@app.tool()
async def list_files(directory: str = ".") -> list:
    """List files in a directory.

    Args:
        directory: Relative path to directory
    """
    dir_path = validate_path(directory)
    return [f.name for f in dir_path.iterdir()]

@app.tool()
async def search_files(pattern: str, directory: str = ".") -> list:
    """Search for files matching a pattern.

    Args:
        pattern: Glob pattern (e.g., "*.py")
        directory: Directory to search in
    """
    dir_path = validate_path(directory)
    return [str(f.relative_to(dir_path)) for f in dir_path.glob(pattern)]
```

---

## Error Handling

```python
@app.tool()
async def safe_operation(input: str) -> dict:
    """Perform an operation with error handling."""
    try:
        result = perform_operation(input)
        return {"status": "success", "result": result}
    except ValueError as e:
        return {"status": "error", "message": f"Invalid input: {e}"}
    except Exception as e:
        return {"status": "error", "message": f"Operation failed: {e}"}
```

---

## Async Operations

```python
import asyncio

@app.tool()
async def parallel_fetch(urls: list) -> list:
    """Fetch multiple URLs in parallel.

    Args:
        urls: List of URLs to fetch
    """
    async with httpx.AsyncClient() as client:
        tasks = [client.get(url) for url in urls]
        responses = await asyncio.gather(*tasks, return_exceptions=True)
        return [
            {"url": url, "status": r.status_code if hasattr(r, 'status_code') else str(r)}
            for url, r in zip(urls, responses)
        ]
```

---

## Testing Your Server

### Manual Testing

```bash
# Run the server directly
python -m my_mcp_server

# In another terminal, send a test request
echo '{"jsonrpc": "2.0", "method": "tools/list", "id": 1}' | python -m my_mcp_server
```

### Integration Testing

```python
import pytest
from my_mcp_server import greet, calculate

@pytest.mark.asyncio
async def test_greet():
    result = await greet("Alice")
    assert result == "Hello, Alice!"

@pytest.mark.asyncio
async def test_calculate():
    result = await calculate("add", 5, 3)
    assert result == 8
```

---

## Packaging

### Project Structure

```
my_mcp_tools/
├── pyproject.toml
├── src/
│   └── my_mcp_tools/
│       ├── __init__.py
│       ├── __main__.py
│       └── server.py
└── tests/
    └── test_server.py
```

### pyproject.toml

```toml
[project]
name = "my-mcp-tools"
version = "0.1.0"
dependencies = ["mcp>=0.1.0"]

[project.scripts]
my-mcp-tools = "my_mcp_tools:main"
```

### __main__.py

```python
from .server import main
import asyncio

if __name__ == "__main__":
    asyncio.run(main())
```

---

## Best Practices

1. **Clear Docstrings** - Models use docstrings to understand tools
2. **Type Hints** - Enable automatic schema generation
3. **Error Handling** - Return structured errors, don't raise
4. **Validation** - Validate inputs, especially file paths
5. **Async** - Use async for I/O operations
6. **Security** - Never trust model-provided inputs blindly

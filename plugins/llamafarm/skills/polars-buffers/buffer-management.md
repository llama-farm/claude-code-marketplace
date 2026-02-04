# Buffer Management

## Creating Buffers

### Basic Creation

```bash
curl -X POST http://localhost:14345/v1/ml/buffers \
  -H "Content-Type: application/json" \
  -d '{
    "name": "server_metrics",
    "max_size": 10000,
    "columns": ["cpu", "memory", "disk_io", "network"],
    "rolling_windows": [10, 50, 200],
    "lag_periods": [1, 5, 10]
  }'
```

**Response:**
```json
{
  "name": "server_metrics",
  "max_size": 10000,
  "columns": ["cpu", "memory", "disk_io", "network"],
  "rolling_windows": [10, 50, 200],
  "lag_periods": [1, 5, 10],
  "current_size": 0,
  "created_at": "2025-03-15T08:30:00Z"
}
```

### Creation Parameters

| Parameter | Type | Required | Default | Description |
|-----------|------|----------|---------|-------------|
| `name` | string | yes | -- | Unique buffer identifier |
| `max_size` | int | no | 10000 | Maximum rows before oldest are dropped |
| `columns` | array | yes | -- | Column names for the buffer schema |
| `rolling_windows` | array | no | [] | Window sizes for rolling statistics |
| `lag_periods` | array | no | [] | Periods for lag features |

### Minimal Creation

Only `name` and `columns` are required. Defaults apply for everything else:

```bash
curl -X POST http://localhost:14345/v1/ml/buffers \
  -H "Content-Type: application/json" \
  -d '{
    "name": "simple_buffer",
    "columns": ["value"]
  }'
```

---

## Appending Data

### Single Row Append

```bash
curl -X POST http://localhost:14345/v1/ml/buffers/server_metrics/append \
  -H "Content-Type: application/json" \
  -d '{
    "data": [
      {"cpu": 45.2, "memory": 62.1, "disk_io": 120, "network": 500}
    ]
  }'
```

**Response:**
```json
{
  "rows_appended": 1,
  "current_size": 1,
  "rows_dropped": 0
}
```

### Batch Append

Send multiple rows in a single request for better throughput:

```bash
curl -X POST http://localhost:14345/v1/ml/buffers/server_metrics/append \
  -H "Content-Type: application/json" \
  -d '{
    "data": [
      {"cpu": 45.2, "memory": 62.1, "disk_io": 120, "network": 500},
      {"cpu": 47.8, "memory": 63.0, "disk_io": 115, "network": 520},
      {"cpu": 46.1, "memory": 62.5, "disk_io": 118, "network": 510}
    ]
  }'
```

**Response:**
```json
{
  "rows_appended": 3,
  "current_size": 4,
  "rows_dropped": 0
}
```

### Single vs Batch Performance

| Method | Rows | Latency | Throughput |
|--------|------|---------|------------|
| Single append | 1 | < 1ms | ~1K rows/sec |
| Batch 100 | 100 | < 2ms | ~50K rows/sec |
| Batch 1K | 1000 | < 5ms | ~200K rows/sec |
| Batch 10K | 10000 | < 20ms | ~500K rows/sec |

**Guidance:** Use single appends for real-time streaming where each row arrives individually. Use batch appends when you have multiple rows available at once (periodic collection, log ingestion, bulk loads).

---

## Reading Buffer Contents

### Read Raw Data

```bash
curl http://localhost:14345/v1/ml/buffers/server_metrics/data
```

**Response:**
```json
{
  "name": "server_metrics",
  "current_size": 150,
  "columns": ["cpu", "memory", "disk_io", "network"],
  "data": [
    {"cpu": 45.2, "memory": 62.1, "disk_io": 120, "network": 500},
    {"cpu": 47.8, "memory": 63.0, "disk_io": 115, "network": 520}
  ]
}
```

### Pagination

For large buffers, use `offset` and `limit` query parameters:

```bash
# Get rows 100-199
curl "http://localhost:14345/v1/ml/buffers/server_metrics/data?offset=100&limit=100"
```

### Read Latest Rows

Use the `last` query parameter to read the most recent rows:

```bash
# Get the 10 most recent rows
curl "http://localhost:14345/v1/ml/buffers/server_metrics/data?last=10"
```

**Response:**
```json
{
  "name": "server_metrics",
  "current_size": 5000,
  "returned_rows": 10,
  "offset": 4990,
  "data": [...]
}
```

---

## Sliding Window Behavior

Buffers operate as fixed-capacity sliding windows. When `max_size` is reached, the oldest rows are dropped to make room for new data.

### Example Progression

```
max_size = 5

Append [A, B, C]:       [A, B, C]           size=3
Append [D, E]:           [A, B, C, D, E]     size=5 (full)
Append [F]:              [B, C, D, E, F]     size=5 (A dropped)
Append [G, H]:           [D, E, F, G, H]     size=5 (B, C dropped)
```

The append response includes `rows_dropped` so you can track evictions:

```json
{
  "rows_appended": 2,
  "current_size": 5,
  "rows_dropped": 2
}
```

### Choosing max_size

| Use Case | Recommended max_size | Reasoning |
|----------|---------------------|-----------|
| Real-time dashboard | 500-2000 | Recent data only |
| Hourly metrics (1/sec) | 3600 | One hour of history |
| Daily metrics (1/min) | 1440 | One day of history |
| Training data export | 50000-100000 | Larger window for ML |
| Memory-constrained | 1000-5000 | Limit footprint |

---

## Memory Management

### Estimating Buffer Memory

Each buffer stores a Polars DataFrame in memory. Approximate memory usage:

```
memory_bytes ~ max_size * num_columns * 8 bytes (float64)
```

| max_size | Columns | Approximate Memory |
|----------|---------|-------------------|
| 1,000 | 4 | ~32 KB |
| 10,000 | 4 | ~320 KB |
| 10,000 | 20 | ~1.6 MB |
| 100,000 | 4 | ~3.2 MB |
| 100,000 | 20 | ~16 MB |

### max_size Guidance

- Start with 10,000 for general use
- Use smaller values (1,000-5,000) if running many buffers concurrently
- Use larger values (50,000-100,000) only when you need deep history for ML training
- Monitor total memory if running more than 50 concurrent buffers

### Monitoring Buffer Memory

List all buffers with their metadata to assess total memory usage:

```bash
curl http://localhost:14345/v1/ml/buffers
```

**Response:**
```json
{
  "buffers": [
    {
      "name": "server_metrics",
      "max_size": 10000,
      "current_size": 8542,
      "columns": ["cpu", "memory", "disk_io", "network"],
      "rolling_windows": [10, 50, 200],
      "lag_periods": [1, 5, 10],
      "memory_bytes": 273344,
      "created_at": "2025-03-15T08:30:00Z"
    },
    {
      "name": "request_latency",
      "max_size": 5000,
      "current_size": 5000,
      "columns": ["p50", "p95", "p99"],
      "rolling_windows": [20, 100],
      "lag_periods": [1],
      "memory_bytes": 120000,
      "created_at": "2025-03-15T09:00:00Z"
    }
  ],
  "total_buffers": 2,
  "total_memory_bytes": 393344
}
```

---

## Deleting Buffers

### Delete a Single Buffer

```bash
curl -X DELETE http://localhost:14345/v1/ml/buffers/server_metrics
```

**Response:**
```json
{
  "deleted": "server_metrics",
  "memory_freed_bytes": 273344
}
```

### Confirming Deletion

After deletion, attempting to access the buffer returns a 404:

```bash
curl http://localhost:14345/v1/ml/buffers/server_metrics/data
```

```json
{
  "error": {
    "code": "not_found",
    "message": "Buffer 'server_metrics' not found"
  }
}
```

---

## Listing All Buffers

```bash
curl http://localhost:14345/v1/ml/buffers
```

Returns all active buffers with metadata including current size, column schema, configured windows, lag periods, memory usage, and creation timestamp. See the response format under [Monitoring Buffer Memory](#monitoring-buffer-memory) above.

---

## Error Handling

### Buffer Not Found

Returned when accessing a buffer that does not exist (HTTP 404):

```json
{
  "error": {
    "code": "not_found",
    "message": "Buffer 'unknown_buffer' not found"
  }
}
```

### Duplicate Buffer Name

Returned when creating a buffer with a name that already exists (HTTP 409):

```json
{
  "error": {
    "code": "conflict",
    "message": "Buffer 'server_metrics' already exists"
  }
}
```

### Schema Mismatch on Append

Returned when appended data does not match the buffer's column schema (HTTP 400):

```json
{
  "error": {
    "code": "invalid_request",
    "message": "Schema mismatch: buffer expects columns ['cpu', 'memory', 'disk_io', 'network'], got ['cpu', 'memory', 'latency']"
  }
}
```

**Common causes:**
- Missing columns in the appended data
- Extra columns not in the buffer schema
- Misspelled column names

### Invalid Data Types

Returned when column values cannot be cast to float64 (HTTP 400):

```json
{
  "error": {
    "code": "invalid_request",
    "message": "Column 'cpu' contains non-numeric value: 'high'"
  }
}
```

### Empty Data

Returned when the `data` array is empty (HTTP 400):

```json
{
  "error": {
    "code": "invalid_request",
    "message": "Data array must contain at least one row"
  }
}
```

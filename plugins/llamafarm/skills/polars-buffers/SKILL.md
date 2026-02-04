---
description: Polars-based feature engineering buffers. Create sliding windows, compute rolling statistics, and generate lag features for ML pipelines.
user-invocable: false
---

# Polars Buffers Skill

Guide for using Polars-based feature engineering buffers in LlamaFarm.

## When to Load

Load this skill when the user:
- Asks about feature engineering
- Wants to compute rolling statistics
- Needs sliding window calculations
- Asks about lag features
- Wants to pre-process data for anomaly detection
- Needs time-series feature generation

## API Overview

| Endpoint | Method | Description |
|----------|--------|-------------|
| `/v1/ml/buffers` | GET | List all active buffers |
| `/v1/ml/buffers` | POST | Create a new buffer |
| `/v1/ml/buffers/{name}/append` | POST | Append data to a buffer |
| `/v1/ml/buffers/{name}/features` | GET | Compute and retrieve features |
| `/v1/ml/buffers/{name}/data` | GET | Read raw buffer contents |
| `/v1/ml/buffers/{name}` | DELETE | Delete a buffer |

## Feature Types

| Feature | Description | Example |
|---------|-------------|---------|
| Rolling Mean | Average over a window | 5-min avg CPU usage |
| Rolling Std | Standard deviation over window | Volatility measure |
| Rolling Min | Minimum in window | Baseline detection |
| Rolling Max | Maximum in window | Peak detection |
| Lag | Value from N periods ago | Previous hour's value |
| Delta | Change from N periods ago | Rate of change |

## Performance Characteristics

| Operation | Typical Latency | Notes |
|-----------|----------------|-------|
| Append (single) | < 1ms | Lock-free for single writer |
| Append (batch 1K) | < 5ms | Batch preferred for throughput |
| Feature compute | 1-10ms | Depends on window count |
| Buffer read | 1-5ms | Depends on buffer size |

## Use Cases

1. **Pre-process for anomaly detection** -- Compute rolling stats, feed to streaming detector
2. **External ML pipeline** -- Generate features, export for training
3. **Time series monitoring** -- Track trends, detect regime changes

## Quick Start

### Create a buffer

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

### Append data

```bash
curl -X POST http://localhost:14345/v1/ml/buffers/server_metrics/append \
  -H "Content-Type: application/json" \
  -d '{
    "data": [
      {"cpu": 45.2, "memory": 62.1, "disk_io": 120, "network": 500},
      {"cpu": 47.8, "memory": 63.0, "disk_io": 115, "network": 520}
    ]
  }'
```

### Get computed features

```bash
curl http://localhost:14345/v1/ml/buffers/server_metrics/features
```

## Progressive Disclosure

For detailed guidance:
- `buffer-management.md` - Creating, appending, reading, memory management, deletion
- `feature-engineering.md` - Rolling stats, window selection, lag design, ML integration

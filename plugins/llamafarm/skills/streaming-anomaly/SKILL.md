---
description: Real-time streaming anomaly detection. Monitor data streams with automatic model updates, cold-start handling, and configurable retraining.
---

# Streaming Anomaly Detection Skill

Guide for real-time anomaly detection on data streams in LlamaFarm.

## When to Load

Load this skill when the user:
- Wants real-time or streaming anomaly detection
- Asks about monitoring data streams
- Needs continuous anomaly detection
- Asks about cold start or warm-up periods
- Wants to configure streaming detectors
- Asks about online/incremental learning

## API Overview

| Endpoint | Method | Description |
|----------|--------|-------------|
| `/v1/ml/anomaly/stream` | POST | Send data point to streaming detector |
| `/v1/ml/anomaly/detectors` | GET | List active streaming detectors |
| `/v1/ml/anomaly/detectors/{name}/status` | GET | Get detector health and stats |
| `/v1/ml/anomaly/detectors/{name}/reset` | POST | Reset a detector to initial state |
| `/v1/ml/anomaly/detectors/{name}` | DELETE | Remove a streaming detector |

## Streaming vs Batch

| Factor | Streaming | Batch |
|--------|-----------|-------|
| Latency | Milliseconds per point | Seconds to minutes per batch |
| Model updates | Automatic rolling retrain | Manual retrain required |
| Cold start | Needs warm-up period | Full dataset available |
| Memory | Fixed window | Full dataset in memory |
| Best for | Monitoring, alerting | Analysis, reporting |

**Use streaming when:** Data arrives continuously, you need immediate alerts, or data volume is too large for batch.

**Use batch when:** You have a complete dataset, need highest accuracy, or are doing exploratory analysis.

## Recommended Streaming Backends

| Backend | Speed | Memory | Cold Start |
|---------|-------|--------|------------|
| `ecod` | Very Fast | Low | Quick |
| `hbos` | Very Fast | Low | Quick |
| `loda` | Very Fast | Very Low | Quick |

## Cold-Start Lifecycle

```
1. COLLECTING  →  Not enough data, no detection
                   (accumulating min_samples)
2. TRAINING    →  First model being trained
3. DETECTING   →  Active detection, periodic retrain
4. RETRAINING  →  Background model update (tick-tock)
```

## Key Parameters

| Parameter | Default | Description |
|-----------|---------|-------------|
| `backend` | `ecod` | Detection algorithm |
| `min_samples` | `100` | Samples before first detection |
| `window_size` | `1000` | Rolling window size |
| `retrain_interval` | `500` | Points between retrains |
| `contamination` | `0.05` | Expected anomaly fraction |
| `rolling_windows` | `[]` | Rolling feature windows |
| `lag_periods` | `[]` | Lag feature periods |

## Quick Start

### Create a streaming detector

```bash
curl -X POST http://localhost:14345/v1/ml/anomaly/stream \
  -H "Content-Type: application/json" \
  -d '{
    "detector_name": "server_monitor",
    "backend": "ecod",
    "min_samples": 50,
    "window_size": 500,
    "retrain_interval": 250,
    "contamination": 0.05,
    "data": [{"cpu": 45.2, "memory": 62.1, "requests": 150}]
  }'
```

### Check detector status

```bash
curl http://localhost:14345/v1/ml/anomaly/detectors/server_monitor/status
```

### Send more data

```bash
curl -X POST http://localhost:14345/v1/ml/anomaly/stream \
  -H "Content-Type: application/json" \
  -d '{
    "detector_name": "server_monitor",
    "data": [{"cpu": 99.8, "memory": 95.0, "requests": 5000}]
  }'
```

## Progressive Disclosure

Load these files for detailed guidance:

- **cold-start.md** - Managing the cold-start period and seeding with history
- **configuration.md** - Full parameter reference, window and lag features, tick-tock retraining
- **monitoring.md** - Health checks, drift detection, reset strategies, alerting

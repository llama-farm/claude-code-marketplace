---
description: Check health and status of all ML services and models
---

# /llamafarm:ml-status - ML Services Health Check

Check the health and status of all LlamaFarm ML components — runtime, anomaly models, classifiers, streaming detectors, and data buffers.

## Usage

```
/llamafarm:ml-status                # Full ML status report
/llamafarm:ml-status --component anomaly    # Check anomaly models only
/llamafarm:ml-status --component classify   # Check classifiers only
/llamafarm:ml-status --component buffers    # Check Polars buffers only
```

## What This Command Does

1. **Checks runtime health** — verifies the Universal Runtime is responding
2. **Lists anomaly models** — trained batch models and their metadata
3. **Lists streaming detectors** — active streaming anomaly detectors
4. **Lists classifiers** — trained text classification models
5. **Lists data buffers** — active Polars buffers and their sizes
6. **Presents formatted report** — consolidated view of all ML resources

## Implementation

When the user runs `/llamafarm:ml-status`, follow these steps:

### Step 1: Check Universal Runtime Health

```bash
curl -s http://localhost:14345/health | jq .
```

Expected healthy response:
```json
{
  "status": "healthy",
  "components": {
    "server": "healthy",
    "celery": "healthy",
    "database": "healthy"
  },
  "version": "0.x.x"
}
```

If the health check fails, report the runtime as unavailable and suggest running `/llamafarm:start --runtime`.

### Step 2: List Anomaly Models

```bash
curl -s http://localhost:14345/v1/ml/anomaly/models | jq .
```

### Step 3: List Streaming Detectors

```bash
curl -s http://localhost:14345/v1/ml/anomaly/detectors | jq .
```

### Step 4: List Classifiers

```bash
curl -s http://localhost:14345/v1/classifier/models | jq .
```

### Step 5: List Polars Buffers

```bash
curl -s http://localhost:14345/v1/ml/buffers | jq .
```

### Step 6: Present Comprehensive Status

```
LlamaFarm ML Status Report
===========================

Runtime: HEALTHY (http://localhost:14345)

Anomaly Models (batch):
  Name              Backend     Version  Trained
  sensor_model      iforest     3        2024-01-14 10:30
  fraud_detector    autoencoder 1        2024-01-13 15:22

Streaming Detectors:
  Name              Backend   Window   Samples Seen
  live_sensor       ecod      1000     4,521
  network_monitor   hbos      500      12,034

Classifiers:
  Name              Base Model              Labels  Accuracy
  support_tickets   distilbert-base-uncased 5       92.3%
  doc_classifier    roberta-base            3       96.1%

Polars Buffers:
  Name              Rows     Columns  Memory
  sensor_data       10,000   12       1.2 MB
  transactions      50,000   8        3.8 MB

Summary:
  Anomaly Models:      2
  Streaming Detectors: 2
  Classifiers:         2
  Polars Buffers:      2
  Overall:             All systems operational
```

## Component Details

### Runtime Health

| Status | Meaning |
|--------|---------|
| `healthy` | Runtime responding, all ML endpoints available |
| `degraded` | Runtime running but some backends unavailable |
| `failed` | Runtime not responding |

### Model States

| State | Meaning |
|-------|---------|
| `ready` | Model loaded and ready for inference |
| `training` | Model currently being trained |
| `error` | Model failed to load or train |

## Troubleshooting Output

When issues are detected:

```
LlamaFarm ML Status Report
===========================

Runtime: DEGRADED

Issues Detected:

1. Universal Runtime Not Responding
   - Health endpoint returned connection refused
   - Fix: lf runtime start or /llamafarm:start --runtime

2. Anomaly Model Load Error
   - Model "sensor_model" failed to load: missing backend dependency
   - Fix: Install required backend: pip install pyod

Recommendations:
- Run /llamafarm:logs --service runtime to see error details
- Run /llamafarm:start --runtime to restart the ML runtime
```

## Related Commands

- `/llamafarm:status` - Check overall service health
- `/llamafarm:anomaly` - Anomaly detection workflows
- `/llamafarm:classify` - Text classification workflows
- `/llamafarm:ocr` - OCR text extraction

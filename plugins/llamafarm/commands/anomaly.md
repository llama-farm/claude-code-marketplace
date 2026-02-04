---
description: Detect anomalies in data using batch or streaming ML models
---

# /llamafarm:anomaly - Anomaly Detection

Guide users through anomaly detection workflows — training models, detecting outliers, and monitoring streams.

## Usage

```
/llamafarm:anomaly fit              # Train a new anomaly model
/llamafarm:anomaly detect           # Detect anomalies with a trained model
/llamafarm:anomaly stream           # Set up streaming anomaly detection
/llamafarm:anomaly backends         # List available backends
/llamafarm:anomaly models           # List trained models
```

## What This Command Does

1. **Identifies the workflow** — fit, detect, stream, or informational
2. **Gathers requirements** — data source, backend preference, parameters
3. **Recommends configuration** — backend, contamination, normalization
4. **Executes the workflow** — trains model or runs detection
5. **Presents results** — scores, labels, interpretation

## Implementation

When the user runs `/llamafarm:anomaly`, follow these steps:

### Subcommand: `fit`

#### Step 1: Gather Requirements

Ask the user:
1. Where is your data? (file path, API endpoint, or inline)
2. What type of data? (numeric, mixed, time series)
3. How much data? (rows and columns)

#### Step 2: Recommend Backend

Use this decision tree:
- General tabular data → `iforest`
- Need speed → `ecod`
- High-dimensional → `ecod` or `pca`
- Complex patterns → `autoencoder`

#### Step 3: Set Contamination

| Data Quality | Recommended |
|-------------|-------------|
| Clean data, few anomalies | 0.01 - 0.05 |
| Moderate noise | 0.05 - 0.10 |
| Noisy / unknown | 0.10 - 0.15 |

#### Step 4: Execute Fit

```bash
curl -X POST http://localhost:14345/v1/ml/anomaly/fit \
  -H "Content-Type: application/json" \
  -d '{
    "data": <user_data>,
    "backend": "<recommended>",
    "contamination": <recommended>,
    "normalization": "standardize",
    "model_name": "<user_chosen_name>"
  }'
```

#### Step 5: Report Results

Present:
- Model name and version
- Backend used
- Training data shape
- Next steps: run detection or save model

### Subcommand: `detect`

#### Step 1: List Available Models

```bash
curl http://localhost:14345/v1/ml/anomaly/models
```

#### Step 2: Score Data

```bash
curl -X POST http://localhost:14345/v1/ml/anomaly/detect \
  -H "Content-Type: application/json" \
  -d '{
    "model_name": "<selected_model>",
    "data": <new_data>
  }'
```

#### Step 3: Present Results

Show:
- Number of anomalies found
- Anomaly indices and scores
- Score distribution summary
- Interpretation guidance

### Subcommand: `stream`

#### Step 1: Recommend Backend

For streaming, recommend: `ecod` (default), `hbos`, or `loda`

#### Step 2: Configure Detector

Ask about:
- Expected data rate
- Acceptable detection delay
- Data dimensions

#### Step 3: Create Detector

```bash
curl -X POST http://localhost:14345/v1/ml/anomaly/stream \
  -H "Content-Type: application/json" \
  -d '{
    "detector_name": "<name>",
    "backend": "ecod",
    "min_samples": 100,
    "window_size": 1000,
    "retrain_interval": 500,
    "contamination": 0.05,
    "data": <initial_data>
  }'
```

### Subcommand: `backends`

```bash
curl http://localhost:14345/v1/ml/anomaly/backends
```

Present the backend quick reference table from the anomaly-detection skill.

### Subcommand: `models`

```bash
curl http://localhost:14345/v1/ml/anomaly/models
```

Present models with name, backend, version, training date.

## Error Handling

| Error | Cause | Fix |
|-------|-------|-----|
| `422 Unprocessable Entity` | Invalid data format | Check data is numeric array |
| `404 Not Found` | Model doesn't exist | Run `/llamafarm:anomaly models` to list |
| `503 Service Unavailable` | ML runtime not running | Run `/llamafarm:start` |
| `400 Bad Request` | Invalid backend name | Run `/llamafarm:anomaly backends` |

## Skills to Load

- `anomaly-detection` - Detailed backend reference and tuning
- `streaming-anomaly` - Streaming configuration and monitoring
- `polars-buffers` - Feature engineering before detection

## Related Commands

- `/llamafarm:ml-status` - Check ML service health
- `/llamafarm:status` - Check overall service health
- `/llamafarm:classify` - Text classification

# Configuration Reference

Full parameter reference and advanced configuration for streaming anomaly detectors.

## Parameter Reference

| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| `detector_name` | string | *required* | Unique name for the detector instance |
| `backend` | string | `"ecod"` | Detection algorithm (`ecod`, `hbos`, `loda`, `iforest`, `knn`, `lof`) |
| `min_samples` | integer | `100` | Data points required before first model training |
| `window_size` | integer | `1000` | Maximum points in the rolling window |
| `retrain_interval` | integer | `500` | New points between automatic retrains |
| `contamination` | float | `0.05` | Expected fraction of anomalies (0.0 to 0.5) |
| `rolling_windows` | array[int] | `[]` | Window sizes for rolling statistics features |
| `lag_periods` | array[int] | `[]` | Lag periods for autoregressive features |
| `feature_columns` | array[string] | `null` | Columns to use (null = all numeric columns) |
| `score_threshold` | float | `null` | Custom score threshold (overrides contamination-based threshold) |

## Backend Selection

| Backend | Algorithm | Best For | Limitations |
|---------|-----------|----------|-------------|
| `ecod` | Empirical CDF | General purpose, fast | Assumes feature independence |
| `hbos` | Histogram-based | High-dimensional data | Assumes feature independence |
| `loda` | Lightweight online | Very high throughput | Lower accuracy on complex patterns |
| `iforest` | Isolation Forest | Mixed feature types | Slower retrain, higher memory |
| `knn` | K-Nearest Neighbors | Small datasets, clusters | Slow on large windows |
| `lof` | Local Outlier Factor | Density-based clusters | Slow on large windows, sensitive to k |

## Rolling Windows and Lag Periods

Rolling windows and lag periods generate derived features that help the detector capture temporal patterns. They combine multiplicatively with your base features.

### How features are generated

Given base features `[cpu, memory]` with `rolling_windows: [5, 10]` and `lag_periods: [1, 3]`:

```
Base features (2):
  cpu, memory

Rolling window features (2 base x 2 windows x 3 stats = 12):
  cpu_mean_5, cpu_std_5, cpu_max_5
  cpu_mean_10, cpu_std_10, cpu_max_10
  memory_mean_5, memory_std_5, memory_max_5
  memory_mean_10, memory_std_10, memory_max_10

Lag features (2 base x 2 lags = 4):
  cpu_lag_1, cpu_lag_3
  memory_lag_1, memory_lag_3

Total features: 2 + 12 + 4 = 18
```

Rolling statistics computed per window: `mean`, `std`, `max`.

### Feature count formula

```
total_features = base_features
               + (base_features x len(rolling_windows) x 3)
               + (base_features x len(lag_periods))
```

### Choosing window sizes

| Pattern to Capture | Rolling Windows | Lag Periods |
|-------------------|-----------------|-------------|
| Short-term spikes | `[3, 5]` | `[1]` |
| Gradual drift | `[10, 30, 60]` | `[1, 5]` |
| Periodic patterns | `[cycle_length]` | `[1, cycle_length]` |
| Multi-scale | `[5, 15, 60]` | `[1, 5, 15]` |

**Warning:** Large rolling windows and many lag periods multiply the feature count quickly. With 10 base features, `rolling_windows: [5, 15, 60]` and `lag_periods: [1, 5, 15]` produces 10 + 90 + 30 = 130 features. Increase `min_samples` accordingly (at least 10x feature count).

## Polars Buffer Integration

Streaming detectors use Polars DataFrames internally as the rolling buffer. When sending data, each JSON object in the `data` array becomes a row in the buffer.

### Buffer behavior

- The buffer holds at most `window_size` rows
- When the buffer is full, oldest rows are evicted (FIFO)
- Rolling window and lag features are computed directly on the Polars buffer
- All numeric columns are cast to `Float64` automatically
- Non-numeric columns are ignored during detection but preserved in the buffer

### Sending structured data

```bash
curl -X POST http://localhost:14345/v1/ml/anomaly/stream \
  -H "Content-Type: application/json" \
  -d '{
    "detector_name": "metrics_monitor",
    "data": [
      {"timestamp": "2025-01-15T10:00:00Z", "cpu": 45.2, "memory": 62.1, "host": "web-01"},
      {"timestamp": "2025-01-15T10:01:00Z", "cpu": 47.8, "memory": 63.5, "host": "web-01"}
    ]
  }'
```

In this example, `timestamp` and `host` are non-numeric and ignored for detection. Only `cpu` and `memory` are used as base features.

### Selecting specific columns

Use `feature_columns` to control which columns are used:

```bash
curl -X POST http://localhost:14345/v1/ml/anomaly/stream \
  -H "Content-Type: application/json" \
  -d '{
    "detector_name": "metrics_monitor",
    "feature_columns": ["cpu", "memory"],
    "data": [
      {"cpu": 45.2, "memory": 62.1, "disk_io": 1200, "network": 500}
    ]
  }'
```

Only `cpu` and `memory` will be used. `disk_io` and `network` are buffered but excluded from detection.

## Window Size vs Retrain Interval

These two parameters interact to control model freshness and stability.

| | Small `window_size` | Large `window_size` |
|--|---------------------|---------------------|
| **Small `retrain_interval`** | Very adaptive, may oscillate | Adaptive with stability |
| **Large `retrain_interval`** | Stale model, wastes memory | Stable, slow to adapt |

### Tradeoffs

**window_size** controls how much history the model trains on:
- **Larger** (2000-10000): More stable baselines, better at filtering noise, slower to adapt to legitimate distribution shifts
- **Smaller** (200-500): Adapts quickly to changes, more susceptible to noise, lower memory usage

**retrain_interval** controls how often the model is rebuilt:
- **Larger** (1000-5000): Less CPU overhead, more stable scores, slower to incorporate new patterns
- **Smaller** (100-250): Faster adaptation, more CPU usage, scores may fluctuate between retrains

### Recommended combinations

| Use Case | window_size | retrain_interval | Reasoning |
|----------|-------------|------------------|-----------|
| Server monitoring (1-min intervals) | 1440 | 360 | 24h window, retrain every 6h |
| IoT sensor (1-sec intervals) | 3600 | 900 | 1h window, retrain every 15min |
| Financial ticks (sub-second) | 10000 | 2000 | Large window for market patterns |
| Batch job metrics (hourly) | 168 | 24 | 1-week window, retrain daily |

## Tick-Tock Retraining

Streaming detectors use a tick-tock (double-buffer) strategy for zero-downtime model updates.

### How it works

```
Slot A (active): Serving predictions
Slot B (standby): Idle

  ↓ retrain_interval reached

Slot A (active): Still serving predictions
Slot B (standby): Training new model on current window

  ↓ training complete

Slot A (standby): Idle (previous model discarded)
Slot B (active): Now serving predictions

  ↓ retrain_interval reached again

Slot A (standby): Training new model
Slot B (active): Still serving predictions

  ... (alternates)
```

**Key properties:**
- No prediction gaps during retraining
- No lock contention between prediction and training
- Training always uses the latest rolling window snapshot
- If training fails, the active slot continues serving with the previous model

### Monitoring retrain cycles

```bash
curl http://localhost:14345/v1/ml/anomaly/detectors/server_monitor/status
```

```json
{
  "detector_name": "server_monitor",
  "state": "detecting",
  "active_slot": "B",
  "retrain_count": 14,
  "last_retrain": "2025-01-15T14:30:00Z",
  "last_retrain_duration_ms": 120,
  "next_retrain_at_sample": 7500
}
```

## Advanced Configuration Examples

### IoT Sensor Monitoring

High-frequency data from multiple sensors with gradual drift detection:

```bash
curl -X POST http://localhost:14345/v1/ml/anomaly/stream \
  -H "Content-Type: application/json" \
  -d '{
    "detector_name": "iot_sensors",
    "backend": "ecod",
    "min_samples": 300,
    "window_size": 3600,
    "retrain_interval": 900,
    "contamination": 0.02,
    "rolling_windows": [10, 60, 300],
    "lag_periods": [1, 10],
    "data": [
      {"temperature": 22.5, "humidity": 45.0, "pressure": 1013.2, "vibration": 0.03}
    ]
  }'
```

### Financial Transaction Monitoring

Low contamination, large window for capturing market patterns:

```bash
curl -X POST http://localhost:14345/v1/ml/anomaly/stream \
  -H "Content-Type: application/json" \
  -d '{
    "detector_name": "transaction_monitor",
    "backend": "iforest",
    "min_samples": 500,
    "window_size": 10000,
    "retrain_interval": 2000,
    "contamination": 0.01,
    "rolling_windows": [10, 50, 200],
    "lag_periods": [1, 5, 20],
    "feature_columns": ["amount", "latency_ms", "error_rate"],
    "data": [
      {"amount": 150.00, "latency_ms": 45, "error_rate": 0.001, "merchant_id": "M001"}
    ]
  }'
```

### Web Traffic Monitoring

Moderate window with short-term spike detection:

```bash
curl -X POST http://localhost:14345/v1/ml/anomaly/stream \
  -H "Content-Type: application/json" \
  -d '{
    "detector_name": "web_traffic",
    "backend": "hbos",
    "min_samples": 100,
    "window_size": 1440,
    "retrain_interval": 360,
    "contamination": 0.05,
    "rolling_windows": [5, 15],
    "lag_periods": [1, 5],
    "data": [
      {"requests_per_sec": 1200, "p99_latency_ms": 89, "error_5xx": 2, "active_connections": 340}
    ]
  }'
```

## Configuration Validation

Common mistakes and how to avoid them:

| Mistake | Symptom | Fix |
|---------|---------|-----|
| `min_samples` > `window_size` | Detector never trains | Set `window_size` >= 2x `min_samples` |
| `retrain_interval` > `window_size` | Stale model, wasted buffer | Set `retrain_interval` <= `window_size / 2` |
| Too many rolling windows + lags | Slow training, poor accuracy | Keep total features under 200 |
| `contamination` = 0.0 | No anomalies ever detected | Use at least 0.001 |
| `contamination` > 0.3 | Too many false positives | Keep between 0.01 and 0.1 for most use cases |
| Mismatched feature names | Features silently dropped | Use consistent keys in every `data` payload |

# Feature Engineering

## Rolling Statistics Configuration

When a buffer is created with `rolling_windows`, the feature endpoint computes rolling statistics for every column across every window size.

### Available Rolling Statistics

| Statistic | Feature Name Pattern | Description |
|-----------|---------------------|-------------|
| Mean | `{col}_rolling_mean_{window}` | Average over the window |
| Std | `{col}_rolling_std_{window}` | Standard deviation over the window |
| Min | `{col}_rolling_min_{window}` | Minimum value in the window |
| Max | `{col}_rolling_max_{window}` | Maximum value in the window |

### Example

A buffer with `columns: ["cpu", "memory"]` and `rolling_windows: [10, 50]` produces these features:

```
cpu_rolling_mean_10     memory_rolling_mean_10
cpu_rolling_std_10      memory_rolling_std_10
cpu_rolling_min_10      memory_rolling_min_10
cpu_rolling_max_10      memory_rolling_max_10
cpu_rolling_mean_50     memory_rolling_mean_50
cpu_rolling_std_50      memory_rolling_std_50
cpu_rolling_min_50      memory_rolling_min_50
cpu_rolling_max_50      memory_rolling_max_50
```

Total rolling features = `num_columns * num_windows * 4 stats`
In this example: 2 columns * 2 windows * 4 = 16 features.

### Retrieving Features

```bash
curl http://localhost:14345/v1/ml/buffers/server_metrics/features
```

**Response:**
```json
{
  "name": "server_metrics",
  "row_count": 500,
  "feature_count": 40,
  "features": {
    "cpu_rolling_mean_10": 46.3,
    "cpu_rolling_std_10": 2.1,
    "cpu_rolling_min_10": 42.0,
    "cpu_rolling_max_10": 51.5,
    "cpu_rolling_mean_50": 45.8,
    "cpu_lag_1": 47.2,
    "cpu_lag_5": 44.9,
    "cpu_delta_1": -1.0,
    "memory_rolling_mean_10": 63.2
  }
}
```

Features are computed from the most recent row's perspective. Values are `null` when the buffer does not yet contain enough rows for a given window or lag period.

---

## Window Size Selection

### Short Windows (5-20)

**Purpose:** Capture fast-moving signals and recent trends.

| Window | Use Case |
|--------|----------|
| 5 | Spike detection, rapid oscillation |
| 10 | Short-term trend, noise smoothing |
| 20 | Recent baseline, local averages |

**Best for:** Real-time alerting, detecting sudden anomalies, high-frequency monitoring.

### Medium Windows (50-200)

**Purpose:** Capture trends and sustained shifts.

| Window | Use Case |
|--------|----------|
| 50 | Trend identification, moderate smoothing |
| 100 | Hourly patterns (at 1 sample/sec for ~2 min) |
| 200 | Sustained behavior change detection |

**Best for:** Trend analysis, capacity planning, detecting gradual degradation.

### Long Windows (500-2000)

**Purpose:** Establish baselines and detect regime changes.

| Window | Use Case |
|--------|----------|
| 500 | Long-term baseline |
| 1000 | Historical norm, slow drift detection |
| 2000 | Seasonal baseline (at 1 sample/min for ~33 hrs) |

**Best for:** Baseline comparison, seasonal analysis, long-term drift detection.

### Recommended Combinations

| Scenario | Windows | Reasoning |
|----------|---------|-----------|
| Real-time alerting | [5, 20, 100] | Fast signal + context + baseline |
| Capacity planning | [50, 200, 1000] | Trends at multiple scales |
| Anomaly detection | [10, 50, 200] | Balanced short/medium/long |
| High-frequency trading | [5, 10, 50] | All short for speed |

---

## Lag Feature Design

Lag features capture the value from N rows ago, useful for comparing current state to historical state.

### Configuration

Set `lag_periods` during buffer creation:

```bash
curl -X POST http://localhost:14345/v1/ml/buffers \
  -H "Content-Type: application/json" \
  -d '{
    "name": "cpu_monitor",
    "columns": ["cpu"],
    "lag_periods": [1, 5, 10, 60]
  }'
```

### Feature Names

| Period | Feature Name | Meaning |
|--------|-------------|---------|
| 1 | `cpu_lag_1` | Previous row's CPU value |
| 5 | `cpu_lag_5` | CPU value 5 rows ago |
| 10 | `cpu_lag_10` | CPU value 10 rows ago |
| 60 | `cpu_lag_60` | CPU value 60 rows ago |

### Choosing Lag Periods

| Period Range | Use Case |
|-------------|----------|
| 1-5 | Autocorrelation, short-term memory |
| 10-30 | Medium-term comparison |
| 60-360 | Periodic pattern capture (hourly/daily at known intervals) |

**Guidance:** Choose lag periods that align with your data's sampling rate and expected periodic patterns. If you sample once per second and expect hourly cycles, use `lag_periods: [1, 10, 60, 3600]`.

### Combining Lags with Rolling Stats

Lags and rolling stats complement each other. Rolling stats summarize recent behavior; lags provide point-in-time comparisons:

```bash
curl -X POST http://localhost:14345/v1/ml/buffers \
  -H "Content-Type: application/json" \
  -d '{
    "name": "combined_features",
    "columns": ["cpu", "memory"],
    "rolling_windows": [10, 50],
    "lag_periods": [1, 10]
  }'
```

This produces rolling features (mean, std, min, max for each window) plus lag features for each column and period.

---

## Delta Features

Delta features compute the difference between the current value and the value N rows ago. They are derived from lag features automatically.

### Feature Names

For each lag period, a corresponding delta is produced:

| Lag Period | Delta Feature | Computation |
|-----------|---------------|-------------|
| 1 | `cpu_delta_1` | `cpu_current - cpu_lag_1` |
| 5 | `cpu_delta_5` | `cpu_current - cpu_lag_5` |
| 10 | `cpu_delta_10` | `cpu_current - cpu_lag_10` |

### Interpreting Deltas

| Delta Value | Meaning |
|-------------|---------|
| Positive | Value increased over the period |
| Negative | Value decreased over the period |
| Near zero | Stable over the period |
| Large absolute | Rapid change, potential anomaly |

Delta features are especially useful for rate-of-change detection. A `cpu_delta_1` spike indicates a sudden jump, while a sustained positive `cpu_delta_60` indicates a steady upward trend.

---

## Combining Features for ML

### Feature Matrix Construction

The features endpoint returns the latest computed features as a flat dictionary. To build a feature matrix for ML, poll the endpoint at your desired interval:

```python
import requests
import polars as pl
import time

rows = []
for _ in range(1000):
    resp = requests.get("http://localhost:14345/v1/ml/buffers/server_metrics/features")
    features = resp.json()["features"]
    rows.append(features)
    time.sleep(1)

df = pl.DataFrame(rows)
# df now has 1000 rows, one per sample, with all rolling/lag/delta features as columns
```

### Selecting Features

Not all generated features may be useful. After building a feature matrix, apply standard feature selection:

```python
# Drop features with too many nulls (buffer not full enough for long windows)
df = df.drop_nulls()

# Check variance -- drop near-constant features
for col in df.columns:
    if df[col].std() < 0.001:
        df = df.drop(col)
```

### Feature Count Estimation

| Columns | Windows | Lags | Rolling Features | Lag Features | Delta Features | Total |
|---------|---------|------|-----------------|-------------|---------------|-------|
| 4 | 3 | 3 | 48 | 12 | 12 | 72 |
| 4 | 2 | 2 | 32 | 8 | 8 | 48 |
| 10 | 3 | 3 | 120 | 30 | 30 | 180 |

Formula: `columns * windows * 4` (rolling) + `columns * lags` (lag) + `columns * lags` (delta)

---

## Integration with Anomaly Detection

Polars buffers pair naturally with LlamaFarm's streaming anomaly detection. Compute features in a buffer, then feed them to a detector.

### Pipeline Pattern

```
Raw Data --> Buffer (append) --> Features (compute) --> Anomaly Detector
```

### Example: Streaming CPU Anomaly Detection

```python
import requests
import time

BUFFER_URL = "http://localhost:14345/v1/ml/buffers"
BUFFER_NAME = "cpu_anomaly"

# 1. Create buffer with windows tuned for anomaly detection
requests.post(BUFFER_URL, json={
    "name": BUFFER_NAME,
    "columns": ["cpu"],
    "max_size": 5000,
    "rolling_windows": [10, 50, 200],
    "lag_periods": [1, 5]
})

# 2. Stream data and check features
while True:
    # Append latest metric
    cpu_value = get_current_cpu()  # your data source
    requests.post(f"{BUFFER_URL}/{BUFFER_NAME}/append", json={
        "data": [{"cpu": cpu_value}]
    })

    # Get features
    resp = requests.get(f"{BUFFER_URL}/{BUFFER_NAME}/features")
    features = resp.json()["features"]

    # Simple anomaly check: current value vs rolling mean/std
    mean_50 = features.get("cpu_rolling_mean_50")
    std_50 = features.get("cpu_rolling_std_50")
    if mean_50 is not None and std_50 is not None and std_50 > 0:
        z_score = abs(cpu_value - mean_50) / std_50
        if z_score > 3.0:
            print(f"ANOMALY: cpu={cpu_value}, z_score={z_score:.2f}")

    time.sleep(1)
```

---

## Integration with External ML Tools

### Exporting Feature DataFrames

Retrieve raw buffer data and compute features externally for full control:

```python
import requests
import polars as pl

# Get raw data
resp = requests.get("http://localhost:14345/v1/ml/buffers/server_metrics/data")
raw = resp.json()["data"]
df = pl.DataFrame(raw)

# Compute custom features with Polars
df = df.with_columns([
    pl.col("cpu").rolling_mean(window_size=10).alias("cpu_ma_10"),
    pl.col("cpu").rolling_std(window_size=10).alias("cpu_std_10"),
    pl.col("cpu").shift(1).alias("cpu_prev"),
    (pl.col("cpu") - pl.col("cpu").shift(1)).alias("cpu_change"),
])

# Export for scikit-learn, XGBoost, etc.
X = df.drop_nulls().to_pandas()
```

### Exporting to CSV

```python
resp = requests.get("http://localhost:14345/v1/ml/buffers/server_metrics/data")
df = pl.DataFrame(resp.json()["data"])
df.write_csv("server_metrics_export.csv")
```

### Using with scikit-learn

```python
from sklearn.ensemble import IsolationForest
import requests
import polars as pl

# Collect feature snapshots
resp = requests.get("http://localhost:14345/v1/ml/buffers/server_metrics/features")
# ... collect multiple snapshots into a DataFrame ...

X = feature_df.drop_nulls().to_pandas()
model = IsolationForest(contamination=0.05)
model.fit(X)
predictions = model.predict(X)
```

---

## Full Pipeline Example

End-to-end pipeline: raw metrics to features to anomaly detection.

### 1. Create the Buffer

```bash
curl -X POST http://localhost:14345/v1/ml/buffers \
  -H "Content-Type: application/json" \
  -d '{
    "name": "app_health",
    "max_size": 10000,
    "columns": ["request_latency", "error_rate", "throughput", "memory_usage"],
    "rolling_windows": [10, 50, 200],
    "lag_periods": [1, 5, 30]
  }'
```

### 2. Stream Metrics

```python
import requests
import time

while True:
    metrics = collect_app_metrics()  # your instrumentation
    requests.post(
        "http://localhost:14345/v1/ml/buffers/app_health/append",
        json={"data": [metrics]}
    )
    time.sleep(1)
```

### 3. Compute and Use Features

```python
import requests

resp = requests.get("http://localhost:14345/v1/ml/buffers/app_health/features")
features = resp.json()["features"]

# Features available per column (request_latency example):
# request_latency_rolling_mean_10, request_latency_rolling_std_10
# request_latency_rolling_min_10,  request_latency_rolling_max_10
# request_latency_rolling_mean_50, request_latency_rolling_std_50
# ... (same for window 200)
# request_latency_lag_1, request_latency_lag_5, request_latency_lag_30
# request_latency_delta_1, request_latency_delta_5, request_latency_delta_30
```

### 4. Detect Anomalies

```python
def check_anomalies(features):
    alerts = []

    # Latency spike: short-term mean exceeds long-term mean + 2 std
    short_mean = features.get("request_latency_rolling_mean_10")
    long_mean = features.get("request_latency_rolling_mean_200")
    long_std = features.get("request_latency_rolling_std_200")

    if all(v is not None for v in [short_mean, long_mean, long_std]) and long_std > 0:
        if short_mean > long_mean + 2 * long_std:
            alerts.append(f"Latency spike: short_mean={short_mean:.1f}, baseline={long_mean:.1f}")

    # Error rate increase: delta over 30 periods
    error_delta = features.get("error_rate_delta_30")
    if error_delta is not None and error_delta > 0.05:
        alerts.append(f"Error rate rising: +{error_delta:.3f} over 30 periods")

    # Memory leak: sustained positive delta
    mem_delta_5 = features.get("memory_usage_delta_5")
    mem_delta_30 = features.get("memory_usage_delta_30")
    if mem_delta_5 is not None and mem_delta_30 is not None:
        if mem_delta_5 > 0 and mem_delta_30 > 0:
            alerts.append(f"Possible memory leak: mem growing over 5 and 30 periods")

    return alerts
```

### 5. Clean Up

```bash
curl -X DELETE http://localhost:14345/v1/ml/buffers/app_health
```

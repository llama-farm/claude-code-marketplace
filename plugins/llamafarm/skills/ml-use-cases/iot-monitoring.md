# IoT Monitoring Pipeline

End-to-end pipeline for real-time IoT sensor monitoring using Polars buffers, rolling feature extraction, and streaming anomaly detection.

## Architecture Overview

```
Sensors → Polars Buffer → Rolling Features → Streaming ECOD → Alert Engine
           (raw data)     (mean, std, etc.)   (anomaly score)   (webhook/log)
```

**Why this architecture:**
- Polars buffers provide efficient columnar storage for time-series data
- Rolling features capture temporal patterns (trends, volatility)
- Streaming ECOD detects anomalies without retraining on every observation
- Decoupled stages allow independent scaling and tuning

---

## Step 1: Create a Polars Buffer for Metrics

Create a named buffer to accumulate raw sensor readings.

```bash
curl -X POST http://localhost:14345/v1/ml/polars/buffers \
  -H "Content-Type: application/json" \
  -d '{
    "name": "sensor_metrics",
    "schema": {
      "timestamp": "datetime",
      "sensor_id": "str",
      "temperature": "float",
      "humidity": "float",
      "pressure": "float"
    },
    "max_rows": 10000
  }'
```

**Response:**
```json
{
  "name": "sensor_metrics",
  "status": "created",
  "schema": {
    "timestamp": "datetime",
    "sensor_id": "str",
    "temperature": "float",
    "humidity": "float",
    "pressure": "float"
  },
  "max_rows": 10000,
  "current_rows": 0
}
```

The buffer holds up to `max_rows` in memory. Older rows are evicted when the limit is reached, keeping a sliding window of recent data.

---

## Step 2: Configure Rolling Features

Define rolling windows that compute statistical features over the buffer contents.

```bash
curl -X POST http://localhost:14345/v1/ml/polars/buffers/sensor_metrics/features \
  -H "Content-Type: application/json" \
  -d '{
    "features": [
      {
        "name": "temp_mean_10",
        "column": "temperature",
        "function": "mean",
        "window": 10
      },
      {
        "name": "temp_std_10",
        "column": "temperature",
        "function": "std",
        "window": 10
      },
      {
        "name": "temp_mean_50",
        "column": "temperature",
        "function": "mean",
        "window": 50
      },
      {
        "name": "temp_std_50",
        "column": "temperature",
        "function": "std",
        "window": 50
      },
      {
        "name": "temp_mean_200",
        "column": "temperature",
        "function": "mean",
        "window": 200
      },
      {
        "name": "temp_std_200",
        "column": "temperature",
        "function": "std",
        "window": 200
      },
      {
        "name": "humidity_mean_50",
        "column": "humidity",
        "function": "mean",
        "window": 50
      },
      {
        "name": "pressure_mean_50",
        "column": "pressure",
        "function": "mean",
        "window": 50
      }
    ]
  }'
```

**Window size guidance:**
| Window | Purpose | Example |
|--------|---------|---------|
| 10 | Short-term spikes | Sudden sensor failure |
| 50 | Medium-term trends | Gradual drift |
| 200 | Long-term baseline | Seasonal patterns |

Use multiple windows to capture anomalies at different time scales.

---

## Step 3: Set Up Streaming Anomaly Detector

Create a streaming ECOD (Empirical Cumulative Distribution) detector that scores each observation in real time.

```bash
curl -X POST http://localhost:14345/v1/ml/anomaly/streaming \
  -H "Content-Type: application/json" \
  -d '{
    "name": "sensor_anomaly",
    "algorithm": "ecod",
    "features": [
      "temp_mean_10", "temp_std_10",
      "temp_mean_50", "temp_std_50",
      "temp_mean_200", "temp_std_200",
      "humidity_mean_50", "pressure_mean_50"
    ],
    "contamination": 0.05,
    "warmup_period": 200
  }'
```

**Response:**
```json
{
  "name": "sensor_anomaly",
  "algorithm": "ecod",
  "status": "created",
  "warmup_period": 200,
  "observations_seen": 0,
  "is_warm": false
}
```

**Key parameters:**
- `contamination`: Expected fraction of anomalies (0.05 = 5%). Start here and tune down for fewer false positives.
- `warmup_period`: Number of observations before the detector starts scoring. Set to at least the size of your largest rolling window.
- `algorithm`: ECOD is recommended for IoT because it handles multivariate data well and requires no explicit training.

---

## Step 4: Continuous Monitoring Loop

Feed sensor data through the pipeline: append to buffer, extract features, score for anomalies.

```python
import requests
import time
from datetime import datetime

BASE = "http://localhost:14345/v1/ml"

def monitor_sensors(read_sensor_fn, interval=5):
    """Continuous monitoring loop.

    Args:
        read_sensor_fn: Callable returning dict with temperature,
                        humidity, pressure keys.
        interval: Seconds between readings.
    """
    while True:
        reading = read_sensor_fn()
        reading["timestamp"] = datetime.utcnow().isoformat()
        reading["sensor_id"] = "sensor-01"

        # 1. Append raw reading to buffer
        requests.post(
            f"{BASE}/polars/buffers/sensor_metrics/append",
            json={"rows": [reading]},
        )

        # 2. Get rolling features from buffer
        features_resp = requests.get(
            f"{BASE}/polars/buffers/sensor_metrics/features"
        )
        features = features_resp.json()["features"]

        # 3. Score with streaming anomaly detector
        score_resp = requests.post(
            f"{BASE}/anomaly/streaming/sensor_anomaly/score",
            json={"features": features},
        )
        result = score_resp.json()

        # 4. Alert if anomaly detected
        if result["is_anomaly"]:
            alert(
                sensor_id=reading["sensor_id"],
                score=result["anomaly_score"],
                features=features,
            )

        time.sleep(interval)


def alert(sensor_id, score, features):
    """Handle anomaly alert."""
    print(
        f"ANOMALY sensor={sensor_id} "
        f"score={score:.4f} "
        f"temp_mean_10={features.get('temp_mean_10', 'N/A')}"
    )
    # Send webhook, write to log, trigger incident, etc.
```

---

## Cold Start Handling

Streaming detectors need a warmup period before they produce reliable scores. For IoT systems that cannot afford blind spots during startup, seed the buffer with historical data.

```python
import requests

BASE = "http://localhost:14345/v1/ml"

def seed_from_history(historical_readings):
    """Load historical data into the buffer before starting the
    monitoring loop. Use at least 1 hour of data (or 200+ readings)
    to satisfy the warmup period.

    Args:
        historical_readings: List of dicts with timestamp, sensor_id,
                             temperature, humidity, pressure keys.
    """
    # Batch append for efficiency
    BATCH_SIZE = 100
    for i in range(0, len(historical_readings), BATCH_SIZE):
        batch = historical_readings[i : i + BATCH_SIZE]
        requests.post(
            f"{BASE}/polars/buffers/sensor_metrics/append",
            json={"rows": batch},
        )

    # Run a warmup pass through the detector
    for i in range(0, len(historical_readings), BATCH_SIZE):
        features_resp = requests.get(
            f"{BASE}/polars/buffers/sensor_metrics/features"
        )
        features = features_resp.json()["features"]
        requests.post(
            f"{BASE}/anomaly/streaming/sensor_anomaly/score",
            json={"features": features},
        )

    # Verify warmup complete
    status = requests.get(
        f"{BASE}/anomaly/streaming/sensor_anomaly"
    ).json()
    print(f"Detector warm: {status['is_warm']}")
```

---

## Alerting Patterns

### Webhook Alert

```python
def alert_webhook(sensor_id, score, features):
    requests.post(
        "https://hooks.example.com/alerts",
        json={
            "source": "iot-monitor",
            "sensor_id": sensor_id,
            "anomaly_score": score,
            "features": features,
            "timestamp": datetime.utcnow().isoformat(),
        },
    )
```

### Threshold-Based Severity

```python
def alert_with_severity(sensor_id, score, features):
    if score > 0.95:
        severity = "critical"
    elif score > 0.85:
        severity = "warning"
    else:
        severity = "info"

    print(f"[{severity.upper()}] sensor={sensor_id} score={score:.4f}")
```

### Debounced Alerts

Avoid flooding on sustained anomalies by tracking last alert time per sensor.

```python
from collections import defaultdict

last_alert = defaultdict(float)
COOLDOWN = 300  # seconds

def alert_debounced(sensor_id, score, features):
    now = time.time()
    if now - last_alert[sensor_id] < COOLDOWN:
        return
    last_alert[sensor_id] = now
    alert_webhook(sensor_id, score, features)
```

---

## Scaling Considerations

### Multiple Sensors, Multiple Detectors

Group related sensors and create one detector per group. This keeps feature dimensions manageable and allows per-group contamination tuning.

```python
SENSOR_GROUPS = {
    "hvac": ["temp_indoor", "temp_outdoor", "humidity", "fan_speed"],
    "power": ["voltage", "current", "power_factor"],
    "network": ["latency", "packet_loss", "throughput"],
}

# Create one buffer + detector per group
for group, columns in SENSOR_GROUPS.items():
    requests.post(
        f"{BASE}/polars/buffers",
        json={
            "name": f"{group}_metrics",
            "schema": {col: "float" for col in columns}
            | {"timestamp": "datetime"},
            "max_rows": 10000,
        },
    )
    requests.post(
        f"{BASE}/anomaly/streaming",
        json={
            "name": f"{group}_anomaly",
            "algorithm": "ecod",
            "features": [f"{col}_mean_50" for col in columns],
            "contamination": 0.05,
            "warmup_period": 200,
        },
    )
```

### Buffer Sizing

| Reading Interval | 1 Hour | 8 Hours | 24 Hours |
|-----------------|--------|---------|----------|
| 1s | 3,600 | 28,800 | 86,400 |
| 5s | 720 | 5,760 | 17,280 |
| 30s | 120 | 960 | 2,880 |

Set `max_rows` to hold at least enough data for your largest rolling window plus a safety margin.

---

## Full Working Example

Complete script that sets up and runs an IoT monitoring pipeline.

```python
import requests
import time
import random
from datetime import datetime

BASE = "http://localhost:14345/v1/ml"


def setup():
    """Create buffer, features, and detector."""
    # Buffer
    requests.post(f"{BASE}/polars/buffers", json={
        "name": "sensor_metrics",
        "schema": {
            "timestamp": "datetime",
            "temperature": "float",
            "humidity": "float",
        },
        "max_rows": 5000,
    })

    # Rolling features
    requests.post(
        f"{BASE}/polars/buffers/sensor_metrics/features",
        json={"features": [
            {"name": "temp_mean", "column": "temperature",
             "function": "mean", "window": 50},
            {"name": "temp_std", "column": "temperature",
             "function": "std", "window": 50},
            {"name": "humid_mean", "column": "humidity",
             "function": "mean", "window": 50},
        ]},
    )

    # Streaming detector
    requests.post(f"{BASE}/anomaly/streaming", json={
        "name": "sensor_anomaly",
        "algorithm": "ecod",
        "features": ["temp_mean", "temp_std", "humid_mean"],
        "contamination": 0.05,
        "warmup_period": 100,
    })


def simulate_reading():
    """Generate a simulated sensor reading."""
    return {
        "temperature": 22.0 + random.gauss(0, 0.5),
        "humidity": 45.0 + random.gauss(0, 2.0),
    }


def run():
    """Main monitoring loop."""
    setup()
    print("IoT monitoring started. Press Ctrl+C to stop.")

    while True:
        reading = simulate_reading()
        reading["timestamp"] = datetime.utcnow().isoformat()

        requests.post(
            f"{BASE}/polars/buffers/sensor_metrics/append",
            json={"rows": [reading]},
        )

        features = requests.get(
            f"{BASE}/polars/buffers/sensor_metrics/features"
        ).json()["features"]

        result = requests.post(
            f"{BASE}/anomaly/streaming/sensor_anomaly/score",
            json={"features": features},
        ).json()

        status = "ANOMALY" if result["is_anomaly"] else "OK"
        print(
            f"[{status}] temp={reading['temperature']:.1f} "
            f"humid={reading['humidity']:.1f} "
            f"score={result.get('anomaly_score', 0):.4f}"
        )

        time.sleep(5)


if __name__ == "__main__":
    run()
```

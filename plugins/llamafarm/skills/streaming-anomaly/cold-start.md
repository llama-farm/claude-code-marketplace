# Cold-Start Management

Strategies for handling the cold-start period when a streaming detector has insufficient data.

## Cold-Start Lifecycle

Every streaming detector progresses through these states:

```
COLLECTING → TRAINING → DETECTING → RETRAINING → DETECTING → ...
```

| State | Behavior | Duration |
|-------|----------|----------|
| COLLECTING | Accumulating data, no anomaly scores returned | Until `min_samples` reached |
| TRAINING | First model being fitted on collected data | Seconds (depends on backend) |
| DETECTING | Active detection, anomaly scores returned | Until `retrain_interval` reached |
| RETRAINING | Background model update via tick-tock swap | Milliseconds (zero downtime) |

During COLLECTING, the API response includes `"state": "collecting"` and `"scores": null`. No anomaly detection occurs.

## Collecting State Behavior

When a detector is in the COLLECTING state:

- Data points are buffered in the rolling window
- No anomaly scores are returned in the response
- The `samples_seen` counter increments with each point
- Detection begins automatically once `min_samples` is reached

```bash
# Response during COLLECTING state
curl -X POST http://localhost:14345/v1/ml/anomaly/stream \
  -H "Content-Type: application/json" \
  -d '{
    "detector_name": "server_monitor",
    "data": [{"cpu": 45.2, "memory": 62.1}]
  }'
```

```json
{
  "detector_name": "server_monitor",
  "state": "collecting",
  "samples_seen": 23,
  "min_samples": 100,
  "scores": null,
  "message": "Collecting data (23/100 samples)"
}
```

## Choosing min_samples

The `min_samples` parameter controls how many data points are collected before the first model is trained. Choosing the right value depends on data complexity.

| Data Type | Recommended min_samples | Reasoning |
|-----------|------------------------|-----------|
| Univariate (single metric) | 30-50 | Simple distribution, converges quickly |
| Low-dimensional (2-5 features) | 50-100 | Needs enough to capture correlations |
| Multivariate (5-20 features) | 100-200 | Higher dimensions need more samples |
| Complex patterns (20+ features) | 500+ | Curse of dimensionality, need dense coverage |
| Seasonal data | 1-2 full cycles | Must see complete pattern before detecting |

**Rules of thumb:**
- At minimum, set `min_samples` to 10x the number of features
- For data with daily seasonality arriving every minute, use at least 1440 (one full day)
- When unsure, start with 100 and adjust based on false positive rate

## Seeding with Historical Data

The fastest way to exit the cold-start phase is to bulk-send historical data points. This bootstraps the detector with a trained model immediately.

### Bulk seed request

```bash
curl -X POST http://localhost:14345/v1/ml/anomaly/stream \
  -H "Content-Type: application/json" \
  -d '{
    "detector_name": "server_monitor",
    "backend": "ecod",
    "min_samples": 100,
    "window_size": 1000,
    "retrain_interval": 500,
    "contamination": 0.05,
    "data": [
      {"cpu": 42.1, "memory": 58.3, "requests": 120},
      {"cpu": 45.7, "memory": 61.0, "requests": 135},
      {"cpu": 43.9, "memory": 59.8, "requests": 128},
      "... (100+ historical points)"
    ]
  }'
```

When you send more data points than `min_samples` in a single request, the detector:
1. Buffers all points into the rolling window
2. Trains the initial model immediately
3. Returns anomaly scores for points beyond `min_samples`

### Seeding from a file

```bash
# Prepare historical data as JSON array
cat historical_data.json | curl -X POST http://localhost:14345/v1/ml/anomaly/stream \
  -H "Content-Type: application/json" \
  -d @-
```

### Verify seeding succeeded

```bash
curl http://localhost:14345/v1/ml/anomaly/detectors/server_monitor/status
```

```json
{
  "detector_name": "server_monitor",
  "state": "detecting",
  "samples_seen": 500,
  "model_age": 0,
  "window_fill": 500,
  "window_size": 1000,
  "backend": "ecod",
  "last_retrain": "2025-01-15T10:30:00Z"
}
```

Confirm that `state` is `"detecting"` and `samples_seen` meets or exceeds `min_samples`.

## Handling False Negatives During Ramp-Up

The first model trained on `min_samples` data points may not be fully representative. During the ramp-up period (first 1-3 retrain cycles after cold start), expect:

- **Higher false negative rate** - The model has not seen enough variety to establish tight boundaries
- **Unstable score distributions** - Scores shift as the model retrains on more data
- **Missed seasonal patterns** - If the training window does not cover a full cycle

### Mitigation strategies

**1. Buffer alerts during ramp-up**

Do not page on-call during the first few retrain cycles. Instead, log anomalies for review:

```bash
# Check detector state before acting on alerts
STATUS=$(curl -s http://localhost:14345/v1/ml/anomaly/detectors/server_monitor/status)
SAMPLES=$(echo $STATUS | jq '.samples_seen')
RETRAINS=$(echo $STATUS | jq '.retrain_count')

if [ "$RETRAINS" -lt 3 ]; then
  echo "Ramp-up period: logging anomaly, not alerting"
fi
```

**2. Use conservative contamination during ramp-up**

Start with a higher contamination value (more permissive), then tighten after the model stabilizes:

```bash
# Create with conservative threshold
curl -X POST http://localhost:14345/v1/ml/anomaly/stream \
  -H "Content-Type: application/json" \
  -d '{
    "detector_name": "server_monitor",
    "backend": "ecod",
    "min_samples": 100,
    "contamination": 0.10,
    "data": [...]
  }'
```

After 3-5 retrain cycles, reset the detector with a tighter contamination:

```bash
# Reset and re-seed with tighter threshold
curl -X POST http://localhost:14345/v1/ml/anomaly/detectors/server_monitor/reset \
  -H "Content-Type: application/json" \
  -d '{"contamination": 0.05}'
```

**3. Seed with representative historical data**

The most effective mitigation is seeding with data that covers the full range of normal behavior. Include weekday and weekend data, peak and off-peak hours, and any known seasonal patterns.

## Example Workflow: Seed, Verify, Detect

Complete workflow from cold start to live detection:

```bash
# Step 1: Create detector and seed with historical data
curl -X POST http://localhost:14345/v1/ml/anomaly/stream \
  -H "Content-Type: application/json" \
  -d '{
    "detector_name": "web_traffic",
    "backend": "ecod",
    "min_samples": 200,
    "window_size": 2000,
    "retrain_interval": 500,
    "contamination": 0.05,
    "data": '"$(cat historical_24h.json)"'
  }'

# Step 2: Verify detector is in DETECTING state
curl -s http://localhost:14345/v1/ml/anomaly/detectors/web_traffic/status \
  | jq '{state, samples_seen, model_age}'

# Expected: {"state": "detecting", "samples_seen": 1440, "model_age": 0}

# Step 3: Begin live detection loop
while true; do
  METRICS=$(collect_current_metrics)
  RESULT=$(curl -s -X POST http://localhost:14345/v1/ml/anomaly/stream \
    -H "Content-Type: application/json" \
    -d "{
      \"detector_name\": \"web_traffic\",
      \"data\": [$METRICS]
    }")

  SCORE=$(echo $RESULT | jq '.scores[0]')
  IS_ANOMALY=$(echo $RESULT | jq '.anomalies[0]')

  if [ "$IS_ANOMALY" = "true" ]; then
    echo "ANOMALY detected: score=$SCORE"
    # Trigger alert logic here
  fi

  sleep 60
done
```

## Cold-Start Quick Reference

| Scenario | Recommended Approach |
|----------|---------------------|
| New detector, no history | Set appropriate `min_samples`, wait for COLLECTING to complete |
| New detector, history available | Bulk-seed historical data in first request |
| Detector reset | Re-seed with recent window of data |
| Seasonal data | Seed with at least one full cycle of data |
| High-stakes monitoring | Seed + buffer alerts for first 3 retrain cycles |

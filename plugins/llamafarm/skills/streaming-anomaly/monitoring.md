# Monitoring and Operations

Health checks, drift detection, reset strategies, and alerting integration for streaming detectors.

## Health Check Endpoints

### List all detectors

```bash
curl http://localhost:14345/v1/ml/anomaly/detectors
```

```json
{
  "detectors": [
    {
      "name": "server_monitor",
      "state": "detecting",
      "backend": "ecod",
      "samples_seen": 4523
    },
    {
      "name": "web_traffic",
      "state": "collecting",
      "backend": "hbos",
      "samples_seen": 67
    }
  ]
}
```

### Detector status

```bash
curl http://localhost:14345/v1/ml/anomaly/detectors/server_monitor/status
```

```json
{
  "detector_name": "server_monitor",
  "state": "detecting",
  "backend": "ecod",
  "samples_seen": 4523,
  "window_fill": 1000,
  "window_size": 1000,
  "min_samples": 100,
  "retrain_interval": 500,
  "retrain_count": 8,
  "last_retrain": "2025-01-15T14:30:00Z",
  "last_retrain_duration_ms": 135,
  "next_retrain_at_sample": 4500,
  "active_slot": "A",
  "contamination": 0.05,
  "model_age": 23,
  "drift_score": 0.12,
  "feature_count": 18,
  "anomaly_rate_last_100": 0.04
}
```

### Status field reference

| Field | Type | Description |
|-------|------|-------------|
| `state` | string | Current lifecycle state: `collecting`, `training`, `detecting`, `retraining` |
| `samples_seen` | integer | Total data points received since creation |
| `window_fill` | integer | Current number of points in the rolling window |
| `window_size` | integer | Maximum rolling window capacity |
| `retrain_count` | integer | Number of completed retrain cycles |
| `last_retrain` | string | ISO timestamp of most recent retrain |
| `last_retrain_duration_ms` | integer | Milliseconds taken by last retrain |
| `next_retrain_at_sample` | integer | Sample count that triggers next retrain |
| `active_slot` | string | Current tick-tock slot (`A` or `B`) |
| `model_age` | integer | Points received since last retrain |
| `drift_score` | float | Score distribution drift metric (0.0 = no drift) |
| `feature_count` | integer | Total features including rolling and lag |
| `anomaly_rate_last_100` | float | Fraction of anomalies in last 100 points |

## Interpreting Status Responses

### Healthy detector

```
state: "detecting"
model_age: < retrain_interval
drift_score: < 0.3
anomaly_rate_last_100: close to contamination
window_fill: == window_size (buffer full)
```

A healthy detector is actively detecting, its model is recently trained, drift is low, and the observed anomaly rate roughly matches the configured contamination.

### Detector needing attention

| Indicator | Meaning | Action |
|-----------|---------|--------|
| `drift_score` > 0.5 | Score distribution shifting significantly | Monitor closely, may need reset |
| `drift_score` > 0.8 | Major distribution shift | Reset and re-seed the detector |
| `anomaly_rate_last_100` > 2x `contamination` | Too many anomalies flagged | Check for real incident or raise contamination |
| `anomaly_rate_last_100` = 0.0 | No anomalies detected recently | Check data pipeline, lower contamination |
| `model_age` >> `retrain_interval` | Retrain overdue | Check for training errors in logs |
| `last_retrain_duration_ms` increasing | Training getting slower | Reduce window_size or feature count |

## Drift Detection

Drift occurs when the underlying data distribution changes over time, making the current model less effective.

### Types of drift

| Drift Type | Description | Example |
|------------|-------------|---------|
| Sudden | Abrupt distribution change | Deployment changes server behavior |
| Gradual | Slow shift over time | Growing user base increases baseline traffic |
| Recurring | Periodic distribution changes | Day/night traffic patterns |
| Incremental | Small cumulative changes | Gradual memory leak |

### Monitoring drift with the API

The `drift_score` field in the status response quantifies how much the recent score distribution differs from the training distribution.

```bash
# Poll drift score periodically
DRIFT=$(curl -s http://localhost:14345/v1/ml/anomaly/detectors/server_monitor/status \
  | jq '.drift_score')

echo "Drift score: $DRIFT"
```

### Drift score interpretation

| drift_score | Meaning | Recommendation |
|-------------|---------|----------------|
| 0.0 - 0.2 | Stable, no significant drift | No action needed |
| 0.2 - 0.5 | Mild drift, model still effective | Monitor, allow retrain to adapt |
| 0.5 - 0.8 | Significant drift, model degrading | Reduce retrain_interval, consider reset |
| 0.8 - 1.0 | Severe drift, model unreliable | Reset detector and re-seed |

### Detecting drift patterns

**Rising false positive rate:**

```bash
# Check anomaly rate trend
for i in $(seq 1 10); do
  RATE=$(curl -s http://localhost:14345/v1/ml/anomaly/detectors/server_monitor/status \
    | jq '.anomaly_rate_last_100')
  echo "$(date +%H:%M) anomaly_rate=$RATE"
  sleep 300
done
```

If `anomaly_rate_last_100` trends upward over several checks without a real incident, the model is drifting.

**Score distribution shift:**

Compare scores from recent detection responses. If scores that were previously low (normal) are now clustering near the threshold, the baseline has shifted.

## Resetting a Detector

### When to reset

- Drift score exceeds 0.8 and does not recover after retrain
- Known infrastructure change invalidates the baseline (e.g., major deployment)
- Detector was seeded with unrepresentative data
- Changing backend or contamination threshold

### Reset endpoint

```bash
curl -X POST http://localhost:14345/v1/ml/anomaly/detectors/server_monitor/reset \
  -H "Content-Type: application/json" \
  -d '{}'
```

This clears the rolling window and model, returning the detector to `collecting` state.

### Reset with new parameters

```bash
curl -X POST http://localhost:14345/v1/ml/anomaly/detectors/server_monitor/reset \
  -H "Content-Type: application/json" \
  -d '{
    "contamination": 0.03,
    "window_size": 2000,
    "retrain_interval": 500
  }'
```

Parameters not specified retain their original values.

### Reset and re-seed

```bash
# Step 1: Reset
curl -X POST http://localhost:14345/v1/ml/anomaly/detectors/server_monitor/reset

# Step 2: Re-seed with recent data
curl -X POST http://localhost:14345/v1/ml/anomaly/stream \
  -H "Content-Type: application/json" \
  -d '{
    "detector_name": "server_monitor",
    "data": '"$(cat recent_24h.json)"'
  }'

# Step 3: Verify state
curl -s http://localhost:14345/v1/ml/anomaly/detectors/server_monitor/status \
  | jq '{state, samples_seen, model_age}'
```

### Removing a detector

To permanently remove a detector and free its resources:

```bash
curl -X DELETE http://localhost:14345/v1/ml/anomaly/detectors/server_monitor
```

## Alerting Integration

### Webhook alerting pattern

Push anomaly results to a webhook endpoint after each detection:

```bash
while true; do
  METRICS=$(collect_current_metrics)
  RESULT=$(curl -s -X POST http://localhost:14345/v1/ml/anomaly/stream \
    -H "Content-Type: application/json" \
    -d "{
      \"detector_name\": \"server_monitor\",
      \"data\": [$METRICS]
    }")

  IS_ANOMALY=$(echo $RESULT | jq '.anomalies[0]')

  if [ "$IS_ANOMALY" = "true" ]; then
    SCORE=$(echo $RESULT | jq '.scores[0]')
    curl -X POST https://hooks.example.com/alerts \
      -H "Content-Type: application/json" \
      -d "{
        \"detector\": \"server_monitor\",
        \"score\": $SCORE,
        \"data\": $METRICS,
        \"timestamp\": \"$(date -u +%Y-%m-%dT%H:%M:%SZ)\"
      }"
  fi

  sleep 60
done
```

### Log-based alerting

Write anomaly events to a structured log for consumption by log aggregators (ELK, Datadog, Splunk):

```bash
RESULT=$(curl -s -X POST http://localhost:14345/v1/ml/anomaly/stream \
  -H "Content-Type: application/json" \
  -d '{
    "detector_name": "server_monitor",
    "data": [{"cpu": 99.1, "memory": 94.5}]
  }')

IS_ANOMALY=$(echo $RESULT | jq '.anomalies[0]')
SCORE=$(echo $RESULT | jq '.scores[0]')
STATE=$(echo $RESULT | jq -r '.state')

# Structured JSON log line
echo "{\"level\":\"info\",\"detector\":\"server_monitor\",\"anomaly\":$IS_ANOMALY,\"score\":$SCORE,\"state\":\"$STATE\",\"ts\":\"$(date -u +%Y-%m-%dT%H:%M:%SZ)\"}" \
  >> /var/log/anomaly-detection.json
```

### Threshold-based alerting

Layer severity levels on top of anomaly scores:

```bash
SCORE=$(echo $RESULT | jq '.scores[0]')

# Define severity thresholds
WARNING_THRESHOLD=0.7
CRITICAL_THRESHOLD=0.9

SEVERITY="normal"
if (( $(echo "$SCORE > $CRITICAL_THRESHOLD" | bc -l) )); then
  SEVERITY="critical"
elif (( $(echo "$SCORE > $WARNING_THRESHOLD" | bc -l) )); then
  SEVERITY="warning"
fi

if [ "$SEVERITY" != "normal" ]; then
  echo "[$SEVERITY] Anomaly score: $SCORE"
  # Route to appropriate channel based on severity
fi
```

## Operational Runbook

### Daily checks

```bash
# 1. Verify all detectors are active
curl -s http://localhost:14345/v1/ml/anomaly/detectors \
  | jq '.detectors[] | {name, state, samples_seen}'

# 2. Check for high drift scores
curl -s http://localhost:14345/v1/ml/anomaly/detectors \
  | jq '.detectors[].name' -r \
  | while read NAME; do
      DRIFT=$(curl -s http://localhost:14345/v1/ml/anomaly/detectors/$NAME/status \
        | jq '.drift_score')
      echo "$NAME: drift=$DRIFT"
    done

# 3. Verify anomaly rates are reasonable
curl -s http://localhost:14345/v1/ml/anomaly/detectors \
  | jq '.detectors[].name' -r \
  | while read NAME; do
      RATE=$(curl -s http://localhost:14345/v1/ml/anomaly/detectors/$NAME/status \
        | jq '.anomaly_rate_last_100')
      echo "$NAME: anomaly_rate=$RATE"
    done
```

### Weekly reviews

| Check | What to Look For | Action if Failed |
|-------|-----------------|------------------|
| Retrain duration trend | `last_retrain_duration_ms` increasing over time | Reduce `window_size` or feature count |
| Anomaly rate stability | Rate swinging widely between checks | Increase `window_size` for stability |
| Detector count | Orphaned detectors consuming resources | DELETE unused detectors |
| Drift score trend | Scores trending upward across detectors | Investigate data pipeline changes |
| False positive review | Review flagged anomalies for accuracy | Adjust `contamination` or reset |

### Incident response

When a detector flags a sustained anomaly burst:

```bash
# 1. Check if the anomaly is real
curl -s http://localhost:14345/v1/ml/anomaly/detectors/server_monitor/status \
  | jq '{state, drift_score, anomaly_rate_last_100, model_age}'

# 2. If drift_score is low and anomaly_rate is high → likely real incident
#    Investigate the underlying system

# 3. If drift_score is high → likely model drift, not real anomalies
#    Reset the detector
curl -X POST http://localhost:14345/v1/ml/anomaly/detectors/server_monitor/reset

# 4. After incident resolution, re-seed if needed
curl -X POST http://localhost:14345/v1/ml/anomaly/stream \
  -H "Content-Type: application/json" \
  -d '{
    "detector_name": "server_monitor",
    "data": '"$(cat post_incident_baseline.json)"'
  }'
```

### Capacity planning

| Detector Count | Memory Estimate | CPU Estimate |
|---------------|-----------------|--------------|
| 1-10 | < 100 MB | Negligible |
| 10-50 | 100-500 MB | Low (depends on retrain frequency) |
| 50-200 | 500 MB - 2 GB | Moderate (stagger retrain intervals) |
| 200+ | 2 GB+ | Consider dedicated worker process |

Memory scales primarily with `window_size x feature_count x detector_count`. Plan accordingly when deploying many detectors with large windows.

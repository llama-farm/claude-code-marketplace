# Tuning Anomaly Detection Models

Guide for optimizing contamination rates, thresholds, validation, and A/B testing anomaly detection backends.

---

## Contamination Rate Tuning

The `contamination` parameter tells the model what fraction of the training data is expected to be anomalous. This directly affects detection sensitivity.

### Recommended Ranges by Dataset Type

| Dataset Type | Recommended Range | Reasoning |
|-------------|-------------------|-----------|
| Fraud detection | 0.001 - 0.01 | Fraud is rare, <1% of transactions |
| Network intrusion | 0.01 - 0.05 | Attacks are infrequent but not extremely rare |
| Manufacturing defects | 0.02 - 0.10 | Defect rates vary by process maturity |
| Application errors | 0.01 - 0.05 | Errors should be uncommon in healthy systems |
| Sensor drift | 0.05 - 0.15 | Sensor degradation can be gradual and common |
| User behavior | 0.01 - 0.05 | Unusual behavior is relatively rare |
| Log anomalies | 0.05 - 0.10 | Log noise is common |
| Unknown / exploratory | 0.05 - 0.10 | Start in the middle and adjust |

### Tuning Strategy

1. **Start with domain knowledge.** If you know the anomaly rate, set contamination to match.
2. **If unknown, start at 0.05.** This is a conservative middle ground.
3. **Too many false positives?** Lower contamination (e.g., 0.05 to 0.02).
4. **Missing real anomalies?** Raise contamination (e.g., 0.05 to 0.10).
5. **Iterate with labeled samples** if available (see Validation below).

### Example: Adjusting Contamination

```bash
# Conservative (fewer detections, higher precision)
curl -X POST http://localhost:14345/v1/ml/anomaly/fit \
  -H "Content-Type: application/json" \
  -d '{
    "data": [[1.0, 2.0], [1.1, 2.1], [1.2, 1.9], [10.0, 10.0]],
    "backend": "iforest",
    "contamination": 0.01
  }'

# Aggressive (more detections, higher recall)
curl -X POST http://localhost:14345/v1/ml/anomaly/fit \
  -H "Content-Type: application/json" \
  -d '{
    "data": [[1.0, 2.0], [1.1, 2.1], [1.2, 1.9], [10.0, 10.0]],
    "backend": "iforest",
    "contamination": 0.15
  }'
```

---

## Threshold Adjustment

The `/v1/ml/anomaly/detect` endpoint uses a threshold to convert raw anomaly scores into binary labels (anomaly / normal). The threshold behavior depends on the normalization method.

### Threshold by Normalization Method

| Normalization | Default Threshold | Score Range | Interpretation |
|---------------|-------------------|-------------|----------------|
| `standardize` | 0.0 | Typically -1.0 to 1.0 | Scores > threshold are anomalies |
| `zscore` | 0.0 | Typically -1.0 to 1.0 | Scores > threshold are anomalies |
| `raw` | Backend-dependent | Varies | Scores > threshold are anomalies |

### Custom Threshold

Override the default threshold when detecting anomalies.

```bash
curl -X POST http://localhost:14345/v1/ml/anomaly/detect \
  -H "Content-Type: application/json" \
  -d '{
    "model_name": "http_anomaly",
    "data": [[1.0, 2.0], [50.0, 50.0]],
    "threshold": 0.5
  }'
```

**Response:**

```json
{
  "results": [
    {"index": 0, "label": "normal", "score": -0.12},
    {"index": 1, "label": "anomaly", "score": 0.87}
  ],
  "threshold": 0.5,
  "anomaly_count": 1,
  "total_count": 2
}
```

### Threshold Tuning Approach

1. **Score your data** with `/v1/ml/anomaly/score` to get raw scores.
2. **Examine the score distribution** -- look for a natural gap between normal and anomalous points.
3. **Set the threshold** at the gap, or use a percentile (e.g., 95th percentile for 5% anomaly rate).
4. **Validate** with known anomalies if available.

---

## Validation with Labeled Data

When you have ground-truth labels (known anomalies), use them to measure model quality.

### Score and Compare

```bash
# Step 1: Score labeled data
curl -X POST http://localhost:14345/v1/ml/anomaly/score \
  -H "Content-Type: application/json" \
  -d '{
    "model_name": "http_anomaly",
    "data": [[1.0, 2.0], [1.1, 2.1], [50.0, 50.0], [1.2, 1.9]],
    "labels": [0, 0, 1, 0]
  }'
```

**Response with Metrics:**

```json
{
  "scores": [-0.12, -0.08, 0.87, -0.10],
  "metrics": {
    "precision": 1.0,
    "recall": 1.0,
    "f1_score": 1.0,
    "auc_roc": 1.0,
    "average_precision": 1.0
  }
}
```

### Key Metrics

| Metric | What It Tells You | Good Value |
|--------|-------------------|------------|
| Precision | Of detected anomalies, how many are real? | >0.80 |
| Recall | Of real anomalies, how many did we catch? | >0.80 |
| F1 Score | Harmonic mean of precision and recall | >0.80 |
| AUC-ROC | Overall ranking quality across all thresholds | >0.90 |
| Average Precision | Area under precision-recall curve | >0.70 |

### When to Prioritize Precision vs Recall

| Scenario | Priority | Why |
|----------|----------|-----|
| Security alerts | Recall | Missing an attack is worse than a false alarm |
| Fraud detection | Recall | Missing fraud is costly |
| Production alerts | Precision | Alert fatigue from false positives degrades response |
| Data quality | Balance (F1) | Both false positives and negatives waste effort |

---

## Validation Without Labels

When you have no labeled data, use score distribution analysis to evaluate model quality.

### Score Distribution Analysis

1. **Score all your data** with `/v1/ml/anomaly/score`.
2. **Plot or examine the distribution** of scores.
3. **Look for bimodality** -- a clear separation between a large cluster (normal) and a small cluster (anomalies) is a good sign.
4. **Check the top-scoring points** manually to see if they are genuinely anomalous.

### Example Workflow

```bash
# Score the full dataset
curl -X POST http://localhost:14345/v1/ml/anomaly/score \
  -H "Content-Type: application/json" \
  -d '{
    "model_name": "http_anomaly",
    "data": [
      [100, 0.5, 200],
      [150, 0.6, 201],
      [5000, 15.0, 500],
      [120, 0.4, 200]
    ],
    "return_stats": true
  }'
```

**Response:**

```json
{
  "scores": [-0.12, -0.08, 0.92, -0.15],
  "stats": {
    "mean": 0.1425,
    "std": 0.4876,
    "min": -0.15,
    "max": 0.92,
    "p25": -0.13,
    "p50": -0.10,
    "p75": 0.12,
    "p95": 0.76,
    "p99": 0.89
  }
}
```

### Healthy Score Distribution Signs

- **Clear gap** between p95/p99 and the max score
- **Low standard deviation** relative to the mean (tight normal cluster)
- **Small number of high-scoring points** consistent with expected contamination

### Warning Signs

- **Flat distribution** (no clear separation) -- consider a different backend or more features
- **Many high scores** -- contamination may be set too high, or the model is not learning normal patterns well
- **All scores near zero** -- data may lack enough variation for the backend to find anomalies

---

## A/B Testing Backends

Compare multiple backends on the same data to find the best one for your use case.

### Side-by-Side Comparison

```bash
# Train with iforest
curl -X POST http://localhost:14345/v1/ml/anomaly/fit \
  -H "Content-Type: application/json" \
  -d '{
    "data": [[1.0, 2.0], [1.1, 2.1], [1.2, 1.9], [10.0, 10.0]],
    "backend": "iforest",
    "contamination": 0.1,
    "model_name": "test_iforest"
  }'

# Train with ecod
curl -X POST http://localhost:14345/v1/ml/anomaly/fit \
  -H "Content-Type: application/json" \
  -d '{
    "data": [[1.0, 2.0], [1.1, 2.1], [1.2, 1.9], [10.0, 10.0]],
    "backend": "ecod",
    "contamination": 0.1,
    "model_name": "test_ecod"
  }'

# Score with both and compare
curl -X POST http://localhost:14345/v1/ml/anomaly/score \
  -H "Content-Type: application/json" \
  -d '{
    "model_name": "test_iforest",
    "data": [[1.0, 2.0], [50.0, 50.0]],
    "labels": [0, 1]
  }'

curl -X POST http://localhost:14345/v1/ml/anomaly/score \
  -H "Content-Type: application/json" \
  -d '{
    "model_name": "test_ecod",
    "data": [[1.0, 2.0], [50.0, 50.0]],
    "labels": [0, 1]
  }'
```

### What to Compare

| Metric | How to Compare |
|--------|---------------|
| F1 / AUC-ROC | Higher is better (if labels available) |
| Training time | Check response time headers or log timestamps |
| Score separation | Larger gap between normal and anomaly scores is better |
| Model size | Smaller models load faster and use less memory |
| False positive rate | Lower is better for alert-heavy pipelines |

### Recommended A/B Test Order

1. Start with `iforest` (strong general baseline)
2. Try `ecod` (if speed matters or data is high-dimensional)
3. Try `lof` or `knn` (if cluster structure is present)
4. Try `autoencoder` (if other methods underperform and you have enough data)

---

## Common Mistakes and How to Avoid Them

### 1. Setting contamination too high

**Symptom:** Too many normal points flagged as anomalies.
**Fix:** Lower contamination. Start at 0.01-0.05 for most real-world data.

### 2. Not normalizing data

**Symptom:** Features with large magnitudes dominate detection. Anomalies in small-scale features are missed.
**Fix:** Use `normalization: "standardize"` (the default).

### 3. Training on data that includes many anomalies

**Symptom:** Model learns anomalies as normal, reducing detection quality.
**Fix:** Clean obvious anomalies from training data, or use a low contamination rate so the model treats them as outliers during training.

### 4. Using too few features

**Symptom:** Low separation between normal and anomalous scores.
**Fix:** Add more relevant features. Anomaly detection works better with richer feature sets.

### 5. Using the wrong backend for the data shape

**Symptom:** Poor detection despite good data and reasonable contamination.
**Fix:** Try a different backend. Use the decision tree in SKILL.md, or A/B test multiple backends.

### 6. Never retraining

**Symptom:** Model performance degrades over time as data patterns shift (concept drift).
**Fix:** Retrain periodically (weekly or monthly depending on data velocity). Save new versions and compare metrics before promoting.

### 7. Ignoring score distributions

**Symptom:** Binary labels look wrong, but you do not know why.
**Fix:** Use `/v1/ml/anomaly/score` with `return_stats: true` to examine score distributions before relying on `/detect`.

### 8. Overfitting the autoencoder

**Symptom:** Autoencoder reconstruction error is low for both normal and anomalous data.
**Fix:** Increase `dropout_rate`, reduce `epochs`, or use a smaller bottleneck (fewer hidden neurons in the middle layers).

# Fraud Detection Pipeline

Hybrid real-time and batch fraud detection using streaming pre-screening, daily batch analysis, and ensemble scoring.

## Architecture Overview

```
Transaction
    │
    ├─→ Streaming Pre-Screen (HBOS) ─→ Instant block / flag
    │         fast, catches obvious fraud
    │
    └─→ Per-User Feature Buffer (Polars) ─→ Daily Batch (Isolation Forest)
              accumulates patterns               thorough, catches subtle fraud
                                                        │
                                                        ▼
                                              Ensemble Risk Score
                                          (streaming + batch combined)
```

**Why a hybrid approach:**
- **Streaming only** catches obvious outliers but misses subtle, evolving patterns that require historical context.
- **Batch only** catches subtle patterns but introduces latency, allowing fraud to proceed before detection.
- **Hybrid** combines the speed of streaming with the depth of batch. Streaming blocks obvious fraud instantly, while batch catches sophisticated schemes in the next scoring cycle.

---

## Step 1: Set Up Per-User Feature Buffers

Track each user's transaction history in a Polars buffer. These buffers feed rolling features into both the streaming and batch detectors.

```bash
curl -X POST http://localhost:14345/v1/ml/polars/buffers \
  -H "Content-Type: application/json" \
  -d '{
    "name": "user_transactions",
    "schema": {
      "timestamp": "datetime",
      "user_id": "str",
      "amount": "float",
      "merchant_category": "str",
      "location_lat": "float",
      "location_lon": "float",
      "is_international": "bool",
      "time_since_last_txn": "float"
    },
    "max_rows": 100000
  }'
```

Configure rolling features that capture behavioral patterns per user.

```bash
curl -X POST http://localhost:14345/v1/ml/polars/buffers/user_transactions/features \
  -H "Content-Type: application/json" \
  -d '{
    "features": [
      {
        "name": "amt_mean_10",
        "column": "amount",
        "function": "mean",
        "window": 10
      },
      {
        "name": "amt_std_10",
        "column": "amount",
        "function": "std",
        "window": 10
      },
      {
        "name": "amt_max_10",
        "column": "amount",
        "function": "max",
        "window": 10
      },
      {
        "name": "amt_mean_100",
        "column": "amount",
        "function": "mean",
        "window": 100
      },
      {
        "name": "amt_std_100",
        "column": "amount",
        "function": "std",
        "window": 100
      },
      {
        "name": "txn_frequency_10",
        "column": "time_since_last_txn",
        "function": "mean",
        "window": 10
      },
      {
        "name": "txn_frequency_100",
        "column": "time_since_last_txn",
        "function": "mean",
        "window": 100
      },
      {
        "name": "intl_ratio_50",
        "column": "is_international",
        "function": "mean",
        "window": 50
      }
    ]
  }'
```

---

## Step 2: Configure Streaming Pre-Screen (HBOS)

HBOS (Histogram-Based Outlier Score) is fast and lightweight, making it ideal for real-time pre-screening. It flags transactions that are obviously anomalous based on current feature distributions.

```bash
curl -X POST http://localhost:14345/v1/ml/anomaly/streaming \
  -H "Content-Type: application/json" \
  -d '{
    "name": "fraud_prescreen",
    "algorithm": "hbos",
    "features": [
      "amt_mean_10", "amt_std_10", "amt_max_10",
      "txn_frequency_10", "intl_ratio_50"
    ],
    "contamination": 0.02,
    "warmup_period": 500
  }'
```

**Why HBOS for pre-screening:**
- Sub-millisecond scoring latency
- No complex model fitting needed
- Good at catching extreme outliers (large amounts, impossible velocities)
- Low contamination (0.02) keeps false positives minimal for real-time blocking

---

## Step 3: Daily Batch Analysis (Isolation Forest)

Run a thorough batch analysis once daily using Isolation Forest. This catches subtle patterns that the streaming pre-screen may miss.

```bash
# Train on the full buffer of recent transactions
curl -X POST http://localhost:14345/v1/ml/anomaly/batch/train \
  -H "Content-Type: application/json" \
  -d '{
    "name": "fraud_batch",
    "algorithm": "iforest",
    "features": [
      "amt_mean_10", "amt_std_10", "amt_max_10",
      "amt_mean_100", "amt_std_100",
      "txn_frequency_10", "txn_frequency_100",
      "intl_ratio_50"
    ],
    "contamination": 0.03,
    "n_estimators": 200,
    "source": {
      "type": "buffer",
      "name": "user_transactions"
    }
  }'
```

```bash
# Score all recent transactions
curl -X POST http://localhost:14345/v1/ml/anomaly/batch/score \
  -H "Content-Type: application/json" \
  -d '{
    "name": "fraud_batch",
    "source": {
      "type": "buffer",
      "name": "user_transactions"
    }
  }'
```

**Response:**
```json
{
  "name": "fraud_batch",
  "scored_count": 5432,
  "anomaly_count": 163,
  "results_key": "fraud_batch_20250204"
}
```

**Why Isolation Forest for batch:**
- Handles high-dimensional feature spaces well
- No assumptions about data distribution
- Robust to varying transaction volumes
- `n_estimators: 200` increases accuracy for production use

---

## Step 4: Ensemble Scoring

Combine streaming and batch scores into a single risk score. The ensemble weights can be tuned based on your false-positive tolerance.

```python
import requests

BASE = "http://localhost:14345/v1/ml"

STREAMING_WEIGHT = 0.4
BATCH_WEIGHT = 0.6


def score_transaction(transaction, user_id):
    """Score a single transaction using the hybrid pipeline.

    Returns a risk score between 0 and 1, and a decision.
    """
    # 1. Append transaction to buffer
    requests.post(
        f"{BASE}/polars/buffers/user_transactions/append",
        json={"rows": [transaction]},
    )

    # 2. Get rolling features
    features = requests.get(
        f"{BASE}/polars/buffers/user_transactions/features"
    ).json()["features"]

    # 3. Streaming pre-screen
    streaming_result = requests.post(
        f"{BASE}/anomaly/streaming/fraud_prescreen/score",
        json={"features": features},
    ).json()

    # Instant block on high-confidence streaming anomaly
    if streaming_result["anomaly_score"] > 0.98:
        return {
            "risk_score": streaming_result["anomaly_score"],
            "decision": "block",
            "reason": "streaming_high_confidence",
        }

    # 4. Get latest batch score for this user (if available)
    batch_result = requests.get(
        f"{BASE}/anomaly/batch/fraud_batch/scores/{user_id}"
    ).json()

    batch_score = batch_result.get("anomaly_score", 0.0)
    streaming_score = streaming_result["anomaly_score"]

    # 5. Ensemble: weighted combination
    ensemble_score = (
        STREAMING_WEIGHT * streaming_score
        + BATCH_WEIGHT * batch_score
    )

    # 6. Decision based on ensemble score
    if ensemble_score > 0.85:
        decision = "block"
    elif ensemble_score > 0.60:
        decision = "review"
    else:
        decision = "allow"

    return {
        "risk_score": ensemble_score,
        "decision": decision,
        "streaming_score": streaming_score,
        "batch_score": batch_score,
    }
```

---

## Threshold Tuning

Fraud detection has asymmetric costs: a missed fraud is far more expensive than a false positive that triggers manual review. Tune accordingly.

### Contamination Rate

| Setting | Use Case |
|---------|----------|
| 0.01 | Low fraud rate, conservative (credit cards) |
| 0.03 | Moderate fraud rate (e-commerce) |
| 0.05 | Higher fraud rate or during attacks |

Start with 0.02 for streaming (minimize false blocks) and 0.03 for batch (cast a wider net for review).

### Decision Thresholds

| Ensemble Score | Action | Rationale |
|----------------|--------|-----------|
| > 0.85 | Block | High confidence, auto-decline |
| 0.60 - 0.85 | Review | Flag for manual review |
| < 0.60 | Allow | Normal transaction |

Adjust based on your review capacity. If your team can handle more reviews, lower the review threshold.

---

## Feature Engineering for Fraud

Effective fraud detection depends on well-chosen features. Here are the most impactful categories.

### Amount Deviation

How does this transaction compare to the user's history?

```json
{
  "name": "amt_zscore",
  "column": "amount",
  "function": "zscore",
  "window": 100
}
```

A z-score above 3 means the amount is 3 standard deviations from the user's norm.

### Transaction Frequency

Is the user transacting faster than usual?

```json
{
  "name": "velocity_10",
  "column": "time_since_last_txn",
  "function": "mean",
  "window": 10
}
```

A sudden drop in mean time-between-transactions can indicate card testing.

### Location Changes

Is the user transacting from a new or unusual location?

```json
{
  "name": "location_variance",
  "column": "location_lat",
  "function": "std",
  "window": 20
}
```

High location variance over a short window suggests impossible travel.

### Time-of-Day Patterns

Is the user transacting at unusual hours?

```json
{
  "name": "hour_mean_50",
  "column": "transaction_hour",
  "function": "mean",
  "window": 50
}
```

Transactions outside the user's normal hours are a soft signal.

### International Ratio

Is the user suddenly making more international transactions?

```json
{
  "name": "intl_ratio_20",
  "column": "is_international",
  "function": "mean",
  "window": 20
}
```

A spike in international ratio relative to the long-term average is a common fraud indicator.

---

## Full Working Example

Complete script demonstrating the hybrid fraud detection workflow.

```python
import requests
import time
import random
from datetime import datetime

BASE = "http://localhost:14345/v1/ml"


def setup():
    """Create buffer, features, streaming detector, and batch model."""
    # Transaction buffer
    requests.post(f"{BASE}/polars/buffers", json={
        "name": "user_transactions",
        "schema": {
            "timestamp": "datetime",
            "user_id": "str",
            "amount": "float",
            "is_international": "bool",
            "time_since_last_txn": "float",
        },
        "max_rows": 50000,
    })

    # Rolling features
    requests.post(
        f"{BASE}/polars/buffers/user_transactions/features",
        json={"features": [
            {"name": "amt_mean", "column": "amount",
             "function": "mean", "window": 50},
            {"name": "amt_std", "column": "amount",
             "function": "std", "window": 50},
            {"name": "amt_max", "column": "amount",
             "function": "max", "window": 10},
            {"name": "velocity", "column": "time_since_last_txn",
             "function": "mean", "window": 10},
            {"name": "intl_ratio", "column": "is_international",
             "function": "mean", "window": 50},
        ]},
    )

    # Streaming pre-screen
    requests.post(f"{BASE}/anomaly/streaming", json={
        "name": "fraud_prescreen",
        "algorithm": "hbos",
        "features": ["amt_mean", "amt_std", "amt_max",
                      "velocity", "intl_ratio"],
        "contamination": 0.02,
        "warmup_period": 200,
    })


def simulate_transaction(user_id, fraudulent=False):
    """Generate a simulated transaction."""
    if fraudulent:
        amount = random.uniform(2000, 10000)
        is_intl = random.random() < 0.8
        gap = random.uniform(1, 30)
    else:
        amount = random.uniform(10, 200)
        is_intl = random.random() < 0.05
        gap = random.uniform(60, 7200)

    return {
        "timestamp": datetime.utcnow().isoformat(),
        "user_id": user_id,
        "amount": amount,
        "is_international": is_intl,
        "time_since_last_txn": gap,
    }


def process_transaction(txn):
    """Run a transaction through the streaming pipeline."""
    # Append to buffer
    requests.post(
        f"{BASE}/polars/buffers/user_transactions/append",
        json={"rows": [txn]},
    )

    # Get features
    features = requests.get(
        f"{BASE}/polars/buffers/user_transactions/features"
    ).json()["features"]

    # Score
    result = requests.post(
        f"{BASE}/anomaly/streaming/fraud_prescreen/score",
        json={"features": features},
    ).json()

    score = result.get("anomaly_score", 0)
    if score > 0.85:
        decision = "BLOCK"
    elif score > 0.60:
        decision = "REVIEW"
    else:
        decision = "ALLOW"

    print(
        f"[{decision}] user={txn['user_id']} "
        f"amount=${txn['amount']:.2f} "
        f"intl={txn['is_international']} "
        f"score={score:.4f}"
    )
    return decision


def run_daily_batch():
    """Run daily batch analysis with Isolation Forest."""
    print("\n--- Running daily batch analysis ---")
    requests.post(f"{BASE}/anomaly/batch/train", json={
        "name": "fraud_batch",
        "algorithm": "iforest",
        "features": ["amt_mean", "amt_std", "amt_max",
                      "velocity", "intl_ratio"],
        "contamination": 0.03,
        "n_estimators": 200,
        "source": {"type": "buffer", "name": "user_transactions"},
    })

    result = requests.post(f"{BASE}/anomaly/batch/score", json={
        "name": "fraud_batch",
        "source": {"type": "buffer", "name": "user_transactions"},
    }).json()

    print(f"Batch scored: {result.get('scored_count', 0)} transactions")
    print(f"Anomalies found: {result.get('anomaly_count', 0)}")


def run():
    """Main simulation loop."""
    setup()
    print("Fraud detection pipeline started.\n")

    # Simulate normal traffic with occasional fraud
    for i in range(500):
        is_fraud = random.random() < 0.03
        txn = simulate_transaction("user-001", fraudulent=is_fraud)
        process_transaction(txn)
        time.sleep(0.1)

        # Run batch every 100 transactions (simulating daily)
        if (i + 1) % 100 == 0:
            run_daily_batch()
            print()


if __name__ == "__main__":
    run()
```

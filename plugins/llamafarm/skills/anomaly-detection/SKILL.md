---
description: Batch anomaly detection with 12+ backends. Train models, score data, detect outliers using isolation forests, autoencoders, and more.
---

# Anomaly Detection Skill

Guide for training and using anomaly detection models in LlamaFarm.

## When to Load

Load this skill when the user:
- Wants to detect anomalies or outliers in data
- Asks about anomaly detection backends or algorithms
- Needs to train an anomaly model
- Wants to score data for anomalies
- Asks about contamination rates or thresholds
- Needs to manage anomaly detection models

## API Overview

| Endpoint | Method | Description |
|----------|--------|-------------|
| `/v1/ml/anomaly/fit` | POST | Train a model on data |
| `/v1/ml/anomaly/score` | POST | Score data points (anomaly likelihood) |
| `/v1/ml/anomaly/detect` | POST | Detect anomalies (binary labels + scores) |
| `/v1/ml/anomaly/save` | POST | Save a trained model |
| `/v1/ml/anomaly/load` | POST | Load a previously saved model |
| `/v1/ml/anomaly/models` | GET | List available trained models |
| `/v1/ml/anomaly/backends` | GET | List available detection backends |

## Backend Quick Reference

| Backend | Speed | Accuracy | Best For |
|---------|-------|----------|----------|
| `iforest` | Fast | High | General purpose, tabular data |
| `ecod` | Very Fast | Good | High-dimensional data, streaming |
| `lof` | Medium | High | Cluster-based anomalies |
| `knn` | Medium | High | Local density anomalies |
| `ocsvm` | Medium | High | Well-separated anomalies |
| `copod` | Very Fast | Good | Multivariate data, fast screening |
| `hbos` | Very Fast | Moderate | Histogram-based, fast baseline |
| `loda` | Very Fast | Moderate | Lightweight, streaming-friendly |
| `cblof` | Medium | High | Cluster-based with labels |
| `mcd` | Medium | High | Gaussian-distributed data |
| `pca` | Fast | Good | Dimensionality reduction |
| `autoencoder` | Slow | Very High | Complex patterns, deep learning |

## Decision Tree: Choosing a Backend

1. **Need speed above all?** -> `ecod` or `hbos`
2. **General tabular data?** -> `iforest` (best default)
3. **High-dimensional data?** -> `ecod` or `pca`
4. **Cluster-shaped anomalies?** -> `lof` or `cblof`
5. **Complex non-linear patterns?** -> `autoencoder`
6. **Need interpretability?** -> `ecod` or `copod`
7. **Streaming use case?** -> `ecod`, `hbos`, or `loda`

## Quick Start

### Train a model

```bash
curl -X POST http://localhost:14345/v1/ml/anomaly/fit \
  -H "Content-Type: application/json" \
  -d '{
    "data": [[1.0, 2.0], [1.1, 2.1], [1.2, 1.9], [10.0, 10.0]],
    "backend": "iforest",
    "contamination": 0.1,
    "normalization": "standardize"
  }'
```

### Detect anomalies

```bash
curl -X POST http://localhost:14345/v1/ml/anomaly/detect \
  -H "Content-Type: application/json" \
  -d '{
    "model_name": "my_model",
    "data": [[1.0, 2.0], [50.0, 50.0]]
  }'
```

### List models

```bash
curl http://localhost:14345/v1/ml/anomaly/models
```

## Normalization Methods

| Method | When to Use |
|--------|------------|
| `standardize` | Default -- centers data, works for most cases |
| `zscore` | When you need strict z-score scaling |
| `raw` | When data is already normalized or normalization would distort it |

## Progressive Disclosure

For detailed guidance:
- **backends.md** - Per-backend algorithm details, parameters, and examples
- **data-preparation.md** - Normalization, encoding, handling mixed data types
- **model-lifecycle.md** - Saving, loading, versioning, managing models
- **tuning.md** - Contamination rates, thresholds, validation, A/B testing

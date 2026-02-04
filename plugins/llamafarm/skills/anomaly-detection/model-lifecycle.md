# Model Lifecycle Management

Guide for saving, loading, versioning, and managing anomaly detection models in LlamaFarm.

---

## Model Versioning

LlamaFarm uses timestamp-based versioning for anomaly detection models. Each saved model receives a version identifier based on the save time.

### Version Format

```
{model_name}-{timestamp}
```

Example: `http_anomaly-20240115T083000Z`

### Latest Alias

Every model name automatically maintains a `-latest` alias that points to the most recently saved version. When loading a model without specifying a version, the latest version is used.

```
http_anomaly-latest  ->  http_anomaly-20240115T083000Z
```

---

## Save Workflow

After training a model with `/v1/ml/anomaly/fit`, save it for later use.

### Save a Trained Model

```bash
curl -X POST http://localhost:14345/v1/ml/anomaly/save \
  -H "Content-Type: application/json" \
  -d '{
    "model_name": "http_anomaly",
    "description": "HTTP request anomaly detector trained on 30 days of access logs",
    "tags": ["production", "http", "v2"]
  }'
```

**Response:**

```json
{
  "model_name": "http_anomaly",
  "version": "http_anomaly-20240115T083000Z",
  "backend": "iforest",
  "created_at": "2024-01-15T08:30:00Z",
  "size_bytes": 245760,
  "description": "HTTP request anomaly detector trained on 30 days of access logs",
  "tags": ["production", "http", "v2"]
}
```

### Save with Custom Version Tag

```bash
curl -X POST http://localhost:14345/v1/ml/anomaly/save \
  -H "Content-Type: application/json" \
  -d '{
    "model_name": "http_anomaly",
    "version_tag": "v2.1-stable"
  }'
```

This creates the version `http_anomaly-v2.1-stable` in addition to the timestamp-based version.

---

## Load Workflow

Load a previously saved model to use for scoring or detection.

### Load Latest Version

```bash
curl -X POST http://localhost:14345/v1/ml/anomaly/load \
  -H "Content-Type: application/json" \
  -d '{
    "model_name": "http_anomaly"
  }'
```

This loads `http_anomaly-latest`, which resolves to the most recently saved version.

### Load Specific Version

```bash
curl -X POST http://localhost:14345/v1/ml/anomaly/load \
  -H "Content-Type: application/json" \
  -d '{
    "model_name": "http_anomaly",
    "version": "http_anomaly-20240115T083000Z"
  }'
```

### Load by Version Tag

```bash
curl -X POST http://localhost:14345/v1/ml/anomaly/load \
  -H "Content-Type: application/json" \
  -d '{
    "model_name": "http_anomaly",
    "version": "http_anomaly-v2.1-stable"
  }'
```

**Response:**

```json
{
  "model_name": "http_anomaly",
  "version": "http_anomaly-20240115T083000Z",
  "backend": "iforest",
  "loaded_at": "2024-01-16T10:00:00Z",
  "status": "ready"
}
```

---

## Listing Models

### List All Models

```bash
curl http://localhost:14345/v1/ml/anomaly/models
```

**Response:**

```json
{
  "models": [
    {
      "model_name": "http_anomaly",
      "versions": [
        "http_anomaly-20240115T083000Z",
        "http_anomaly-20240114T120000Z"
      ],
      "latest": "http_anomaly-20240115T083000Z",
      "backend": "iforest",
      "tags": ["production", "http", "v2"]
    },
    {
      "model_name": "payment_fraud",
      "versions": [
        "payment_fraud-20240115T090000Z"
      ],
      "latest": "payment_fraud-20240115T090000Z",
      "backend": "autoencoder",
      "tags": ["production", "payments"]
    }
  ]
}
```

### Filter by Tag

```bash
curl "http://localhost:14345/v1/ml/anomaly/models?tag=production"
```

### Filter by Backend

```bash
curl "http://localhost:14345/v1/ml/anomaly/models?backend=iforest"
```

---

## Deleting Models

### Delete a Specific Version

```bash
curl -X DELETE "http://localhost:14345/v1/ml/anomaly/models/http_anomaly/versions/http_anomaly-20240114T120000Z"
```

### Delete All Versions of a Model

```bash
curl -X DELETE "http://localhost:14345/v1/ml/anomaly/models/http_anomaly"
```

### Delete Old Versions (Keep Latest N)

```bash
curl -X POST http://localhost:14345/v1/ml/anomaly/models/http_anomaly/prune \
  -H "Content-Type: application/json" \
  -d '{
    "keep": 3
  }'
```

This deletes all versions except the 3 most recent ones.

---

## Model Metadata Inspection

### Get Model Details

```bash
curl http://localhost:14345/v1/ml/anomaly/models/http_anomaly
```

**Response:**

```json
{
  "model_name": "http_anomaly",
  "latest_version": "http_anomaly-20240115T083000Z",
  "backend": "iforest",
  "created_at": "2024-01-15T08:30:00Z",
  "description": "HTTP request anomaly detector trained on 30 days of access logs",
  "tags": ["production", "http", "v2"],
  "training_info": {
    "n_samples": 50000,
    "n_features": 8,
    "contamination": 0.05,
    "normalization": "standardize",
    "params": {
      "n_estimators": 200,
      "max_samples": 256
    }
  },
  "versions": [
    {
      "version": "http_anomaly-20240115T083000Z",
      "created_at": "2024-01-15T08:30:00Z",
      "size_bytes": 245760,
      "version_tag": "v2.1-stable"
    },
    {
      "version": "http_anomaly-20240114T120000Z",
      "created_at": "2024-01-14T12:00:00Z",
      "size_bytes": 230400,
      "version_tag": null
    }
  ]
}
```

---

## Best Practices

### Naming Conventions

- Use descriptive, lowercase names with underscores: `http_anomaly`, `payment_fraud`, `sensor_drift`
- Include the domain in the name so models are easy to find when listed
- Avoid generic names like `model1` or `test`

### Version Management

- Save a new version after each retraining, do not overwrite
- Use `version_tag` for milestone versions (e.g., `v2.1-stable`, `pre-migration`)
- Prune old versions periodically with the `/prune` endpoint, keeping at least 3 recent versions
- Always test a new version with `/v1/ml/anomaly/score` before promoting it

### Production Workflow

1. **Train** on fresh data with `/v1/ml/anomaly/fit`
2. **Score** a validation set with `/v1/ml/anomaly/score` to verify quality
3. **Save** the model with `/v1/ml/anomaly/save` and a descriptive tag
4. **Load** the new version in your scoring pipeline with `/v1/ml/anomaly/load`
5. **Prune** old versions to reclaim storage

### Storage Considerations

- Model size depends on the backend: `ecod` and `hbos` produce small models (<100KB), `autoencoder` models can be several MB
- Monitor disk usage if you retrain frequently
- Use the `/prune` endpoint to automate cleanup

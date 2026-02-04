# Data Preparation for Anomaly Detection

Guide for preparing data before training or scoring with anomaly detection models.

---

## Normalization Methods

All numeric features should be normalized before training. The `normalization` parameter on the `/v1/ml/anomaly/fit` endpoint controls this.

### Method Comparison

| Method | Formula | When to Use |
|--------|---------|-------------|
| `standardize` | `(x - mean) / std` | Default. Centers and scales data. Works for most cases. |
| `zscore` | `(x - mean) / std` with outlier-aware stats | When you need strict statistical z-score scaling with robust estimates. |
| `raw` | No transformation | When data is already normalized, or normalization would distort meaningful scales. |

### Decision Guide

1. **Unsure which to pick?** Use `standardize`. It is the default for a reason.
2. **Features on wildly different scales?** Use `standardize` -- it prevents large-magnitude features from dominating.
3. **Data already in [0, 1] or standardized?** Use `raw` to avoid double-normalization.
4. **Suspect extreme outliers in training data?** Use `zscore` for more robust scaling.
5. **Features represent physical units you want preserved?** Use `raw` but ensure comparable magnitudes.

### Example: Standardize (Default)

```bash
curl -X POST http://localhost:14345/v1/ml/anomaly/fit \
  -H "Content-Type: application/json" \
  -d '{
    "data": [[100, 0.5], [200, 0.6], [150, 0.55]],
    "backend": "iforest",
    "normalization": "standardize"
  }'
```

### Example: Raw (Pre-Normalized Data)

```bash
curl -X POST http://localhost:14345/v1/ml/anomaly/fit \
  -H "Content-Type: application/json" \
  -d '{
    "data": [[0.1, 0.5], [0.2, 0.6], [0.15, 0.55]],
    "backend": "iforest",
    "normalization": "raw"
  }'
```

---

## Mixed-Type Encoding

When your data contains non-numeric features (strings, categories), use the `encoding` parameter to convert them to numeric representations.

### Encoding Methods

| Method | Output | Best For |
|--------|--------|----------|
| `hash` | Fixed-width hash vector | High-cardinality categories (e.g., user IDs, URLs) |
| `label` | Integer labels | Ordinal categories (e.g., severity: low/medium/high) |
| `onehot` | Binary columns per category | Low-cardinality nominal categories (<20 unique values) |
| `binary` | Binary encoding of label integers | Medium-cardinality categories (20-100 unique values) |
| `frequency` | Frequency of each category in the dataset | When category frequency carries meaning |

### Decision Table

| Cardinality | Ordinal? | Recommended Encoding |
|-------------|----------|---------------------|
| Low (<20) | No | `onehot` |
| Low (<20) | Yes | `label` |
| Medium (20-100) | No | `binary` |
| Medium (20-100) | Yes | `label` |
| High (100+) | No | `hash` |
| High (100+) | Yes | `hash` |
| Frequency matters | Either | `frequency` |

### Example: Mixed Data with Encoding

```bash
curl -X POST http://localhost:14345/v1/ml/anomaly/fit \
  -H "Content-Type: application/json" \
  -d '{
    "data": [
      [100, "GET", "/api/users", 200],
      [150, "POST", "/api/users", 201],
      [200, "GET", "/api/health", 200],
      [5000, "DELETE", "/api/admin", 500]
    ],
    "columns": ["response_time_ms", "method", "path", "status_code"],
    "encoding": "hash",
    "backend": "iforest",
    "normalization": "standardize"
  }'
```

---

## Handling Categorical Features

### Specifying Categorical Columns

Use the `categorical_columns` parameter to identify which columns are categorical so the system applies encoding only to those columns.

```bash
curl -X POST http://localhost:14345/v1/ml/anomaly/fit \
  -H "Content-Type: application/json" \
  -d '{
    "data": [
      [100, "GET", 200, 1.5],
      [150, "POST", 201, 2.0],
      [200, "GET", 200, 1.8]
    ],
    "columns": ["response_time_ms", "method", "status_code", "cpu_usage"],
    "categorical_columns": ["method"],
    "encoding": "onehot",
    "backend": "iforest"
  }'
```

### Mixed Encoding Strategies

When different categorical columns need different encoding methods, use the `column_encodings` parameter.

```bash
curl -X POST http://localhost:14345/v1/ml/anomaly/fit \
  -H "Content-Type: application/json" \
  -d '{
    "data": [
      [100, "GET", "/api/users/12345", "us-east-1"],
      [150, "POST", "/api/users/67890", "eu-west-1"]
    ],
    "columns": ["response_time_ms", "method", "path", "region"],
    "column_encodings": {
      "method": "onehot",
      "path": "hash",
      "region": "label"
    },
    "backend": "iforest"
  }'
```

---

## Handling Timestamps

Timestamps should be converted to numeric features before passing to the anomaly detection API. Two approaches are supported.

### Extract Components

Break timestamps into meaningful numeric components (hour, day of week, month, etc.). Use this when temporal patterns matter.

```bash
curl -X POST http://localhost:14345/v1/ml/anomaly/fit \
  -H "Content-Type: application/json" \
  -d '{
    "data": [
      ["2024-01-15T08:30:00Z", 100, 200],
      ["2024-01-15T14:00:00Z", 150, 201],
      ["2024-01-15T03:00:00Z", 5000, 500]
    ],
    "columns": ["timestamp", "response_time_ms", "status_code"],
    "timestamp_columns": ["timestamp"],
    "timestamp_features": ["hour", "day_of_week", "month"],
    "backend": "iforest"
  }'
```

Available timestamp features: `hour`, `day_of_week`, `day_of_month`, `month`, `year`, `minute`, `is_weekend`.

### Unix Epoch

Convert timestamps to seconds since epoch. Use this when absolute time matters more than cyclical patterns.

```bash
curl -X POST http://localhost:14345/v1/ml/anomaly/fit \
  -H "Content-Type: application/json" \
  -d '{
    "data": [
      ["2024-01-15T08:30:00Z", 100],
      ["2024-01-15T14:00:00Z", 150]
    ],
    "columns": ["timestamp", "response_time_ms"],
    "timestamp_columns": ["timestamp"],
    "timestamp_features": ["epoch"],
    "backend": "iforest"
  }'
```

---

## Handling Missing Values

Missing values must be addressed before training. Use the `imputation` parameter to control how missing values are filled.

### Imputation Strategies

| Strategy | Description | When to Use |
|----------|-------------|-------------|
| `mean` | Replace with column mean | Default. General-purpose numeric data. |
| `median` | Replace with column median | Numeric data with outliers in the training set. |
| `mode` | Replace with most frequent value | Categorical columns. |
| `zero` | Replace with zero | When zero is a meaningful default. |
| `drop` | Remove rows with any missing value | When missing rows are rare and not informative. |

### Example

```bash
curl -X POST http://localhost:14345/v1/ml/anomaly/fit \
  -H "Content-Type: application/json" \
  -d '{
    "data": [
      [100, 0.5, 200],
      [150, null, 201],
      [null, 0.6, 200],
      [200, 0.55, 200]
    ],
    "columns": ["response_time_ms", "cpu_usage", "status_code"],
    "imputation": "median",
    "backend": "iforest"
  }'
```

### Per-Column Imputation

```bash
curl -X POST http://localhost:14345/v1/ml/anomaly/fit \
  -H "Content-Type: application/json" \
  -d '{
    "data": [
      [100, "GET", null],
      [null, "POST", 201]
    ],
    "columns": ["response_time_ms", "method", "status_code"],
    "column_imputation": {
      "response_time_ms": "median",
      "method": "mode",
      "status_code": "zero"
    },
    "backend": "iforest"
  }'
```

---

## Data Preparation Checklist

1. **Identify column types** -- numeric, categorical, timestamp
2. **Choose normalization** -- `standardize` unless you have a reason not to
3. **Choose encoding** for categorical columns -- `onehot` for low cardinality, `hash` for high
4. **Handle timestamps** -- extract components for cyclical patterns, epoch for absolute
5. **Handle missing values** -- `median` for numeric, `mode` for categorical
6. **Verify feature count** -- ensure all rows have the same number of features after transformation
7. **Check data volume** -- more samples generally produce better models, aim for 100+ rows minimum

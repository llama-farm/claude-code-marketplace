# Anomaly Detection Backends Reference

Detailed reference for each anomaly detection backend available in LlamaFarm.

---

## iforest (Isolation Forest)

**Algorithm:** Builds an ensemble of random trees that isolate observations. Anomalies are isolated in fewer splits, producing shorter path lengths.

**When to use:** General-purpose anomaly detection on tabular data. Best default choice.
**When not to use:** Very high-dimensional sparse data or when you need sub-second training on millions of rows.

### Key Parameters

| Parameter | Default | Description |
|-----------|---------|-------------|
| `n_estimators` | 100 | Number of trees in the ensemble |
| `max_samples` | `auto` | Samples per tree (`auto` = min(256, n_samples)) |
| `max_features` | 1.0 | Fraction of features per tree |
| `contamination` | 0.1 | Expected proportion of anomalies |

### Example

```bash
curl -X POST http://localhost:14345/v1/ml/anomaly/fit \
  -H "Content-Type: application/json" \
  -d '{
    "data": [[1.0, 2.0], [1.1, 2.1], [1.2, 1.9], [10.0, 10.0]],
    "backend": "iforest",
    "contamination": 0.1,
    "params": {
      "n_estimators": 200,
      "max_samples": 256
    }
  }'
```

### Performance

| Dataset Size | Training Time | Memory |
|-------------|---------------|--------|
| Small (<1K) | <1s | Low |
| Medium (1K-100K) | 1-10s | Low |
| Large (100K+) | 10-60s | Moderate |

---

## ecod (Empirical Cumulative Distribution)

**Algorithm:** Uses empirical cumulative distribution functions to estimate tail probabilities. Anomalies are points with low cumulative probability across features.

**When to use:** High-dimensional data, streaming use cases, or when you need the fastest possible training.
**When not to use:** When features have complex non-linear dependencies.

### Key Parameters

| Parameter | Default | Description |
|-----------|---------|-------------|
| `contamination` | 0.1 | Expected proportion of anomalies |

ECOD is parameter-free beyond contamination, which makes it simple to deploy.

### Example

```bash
curl -X POST http://localhost:14345/v1/ml/anomaly/fit \
  -H "Content-Type: application/json" \
  -d '{
    "data": [[1.0, 2.0], [1.1, 2.1], [1.2, 1.9], [10.0, 10.0]],
    "backend": "ecod",
    "contamination": 0.05
  }'
```

### Performance

| Dataset Size | Training Time | Memory |
|-------------|---------------|--------|
| Small (<1K) | <1s | Very Low |
| Medium (1K-100K) | <5s | Low |
| Large (100K+) | 5-30s | Low |

---

## lof (Local Outlier Factor)

**Algorithm:** Measures the local density deviation of a point relative to its neighbors. Points with substantially lower density than their neighbors are anomalies.

**When to use:** Data with clusters of varying density where anomalies sit in sparse regions.
**When not to use:** Very large datasets (quadratic neighbor search), or uniformly distributed data.

### Key Parameters

| Parameter | Default | Description |
|-----------|---------|-------------|
| `n_neighbors` | 20 | Number of neighbors for density estimation |
| `algorithm` | `auto` | Nearest-neighbor algorithm (`ball_tree`, `kd_tree`, `brute`, `auto`) |
| `leaf_size` | 30 | Leaf size for tree-based algorithms |
| `contamination` | 0.1 | Expected proportion of anomalies |

### Example

```bash
curl -X POST http://localhost:14345/v1/ml/anomaly/fit \
  -H "Content-Type: application/json" \
  -d '{
    "data": [[1.0, 2.0], [1.1, 2.1], [1.2, 1.9], [10.0, 10.0]],
    "backend": "lof",
    "contamination": 0.1,
    "params": {
      "n_neighbors": 30,
      "algorithm": "auto"
    }
  }'
```

### Performance

| Dataset Size | Training Time | Memory |
|-------------|---------------|--------|
| Small (<1K) | <1s | Low |
| Medium (1K-100K) | 5-30s | Moderate |
| Large (100K+) | Minutes+ | High |

---

## knn (K-Nearest Neighbors)

**Algorithm:** Computes the distance to the k-th nearest neighbor for each point. Points with large distances to their neighbors are anomalies.

**When to use:** When anomalies are isolated points far from any cluster. Good for local density-based detection.
**When not to use:** Very large datasets due to distance computation cost, or very high-dimensional data.

### Key Parameters

| Parameter | Default | Description |
|-----------|---------|-------------|
| `n_neighbors` | 5 | Number of nearest neighbors |
| `method` | `largest` | Distance method (`largest`, `mean`, `median`) |
| `algorithm` | `auto` | Nearest-neighbor algorithm |
| `contamination` | 0.1 | Expected proportion of anomalies |

### Example

```bash
curl -X POST http://localhost:14345/v1/ml/anomaly/fit \
  -H "Content-Type: application/json" \
  -d '{
    "data": [[1.0, 2.0], [1.1, 2.1], [1.2, 1.9], [10.0, 10.0]],
    "backend": "knn",
    "contamination": 0.1,
    "params": {
      "n_neighbors": 10,
      "method": "mean"
    }
  }'
```

### Performance

| Dataset Size | Training Time | Memory |
|-------------|---------------|--------|
| Small (<1K) | <1s | Low |
| Medium (1K-100K) | 5-30s | Moderate |
| Large (100K+) | Minutes+ | High |

---

## ocsvm (One-Class SVM)

**Algorithm:** Fits a hyperplane that separates normal data from the origin in a high-dimensional feature space using a kernel function. Points on the wrong side are anomalies.

**When to use:** Well-separated anomalies with clear decision boundaries. Works well with RBF kernel for non-linear boundaries.
**When not to use:** Very large datasets (kernel computation is expensive), or when anomalies are close to normal data.

### Key Parameters

| Parameter | Default | Description |
|-----------|---------|-------------|
| `kernel` | `rbf` | Kernel type (`rbf`, `linear`, `poly`, `sigmoid`) |
| `nu` | 0.1 | Upper bound on fraction of anomalies (similar to contamination) |
| `gamma` | `scale` | Kernel coefficient (`scale`, `auto`, or float) |
| `contamination` | 0.1 | Expected proportion of anomalies |

### Example

```bash
curl -X POST http://localhost:14345/v1/ml/anomaly/fit \
  -H "Content-Type: application/json" \
  -d '{
    "data": [[1.0, 2.0], [1.1, 2.1], [1.2, 1.9], [10.0, 10.0]],
    "backend": "ocsvm",
    "contamination": 0.1,
    "params": {
      "kernel": "rbf",
      "nu": 0.1,
      "gamma": "scale"
    }
  }'
```

### Performance

| Dataset Size | Training Time | Memory |
|-------------|---------------|--------|
| Small (<1K) | <1s | Low |
| Medium (1K-100K) | 10-60s | Moderate |
| Large (100K+) | Minutes+ | High |

---

## copod (Copula-Based Outlier Detection)

**Algorithm:** Uses empirical copula models to estimate the probability of each observation. Captures multivariate dependencies without assuming a specific distribution.

**When to use:** Multivariate data where feature dependencies matter. Good for fast screening with interpretable results.
**When not to use:** When features are truly independent (simpler methods suffice) or data has complex non-linear structure.

### Key Parameters

| Parameter | Default | Description |
|-----------|---------|-------------|
| `contamination` | 0.1 | Expected proportion of anomalies |

COPOD is parameter-free beyond contamination, similar to ECOD.

### Example

```bash
curl -X POST http://localhost:14345/v1/ml/anomaly/fit \
  -H "Content-Type: application/json" \
  -d '{
    "data": [[1.0, 2.0], [1.1, 2.1], [1.2, 1.9], [10.0, 10.0]],
    "backend": "copod",
    "contamination": 0.05
  }'
```

### Performance

| Dataset Size | Training Time | Memory |
|-------------|---------------|--------|
| Small (<1K) | <1s | Very Low |
| Medium (1K-100K) | <5s | Low |
| Large (100K+) | 5-30s | Low |

---

## hbos (Histogram-Based Outlier Score)

**Algorithm:** Builds histograms for each feature independently and scores each observation based on the inverse of the histogram bin height. Fast and simple.

**When to use:** Quick baseline, streaming data, or when features are relatively independent.
**When not to use:** When feature correlations are important for detection, or when data has complex multivariate structure.

### Key Parameters

| Parameter | Default | Description |
|-----------|---------|-------------|
| `n_bins` | 10 | Number of histogram bins |
| `alpha` | 0.1 | Regularization parameter for bin density |
| `tol` | 0.5 | Tolerance for bin edge comparison |
| `contamination` | 0.1 | Expected proportion of anomalies |

### Example

```bash
curl -X POST http://localhost:14345/v1/ml/anomaly/fit \
  -H "Content-Type: application/json" \
  -d '{
    "data": [[1.0, 2.0], [1.1, 2.1], [1.2, 1.9], [10.0, 10.0]],
    "backend": "hbos",
    "contamination": 0.1,
    "params": {
      "n_bins": 20,
      "alpha": 0.1
    }
  }'
```

### Performance

| Dataset Size | Training Time | Memory |
|-------------|---------------|--------|
| Small (<1K) | <1s | Very Low |
| Medium (1K-100K) | <2s | Very Low |
| Large (100K+) | <10s | Low |

---

## loda (Lightweight Online Detector of Anomalies)

**Algorithm:** Uses an ensemble of sparse random projections and one-dimensional histograms. Designed for streaming and incremental scenarios.

**When to use:** Streaming data, resource-constrained environments, or when you need lightweight online detection.
**When not to use:** When maximum accuracy is required or when data has very few features.

### Key Parameters

| Parameter | Default | Description |
|-----------|---------|-------------|
| `n_bins` | 10 | Number of histogram bins per projection |
| `n_random_cuts` | 100 | Number of random projections |
| `contamination` | 0.1 | Expected proportion of anomalies |

### Example

```bash
curl -X POST http://localhost:14345/v1/ml/anomaly/fit \
  -H "Content-Type: application/json" \
  -d '{
    "data": [[1.0, 2.0], [1.1, 2.1], [1.2, 1.9], [10.0, 10.0]],
    "backend": "loda",
    "contamination": 0.1,
    "params": {
      "n_bins": 15,
      "n_random_cuts": 150
    }
  }'
```

### Performance

| Dataset Size | Training Time | Memory |
|-------------|---------------|--------|
| Small (<1K) | <1s | Very Low |
| Medium (1K-100K) | <3s | Very Low |
| Large (100K+) | <15s | Low |

---

## cblof (Cluster-Based Local Outlier Factor)

**Algorithm:** First clusters the data, then computes anomaly scores based on a point's distance to its cluster center, weighted by cluster size. Small clusters and distant points score high.

**When to use:** Data with natural cluster structure where anomalies are in small or distant clusters.
**When not to use:** Data without cluster structure, or when the number of clusters is unknown and hard to estimate.

### Key Parameters

| Parameter | Default | Description |
|-----------|---------|-------------|
| `n_clusters` | 8 | Number of clusters |
| `alpha` | 0.9 | Proportion of data in large clusters |
| `beta` | 5 | Ratio threshold for large vs small clusters |
| `clustering_estimator` | None | Custom clustering algorithm (defaults to KMeans) |
| `contamination` | 0.1 | Expected proportion of anomalies |

### Example

```bash
curl -X POST http://localhost:14345/v1/ml/anomaly/fit \
  -H "Content-Type: application/json" \
  -d '{
    "data": [[1.0, 2.0], [1.1, 2.1], [1.2, 1.9], [10.0, 10.0]],
    "backend": "cblof",
    "contamination": 0.1,
    "params": {
      "n_clusters": 5,
      "alpha": 0.9,
      "beta": 5
    }
  }'
```

### Performance

| Dataset Size | Training Time | Memory |
|-------------|---------------|--------|
| Small (<1K) | <1s | Low |
| Medium (1K-100K) | 5-20s | Moderate |
| Large (100K+) | 30s-Minutes | Moderate |

---

## mcd (Minimum Covariance Determinant)

**Algorithm:** Estimates a robust covariance matrix by finding the subset of data that minimizes the determinant of the covariance matrix. Points far from the robust center (high Mahalanobis distance) are anomalies.

**When to use:** Data that is approximately Gaussian where you need robust covariance estimation.
**When not to use:** Non-Gaussian data, very high-dimensional data (covariance estimation degrades), or when n_samples < n_features.

### Key Parameters

| Parameter | Default | Description |
|-----------|---------|-------------|
| `support_fraction` | None | Fraction of data for robust estimate (None = automatic) |
| `assume_centered` | false | Whether to assume data is centered at origin |
| `contamination` | 0.1 | Expected proportion of anomalies |

### Example

```bash
curl -X POST http://localhost:14345/v1/ml/anomaly/fit \
  -H "Content-Type: application/json" \
  -d '{
    "data": [[1.0, 2.0], [1.1, 2.1], [1.2, 1.9], [10.0, 10.0]],
    "backend": "mcd",
    "contamination": 0.1,
    "params": {
      "support_fraction": 0.75
    }
  }'
```

### Performance

| Dataset Size | Training Time | Memory |
|-------------|---------------|--------|
| Small (<1K) | <1s | Low |
| Medium (1K-100K) | 5-30s | Moderate |
| Large (100K+) | Minutes+ | High |

---

## pca (Principal Component Analysis)

**Algorithm:** Projects data onto principal components and measures reconstruction error. Anomalies have high reconstruction error because they do not conform to the dominant variance directions.

**When to use:** High-dimensional data with linear correlations, dimensionality reduction scenarios.
**When not to use:** Non-linear relationships (use autoencoder instead), or very low-dimensional data.

### Key Parameters

| Parameter | Default | Description |
|-----------|---------|-------------|
| `n_components` | None | Number of principal components (None = automatic) |
| `whiten` | false | Whether to whiten the components |
| `svd_solver` | `auto` | SVD solver (`auto`, `full`, `randomized`) |
| `contamination` | 0.1 | Expected proportion of anomalies |

### Example

```bash
curl -X POST http://localhost:14345/v1/ml/anomaly/fit \
  -H "Content-Type: application/json" \
  -d '{
    "data": [[1.0, 2.0, 3.0], [1.1, 2.1, 3.1], [1.2, 1.9, 2.8], [10.0, 10.0, 10.0]],
    "backend": "pca",
    "contamination": 0.1,
    "params": {
      "n_components": 2,
      "whiten": true
    }
  }'
```

### Performance

| Dataset Size | Training Time | Memory |
|-------------|---------------|--------|
| Small (<1K) | <1s | Low |
| Medium (1K-100K) | 1-10s | Low |
| Large (100K+) | 10-60s | Moderate |

---

## autoencoder (Deep Autoencoder)

**Algorithm:** Trains a neural network to compress data into a low-dimensional latent space and reconstruct it. Anomalies have high reconstruction error because the network learns to reconstruct only normal patterns.

**When to use:** Complex non-linear patterns, large datasets, when other methods underperform.
**When not to use:** Small datasets (<500 samples), when training time is critical, or when interpretability is required.

### Key Parameters

| Parameter | Default | Description |
|-----------|---------|-------------|
| `hidden_neurons` | `[64, 32, 32, 64]` | Layer sizes for encoder and decoder |
| `epochs` | 50 | Number of training epochs |
| `batch_size` | 32 | Training batch size |
| `learning_rate` | 0.001 | Optimizer learning rate |
| `dropout_rate` | 0.2 | Dropout rate for regularization |
| `validation_size` | 0.1 | Fraction of data for validation |
| `contamination` | 0.1 | Expected proportion of anomalies |

### Example

```bash
curl -X POST http://localhost:14345/v1/ml/anomaly/fit \
  -H "Content-Type: application/json" \
  -d '{
    "data": [[1.0, 2.0, 3.0, 4.0], [1.1, 2.1, 3.1, 4.1], [1.2, 1.9, 2.8, 3.9], [10.0, 10.0, 10.0, 10.0]],
    "backend": "autoencoder",
    "contamination": 0.1,
    "normalization": "standardize",
    "params": {
      "hidden_neurons": [32, 16, 16, 32],
      "epochs": 100,
      "batch_size": 32,
      "learning_rate": 0.001,
      "dropout_rate": 0.2
    }
  }'
```

### Performance

| Dataset Size | Training Time | Memory |
|-------------|---------------|--------|
| Small (<1K) | 5-30s | Moderate |
| Medium (1K-100K) | 1-10 min | High |
| Large (100K+) | 10 min+ | Very High |

### Architecture Tips

- **Hidden neurons:** Use a symmetric architecture (e.g., `[64, 32, 32, 64]`). The bottleneck (middle layers) should be significantly smaller than input dimension.
- **Epochs:** Start with 50, increase if validation loss is still decreasing.
- **Dropout:** Use 0.1-0.3 to prevent overfitting. Higher values for smaller datasets.
- **Learning rate:** Start with 0.001, reduce to 0.0001 if training is unstable.

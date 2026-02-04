# Model Management

## Model Save/Load Lifecycle

When you train a classifier with `/v1/classifier/fit`, the model is automatically saved and available for predictions immediately. The lifecycle is:

```
Train (fit) → Saved to server → Available for predict → Delete when no longer needed
```

Models persist across server restarts. Each model is stored with its name as the identifier.

## Listing Trained Classifiers

### GET `/v1/classifier/models`

Returns all trained classifiers with their metadata.

```bash
curl http://localhost:14345/v1/classifier/models
```

**Response:**

```json
{
  "models": [
    {
      "name": "ticket_classifier_v1",
      "labels": ["bug", "feature", "question"],
      "base_model": "sentence-transformers/all-MiniLM-L6-v2",
      "num_examples": 24,
      "created_at": "2025-09-15T10:30:00Z"
    },
    {
      "name": "ticket_classifier_v2",
      "labels": ["bug", "feature", "question"],
      "base_model": "BAAI/bge-base-en-v1.5",
      "num_examples": 48,
      "created_at": "2025-09-20T14:15:00Z"
    },
    {
      "name": "sentiment_model",
      "labels": ["positive", "negative", "neutral"],
      "base_model": "sentence-transformers/all-mpnet-base-v2",
      "num_examples": 60,
      "created_at": "2025-10-01T09:00:00Z"
    }
  ]
}
```

## Model Versioning Patterns

LlamaFarm does not enforce a versioning scheme, but using a naming convention keeps models organized.

### Recommended Naming Convention

```
{task}_{version}
```

**Examples:**
- `ticket_classifier_v1`
- `ticket_classifier_v2`
- `sentiment_prod_v3`
- `urgency_detector_v1`

### Version Promotion Workflow

```
ticket_classifier_v1  (initial model, 24 examples)
ticket_classifier_v2  (retrained with 48 examples, better accuracy)
ticket_classifier_v3  (new base model, production candidate)
```

Train new versions alongside existing ones. Both remain available for prediction until you delete the old one.

```bash
# Train new version
curl -X POST http://localhost:14345/v1/classifier/fit \
  -H "Content-Type: application/json" \
  -d '{
    "model_name": "ticket_classifier_v3",
    "training_data": [...],
    "base_model": "BAAI/bge-base-en-v1.5",
    "num_iterations": 30
  }'

# Verify new version works
curl -X POST http://localhost:14345/v1/classifier/predict \
  -H "Content-Type: application/json" \
  -d '{
    "model_name": "ticket_classifier_v3",
    "text": "The export button does nothing when clicked"
  }'

# Delete old version after validation
curl -X DELETE http://localhost:14345/v1/classifier/models/ticket_classifier_v1
```

## Deleting Models

### DELETE `/v1/classifier/models/{name}`

Remove a trained classifier by name.

```bash
curl -X DELETE http://localhost:14345/v1/classifier/models/ticket_classifier_v1
```

**Response:**

```json
{
  "status": "deleted",
  "model_name": "ticket_classifier_v1"
}
```

**When to delete:**
- Old versions superseded by a better model
- Experimental models no longer needed
- Fold models from cross-validation (clean up after evaluation)
- Freeing disk space on the server

## Comparing Model Versions

Compare two model versions on the same test set to decide which to promote.

```python
import requests

test_data = [
    {"text": "The page loads with a blank screen", "label": "bug"},
    {"text": "Add support for Markdown in comments", "label": "feature"},
    {"text": "How do I change my email address?", "label": "question"},
    {"text": "Timeout error when generating reports", "label": "bug"},
    {"text": "Allow filtering by date range", "label": "feature"},
    {"text": "What integrations are available?", "label": "question"},
]

models = ["ticket_classifier_v1", "ticket_classifier_v2"]

for model_name in models:
    correct = 0
    for item in test_data:
        result = requests.post("http://localhost:14345/v1/classifier/predict", json={
            "model_name": model_name,
            "text": item["text"]
        }).json()
        if result["label"] == item["label"]:
            correct += 1
    accuracy = correct / len(test_data)
    print(f"{model_name}: {accuracy:.2%}")
```

Output:
```
ticket_classifier_v1: 83.33%
ticket_classifier_v2: 100.00%
```

### Detailed Comparison with Per-Class Breakdown

```python
import requests
from collections import defaultdict

test_data = [
    {"text": "The page loads with a blank screen", "label": "bug"},
    {"text": "Crash when uploading PDF files", "label": "bug"},
    {"text": "Add Markdown support in comments", "label": "feature"},
    {"text": "Allow filtering by date range", "label": "feature"},
    {"text": "How do I change my email?", "label": "question"},
    {"text": "What integrations are available?", "label": "question"},
]

models = ["ticket_classifier_v1", "ticket_classifier_v2"]

for model_name in models:
    per_class = defaultdict(lambda: {"correct": 0, "total": 0})
    for item in test_data:
        result = requests.post("http://localhost:14345/v1/classifier/predict", json={
            "model_name": model_name,
            "text": item["text"]
        }).json()
        per_class[item["label"]]["total"] += 1
        if result["label"] == item["label"]:
            per_class[item["label"]]["correct"] += 1

    print(f"\n{model_name}:")
    for label, counts in sorted(per_class.items()):
        acc = counts["correct"] / counts["total"]
        print(f"  {label:15s} {acc:.0%} ({counts['correct']}/{counts['total']})")
```

Output:
```
ticket_classifier_v1:
  bug             100% (2/2)
  feature          50% (1/2)
  question        100% (2/2)

ticket_classifier_v2:
  bug             100% (2/2)
  feature         100% (2/2)
  question        100% (2/2)
```

## Model Export Considerations

Trained classifiers are stored server-side on the LlamaFarm instance. Keep in mind:

- **Backup** - Models are stored in the LlamaFarm data directory. Include this directory in backups.
- **Migration** - To move a model between servers, copy the model files from the data directory or retrain on the new instance.
- **Reproducibility** - Save your training data separately (in version control or a dataset store) so you can retrain any model from scratch.
- **Storage** - Each classifier is relatively small (base model weights are shared). Disk usage scales primarily with the number of distinct base models, not the number of classifiers.

### Reproducibility Checklist

For each production model, record:

| Field | Example |
|-------|---------|
| Model name | `ticket_classifier_v2` |
| Base model | `BAAI/bge-base-en-v1.5` |
| num_iterations | `30` |
| batch_size | `16` |
| Training data hash/version | `training_data_v2.json` (sha256: abc123...) |
| Test accuracy | `95.8%` |
| Date trained | `2025-09-20` |

This ensures any model can be reproduced if the server data is lost.

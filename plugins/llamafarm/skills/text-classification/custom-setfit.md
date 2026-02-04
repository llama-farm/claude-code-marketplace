# Custom Classification with SetFit

## How SetFit Works

SetFit (Sentence Transformer Fine-Tuning) trains a classifier with very few examples by:

1. **Contrastive learning** - Generates sentence pairs from your training data and fine-tunes a sentence transformer to push same-class texts closer together
2. **Classification head** - Trains a lightweight classifier on the fine-tuned embeddings

This approach needs only 8-64 labeled examples per class to achieve strong results.

## Training Workflow

### Step 1: Prepare Training Data

Training data is an array of `{"text": ..., "label": ...}` objects.

**Minimum requirements:**
- At least 2 unique labels
- At least 8 examples per label (recommended)
- Balanced classes when possible

```json
{
  "training_data": [
    {"text": "App crashes when clicking submit", "label": "bug"},
    {"text": "Login fails with SSO enabled", "label": "bug"},
    {"text": "500 error on the dashboard page", "label": "bug"},
    {"text": "Data not saving after form submission", "label": "bug"},
    {"text": "Dropdown menu is unresponsive", "label": "bug"},
    {"text": "Page freezes when uploading large files", "label": "bug"},
    {"text": "Broken link on the settings page", "label": "bug"},
    {"text": "Search returns no results for valid queries", "label": "bug"},
    {"text": "Add dark mode toggle", "label": "feature"},
    {"text": "Support CSV export for reports", "label": "feature"},
    {"text": "Add keyboard shortcuts for common actions", "label": "feature"},
    {"text": "Allow custom color themes", "label": "feature"},
    {"text": "Integrate with Slack notifications", "label": "feature"},
    {"text": "Add two-factor authentication option", "label": "feature"},
    {"text": "Support bulk editing of records", "label": "feature"},
    {"text": "Add an API rate limit dashboard", "label": "feature"},
    {"text": "How do I reset my password?", "label": "question"},
    {"text": "Where can I find the API documentation?", "label": "question"},
    {"text": "What file formats are supported?", "label": "question"},
    {"text": "Is there a free trial available?", "label": "question"},
    {"text": "How do I invite team members?", "label": "question"},
    {"text": "What are the storage limits?", "label": "question"},
    {"text": "Can I change my subscription plan?", "label": "question"},
    {"text": "How do I enable notifications?", "label": "question"}
  ]
}
```

### Step 2: Train the Classifier

#### POST `/v1/classifier/fit`

**Request body:**

| Parameter | Type | Required | Default | Description |
|-----------|------|----------|---------|-------------|
| `model_name` | string | Yes | - | Name for the trained classifier |
| `training_data` | array | Yes | - | Array of `{"text", "label"}` objects |
| `base_model` | string | No | `sentence-transformers/all-MiniLM-L6-v2` | Sentence transformer base model |
| `num_iterations` | integer | No | `20` | Number of contrastive learning iterations |
| `batch_size` | integer | No | `16` | Training batch size |

**Example request:**

```bash
curl -X POST http://localhost:14345/v1/classifier/fit \
  -H "Content-Type: application/json" \
  -d '{
    "model_name": "ticket_classifier_v1",
    "training_data": [
      {"text": "App crashes when clicking submit", "label": "bug"},
      {"text": "Login fails with SSO enabled", "label": "bug"},
      {"text": "Add dark mode toggle", "label": "feature"},
      {"text": "Support CSV export for reports", "label": "feature"},
      {"text": "How do I reset my password?", "label": "question"},
      {"text": "Where can I find the API docs?", "label": "question"}
    ],
    "base_model": "sentence-transformers/all-MiniLM-L6-v2",
    "num_iterations": 20,
    "batch_size": 16
  }'
```

**Response:**

```json
{
  "model_name": "ticket_classifier_v1",
  "status": "trained",
  "labels": ["bug", "feature", "question"],
  "num_examples": 6,
  "base_model": "sentence-transformers/all-MiniLM-L6-v2",
  "training_time_seconds": 12.4
}
```

### Step 3: Predict with the Classifier

#### POST `/v1/classifier/predict`

**Request body:**

| Parameter | Type | Required | Default | Description |
|-----------|------|----------|---------|-------------|
| `model_name` | string | Yes | - | Name of the trained classifier |
| `text` | string | Yes | - | Text to classify |

**Example request:**

```bash
curl -X POST http://localhost:14345/v1/classifier/predict \
  -H "Content-Type: application/json" \
  -d '{
    "model_name": "ticket_classifier_v1",
    "text": "The export button does nothing when clicked"
  }'
```

**Response:**

```json
{
  "label": "bug",
  "scores": {
    "bug": 0.91,
    "feature": 0.06,
    "question": 0.03
  },
  "model_name": "ticket_classifier_v1"
}
```

## Base Model Selection

| Model | Size | Quality | Speed | Best For |
|-------|------|---------|-------|----------|
| `sentence-transformers/all-MiniLM-L6-v2` | 80 MB | Good | Fast | Quick prototyping, resource-constrained |
| `sentence-transformers/all-mpnet-base-v2` | 420 MB | High | Moderate | Balanced accuracy and performance |
| `BAAI/bge-small-en-v1.5` | 130 MB | Good | Fast | Quality-focused with small footprint |
| `BAAI/bge-base-en-v1.5` | 440 MB | Very High | Moderate | Best quality for English text |

**Choosing a base model:**
- **Starting out / prototyping:** `sentence-transformers/all-MiniLM-L6-v2` -- fast iteration, good baseline
- **Production with tight resources:** `BAAI/bge-small-en-v1.5` -- small but high quality
- **Production, best accuracy:** `BAAI/bge-base-en-v1.5` -- highest quality English embeddings
- **General production:** `sentence-transformers/all-mpnet-base-v2` -- well-rounded, widely used

## Training Parameters

### num_iterations

Controls how many contrastive learning pairs are generated and trained on.

| Examples Per Class | Recommended Iterations | Notes |
|-------------------|----------------------|-------|
| 8-16 | 8-20 | Few-shot regime, avoid overfitting |
| 16-32 | 20-35 | Good balance of quality and speed |
| 32-64 | 35-50 | More data benefits from more iterations |
| 64+ | 50+ | Diminishing returns beyond 60 |

**Guidance:** Start with 20 iterations. Increase if accuracy on held-out data is still improving. Decrease if training is slow and you have very few examples.

### batch_size

| Available Memory | Recommended Batch Size |
|-----------------|----------------------|
| < 4 GB RAM | 8 |
| 4-8 GB RAM | 16 |
| 8-16 GB RAM | 32 |
| GPU available | 32-64 |

Larger batch sizes train faster but use more memory. If you encounter out-of-memory errors, reduce batch size.

## Evaluation Strategies

### Train/Test Split

Hold out 20% of your data for evaluation before training.

```python
import requests
import random

all_data = [
    {"text": "App crashes on login", "label": "bug"},
    {"text": "Add CSV export", "label": "feature"},
    # ... more examples
]

random.shuffle(all_data)
split = int(len(all_data) * 0.8)
train_data = all_data[:split]
test_data = all_data[split:]

# Train
requests.post("http://localhost:14345/v1/classifier/fit", json={
    "model_name": "ticket_classifier_v1",
    "training_data": train_data,
    "num_iterations": 20
})

# Evaluate
correct = 0
for item in test_data:
    result = requests.post("http://localhost:14345/v1/classifier/predict", json={
        "model_name": "ticket_classifier_v1",
        "text": item["text"]
    }).json()
    if result["label"] == item["label"]:
        correct += 1

accuracy = correct / len(test_data)
print(f"Accuracy: {accuracy:.2%}")
```

### Cross-Validation

For small datasets, use k-fold cross-validation to get a more reliable accuracy estimate.

```python
import requests
import random

def k_fold_evaluate(data, model_base_name, k=5, num_iterations=20):
    random.shuffle(data)
    fold_size = len(data) // k
    accuracies = []

    for i in range(k):
        test_fold = data[i * fold_size : (i + 1) * fold_size]
        train_fold = data[:i * fold_size] + data[(i + 1) * fold_size:]

        fold_model = f"{model_base_name}_fold{i}"

        requests.post("http://localhost:14345/v1/classifier/fit", json={
            "model_name": fold_model,
            "training_data": train_fold,
            "num_iterations": num_iterations
        })

        correct = 0
        for item in test_fold:
            result = requests.post("http://localhost:14345/v1/classifier/predict", json={
                "model_name": fold_model,
                "text": item["text"]
            }).json()
            if result["label"] == item["label"]:
                correct += 1

        accuracies.append(correct / len(test_fold))

        # Clean up fold model
        requests.delete(f"http://localhost:14345/v1/classifier/models/{fold_model}")

    avg = sum(accuracies) / len(accuracies)
    print(f"Cross-validation accuracy: {avg:.2%} (+/- {max(accuracies) - min(accuracies):.2%})")
    return avg
```

### Holdout Set

Keep a separate holdout set that is never used during training or model selection. Use it only for final evaluation before deploying to production.

```python
import requests

holdout = [
    {"text": "Notification emails are not being sent", "label": "bug"},
    {"text": "Add ability to schedule reports", "label": "feature"},
    {"text": "What is the maximum file upload size?", "label": "question"},
    # ... 10-20 examples reserved for final check
]

correct = 0
for item in holdout:
    result = requests.post("http://localhost:14345/v1/classifier/predict", json={
        "model_name": "ticket_classifier_v1",
        "text": item["text"]
    }).json()
    if result["label"] == item["label"]:
        correct += 1

print(f"Holdout accuracy: {correct / len(holdout):.2%}")
```

## Full Training-to-Deployment Workflow

### 1. Collect and prepare data

Gather at least 8 examples per class. More is better, but SetFit excels with few-shot.

### 2. Evaluate with cross-validation

```python
accuracy = k_fold_evaluate(all_data, "ticket_clf", k=5, num_iterations=20)
```

### 3. Train final model on all data

```bash
curl -X POST http://localhost:14345/v1/classifier/fit \
  -H "Content-Type: application/json" \
  -d '{
    "model_name": "ticket_classifier_v2",
    "training_data": [...],
    "base_model": "BAAI/bge-base-en-v1.5",
    "num_iterations": 30
  }'
```

### 4. Validate on holdout set

Run predictions on reserved holdout examples to confirm accuracy.

### 5. Deploy and monitor

Use the trained model name in your application. Monitor predictions and collect misclassified examples for future retraining.

## Tips for Better Accuracy

- **Diverse examples** - Include different phrasings, lengths, and writing styles for each label
- **Clean labels** - Ensure no mislabeled examples in training data
- **Balanced classes** - Roughly equal examples per class, or at minimum 8 per class
- **Iterate** - Start small, evaluate, add more examples where the model struggles
- **Try different base models** - Performance can vary by domain; test 2-3 models

---
description: Classify text using zero-shot or trained classification models
---

# /llamafarm:classify - Text Classification

Guide users through text classification workflows — zero-shot labeling, training custom classifiers, and batch prediction.

## Usage

```
/llamafarm:classify                 # Zero-shot classification (default)
/llamafarm:classify train           # Train a custom classifier
/llamafarm:classify predict         # Predict with a trained classifier
/llamafarm:classify models          # List trained classifiers
```

## What This Command Does

1. **Identifies the workflow** — zero-shot, train, predict, or informational
2. **Gathers inputs** — text, candidate labels, training examples
3. **Recommends configuration** — base model, label set, parameters
4. **Executes the workflow** — runs classification or trains model
5. **Presents results** — per-label scores and interpretation

## Implementation

When the user runs `/llamafarm:classify`, follow these steps:

### Subcommand: (default) Zero-Shot Classification

#### Step 1: Gather Inputs

Ask the user:
1. What text do you want to classify?
2. What are the candidate labels? (comma-separated list)
3. Is this multi-label? (can multiple labels apply simultaneously)

#### Step 2: Execute Classification

```bash
curl -X POST http://localhost:14345/v1/classify \
  -H "Content-Type: application/json" \
  -d '{
    "text": "<user_text>",
    "labels": ["<label_1>", "<label_2>", "<label_3>"],
    "multi_label": false
  }'
```

#### Step 3: Present Results

```
Classification Results
======================

Text: "<truncated input>"

Scores:
  label_1:  0.87  ████████▋
  label_2:  0.09  ▉
  label_3:  0.04  ▍

Predicted: label_1 (confidence: 87%)
```

Show:
- Per-label scores sorted by confidence
- Top prediction with confidence percentage
- If multi-label, all labels above threshold (default 0.5)

### Subcommand: `train`

#### Step 1: Collect Training Examples

Ask the user:
1. How many labels/classes? (list them)
2. Where are your training examples? (file path, inline, or API)
3. How many examples per class? (recommend minimum 10-20)

#### Step 2: Recommend Base Model

| Base Model | Best For | Speed | Accuracy |
|------------|----------|-------|----------|
| `distilbert-base-uncased` | English general text | Fast | Good |
| `roberta-base` | English, higher accuracy | Medium | High |
| `xlm-roberta-base` | Multilingual text | Medium | High |
| `bert-base-uncased` | English general text | Medium | Good |

#### Step 3: Execute Training

```bash
curl -X POST http://localhost:14345/v1/classifier/fit \
  -H "Content-Type: application/json" \
  -d '{
    "model_name": "<user_chosen_name>",
    "base_model": "<recommended>",
    "training_data": {
      "texts": ["<example_1>", "<example_2>"],
      "labels": ["<label_a>", "<label_b>"]
    },
    "epochs": 3,
    "batch_size": 16,
    "learning_rate": 2e-5
  }'
```

#### Step 4: Evaluate Results

Present:
- Model name and version
- Base model used
- Training accuracy and loss
- Per-class precision, recall, F1
- Next steps: run predictions or fine-tune further

### Subcommand: `predict`

#### Step 1: Select Model

```bash
curl http://localhost:14345/v1/classifier/models
```

Present available models and ask the user to select one.

#### Step 2: Run Prediction

```bash
curl -X POST http://localhost:14345/v1/classifier/predict \
  -H "Content-Type: application/json" \
  -d '{
    "model_name": "<selected_model>",
    "texts": ["<text_1>", "<text_2>"]
  }'
```

#### Step 3: Present Results

Show:
- Per-text predicted label with confidence
- Per-label score breakdown for each text
- Summary statistics across batch

### Subcommand: `models`

```bash
curl http://localhost:14345/v1/classifier/models
```

Present classifiers with name, base model, label count, training date, and accuracy.

## Error Handling

| Error | Cause | Fix |
|-------|-------|-----|
| `422 Unprocessable Entity` | Missing text or labels | Verify request body includes both fields |
| `404 Not Found` | Classifier doesn't exist | Run `/llamafarm:classify models` to list |
| `503 Service Unavailable` | ML runtime not running | Run `/llamafarm:start` |
| `400 Bad Request` | Invalid base model | Check supported models in train step |
| `413 Payload Too Large` | Text exceeds token limit | Truncate or split text before classifying |

## Skills to Load

- `text-classification` - Detailed model reference and tuning guidance

## Related Commands

- `/llamafarm:anomaly` - Anomaly detection
- `/llamafarm:ml-status` - Check ML service health
- `/llamafarm:ocr` - Extract text from images for classification

---
description: Zero-shot and custom text classification. Classify text with pre-trained models or train custom classifiers using SetFit with few-shot learning.
---

# Text Classification Skill

Guide for text classification in LlamaFarm using zero-shot and custom-trained approaches.

## When to Load

Load this skill when the user:
- Wants to classify or categorize text
- Asks about zero-shot classification
- Needs to train a custom text classifier
- Asks about SetFit or few-shot learning
- Wants to label or tag documents
- Needs sentiment analysis or topic classification

## Two Approaches

| Feature | Zero-Shot | Custom (SetFit) |
|---------|-----------|------------------|
| Training data needed | None | 8-64 examples per class |
| Setup time | Instant | Minutes |
| Accuracy | Good for broad categories | Excellent for specific domains |
| Custom labels | Any labels at inference | Fixed at training time |
| Best for | Exploration, prototyping | Production, domain-specific |

## Decision Tree

1. **No training data available?** → Zero-shot
2. **Labels change frequently?** → Zero-shot
3. **Need high accuracy on specific domain?** → Custom SetFit
4. **Have 8+ examples per class?** → Custom SetFit
5. **Quick prototype?** → Start zero-shot, upgrade to SetFit if needed

## API Overview

### Zero-Shot Classification

| Endpoint | Method | Description |
|----------|--------|-------------|
| `/v1/classify` | POST | Classify text with candidate labels |

### Custom Classification (SetFit)

| Endpoint | Method | Description |
|----------|--------|-------------|
| `/v1/classifier/fit` | POST | Train a custom classifier |
| `/v1/classifier/predict` | POST | Predict with trained classifier |
| `/v1/classifier/models` | GET | List trained classifiers |
| `/v1/classifier/models/{name}` | DELETE | Delete a classifier |

## Quick Start

### Zero-shot classification

```bash
curl -X POST http://localhost:14345/v1/classify \
  -H "Content-Type: application/json" \
  -d '{
    "text": "The server crashed and lost all data",
    "labels": ["bug", "feature request", "question", "documentation"]
  }'
```

Response:
```json
{
  "label": "bug",
  "scores": {
    "bug": 0.82,
    "feature request": 0.08,
    "question": 0.06,
    "documentation": 0.04
  }
}
```

### Train a custom classifier

```bash
curl -X POST http://localhost:14345/v1/classifier/fit \
  -H "Content-Type: application/json" \
  -d '{
    "model_name": "ticket_classifier",
    "training_data": [
      {"text": "App crashes on login", "label": "bug"},
      {"text": "Add dark mode", "label": "feature"},
      {"text": "How do I reset password?", "label": "question"}
    ],
    "base_model": "sentence-transformers/all-MiniLM-L6-v2",
    "num_iterations": 20
  }'
```

### Predict with custom classifier

```bash
curl -X POST http://localhost:14345/v1/classifier/predict \
  -H "Content-Type: application/json" \
  -d '{
    "model_name": "ticket_classifier",
    "text": "The export button does nothing when clicked"
  }'
```

## Progressive Disclosure

For detailed guidance:
- **zero-shot.md** - Pre-trained models, crafting labels, confidence interpretation
- **custom-setfit.md** - Training workflow, base model selection, evaluation strategies
- **model-management.md** - Save/load lifecycle, versioning, comparing models

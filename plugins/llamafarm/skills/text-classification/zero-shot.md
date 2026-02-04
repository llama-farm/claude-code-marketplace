# Zero-Shot Classification

## Endpoint Reference

### POST `/v1/classify`

Classify text against candidate labels without any training data.

**Request body:**

| Parameter | Type | Required | Default | Description |
|-----------|------|----------|---------|-------------|
| `text` | string | Yes | - | The text to classify |
| `labels` | array[string] | Yes | - | Candidate labels to classify against |
| `model` | string | No | `facebook/bart-large-mnli` | Pre-trained NLI model to use |
| `multi_label` | boolean | No | `false` | Allow multiple labels to be assigned |
| `hypothesis_template` | string | No | `"This text is about {}"` | Template for NLI hypothesis generation |

**Example request:**

```bash
curl -X POST http://localhost:14345/v1/classify \
  -H "Content-Type: application/json" \
  -d '{
    "text": "The new update includes a redesigned dashboard with dark mode support",
    "labels": ["bug report", "feature announcement", "security advisory", "documentation update"],
    "model": "facebook/bart-large-mnli",
    "multi_label": false
  }'
```

**Response format:**

```json
{
  "label": "feature announcement",
  "scores": {
    "feature announcement": 0.87,
    "documentation update": 0.06,
    "bug report": 0.04,
    "security advisory": 0.03
  },
  "model": "facebook/bart-large-mnli",
  "multi_label": false
}
```

**Multi-label response** (when `multi_label: true`):

```json
{
  "labels": ["feature announcement", "documentation update"],
  "scores": {
    "feature announcement": 0.87,
    "documentation update": 0.72,
    "bug report": 0.04,
    "security advisory": 0.03
  },
  "model": "facebook/bart-large-mnli",
  "multi_label": true
}
```

When `multi_label` is true, each label is scored independently (scores do not sum to 1), and `labels` returns all labels above the default threshold of 0.5.

## Pre-Trained Models

| Model | Size | Quality | Speed | Best For |
|-------|------|---------|-------|----------|
| `facebook/bart-large-mnli` | 1.6 GB | High | Moderate | General-purpose, broad categories |
| `cross-encoder/nli-deberta-v3-base` | 700 MB | Very High | Slow | Maximum accuracy, smaller batches |
| `MoritzLaurer/deberta-v3-large-zeroshot-v2.0` | 1.5 GB | Very High | Slow | Nuanced classification, long text |
| `MoritzLaurer/deberta-v3-base-zeroshot-v2.0` | 700 MB | High | Moderate | Balanced accuracy and speed |
| `typeform/distilbert-base-uncased-mnli` | 250 MB | Good | Fast | High throughput, resource-constrained |
| `valhalla/distilbart-mnli-12-3` | 600 MB | Good | Fast | Good balance for distilled model |

**Choosing a model:**
- **Default / general use:** `facebook/bart-large-mnli` -- widely tested, reliable baseline
- **Best accuracy:** `cross-encoder/nli-deberta-v3-base` -- highest quality but slower
- **Fast throughput:** `typeform/distilbert-base-uncased-mnli` -- smallest, fastest
- **Long documents:** `MoritzLaurer/deberta-v3-large-zeroshot-v2.0` -- handles longer context well

## Crafting Effective Labels

Label quality is the single biggest factor in zero-shot accuracy.

### Use Noun Phrases

```json
// Good - descriptive noun phrases
{"labels": ["bug report", "feature request", "support question", "billing issue"]}

// Bad - verbs or vague terms
{"labels": ["broken", "want", "help", "money"]}
```

### Be Specific

```json
// Good - specific and distinct
{"labels": ["server crash", "UI rendering issue", "authentication failure", "data loss"]}

// Bad - overlapping and vague
{"labels": ["error", "problem", "issue", "broken"]}
```

### Use Consistent Granularity

```json
// Good - same level of specificity
{"labels": ["positive sentiment", "negative sentiment", "neutral sentiment"]}

// Bad - mixed granularity
{"labels": ["happy", "negative sentiment", "meh"]}
```

### Custom Hypothesis Templates

The hypothesis template controls how labels are framed for the NLI model.

```bash
# Default template
curl -X POST http://localhost:14345/v1/classify \
  -H "Content-Type: application/json" \
  -d '{
    "text": "The server response time increased from 200ms to 5 seconds",
    "labels": ["performance degradation", "outage", "normal operation"],
    "hypothesis_template": "This incident describes {}"
  }'
```

**Template examples by domain:**

| Domain | Template |
|--------|----------|
| Support tickets | `"This support ticket is about {}"` |
| Sentiment | `"The sentiment of this text is {}"` |
| Topic | `"The topic of this document is {}"` |
| Intent | `"The user intends to {}"` |
| Urgency | `"The urgency level of this is {}"` |

## Confidence Score Interpretation

| Score Range | Confidence | Recommended Action |
|-------------|------------|-------------------|
| > 0.7 | High | Accept classification directly |
| 0.4 - 0.7 | Moderate | Use with caution, consider manual review |
| < 0.4 | Low | Do not trust, flag for human review |

**Score distribution matters:**
- **Clear winner** (0.85 vs 0.05, 0.05, 0.05): Strong signal, high confidence
- **Close scores** (0.30 vs 0.28 vs 0.22 vs 0.20): Ambiguous, text may span categories
- **Two high scores** (0.45 vs 0.42 vs 0.08 vs 0.05): Consider multi-label classification

## When Zero-Shot Fails

Zero-shot classification may underperform in these scenarios:

| Scenario | Why It Fails | Alternative |
|----------|-------------|-------------|
| Highly technical domains | NLI models trained on general text | Train custom SetFit classifier |
| Ambiguous categories | Model cannot distinguish subtle differences | Refine labels or use custom model |
| Context-dependent labels | Same text means different things in different contexts | Add context to hypothesis template |
| Many labels (>15) | Score distribution becomes flat | Group into hierarchical categories |
| Short text (<10 words) | Insufficient signal for NLI | Combine with metadata features |

## Example Workflows

### Sentiment Analysis

```bash
curl -X POST http://localhost:14345/v1/classify \
  -H "Content-Type: application/json" \
  -d '{
    "text": "This product exceeded my expectations, absolutely love it!",
    "labels": ["positive sentiment", "negative sentiment", "neutral sentiment"],
    "hypothesis_template": "The sentiment of this review is {}"
  }'
```

### Topic Classification

```bash
curl -X POST http://localhost:14345/v1/classify \
  -H "Content-Type: application/json" \
  -d '{
    "text": "The Federal Reserve raised interest rates by 25 basis points",
    "labels": ["monetary policy", "fiscal policy", "stock market", "employment", "housing market"],
    "hypothesis_template": "This article is about {}"
  }'
```

### Intent Detection

```bash
curl -X POST http://localhost:14345/v1/classify \
  -H "Content-Type: application/json" \
  -d '{
    "text": "Can you change my delivery address to 123 Main St?",
    "labels": ["update account information", "track order", "request refund", "product inquiry", "cancel order"],
    "hypothesis_template": "The user wants to {}"
  }'
```

### Urgency Classification

```bash
curl -X POST http://localhost:14345/v1/classify \
  -H "Content-Type: application/json" \
  -d '{
    "text": "Production database is down, all customer-facing services affected",
    "labels": ["critical urgency", "high urgency", "medium urgency", "low urgency"],
    "hypothesis_template": "This incident has {}"
  }'
```

### Multi-Label Document Tagging

```bash
curl -X POST http://localhost:14345/v1/classify \
  -H "Content-Type: application/json" \
  -d '{
    "text": "New HIPAA compliance requirements for cloud-hosted patient data encryption",
    "labels": ["healthcare", "compliance", "security", "cloud infrastructure", "data privacy"],
    "multi_label": true,
    "hypothesis_template": "This document relates to {}"
  }'
```

Response:
```json
{
  "labels": ["compliance", "healthcare", "data privacy", "security"],
  "scores": {
    "compliance": 0.92,
    "healthcare": 0.88,
    "data privacy": 0.85,
    "security": 0.74,
    "cloud infrastructure": 0.41
  },
  "model": "facebook/bart-large-mnli",
  "multi_label": true
}
```

## Batch Classification Pattern

For classifying multiple texts, loop requests client-side:

```python
import requests

texts = [
    "The login page returns a 500 error",
    "Would be great to have CSV export",
    "How do I configure SSO?",
]
labels = ["bug report", "feature request", "support question"]

results = []
for text in texts:
    response = requests.post(
        "http://localhost:14345/v1/classify",
        json={"text": text, "labels": labels}
    )
    results.append(response.json())

for text, result in zip(texts, results):
    print(f"{result['label']:20s} ({result['scores'][result['label']]:.2f})  {text}")
```

Output:
```
bug report           (0.89)  The login page returns a 500 error
feature request      (0.84)  Would be great to have CSV export
support question     (0.91)  How do I configure SSO?
```

# Named Entity Recognition (NER)

## Endpoint

```http
POST /v1/ner
Content-Type: application/json
```

## Parameters

| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| `text` | string | required | Text to analyze for named entities |
| `model` | string | `dslim/bert-base-NER` | NER model to use |
| `confidence_threshold` | float | `0.5` | Minimum confidence score (0.0 - 1.0) |
| `aggregate_strategy` | string | `simple` | How to merge sub-word tokens: `simple`, `first`, `average`, `max` |

## Entity Types

| Entity Type | Label | Description | Examples |
|-------------|-------|-------------|----------|
| Person | `PER` | Names of people | John Smith, Dr. Chen, Marie Curie |
| Organization | `ORG` | Companies, agencies, institutions | Google, FDA, MIT, United Nations |
| Location | `LOC` | Physical locations, geographic regions | Mountain View, California, Sahara Desert |
| Miscellaneous | `MISC` | Nationalities, events, products, works of art | American, Olympics, iPhone, Nobel Prize |

## Basic Usage

### Simple NER Request

```bash
curl -X POST http://localhost:14345/v1/ner \
  -H "Content-Type: application/json" \
  -d '{
    "text": "John Smith works at Google in Mountain View.",
    "model": "dslim/bert-base-NER"
  }'
```

### With Custom Confidence Threshold

```bash
curl -X POST http://localhost:14345/v1/ner \
  -H "Content-Type: application/json" \
  -d '{
    "text": "Apple announced the new iPhone at their Cupertino headquarters.",
    "model": "dslim/bert-base-NER",
    "confidence_threshold": 0.8
  }'
```

## Response Format

```json
{
  "entities": [
    {
      "text": "John Smith",
      "type": "PER",
      "confidence": 0.97,
      "start": 0,
      "end": 10
    },
    {
      "text": "Google",
      "type": "ORG",
      "confidence": 0.99,
      "start": 20,
      "end": 26
    },
    {
      "text": "Mountain View",
      "type": "LOC",
      "confidence": 0.95,
      "start": 30,
      "end": 43
    }
  ],
  "model": "dslim/bert-base-NER",
  "processing_time_ms": 45
}
```

## Model Selection Guide

| Model | Speed | Accuracy | Languages | Memory | Best For |
|-------|-------|----------|-----------|--------|----------|
| `dslim/bert-base-NER` | Fast (~20ms) | High | English | ~400 MB | General English, production use |
| `Jean-Baptiste/camembert-ner` | Fast (~20ms) | High | French | ~400 MB | French text |
| `flair/ner-english-large` | Medium (~80ms) | Very High | English | ~1.5 GB | Maximum accuracy, research |
| `dslim/bert-large-NER` | Medium (~50ms) | Very High | English | ~1.2 GB | Better accuracy, more resources |
| `Davlan/bert-base-multilingual-cased-ner-hrl` | Fast (~25ms) | Good | 10 languages | ~600 MB | Multilingual documents |

### Choosing a Model

- **English, production use:** `dslim/bert-base-NER` -- fast, accurate, low memory
- **Maximum English accuracy:** `flair/ner-english-large` -- best quality, higher latency
- **French text:** `Jean-Baptiste/camembert-ner` -- purpose-built for French
- **Multilingual:** `Davlan/bert-base-multilingual-cased-ner-hrl` -- covers 10 languages

## Confidence Threshold Guidance

| Threshold | Precision | Recall | Use When |
|-----------|-----------|--------|----------|
| 0.3 | Lower | Higher | Exploratory analysis, want all possible entities |
| 0.5 | Balanced | Balanced | General purpose (default) |
| 0.7 | Higher | Lower | Production systems, reducing false positives |
| 0.9 | Very High | Low | Critical applications, only high-confidence entities |

### When to Adjust

**Increase threshold (0.7+):**
- Building structured databases from extracted entities
- Feeding entities into downstream automated systems
- Domain-specific text where false positives are costly

**Decrease threshold (0.3-0.4):**
- Exploratory data analysis
- Finding rare or unusual entity mentions
- Short or informal text (tweets, chat messages)

## Combining NER with Classification

Extract entities first, then use them for document classification or enrichment.

```python
import requests

def extract_and_classify(text):
    # Step 1: Extract entities
    ner_response = requests.post(
        "http://localhost:14345/v1/ner",
        json={
            "text": text,
            "model": "dslim/bert-base-NER",
            "confidence_threshold": 0.6
        }
    ).json()

    entities = ner_response["entities"]

    # Step 2: Classify based on entity composition
    entity_types = {e["type"] for e in entities}
    org_count = sum(1 for e in entities if e["type"] == "ORG")
    per_count = sum(1 for e in entities if e["type"] == "PER")
    loc_count = sum(1 for e in entities if e["type"] == "LOC")

    if org_count > per_count and org_count > loc_count:
        category = "business"
    elif per_count > org_count:
        category = "biographical"
    elif loc_count > per_count:
        category = "geographic"
    else:
        category = "general"

    return {
        "entities": entities,
        "category": category,
        "entity_summary": {
            "persons": per_count,
            "organizations": org_count,
            "locations": loc_count
        }
    }
```

## Batch NER

### Processing Multiple Texts

```python
import requests

def batch_ner(texts, model="dslim/bert-base-NER", confidence_threshold=0.5):
    results = []
    for text in texts:
        response = requests.post(
            "http://localhost:14345/v1/ner",
            json={
                "text": text,
                "model": model,
                "confidence_threshold": confidence_threshold
            }
        )
        results.append(response.json())
    return results

texts = [
    "Apple CEO Tim Cook announced new products in Cupertino.",
    "The European Union approved the merger between Deutsche Bank and Commerzbank.",
    "Dr. Sarah Chen published her research at Stanford University."
]

results = batch_ner(texts)
for i, result in enumerate(results):
    print(f"Text {i+1}: {len(result['entities'])} entities found")
    for entity in result["entities"]:
        print(f"  {entity['type']}: {entity['text']} ({entity['confidence']:.2f})")
```

### Aggregating Entities Across Documents

```python
from collections import defaultdict

def aggregate_entities(batch_results):
    entity_index = defaultdict(lambda: {"count": 0, "types": set(), "max_confidence": 0.0})

    for result in batch_results:
        for entity in result["entities"]:
            key = entity["text"].lower()
            entry = entity_index[key]
            entry["count"] += 1
            entry["types"].add(entity["type"])
            entry["max_confidence"] = max(entry["max_confidence"], entity["confidence"])

    # Convert sets to lists for serialization
    return {
        k: {**v, "types": list(v["types"])}
        for k, v in sorted(entity_index.items(), key=lambda x: -x[1]["count"])
    }
```

## Handling Overlapping Entities

Some models may produce overlapping entity spans. LlamaFarm resolves these using the `aggregate_strategy` parameter.

| Strategy | Behavior | Best For |
|----------|----------|----------|
| `simple` | Merge adjacent sub-word tokens, keep highest confidence on overlap | General use (default) |
| `first` | Use the label of the first sub-word token | Consistent labeling |
| `average` | Average confidence across sub-word tokens | Balanced confidence scores |
| `max` | Use the highest confidence label for each token | Maximizing precision |

**Example with overlapping resolution:**

```bash
curl -X POST http://localhost:14345/v1/ner \
  -H "Content-Type: application/json" \
  -d '{
    "text": "New York City Mayor Eric Adams spoke at the United Nations.",
    "model": "dslim/bert-base-NER",
    "aggregate_strategy": "simple"
  }'
```

In this example, "New York City" is correctly recognized as a single `LOC` entity rather than being split into separate tokens.

## Error Handling

| Error | HTTP Status | Cause | Resolution |
|-------|-------------|-------|------------|
| `invalid_request` | 400 | Missing or empty `text` field | Provide non-empty text |
| `model_not_found` | 404 | Requested model not available | Check available models or use default |
| `text_too_long` | 400 | Text exceeds model max length (typically 512 tokens) | Split text into smaller segments |
| `processing_error` | 500 | Model inference failed | Check server logs, try a different model |

**Example error response:**

```json
{
  "error": {
    "code": "text_too_long",
    "message": "Input text exceeds maximum length of 512 tokens (received 1024 tokens). Split into smaller segments.",
    "details": {
      "max_tokens": 512,
      "received_tokens": 1024
    }
  }
}
```

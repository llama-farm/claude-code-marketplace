---
description: End-to-end ML patterns and use cases. Complete pipelines for IoT monitoring, fraud detection, and document intelligence using LlamaFarm ML features.
---

# ML Use Cases Skill

Reference implementations for common ML patterns built on LlamaFarm.

## When to Load

Load this skill when the user:
- Asks for an end-to-end ML example
- Wants to build a monitoring pipeline
- Asks about fraud detection patterns
- Needs a document processing pipeline
- Wants to combine multiple ML features
- Asks "how do I use anomaly detection for X?"

## Use Case Catalog

| Use Case | ML Features Used | Complexity |
|----------|-----------------|------------|
| IoT Monitoring | Polars buffers → Rolling features → Streaming anomaly | Medium |
| Fraud Detection | Streaming pre-screen → Batch analysis → Ensemble | High |
| Document Intelligence | OCR → NER → Classification → RAG | Medium |

## Pattern Selection Guide

1. **Real-time monitoring?** → IoT Monitoring pattern
2. **Transaction scoring?** → Fraud Detection pattern
3. **Document processing?** → Document Intelligence pattern
4. **Custom pipeline?** → Combine patterns as building blocks

## Quick Reference

### IoT Monitoring Pipeline
```
Raw metrics → Polars buffer → Rolling features → Streaming ecod → Alerts
```

### Fraud Detection Pipeline
```
Transaction → Streaming pre-screen → Daily batch ensemble → Risk score
```

### Document Intelligence Pipeline
```
Document → OCR → NER → Classification → RAG index → Query
```

## Progressive Disclosure

For detailed implementations:
- `iot-monitoring.md` - Complete IoT pipeline with Polars + streaming anomaly
- `fraud-detection.md` - Hybrid real-time + batch fraud scoring
- `document-intelligence.md` - OCR → NER → classification → RAG pipeline

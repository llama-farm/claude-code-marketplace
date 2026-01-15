# Datasets API

Programmatic dataset management for document upload, processing, and organization.

## Overview

The Datasets API allows you to:
- Create and manage datasets
- Upload documents
- Trigger processing pipelines
- Monitor ingestion progress

## Endpoints

| Method | Endpoint | Description |
|--------|----------|-------------|
| GET | `/datasets` | List all datasets |
| POST | `/datasets` | Create dataset |
| GET | `/datasets/{name}` | Get dataset details |
| DELETE | `/datasets/{name}` | Delete dataset |
| POST | `/datasets/{name}/data` | Upload file |
| GET | `/datasets/{name}/data` | List files |
| DELETE | `/datasets/{name}/data/{file}` | Delete file |
| POST | `/datasets/{name}/actions` | Trigger action |
| GET | `/datasets/{name}/status` | Get processing status |

All endpoints are under `/v1/projects/{namespace}/{project}/`.

---

## Create Dataset

```http
POST /v1/projects/{namespace}/{project}/datasets
Content-Type: application/json
```

**Request:**
```json
{
  "name": "research-papers",
  "database": "main_db",
  "data_processing_strategy": "pdf_processor"
}
```

**Python Example:**
```python
import requests

def create_dataset(name, database, strategy, project="my-project"):
    response = requests.post(
        f"http://localhost:8000/v1/projects/default/{project}/datasets",
        json={
            "name": name,
            "database": database,
            "data_processing_strategy": strategy
        }
    )
    response.raise_for_status()
    return response.json()

# Create a dataset for PDFs
create_dataset("papers", "main_db", "pdf_processor")

# Create a dataset for markdown docs
create_dataset("docs", "main_db", "markdown_processor")
```

---

## Upload Files

```http
POST /v1/projects/{namespace}/{project}/datasets/{dataset}/data
Content-Type: multipart/form-data
```

**curl Example:**
```bash
curl -X POST \
  "http://localhost:8000/v1/projects/default/my-project/datasets/papers/data" \
  -F "file=@research_paper.pdf"
```

**Python Example - Single File:**
```python
def upload_file(dataset, file_path, project="my-project"):
    with open(file_path, "rb") as f:
        response = requests.post(
            f"http://localhost:8000/v1/projects/default/{project}/datasets/{dataset}/data",
            files={"file": f}
        )
    response.raise_for_status()
    return response.json()

upload_file("papers", "./documents/paper.pdf")
```

**Python Example - Batch Upload:**
```python
from pathlib import Path
from concurrent.futures import ThreadPoolExecutor

def upload_batch(dataset, directory, pattern="*.pdf", project="my-project"):
    files = list(Path(directory).glob(pattern))
    results = []

    def upload_one(file_path):
        try:
            result = upload_file(dataset, str(file_path), project)
            return {"file": file_path.name, "status": "success"}
        except Exception as e:
            return {"file": file_path.name, "status": "error", "error": str(e)}

    # Upload 5 files in parallel
    with ThreadPoolExecutor(max_workers=5) as executor:
        results = list(executor.map(upload_one, files))

    return results

# Upload all PDFs from a directory
results = upload_batch("papers", "./documents/", "*.pdf")
print(f"Uploaded {len([r for r in results if r['status'] == 'success'])} files")
```

---

## Process Dataset

After uploading files, trigger the ingestion pipeline:

```http
POST /v1/projects/{namespace}/{project}/datasets/{dataset}/actions
Content-Type: application/json
```

**Request:**
```json
{
  "action": "process"
}
```

**Python Example:**
```python
def process_dataset(dataset, project="my-project"):
    response = requests.post(
        f"http://localhost:8000/v1/projects/default/{project}/datasets/{dataset}/actions",
        json={"action": "process"}
    )
    response.raise_for_status()
    return response.json()

# Start processing
result = process_dataset("papers")
print(f"Task ID: {result['task_id']}")
```

---

## Monitor Progress

Check processing status:

```http
GET /v1/projects/{namespace}/{project}/datasets/{dataset}/status
```

**Response:**
```json
{
  "status": "processing",
  "progress": {
    "files_total": 25,
    "files_processed": 18,
    "files_failed": 1,
    "chunks_created": 1560,
    "percent_complete": 72
  },
  "started_at": "2024-01-14T10:00:00Z",
  "errors": [
    {
      "file": "corrupted.pdf",
      "error": "Unable to parse PDF: file is encrypted"
    }
  ]
}
```

**Python Example - Wait for Completion:**
```python
import time

def wait_for_processing(dataset, project="my-project", timeout=600):
    start_time = time.time()

    while time.time() - start_time < timeout:
        response = requests.get(
            f"http://localhost:8000/v1/projects/default/{project}/datasets/{dataset}/status"
        )
        status = response.json()

        if status["status"] == "completed":
            print(f"Processing complete: {status['progress']['chunks_created']} chunks")
            return status

        if status["status"] == "failed":
            raise Exception(f"Processing failed: {status.get('error')}")

        # Show progress
        progress = status.get("progress", {})
        percent = progress.get("percent_complete", 0)
        print(f"Processing: {percent}% complete")

        time.sleep(5)

    raise TimeoutError("Processing did not complete in time")

# Wait for processing to finish
wait_for_processing("papers")
```

---

## Complete Workflow

End-to-end example of uploading and processing documents:

```python
import requests
from pathlib import Path
import time

class LlamaFarmClient:
    def __init__(self, base_url="http://localhost:8000", namespace="default", project="my-project"):
        self.base_url = f"{base_url}/v1/projects/{namespace}/{project}"

    def create_dataset(self, name, database, strategy):
        response = requests.post(
            f"{self.base_url}/datasets",
            json={
                "name": name,
                "database": database,
                "data_processing_strategy": strategy
            }
        )
        if response.status_code == 409:
            print(f"Dataset '{name}' already exists")
            return
        response.raise_for_status()
        print(f"Created dataset: {name}")

    def upload_files(self, dataset, directory, pattern="*"):
        files = list(Path(directory).glob(pattern))
        print(f"Uploading {len(files)} files to '{dataset}'")

        for file_path in files:
            with open(file_path, "rb") as f:
                response = requests.post(
                    f"{self.base_url}/datasets/{dataset}/data",
                    files={"file": f}
                )
            response.raise_for_status()
            print(f"  Uploaded: {file_path.name}")

    def process(self, dataset):
        response = requests.post(
            f"{self.base_url}/datasets/{dataset}/actions",
            json={"action": "process"}
        )
        response.raise_for_status()
        task_id = response.json()["task_id"]
        print(f"Processing started: {task_id}")
        return task_id

    def wait_for_completion(self, dataset, timeout=600):
        start = time.time()
        while time.time() - start < timeout:
            response = requests.get(f"{self.base_url}/datasets/{dataset}/status")
            status = response.json()

            if status["status"] == "completed":
                chunks = status["progress"]["chunks_created"]
                print(f"Complete! {chunks} chunks indexed")
                return status

            if status["status"] == "failed":
                raise Exception(f"Failed: {status.get('error')}")

            percent = status.get("progress", {}).get("percent_complete", 0)
            print(f"Progress: {percent}%")
            time.sleep(5)

        raise TimeoutError("Processing timeout")

# Usage
client = LlamaFarmClient(project="my-project")

# Create dataset
client.create_dataset("research", "main_db", "pdf_processor")

# Upload documents
client.upload_files("research", "./papers/", "*.pdf")

# Process and wait
client.process("research")
client.wait_for_completion("research")
```

---

## List Files in Dataset

```http
GET /v1/projects/{namespace}/{project}/datasets/{dataset}/data
```

**Response:**
```json
{
  "files": [
    {
      "name": "paper1.pdf",
      "size": 1234567,
      "uploaded_at": "2024-01-14T10:00:00Z",
      "status": "processed",
      "chunks": 45
    },
    {
      "name": "paper2.pdf",
      "size": 2345678,
      "uploaded_at": "2024-01-14T10:05:00Z",
      "status": "pending"
    }
  ]
}
```

---

## Delete Dataset

```http
DELETE /v1/projects/{namespace}/{project}/datasets/{dataset}
```

**Options:**
- `?delete_vectors=true` - Also delete vectors from database

```python
def delete_dataset(dataset, delete_vectors=True, project="my-project"):
    response = requests.delete(
        f"http://localhost:8000/v1/projects/default/{project}/datasets/{dataset}",
        params={"delete_vectors": delete_vectors}
    )
    response.raise_for_status()
```

---

## Error Handling

```python
def safe_upload(dataset, file_path, project="my-project"):
    try:
        with open(file_path, "rb") as f:
            response = requests.post(
                f"http://localhost:8000/v1/projects/default/{project}/datasets/{dataset}/data",
                files={"file": f},
                timeout=30
            )
        response.raise_for_status()
        return {"status": "success", "file": file_path}

    except requests.exceptions.HTTPError as e:
        error = e.response.json().get("error", {})
        return {
            "status": "error",
            "file": file_path,
            "code": error.get("code"),
            "message": error.get("message")
        }

    except Exception as e:
        return {"status": "error", "file": file_path, "message": str(e)}
```

Common errors:
| Code | Cause | Solution |
|------|-------|----------|
| `dataset_not_found` | Dataset doesn't exist | Create dataset first |
| `file_too_large` | File exceeds limit | Split or compress file |
| `invalid_file_type` | Unsupported format | Check parser patterns |
| `processing_in_progress` | Already processing | Wait for completion |

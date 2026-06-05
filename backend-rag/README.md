# Backend RAG (Jetson FAISS)

## Overview

Lightweight FAISS-based RAG backend running on Jetson with Docker/GPU support. Single active backend service runs on port **8001**.

```text
http://localhost:8001
```

## Architecture

```text
Frontend / Open WebUI
  -> FastAPI backend-rag (`src/app.py`) on :8001
  -> FAISS text search (`src/rag_service.py`)
  -> Stored source documents in `storage/`
  -> Docling JSON picture extraction + optional Ollama vision description
```

## Picture-aware workflow

FAISS is used only for text search and `source_file` discovery. Pictures are retrieved from the stored Docling JSON source file, not from FAISS.

Picture crops are generated from:

```text
pictures[].prov[].page_no
pictures[].prov[].bbox
pages[page_no].image.uri
```

Cropped pictures can be sent to Ollama vision model `gemma4:e4b`.

## API

```text
GET  /
GET  /api/rag-status
POST /api/clear-rag
POST /api/upload
GET  /api/upload/progress/{job_id}
POST /api/search
GET  /api/document/{file_id}
POST /api/pictures
POST /api/describe-pictures
```

## List picture metadata

```bash
curl -s http://localhost:8001/api/pictures \
  -H 'Content-Type: application/json' \
  -d '{"source_file":"0b3821c1-7632-4d64-b6e8-590977832ef4.json","query":"statue","limit":5}' \
| python3 -m json.tool
```

## Describe cropped picture with Gemma4

```bash
curl -s http://localhost:8001/api/describe-pictures \
  -H 'Content-Type: application/json' \
  -d '{"source_file":"0b3821c1-7632-4d64-b6e8-590977832ef4.json","query":"statue","limit":1,"model":"gemma4:e4b"}' \
| python3 -m json.tool
```

## Expected flow

```text
/api/search finds source_file
-> /api/describe-pictures loads stored Docling JSON
-> reads pictures[]
-> crops page image using bbox
-> sends crop to Ollama gemma4:e4b
-> returns grounded visual evidence
```

## Requirements

Python dependencies are in `requirements.txt`.

Picture description requires:

```text
Pillow
requests
```

Ollama must be running with a vision-capable model:

```bash
ollama list | grep gemma4
```

## Local run

```bash
cd /data/projects/chat-llama-nemotron/backend-rag
source venv/bin/activate
python src/app.py
```

## Docker run Jetson

```bash
docker run -d \
  --name backend-rag \
  --restart unless-stopped \
  --runtime nvidia \
  -p 8001:8001 \
  -v /home/hp/projects/chat-llama-nemotron/backend-rag:/home/hp/projects/chat-llama-nemotron/backend-rag \
  backend-rag:jetson-faiss-host
```

## Docker Compose

```bash
docker compose up -d
```

## Storage

- Uploaded source documents are stored under `storage/`.
- Stored Docling JSON files are served through `/api/document/{file_id}`.
- Avoid container-only storage for FAISS or uploaded documents.

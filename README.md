# Multi-Agent MultiModal Agentic RAG Project

Brand Guardian AI pipeline for auditing video content against compliance rules using a multimodal, agentic RAG workflow.

## What this project does

- Accepts a video URL.
- Indexes the video content (speech/OCR metadata) through Azure Video Indexer.
- Retrieves relevant compliance context from Azure AI Search.
- Uses Azure OpenAI to produce structured compliance findings.
- Returns a `PASS/FAIL` result with issue details through API or CLI flow.

## Project structure

- `backend/src/api/server.py` - FastAPI API (`/audit`, `/health`)
- `backend/src/graph/` - LangGraph workflow state and nodes
- `backend/src/services/video_indexer.py` - Video indexing integration
- `azure_functions/function_app.py` - Azure Functions entrypoint
- `main.py` - Local CLI simulation runner

## Requirements

- Python 3.12+
- `uv` package manager
- Azure services configured in environment variables

## Quick start

```bash
uv sync
uv run uvicorn backend.src.api.server:app --reload
```

API docs will be available at `http://localhost:8000/docs`.

## Example request

```bash
curl -X POST "http://localhost:8000/audit" ^
  -H "Content-Type: application/json" ^
  -d "{\"video_url\":\"https://youtu.be/dT7S75eYhcQ\"}"
```

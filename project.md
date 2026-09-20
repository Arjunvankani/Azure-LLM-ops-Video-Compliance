# Project Details

## Brand Guardian: Azure LLMOps Video Compliance Project

## Overview

Brand Guardian is an AI-powered compliance auditing system for video content. It analyzes a YouTube video, extracts transcript and on-screen text, retrieves relevant policy and advertising guidance from a vector database, and uses an LLM to flag potential compliance or misleading-claim issues.

The goal is to help marketing, legal, and brand teams review video content before publication and detect risky claims early.

---

## Business Problem

Marketing teams often publish video content that includes promotional claims, product promises, or comparisons without a clear compliance check. Manually reviewing every video is slow and error-prone.

This project automates the review workflow by:

- downloading the video
- extracting transcript and OCR text
- indexing policy documents into a retrieval system
- searching for the most relevant compliance rule set
- using LLM reasoning to find violations and summarize risk

---

## System Architecture

The application is built as a retrieval-augmented generation (RAG) pipeline with workflow orchestration.

### Core flow

1. User provides a YouTube URL.
2. The service downloads the video locally.
3. The video is uploaded to Azure Video Indexer.
4. Transcript and OCR text are extracted.
5. The transcript and OCR text are used as a query against Azure AI Search.
6. Relevant compliance documents are retrieved from the indexed knowledge base.
7. Azure OpenAI evaluates the content against those rules.
8. The system returns compliance findings, status, and summary.

High-level flow:

User Input -> FastAPI API / CLI -> LangGraph Workflow -> Video Indexer -> Azure AI Search -> Azure OpenAI -> Compliance Report

---

## Project Structure

```text
ComplianceQAPipeline/
├── main.py
├── pyproject.toml
├── README.md
├── project.md
├── AWS_DEPLOYMENT.md
├── .env.example
├── .gitignore
├── backend/
│   ├── data/
│   ├── scripts/
│   │   ├── explanation.txt
│   │   └── index_documents.py
│   ├── src/
│   │   ├── api/
│   │   │   ├── __init__.py
│   │   │   ├── server.py
│   │   │   └── telemetry.py
│   │   ├── graph/
│   │   │   ├── __init__.py
│   │   │   ├── nodes.py
│   │   │   ├── state.py
│   │   │   └── workflow.py
│   │   ├── services/
│   │   │   ├── __init__.py
│   │   │   └── video_indexer.py
│   │   └── __init__.py
│   └── tests/
```

---

## File-by-File Documentation

### main.py

This is the CLI entry point for running a workflow simulation.

Responsibilities:

- load environment variables
- generate a unique session id
- prepare the input state
- call the LangGraph workflow
- print the final compliance report

### backend/src/graph/state.py

Defines the state schema for the workflow, including:

- video URL and ID
- transcript and OCR data
- metadata
- compliance results
- final status
- final report
- errors

This acts as the memory for the graph execution.

### backend/src/graph/workflow.py

Creates the LangGraph execution flow.

The workflow is structured as:

START -> indexer -> auditor -> END

### backend/src/graph/nodes.py

Contains the two primary graph nodes:

#### index_video_node

- validates the URL
- downloads the YouTube video
- uploads the file to Azure Video Indexer
- waits for processing to complete
- extracts transcript and OCR data

#### audit_content_node

- loads LLM and embedding clients
- performs similarity search in Azure AI Search
- retrieves relevant rules
- analyzes transcript and OCR text
- returns compliance findings and final summary

### backend/src/services/video_indexer.py

Handles interaction with Azure Video Indexer.

Core features:

- Azure token retrieval
- Video Indexer token generation
- YouTube download
- file upload
- polling and processing wait
- transcript/OCR extraction

### backend/src/api/server.py

FastAPI application used to expose the workflow to external clients.

Endpoints:

- POST /audit
- GET /health

The API accepts a video URL and returns structured compliance results.

### backend/src/api/telemetry.py

Configures Azure Monitor OpenTelemetry for application observability.

### backend/scripts/index_documents.py

Indexes compliance PDFs into Azure AI Search.

Responsibilities:

- load PDF documents
- split them into chunks
- create vector embeddings
- store them in the search index

This is the retrieval base used by the auditor.

### backend/data

Stores compliance-related PDF files used to build the knowledge base.

### backend/tests

Contains future automated tests for workflow validation and API testing.

---

## Workflow Details

### 1. Input Stage

The system accepts a video URL from:

- CLI main.py
- FastAPI POST /audit endpoint

### 2. Indexing Stage

The system:

- downloads the video
- uploads to Azure Video Indexer
- waits for processing
- extracts transcript and OCR

### 3. Retrieval Stage

The transcript and OCR text are turned into a query against Azure AI Search. The vector database returns relevant compliance rule fragments from the PDF knowledge base.

### 4. Reasoning Stage

The LLM receives:

- transcript text
- OCR text
- retrieved compliance rules
- metadata

Then it returns structured JSON with:

- compliance_results
- final_status
- final_report

---

## Output Format

Example response:

```json
{
  "session_id": "uuid",
  "video_id": "vid_12345678",
  "status": "FAIL",
  "final_report": "Video contains multiple compliance risks related to unsupported medical and performance claims.",
  "compliance_results": [
    {
      "category": "Claim Validation",
      "severity": "CRITICAL",
      "description": "The video makes an unsupported absolute performance claim."
    }
  ]
}
```

---

## Environment Setup

The project uses environment variables in a `.env` file.

Typical configuration:

```env
AZURE_OPENAI_ENDPOINT=
AZURE_OPENAI_API_KEY=
AZURE_OPENAI_CHAT_DEPLOYMENT=
AZURE_OPENAI_EMBEDDING_DEPLOYMENT=text-embedding-3-small
AZURE_OPENAI_API_VERSION=2024-02-01

AZURE_SEARCH_ENDPOINT=
AZURE_SEARCH_API_KEY=
AZURE_SEARCH_INDEX_NAME=

AZURE_VI_ACCOUNT_ID=
AZURE_VI_LOCATION=
AZURE_SUBSCRIPTION_ID=
AZURE_RESOURCE_GROUP=
AZURE_VI_NAME=

APPLICATIONINSIGHTS_CONNECTION_STRING=
```

A template file is provided in `.env.example`.

---

## Quick Start

### Install dependencies

```bash
uv sync
```

### Build the knowledge base

```bash
uv run python backend/scripts/index_documents.py
```

### Run the CLI simulation

```bash
uv run python main.py
```

### Run the API

```bash
uv run uvicorn backend.src.api.server:app --reload
```

Open the app:

- Swagger UI: http://localhost:8000/docs
- Health check: http://localhost:8000/health

---

## AWS Deployment

This project is currently Azure-first, but the same architecture can be deployed on AWS.

Recommended AWS equivalents:

- Amazon Bedrock for LLM inference and embeddings
- Amazon OpenSearch for vector retrieval
- Amazon S3 for files and documents
- Amazon Rekognition Video for media analysis
- Amazon CloudWatch for logs and monitoring
- IAM + Secrets Manager for credentials

For more details, see [AWS_DEPLOYMENT.md](AWS_DEPLOYMENT.md).

---

## Key Technologies

- Python 3.12
- FastAPI
- LangGraph
- LangChain
- Azure OpenAI
- Azure AI Search
- Azure Video Indexer
- Azure Monitor
- Pydantic
- yt-dlp

---

## Use Cases

- brand compliance review
- advertising claim auditing
- regulated marketing review
- AI-assisted governance and risk detection
- policy-based video screening

---

## Future Enhancements

- add PostgreSQL storage for audit history
- add authentication and role-based access
- add dashboard UI with Streamlit or React
- support multiple video sources beyond YouTube
- deploy to AWS or hybrid cloud
- improve retry logic and event-driven job processing

---

## Summary

Brand Guardian is a practical AI-based compliance workflow that combines document retrieval, video analysis, and LLM-based reasoning to automate the review of marketing content. It provides a strong base for compliance automation and can be extended for enterprise-scale deployment.

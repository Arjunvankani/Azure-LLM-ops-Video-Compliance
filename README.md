# Brand Guardian Compliance QA Pipeline

## Overview

Brand Guardian is an AI-powered video compliance auditing system. It analyzes a YouTube video, extracts transcript and on-screen text, searches a knowledge base of compliance rules, and uses a Large Language Model (LLM) to identify violations such as misleading claims, compliance issues, or policy breaches.

This project combines:

- FastAPI backend for REST API access
- LangGraph workflow orchestration
- Azure AI Search for retrieval of regulations and policy guidance
- Azure OpenAI for compliance reasoning
- Azure Video Indexer for transcription and OCR extraction
- Azure Monitor telemetry for application observability

The output is a structured compliance report with findings, categories, severity, and summary text.

---

## Business Goal

The system is designed to help brands and compliance teams identify risky advertising or promotional content before it is published. It acts as an automated compliance reviewer for video-based media.

Typical use cases:

- review marketing videos for claim validation issues
- detect unapproved product claims or unsupported guarantees
- inspect transcripts and visible text for regulatory concerns
- provide structured, explainable audit results for human review

---

## Project Architecture

The project follows a simple but powerful AI pipeline:

1. User submits a video URL through the API or CLI.
2. The workflow downloads the YouTube video.
3. The video is uploaded to Azure Video Indexer.
4. Transcript, OCR text, and metadata are extracted.
5. The extracted text is used as a search query to Azure AI Search.
6. Relevant compliance rules are retrieved from the indexed PDF knowledge base.
7. The LLM analyzes the transcript and rule context.
8. The system returns a compliance result with status, findings, and summary.

High-level flow:

User Input -> FastAPI API / CLI -> LangGraph Workflow -> Video Indexer -> Azure Search -> Azure OpenAI -> Compliance Report

---

## Folder and File Structure

```text
ComplianceQAPipeline/
├── main.py
├── pyproject.toml
├── README.md
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

## File-by-File Explanation

### 1) main.py

This is the main entry point for running the workflow in a local CLI simulation.

Responsibilities:

- load environment variables from the .env file
- create a unique session ID for each audit run
- prepare the initial input state
- call the LangGraph workflow
- print compliance results to the terminal

Important concept:

- It acts like a runner or orchestrator for the end-to-end compliance audit pipeline.

Example flow inside this file:

- generate UUID
- define video URL and metadata
- invoke the graph with app.invoke(initial_inputs)
- print final_status, issues, and final_report

---

### 2) pyproject.toml

This is the Python project configuration file.

It defines:

- project metadata
- Python version requirement
- runtime dependencies used by the system

Key dependencies include:

- FastAPI for the API layer
- LangGraph and LangChain for orchestration and AI workflow
- OpenAI / Azure OpenAI packages
- Azure Search client
- Azure Blob, Azure Monitor, and Azure Identity
- Redis, SQLAlchemy, psycopg2, pandas, streamlit, uvicorn
- yt-dlp for downloading YouTube videos

This file is the central dependency manifest for the application.

---

### 3) backend/src/graph/state.py

This file defines the workflow state schema using TypedDict.

It contains fields such as:

- video_url
- video_id
- local_file_path
- video_metadata
- transcript
- ocr_text
- compliance_results
- final_status
- final_report
- errors

Purpose:

- ensures the LangGraph nodes pass consistent data
- tracks all intermediate and final information
- allows the system to append compliance findings and errors without losing prior state

The state is the shared memory of the workflow.

---

### 4) backend/src/graph/workflow.py

This file builds the LangGraph workflow.

Workflow definition:

- start node: indexer
- second node: auditor
- final edge: end

Key idea:

- indexer processes the video and extracts insights
- auditor checks the content against retrieved regulatory guidance and policy rules

This creates a simple DAG (Directed Acyclic Graph):

START -> indexer -> auditor -> END

---

### 5) backend/src/graph/nodes.py

This file contains the actual workflow logic for each node.

#### index_video_node

Responsibilities:

- validate the input URL
- download the video from YouTube
- upload the video to Azure Video Indexer
- wait for processing to finish
- extract transcript, OCR text, and metadata
- return the processed state data

If processing fails, it returns an error state and sets final_status to FAIL.

#### audit_content_node

Responsibilities:

- read transcript and OCR data
- initialize LLM and embeddings clients
- query Azure AI Search for relevant rules
- build a compliance evaluation prompt
- call the LLM to parse the transcript and identify rule violations
- return compliance_results, final_status, and final_report

This node is the actual compliance reasoning engine.

---

### 6) backend/src/services/video_indexer.py

This is the service module that handles Azure Video Indexer interactions.

Responsibilities:

- get Azure management access tokens
- generate Video Indexer access token
- download a local YouTube video
- upload the local file to Azure Video Indexer
- poll for processing completion
- parse transcript, OCR, and summary metadata

Core methods:

- get_access_token()
- get_account_token()
- download_youtube_video()
- upload_video()
- wait_for_processing()
- extract_data()

This file is the bridge between raw video input and AI-based analysis.

---

### 7) backend/src/api/server.py

This is the FastAPI application that exposes the project as a backend API.

Main endpoints:

- POST /audit
- GET /health

POST /audit flow:

- receives a video_url in JSON request
- generates a session_id and video_id
- invokes the LangGraph workflow
- returns a structured compliance report

The server uses Pydantic models to validate request and response formats.

This file turns the workflow into a usable web service.

---

### 8) backend/src/api/telemetry.py

This file initializes Azure Monitor OpenTelemetry integration.

Responsibilities:

- read APPLICATIONINSIGHTS_CONNECTION_STRING from environment variables
- configure Azure Monitor tracing
- capture API telemetry and app instrumentation automatically

This is the observability layer for the application.

---

### 9) backend/scripts/index_documents.py

This script indexes policy and regulatory PDF documents into Azure AI Search.

Responsibilities:

- read PDF files from backend/data
- split documents into chunks using RecursiveCharacterTextSplitter
- create embeddings using Azure OpenAI embeddings model
- upload vectors to Azure AI Search index

This is the project’s knowledge base builder.

Why this matters:

- without this step, the LLM would not have a relevant policy base to audit against
- it creates the retrieval layer used by the auditor node

---

### 10) backend/scripts/explanation.txt

This file contains a plain-English explanation of how the system works and documents the project conceptually.

It explains the transformation from:

- raw video content
- to transcript/OCR extraction
- to policy retrieval
- to compliance judgment

This acts as a supporting design note for the architecture and AI pipeline.

---

### 11) backend/data/

This folder is intended to store PDF regulatory/compliance rule documents that are used to build the knowledge base.

Examples of content likely stored here:

- FTC guidance
- platform policy references
- brand compliance rules
- advertising regulations

The script expects to find PDF documents here and index them into Azure AI Search.

---

### 12) backend/tests/

This folder is intended for automated tests and validation logic.

It should later contain:

- workflow validation tests
- API tests
- service integration tests
- rule retrieval tests
- compliance result validation

---

## End-to-End Runtime Flow

### 1. Input
A user sends a request like:

```json
{
  "video_url": "https://youtu.be/dT7S75eYhcQ"
}
```

### 2. API Layer
The FastAPI server receives the request and creates:

- unique session_id
- video_id
- initial graph input state

### 3. Workflow Execution
The LangGraph workflow runs:

- indexer node downloads and processes the video
- auditor node retrieves rules and analyzes content

### 4. Retrieval and Reasoning
The AI uses:

- transcript and OCR text
- vector search against Azure AI Search
- LLM rule-based reasoning

### 5. Output
The final response is structured as:

```json
{
  "session_id": "...",
  "video_id": "...",
  "status": "FAIL",
  "final_report": "...",
  "compliance_results": [
    {
      "category": "Claim Validation",
      "severity": "CRITICAL",
      "description": "..."
    }
  ]
}
```

---

## Environment Variables

The application depends on environment variables stored in a .env file.

Typical variables include:

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

These values are required for:

- OpenAI model access
- vector search setup
- Azure Video Indexer authentication
- telemetry and monitoring

---

## How to Run the Project

### Install dependencies

```bash
uv sync
```

### Run the CLI simulation

```bash
uv run python main.py
```

### Run the FastAPI backend

```bash
uv run uvicorn backend.src.api.server:app --reload
```

Then open:

- http://localhost:8000/docs for Swagger UI
- http://localhost:8000/health for health check

---

## Build the Knowledge Base

Before the auditor can work correctly, the PDF rules should be indexed.

Run:

```bash
uv run python backend/scripts/index_documents.py
```

This loads the PDF documents in backend/data, splits them into chunks, and stores them in Azure AI Search.

---

## AWS Services Needed (for deployment or migration)

This project is currently built on Microsoft Azure services. If you want to run the same solution on Amazon Web Services (AWS), these are the main AWS services you would need.

| Current Azure Service | AWS Equivalent | Purpose |
|---|---|---|
| Azure OpenAI | Amazon Bedrock / SageMaker endpoints | LLM reasoning and embeddings |
| Azure AI Search | Amazon OpenSearch Service / OpenSearch Serverless | vector search over compliance rules |
| Azure Video Indexer | Amazon Rekognition Video / MediaConvert / Elemental services | video processing, transcription, OCR |
| Azure Monitor | Amazon CloudWatch | logs, telemetry, monitoring |
| Azure Identity / App Config | IAM + Secrets Manager + SSM Parameter Store | secure credentials and access control |
| Azure Blob Storage | Amazon S3 | file storage for uploaded videos and PDFs |
| FastAPI app hosting | EC2, ECS, EKS, or App Runner | host the API backend |
| Load balancing / API exposure | Application Load Balancer / API Gateway | public API entry route |
| Container management | ECS / EKS / Fargate | container orchestration |
| CI/CD | CodePipeline / CodeBuild / GitHub Actions | automated deployment |

### Recommended AWS Architecture

For an AWS version of this solution, a practical architecture would be:

- API hosted on ECS or EC2
- video files stored in S3
- video transcription/OCR from Amazon Rekognition or AWS Media services
- embeddings stored in OpenSearch Serverless
- LLM inference via Amazon Bedrock
- logs and metrics sent to CloudWatch
- IAM + Secrets Manager for credentials

This would mirror the Azure architecture while using AWS-native services.

> Important: The current project is Azure-first, but the concepts are portable. The workflow itself is cloud-agnostic as long as the equivalent services are configured correctly.

---

## Key Design Concepts

### 1. Retrieval-Augmented Generation (RAG)

The system does not rely only on the model’s memory. Instead, it first retrieves relevant compliance rules from a vector database and then sends those rules to the LLM.

This improves:

- precision
- factual grounding
- policy relevance
- traceability

### 2. LangGraph Orchestration

The workflow is explicit and structured:

- easier debugging
- better traceability
- manageable state transitions
- clear AI pipeline logic

### 3. Modular Service Design

The project separates:

- API handling
- workflow orchestration
- video processing
- telemetry
- document indexing

This keeps the codebase maintainable and scalable.

---

## Strengths of the Project

- modular architecture
- reusable AI workflow
- cloud integration with Azure services
- API-ready for front-end integration
- policy-guided compliance analysis
- uses retrieval-based reasoning for better accuracy

---

## Possible Future Enhancements

- add user authentication and authorization
- add database storage for audit history
- add dashboard UI with Streamlit or React frontend
- support multiple video sources beyond YouTube
- add policy version tracking and legal-document metadata
- integrate with AWS or multi-cloud deployment options
- add automated retry logic and queue-based processing
- store compliance results in PostgreSQL or Redis

---

## Summary

Brand Guardian is an AI compliance pipeline that converts a YouTube video into a structured compliance assessment by combining:

- video ingestion
- transcript and OCR extraction
- document retrieval from a vector index
- LLM-based reasoning
- API-based workflow execution

It is a practical example of a production-style RAG architecture for media compliance analysis.

---

## Quick Start Summary

```bash
uv sync
uv run python backend/scripts/index_documents.py
uv run python main.py
uv run uvicorn backend.src.api.server:app --reload
```

If you want, the next step can be to add:

1. a full Streamlit dashboard
2. a PostgreSQL audit history storage layer
3. a deployment configuration for AWS or Azure
4. a production-ready Docker setup

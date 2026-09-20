# AWS Deployment Guide for Brand Guardian Video Compliance Project

This guide explains how to deploy the Brand Guardian compliance pipeline on Amazon Web Services (AWS) using AWS-native services that match the Azure architecture in this project.

---

## 1. Overview

The application is an AI-powered video compliance auditor that:

- accepts a YouTube video URL
- downloads the video
- extracts transcript and OCR text
- retrieves relevant compliance rules from a knowledge base
- uses an LLM to determine if the content violates brand or advertising rules
- returns a structured compliance report

The current project is Azure-first, but it can be migrated to AWS by replacing each Azure service with the equivalent AWS service.

---

## 2. AWS Architecture

A recommended AWS architecture is:

- API Layer: Amazon ECS / EC2 / App Runner
- Video Storage: Amazon S3
- Video Analysis: Amazon Rekognition Video or AWS Media services
- LLM: Amazon Bedrock
- Vector Search: Amazon OpenSearch Serverless or Amazon OpenSearch Service
- Secrets: AWS Secrets Manager / IAM
- Logs and Monitoring: Amazon CloudWatch
- Container Registry: Amazon ECR
- Optional Queueing: Amazon SQS or SNS
- Optional orchestration: AWS Step Functions / ECS Tasks

### High-level flow

Client -> API Gateway / ALB -> ECS/EC2 app -> S3 -> Rekognition/Media -> Bedrock -> OpenSearch -> Compliance response

---

## 3. AWS Service Mapping

| Current Azure Component | AWS Equivalent | Use in this project |
|---|---|---|
| Azure OpenAI | Amazon Bedrock / SageMaker | LLM reasoning and embeddings |
| Azure AI Search | Amazon OpenSearch Service | rule retrieval and vector search |
| Azure Video Indexer | Amazon Rekognition Video / MediaConvert | transcription, OCR, video analysis |
| Azure Monitor | Amazon CloudWatch | logging, monitoring, alerts |
| Azure Identity | IAM + Secrets Manager | authentication and secret management |
| Azure Blob Storage | Amazon S3 | storing videos, PDFs, and generated files |
| FastAPI app hosting | ECS Fargate / EC2 / App Runner | host the backend |
| API exposure | Application Load Balancer / API Gateway | expose the API |
| Container registry | Amazon ECR | store Docker images |
| CI/CD | CodePipeline / CodeBuild / GitHub Actions | automation |

---

## 4. AWS Deployment Options

### Option A: ECS Fargate (recommended for production)

Best for:

- container-based deployment
- scalability
- managed infrastructure
- cleaner production setup

### Option B: EC2

Best for:

- simple setup
- developer-friendly deployment
- direct control over runtime environment

### Option C: App Runner

Best for:

- quick deployment
- lightweight API services
- simpler developer workflows

---

## 5. Required AWS Services

### 5.1 Amazon Bedrock

Use Bedrock for:

- LLM inference for compliance analysis
- embeddings generation for RAG-based retrieval

Recommended model patterns:

- Claude 3.x for reasoning
- Titan Embeddings or other embedding model for vector similarity

### 5.2 Amazon OpenSearch Service or OpenSearch Serverless

Use OpenSearch to store indexed docs with vectors for retrieval.

This replaces Azure AI Search.

You will need:

- domain or serverless collection
- index name
- endpoint
- access credentials

### 5.3 Amazon S3

Use S3 for:

- uploaded videos
- downloaded local video storage
- compliance PDFs and source docs
- temporary artifacts

### 5.4 Amazon Rekognition Video

Use Rekognition Video to extract:

- scene information
- labels
- text detection
- media analysis

This can supplement or replace the Azure Video Indexer step depending on architecture.

### 5.5 AWS Secrets Manager

Store:

- Bedrock access credentials
- OpenSearch credentials
- AWS account details
- any API keys

Never hardcode secrets in source code.

### 5.6 Amazon CloudWatch

Use CloudWatch for:

- application logs
- metrics
- alarms
- uptime and health checks

### 5.7 Amazon ECR

Store the Docker image for the application before deployment to ECS or App Runner.

---

## 6. Recommended Project Structure for AWS Deployment

```text
ComplianceQAPipeline/
├── app/
│   ├── main.py
│   ├── backend/
│   └── requirements.txt
├── Dockerfile
├── docker-compose.yml   # optional for local testing
├── .env.example
├── .gitignore
├── README.md
├── AWS_DEPLOYMENT.md
└── infra/
    ├── terraform/
    │   ├── main.tf
    │   ├── variables.tf
    │   └── outputs.tf
    └── cloudformation/
```

---

## 7. Environment Variables for AWS

Set the following variables in AWS Secrets Manager or environment config:

```env
AWS_REGION=us-east-1

BEDROCK_MODEL_ID=anthropic.claude-3-sonnet-20240229-v1:0
BEDROCK_EMBEDDING_MODEL_ID=amazon.titan-embed-text-v1

OPENSEARCH_ENDPOINT=https://search-your-domain.region.es.amazonaws.com
OPENSEARCH_INDEX_NAME=brand-guardian-rules
AWS_ACCESS_KEY_ID=your_access_key
AWS_SECRET_ACCESS_KEY=your_secret_key

S3_BUCKET_NAME=brand-guardian-assets

APPLICATIONINSIGHTS_CONNECTION_STRING=
```

If you keep the current Azure-based project configuration, the AWS deployment still needs equivalent Azure environment values mapped to the new runtime environment.

---

## 8. Containerization for AWS

Create a Dockerfile at the root of the project:

```dockerfile
FROM python:3.12-slim

WORKDIR /app

COPY pyproject.toml ./
COPY . .

RUN pip install --upgrade pip && pip install uv && uv sync --frozen

EXPOSE 8000

CMD ["uv", "run", "uvicorn", "backend.src.api.server:app", "--host", "0.0.0.0", "--port", "8000"]
```

### Build image

```bash
docker build -t brand-guardian-api .
```

### Run locally

```bash
docker run -p 8000:8000 --env-file .env brand-guardian-api
```

---

## 9. Deploy to Amazon ECS Fargate

### Step 1: Create ECR repository

```bash
aws ecr create-repository --repository-name brand-guardian-api
```

### Step 2: Authenticate Docker to ECR

```bash
aws ecr get-login-password --region us-east-1 | docker login --username AWS --password-stdin <aws_account_id>.dkr.ecr.us-east-1.amazonaws.com
```

### Step 3: Tag and push the image

```bash
docker tag brand-guardian-api:latest <aws_account_id>.dkr.ecr.us-east-1.amazonaws.com/brand-guardian-api:latest
docker push <aws_account_id>.dkr.ecr.us-east-1.amazonaws.com/brand-guardian-api:latest
```

### Step 4: Create ECS cluster and task definition

Create:

- ECS cluster
- task definition
- security group
- IAM role
- load balancer

### Step 5: Run the service

Use ECS to run the container with:

- port 8000 exposed
- environment variables from Secrets Manager
- CloudWatch logging enabled

---

## 10. Deploy to EC2

This is a simpler alternative for testing or MVP deployment.

### Install dependencies

```bash
sudo yum update -y
sudo yum install -y python3 git
```

### Clone project

```bash
git clone https://github.com/<your-username>/azure-llmops-video-compliance-project.git
cd azure-llmops-video-compliance-project
```

### Create venv and install

```bash
python3 -m venv .venv
source .venv/bin/activate
pip install -r requirements.txt
```

### Start API

```bash
uvicorn backend.src.api.server:app --host 0.0.0.0 --port 8000
```

### Use a process manager

For production, use:

- systemd
- supervisord
- PM2 (for Node-based apps, less relevant here)

---

## 11. Deploy with AWS App Runner

This is the fastest path for a Python API service.

### Steps

1. Push the code to GitHub
2. Connect the repository to App Runner
3. Select Python runtime
4. Add environment variables
5. Set the startup command

Example start command:

```bash
uv run uvicorn backend.src.api.server:app --host 0.0.0.0 --port 8000
```

This is ideal for a quick production deployment without managing a full ECS cluster.

---

## 12. Indexing Compliance Rules on AWS

Since the project currently indexes PDFs into Azure AI Search, the AWS version should use OpenSearch vector indexing.

### Workflow

1. store PDFs in S3
2. load documents in Python
3. chunk them using RecursiveCharacterTextSplitter
4. generate embeddings with Bedrock embeddings model
5. store vectors in OpenSearch

Example Python logic:

```python
from langchain_community.vectorstores import OpenSearchVectorSearch
from langchain_aws import BedrockEmbeddings
```

This is the AWS replacement for the Azure Search indexing script in `backend/scripts/index_documents.py`.

---

## 13. Security Best Practices

- never commit real `.env` files
- store secrets in AWS Secrets Manager
- use IAM roles, not static access keys when possible
- restrict S3 bucket access by role and VPC
- use VPC private subnets where possible
- enable CloudWatch logging and alarms
- enable HTTPS only through ALB or API Gateway
- rotate keys periodically

---

## 14. Typical Production Deployment Architecture

```text
Users
  |
  v
API Gateway / ALB
  |
  v
ECS Fargate service (FastAPI app)
  |
  +--> Bedrock (LLM + embeddings)
  |
  +--> OpenSearch (vector retrieval)
  |
  +--> S3 (uploaded videos/docs)
  |
  +--> CloudWatch (logs/metrics)
```

This architecture mirrors the Azure version while staying within AWS-native services.

---

## 15. Recommended Final Deployment Pattern

For a robust production deployment, choose:

- ECS Fargate for API hosting
- S3 for file storage
- OpenSearch for vector retrieval
- Bedrock for AI reasoning
- CloudWatch for observability
- Secrets Manager for credentials
- ECR for image management

This is the best balance of scalability, security, and operational simplicity.

---

## 16. Deployment Checklist

Before launch, confirm:

- [ ] AWS account and IAM permissions are ready
- [ ] Bedrock access is enabled
- [ ] OpenSearch cluster is initialized
- [ ] S3 buckets are created
- [ ] Docker image builds successfully
- [ ] environment variables are stored securely
- [ ] health endpoint works
- [ ] API responds to sample video audit requests
- [ ] logs are visible in CloudWatch
- [ ] indexing script can upload PDF rules to OpenSearch

---

## 17. Summary

To deploy this project on AWS, the recommended path is:

- FastAPI app on ECS Fargate or EC2
- Bedrock for LLM and embeddings
- OpenSearch for rule retrieval
- S3 for files and documents
- CloudWatch for monitoring
- Secrets Manager for secure configuration

This architecture closely matches the Azure design and provides a production-ready AWS deployment pattern.

---

## 18. Next recommended step

The next best step is to implement one of the following:

1. Docker-based ECS deployment
2. Terraform infrastructure for AWS resources
3. GitHub Actions CI/CD pipeline for AWS deployment
4. App Runner quick deploy setup

If you want, I can generate the next file for you:

- Dockerfile
- ECS task definition JSON
- Terraform AWS infrastructure
- GitHub Actions deployment workflow for AWS

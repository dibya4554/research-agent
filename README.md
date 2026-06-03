# 🔍 Research Assistant Agent
### Multi-Agent RAG Pipeline with LangGraph + FastAPI + AWS Deployment

A production-grade **agentic AI system** that answers research queries by orchestrating multiple specialised AI agents. It combines **Retrieval-Augmented Generation (RAG)** from your documents with **live web search** to deliver comprehensive, grounded answers — exposed as a REST API deployable on AWS EC2.

---

## 🏗️ Architecture

```
User Query (REST API)
        │
        ▼
┌───────────────────────────────────────────────────────┐
│              LangGraph Orchestrator                    │
│                                                        │
│  ┌────────────┐    ┌─────────────┐    ┌────────────┐  │
│  │  Planner   │───▶│  Retriever  │───▶│  Searcher  │  │
│  │   Agent    │    │   Agent     │    │   Agent    │  │
│  │            │    │ (FAISS RAG) │    │(DuckDuckGo)│  │
│  └────────────┘    └─────────────┘    └────────────┘  │
│        │                  │                  │         │
│        ▼                  ▼                  ▼         │
│  Decomposes query   Semantic search    Live web search │
│  into sub-tasks     over documents     for recency     │
│                                                        │
│                   ┌──────────────┐                     │
│                   │  Synthesis   │◀── Merges all       │
│                   │    Agent     │    context          │
│                   └──────────────┘                     │
│                          │                             │
│                   ┌──────────────┐                     │
│                   │   Summary    │ (optional)          │
│                   │    Agent     │                     │
│                   └──────────────┘                     │
└───────────────────────────────────────────────────────┘
        │
        ▼
  Structured JSON Response
  (answer + sources + summary + agent_trace)
```

### Agent Responsibilities

| Agent | Role | Framework |
|-------|------|-----------|
| **PlannerAgent** | Decomposes the query into 2–4 focused sub-tasks | LangChain + GPT-4o-mini |
| **RetrieverAgent** | Semantic similarity search over FAISS vector store | LangChain + FAISS |
| **SearchAgent** | Live web search for recent information | DuckDuckGo (no API key) |
| **SynthesisAgent** | Combines all context into a coherent answer | LangChain + GPT-4o-mini |
| **SummaryAgent** | Generates 3–5 sentence executive summary | LangChain + GPT-4o-mini |

---

## 📁 Project Structure

```
research-agent/
├── app/
│   ├── main.py                    # FastAPI app + routes
│   ├── agents/
│   │   ├── orchestrator.py        # LangGraph StateGraph pipeline
│   │   ├── planner.py             # Query decomposition agent
│   │   ├── retriever.py           # FAISS semantic retrieval agent
│   │   ├── search.py              # Web search agent
│   │   ├── synthesis.py           # Answer synthesis agent
│   │   └── summary.py             # Executive summary agent
│   ├── rag/
│   │   ├── ingestor.py            # PDF/TXT ingestion + chunking
│   │   └── vectorstore.py         # FAISS vector store manager
│   └── utils/
│       ├── config.py              # Pydantic settings (env vars)
│       └── logger.py              # Centralised logger
├── data/
│   ├── raw/                       # Original uploaded documents
│   ├── processed/                 # Intermediate files
│   └── vectorstore/               # Persisted FAISS index (git-ignored)
├── tests/
│   ├── test_api.py                # API endpoint tests
│   └── test_agents.py             # Unit tests for each agent
├── Dockerfile
├── requirements.txt
├── .env.example
├── .gitignore
└── README.md
```

---

## 🚀 Quick Start (Local)

### 1. Clone & Setup

```bash
git clone https://github.com/<your-username>/research-agent.git
cd research-agent

python -m venv venv
source venv/bin/activate        # Windows: venv\Scripts\activate
pip install -r requirements.txt
```

### 2. Configure Environment

```bash
cp .env.example .env
# Edit .env and add your OpenAI API key
```

```env
OPENAI_API_KEY=sk-your-key-here
LLM_MODEL=gpt-4o-mini
ENABLE_WEB_SEARCH=true
```

### 3. Run the API

```bash
uvicorn app.main:app --host 0.0.0.0 --port 8000 --reload
```

Visit **http://localhost:8000/docs** for the interactive Swagger UI.

---

## 📡 API Reference

### `GET /health`
Health check for load balancers.
```json
{ "status": "healthy", "service": "research-agent" }
```

---

### `POST /ingest`
Upload a PDF or TXT document to index into the vector store.

```bash
curl -X POST http://localhost:8000/ingest \
  -F "file=@my_research_paper.pdf"
```

**Response:**
```json
{
  "status": "success",
  "chunks_indexed": 47,
  "document_name": "my_research_paper.pdf"
}
```

---

### `POST /query`
Submit a research query to the multi-agent pipeline.

```bash
curl -X POST http://localhost:8000/query \
  -H "Content-Type: application/json" \
  -d '{
    "query": "What are the key challenges in deploying LLMs at scale?",
    "max_sources": 5,
    "include_summary": true
  }'
```

**Response:**
```json
{
  "query": "What are the key challenges in deploying LLMs at scale?",
  "answer": "Deploying large language models at scale presents several interconnected challenges...",
  "sources": ["llm_survey.pdf (p.4)", "scaling_paper.pdf (p.12)"],
  "summary": "The primary challenges include GPU memory constraints, inference latency, and cost...",
  "agent_trace": [
    "[Orchestrator] Pipeline started.",
    "[Planner] Decomposed into 3 sub-tasks: ['LLM memory requirements', ...]",
    "[Retriever] Retrieved 5 chunks from vector store.",
    "[Search] Found 4 web results.",
    "[Synthesis] Generated answer from retrieved context.",
    "[Summary] Executive summary generated."
  ]
}
```

---

### `DELETE /vectorstore`
Clear all indexed documents.

```bash
curl -X DELETE http://localhost:8000/vectorstore
```

---

## 🧪 Running Tests

```bash
pytest tests/ -v
```

Expected output:
```
tests/test_api.py::test_health_check PASSED
tests/test_api.py::test_query_returns_answer PASSED
tests/test_api.py::test_ingest_pdf PASSED
tests/test_agents.py::TestPlannerAgent::test_decompose_query PASSED
tests/test_agents.py::TestRetrieverAgent::test_retrieval_deduplicates PASSED
...
```

---

## 🐳 Docker

### Build & Run Locally

```bash
docker build -t research-agent .

docker run -d \
  -p 8000:8000 \
  -e OPENAI_API_KEY=sk-your-key \
  -e ENABLE_WEB_SEARCH=true \
  -v $(pwd)/data:/app/data \
  --name research-agent \
  research-agent
```

---

## ☁️ AWS Deployment (EC2 + API Gateway)

### Step 1 — Launch EC2 Instance

1. Go to **EC2 Console → Launch Instance**
2. Choose **Ubuntu 22.04 LTS**, instance type **t3.medium** (minimum)
3. Create/select a key pair for SSH
4. Security group — add inbound rules:
   - Port **22** (SSH) from your IP
   - Port **8000** (HTTP) from anywhere (or restrict to API Gateway IP range)
5. Launch the instance

### Step 2 — SSH & Install Docker

```bash
ssh -i your-key.pem ubuntu@<EC2-PUBLIC-IP>

# Install Docker
sudo apt update && sudo apt install -y docker.io
sudo systemctl start docker
sudo usermod -aG docker ubuntu
newgrp docker
```

### Step 3 — Deploy the Application

```bash
# Clone your repo on EC2
git clone https://github.com/<your-username>/research-agent.git
cd research-agent

# Create .env with your API key
cp .env.example .env
nano .env   # add OPENAI_API_KEY

# Build and start
docker build -t research-agent .
docker run -d \
  --restart always \
  -p 8000:8000 \
  --env-file .env \
  -v $(pwd)/data:/app/data \
  --name research-agent \
  research-agent

# Verify it's running
curl http://localhost:8000/health
```

### Step 4 — Configure AWS API Gateway

1. Go to **API Gateway Console → Create API → HTTP API**
2. Click **Add Integration → HTTP**
3. Integration URL: `http://<EC2-PUBLIC-IP>:8000`
4. Create routes:
   - `GET /health` → integration
   - `POST /query` → integration
   - `POST /ingest` → integration
   - `DELETE /vectorstore` → integration
5. **Deploy** to a stage (e.g., `prod`)
6. Your public endpoint will be: `https://<api-id>.execute-api.<region>.amazonaws.com/`

### Step 5 — Set Up CloudWatch Monitoring

```bash
# Install CloudWatch Agent on EC2
sudo apt install -y amazon-cloudwatch-agent

# Create config for Docker log monitoring
sudo /opt/aws/amazon-cloudwatch-agent/bin/amazon-cloudwatch-agent-config-wizard
```

Key metrics to monitor via CloudWatch:
- **EC2**: CPUUtilization, NetworkIn/Out
- **API Gateway**: Latency, 4XXError, 5XXError, Count
- Set alarms on latency > 5s and 5XX error rate > 1%

### Step 6 — Store Documents on S3 (Optional)

```bash
pip install boto3

# Upload documents to S3
aws s3 cp my_document.pdf s3://your-bucket/documents/

# Add to .env
AWS_S3_BUCKET=your-bucket
AWS_REGION=ap-south-1
```

---

## ⚙️ Configuration Reference

| Variable | Default | Description |
|----------|---------|-------------|
| `OPENAI_API_KEY` | *(required)* | OpenAI API key |
| `LLM_MODEL` | `gpt-4o-mini` | LLM for agents |
| `EMBEDDING_MODEL` | `text-embedding-3-small` | Embedding model |
| `VECTORSTORE_DIR` | `data/vectorstore` | FAISS index path |
| `CHUNK_SIZE` | `800` | Characters per chunk |
| `CHUNK_OVERLAP` | `150` | Overlap between chunks |
| `ENABLE_WEB_SEARCH` | `true` | Enable DuckDuckGo search |
| `PORT` | `8000` | Server port |

---

## 🛠️ Tech Stack

| Layer | Technology |
|-------|-----------|
| Agent Orchestration | LangGraph |
| LLM & Embeddings | LangChain + OpenAI GPT-4o-mini |
| Vector Store | FAISS (CPU) |
| PDF Parsing | PyMuPDF |
| Web Search | DuckDuckGo Search |
| API Framework | FastAPI + Uvicorn |
| Containerisation | Docker |
| Cloud | AWS EC2 + API Gateway + CloudWatch |
| Testing | pytest + httpx |

---

## 📄 License

MIT License — free to use, modify, and distribute.

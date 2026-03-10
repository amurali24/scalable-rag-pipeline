# Demo Setup & Implementation Plan

## Project Status Summary

The codebase is **well-architected and ~80% implemented**. The core RAG pipeline (API -> LangGraph Agent -> Hybrid Retrieval -> LLM Response) is fully coded. Infrastructure-as-Code (Terraform, Helm, Ray manifests) is production-ready. The main gaps are some document loaders, a few tool stubs, and test coverage.

---

## Phase 0: Local Environment Prerequisites

**Goal:** Get your workstation ready.

| Step | Action | Notes |
|------|--------|-------|
| 0.1 | Install Python 3.10+, Docker Desktop, AWS CLI v2, Terraform 1.5+, kubectl 1.29+, Helm 3.x | All listed in README S3 |
| 0.2 | Clone repo, `cd scalable-rag-pipeline` | |
| 0.3 | `pip install -r requirements.txt` (or `make install`) | 52 pinned deps including Ray, LangChain, FastAPI |
| 0.4 | Create `.env` file from `services/api/app/config.py` Pydantic Settings | Keys: `DATABASE_URL`, `REDIS_URL`, `QDRANT_HOST`, `NEO4J_URI`, `JWT_SECRET_KEY`, `RAY_LLM_ENDPOINT`, `RAY_EMBED_ENDPOINT` |

---

## Phase 1: Local Demo (No AWS, No GPU) -- Best for Quick Demos

**Goal:** Run the full system locally using Docker Compose + local API server.

| Step | Command | What It Does |
|------|---------|--------------|
| 1.1 | `make up` | Starts Postgres, Redis, Qdrant, Neo4j, MinIO via `docker-compose.yml` |
| 1.2 | Verify containers | `docker ps` -- confirm all 5 services are healthy on ports 5432, 6379, 6333, 7687, 9000 |
| 1.3 | Create `.env` with local connection strings | `DATABASE_URL=postgresql+asyncpg://ragadmin:ragadmin@localhost:5432/rag_db`, `REDIS_URL=redis://localhost:6379/0`, `QDRANT_HOST=localhost`, `NEO4J_URI=bolt://localhost:7687` |
| 1.4 | `make dev` | Starts FastAPI on `localhost:8000` with hot reload |
| 1.5 | Test health | `curl http://localhost:8000/health/readiness` |
| 1.6 | Test chat (with auth disabled or dev token) | POST to `/api/v1/chat/stream` |

**Key consideration for local demo:** The LLM and embedding endpoints (`RAY_LLM_ENDPOINT`, `RAY_EMBED_ENDPOINT`) won't have a real Ray cluster. You have two options:

- **Option A -- Mock/Stub:** Point these at a local mock server or use OpenAI API as a drop-in replacement (the `openai` package is already in requirements)
- **Option B -- Local Ray:** Start a single-node Ray cluster (`ray start --head`) and deploy lightweight models (e.g., Llama-7B via `models/llm/llama-7b.yaml` if you have a GPU, or a tiny model for CPU)

---

## Phase 2: Data Ingestion for Demo Content

**Goal:** Load the evaluation dataset so the RAG system has documents to retrieve from.

| Step | Action | Details |
|------|--------|---------|
| 2.1 | Upload eval docs to MinIO (local S3) | Use `eval/datasets/true_data/` -- contains `.pptx`, `.docx`, `.html`, `.txt` files about Kubernetes |
| 2.2 | Run ingestion pipeline | `python -m pipelines.ingestion.main` (reads from S3/MinIO, chunks, embeds, indexes) |
| 2.3 | Verify Qdrant has vectors | Hit Qdrant REST API at `localhost:6333/collections` |
| 2.4 | Verify Neo4j has graph | Open Neo4j browser at `localhost:7474`, run `MATCH (n) RETURN n LIMIT 25` |

**Note:** The PDF loader is fully implemented; HTML/DOCX loaders are stubs. For demo, either:
- Convert all docs to PDF first, or
- Implement the missing loaders (they follow the same pattern as `pipelines/ingestion/loaders/pdf.py`)

---

## Phase 3: Cloud Deployment (Full Production Demo on AWS)

**Goal:** Deploy the complete system on AWS EKS with GPU inference.

### 3A. Terraform Infrastructure (~20 min provision time)

| Step | Command | Resources Created |
|------|---------|-------------------|
| 3A.1 | Create S3 bucket `rag-platform-terraform-state-prod-001` + DynamoDB `terraform-state-lock` manually | Terraform remote state backend |
| 3A.2 | `cd infra/terraform && terraform init` | Initialize providers |
| 3A.3 | `terraform plan -var="db_password=<STRONG_PASS>" -out=tfplan` | Preview: VPC, EKS, Aurora, ElastiCache, S3, IAM |
| 3A.4 | `terraform apply tfplan` | Provision all AWS resources |
| 3A.5 | `aws eks update-kubeconfig --region us-east-1 --name rag-platform-cluster` | Connect kubectl |

### 3B. Cluster Bootstrap

| Step | Command | What It Installs |
|------|---------|------------------|
| 3B.1 | `./scripts/bootstrap_cluster.sh` | Karpenter, KubeRay Operator, External Secrets, Ingress Controller |
| 3B.2 | `kubectl apply -f infra/karpenter/provisioner-cpu.yaml` | CPU autoscaler (m6i/c6i, Spot) |
| 3B.3 | `kubectl apply -f infra/karpenter/provisioner-gpu.yaml` | GPU autoscaler (g5.xlarge, A10G) |

### 3C. Data Plane (Databases + Ray)

| Step | Command | Duration |
|------|---------|----------|
| 3C.1 | `helm upgrade --install qdrant deploy/helm/qdrant` | ~2 min |
| 3C.2 | `helm upgrade --install neo4j deploy/helm/neo4j` | ~2 min |
| 3C.3 | `kubectl apply -f deploy/ray/ray-cluster.yaml` | Ray head + workers |
| 3C.4 | `kubectl apply -f deploy/ray/ray-serve-embed.yaml` | BGE-M3 embedding service |
| 3C.5 | `kubectl apply -f deploy/ray/ray-serve-llm.yaml` | vLLM Llama-70B (~2 min node provision + ~5 min model download) |

### 3D. Control Plane (API)

| Step | Command |
|------|---------|
| 3D.1 | Create `app-env-secret` K8s secret (as in README S7.1) |
| 3D.2 | `helm upgrade --install api deploy/helm/api` |
| 3D.3 | `kubectl apply -f deploy/ingress/nginx.yaml` |
| 3D.4 | Get ALB DNS: `kubectl get ingress` |

### 3E. Data Loading

| Step | Command |
|------|---------|
| 3E.1 | `python scripts/bulk_upload_s3.py ./eval/datasets/true_data $BUCKET_NAME` |
| 3E.2 | Port-forward Ray dashboard: `kubectl port-forward svc/rag-ray-cluster-head-svc 8265:8265` |
| 3E.3 | `python -m pipelines.jobs.s3_event_handler` |

---

## Phase 4: Demo Validation Checklist

| Test | Command/Action | Expected Result |
|------|----------------|-----------------|
| Health check | `curl <endpoint>/health/readiness` | `{"redis": "up", "neo4j": "up"}` |
| Auth flow | Generate JWT via `auth/jwt.py` utility | Valid Bearer token |
| Chat (streaming) | POST `/api/v1/chat/stream` with query | SSE stream with LLM-synthesized answer |
| Retrieval demo | Ask "What are the financial risks in Q3?" | Agent uses vector + graph search |
| Tool use demo | Ask a calculation question | Agent routes to calculator tool |
| Graph demo | Open Neo4j browser, show knowledge graph | Visual entity-relationship display |
| Ray dashboard | `localhost:8265` | Show autoscaling, actor status |
| Load test | `python scripts/load_test.py` | Locust results showing throughput |

---

## Phase 5: Pre-Demo Polish (Optional Improvements)

| Priority | Task | Effort | Impact |
|----------|------|--------|--------|
| **HIGH** | Implement HTML/DOCX loaders (follow `pdf.py` pattern) | 1-2 hrs | Enables all eval dataset documents |
| **HIGH** | Add a simple demo UI (Streamlit/Gradio chat) | 2-3 hrs | Makes demo visually compelling |
| **MEDIUM** | Complete `tools/web_search.py` | 1 hr | Shows tool-use capabilities |
| **MEDIUM** | Run `eval/ragas/run.py` to generate quality report | 30 min | Demonstrates evaluation pipeline |
| **LOW** | Complete `enhancers/hyde.py` | 1 hr | Shows HyDE retrieval improvement |
| **LOW** | Implement `services/sandbox/runner.py` | 2 hrs | Code execution demo |

---

## Cost Awareness (AWS Production)

| Resource | Estimated Cost | Optimization |
|----------|---------------|-------------|
| EKS Control Plane | ~$73/mo | Fixed cost |
| CPU nodes (m6i.large, Spot) | ~$30-50/mo | Karpenter scale-to-zero |
| GPU nodes (g5.xlarge, Spot) | ~$0.50/hr Spot | **Set `min_replicas: 0` in ray-serve-llm.yaml for demos** -- only pay when running |
| Aurora Serverless v2 | ~$40-80/mo | Scales with ACU usage |
| ElastiCache Redis | ~$25/mo | cache.t3.micro for demo |
| **Total (active demo)** | **~$5-10/hr** | Tear down with `scripts/cleanup.sh` after demo |

---

## Recommended Demo Flow (15-minute presentation)

1. **Architecture Overview** (2 min) -- Show the README diagram, explain Control Plane vs Data Plane
2. **Infrastructure** (2 min) -- Show Terraform files, `kubectl get nodes`, Karpenter provisioners
3. **Live Query** (3 min) -- Send a chat request, show SSE streaming response in real time
4. **Agent Reasoning** (3 min) -- Show Ray dashboard, explain Planner -> Retriever -> Responder flow
5. **Knowledge Graph** (2 min) -- Open Neo4j browser, show entity graph from ingested documents
6. **Scaling Demo** (2 min) -- Show Karpenter spinning up GPU nodes, Ray autoscaler metrics
7. **Evaluation** (1 min) -- Show RAGAS report from `eval/reports/latest.html`

---

## Recommendation

Start with **Phase 1 (Local Demo)** to validate the system works end-to-end before investing in AWS infrastructure. The local setup uses the same code paths -- only the backing services change from Docker Compose to managed AWS services.

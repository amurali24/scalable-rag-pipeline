# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

Production-grade agentic RAG (Retrieval-Augmented Generation) platform with event-driven architecture. Decoupled into a **Control Plane** (FastAPI on CPU nodes) for orchestration/auth/state and a **Data Plane** (Ray Serve on GPU nodes) for LLM inference and embeddings.

## Build & Development Commands

```bash
make install      # pip install -r requirements.txt
make dev          # uvicorn with hot reload on port 8000 (loads .env)
make up           # docker-compose up -d (Postgres, Redis, Qdrant, Neo4j, MinIO)
make down         # docker-compose down
make test         # pytest tests/
make deploy       # Helm upgrade (api + ray-cluster charts)
make infra        # terraform init && apply (AWS infrastructure)
```

Local development workflow: `make up` to start databases, then `make dev` to run the API.

## Architecture

### Request Flow (LangGraph Workflow)
```
User Query → Planner (decide: retrieve/direct/tool_use)
           → Retriever (Vector + Graph search in parallel)
           → Responder (Ray LLM synthesis)
           → Streamed SSE Response
```

Defined in `services/api/app/agents/graph.py` with state in `agents/state.py`. Nodes live in `agents/nodes/` (planner, retriever, responder, tool).

### Data Ingestion Pipeline (Ray Distributed)
```
S3 Documents → [Map] Parse & Chunk (CPU)
             → [Fork A] Embed → Qdrant (vectors)
             → [Fork B] Extract entities → Neo4j (graph)
```

Entry point: `pipelines/ingestion/main.py`. Config: `pipelines/ingestion/config.yaml`.

### Key Services
- **API** (`services/api/`) — FastAPI app, main control plane. Entry: `main.py`, config: `app/config.py` (Pydantic Settings from env vars)
- **Gateway** (`services/gateway/`) — Gateway service
- **Sandbox** (`services/sandbox/`) — Isolated code execution container (has its own Dockerfile)

### Databases
- **Aurora Postgres** — Chat history persistence (`memory/postgres.py`, `memory/models.py` with SQLAlchemy ORM)
- **Redis** — Semantic cache, session cache, rate limiting (`cache/redis.py`, `cache/semantic.py`)
- **Qdrant** — Vector search (`clients/qdrant.py`)
- **Neo4j** — Knowledge graph search (`clients/neo4j.py`)

### Clients (all async, in `services/api/app/clients/`)
- `ray_llm.py` — Async HTTP to Ray Serve LLM endpoint with exponential backoff
- `ray_embed.py` — Embedding service client
- `qdrant.py` — AsyncQdrantClient for vector search
- `neo4j.py` — Neo4j graph database client

### Shared Libraries (`libs/`)
- `observability/` — Prometheus metrics + OpenTelemetry tracing
- `retry/` — Exponential backoff strategy
- `schemas/` — Shared Pydantic models (e.g., `chat.py`)
- `utils/` — UUID generation, perf timing

### Query Enhancement (`services/api/app/enhancers/`)
- `hyde.py` — Hypothetical Document Embeddings for better retrieval
- `query_rewriter.py` — LLM-based query refinement

### Tools (`services/api/app/tools/`)
Vector search, graph search (Cypher), calculator, code sandbox, web search.

## Infrastructure

- **Terraform** (`infra/terraform/`) — AWS: EKS v1.29, VPC (10.0.0.0/16 multi-AZ), Aurora Postgres Serverless v2, ElastiCache Redis, S3, IAM (IRSA)
- **Karpenter** (`infra/karpenter/`) — CPU provisioner (m6i/c6i), GPU provisioner (g5.xlarge)
- **Helm Charts** (`deploy/helm/`) — API, Qdrant, Neo4j
- **Ray** (`deploy/ray/`) — Cluster spec, vLLM serving (Llama-70B), embedding service (BGE-M3), autoscaling policies
- **Ingress** (`deploy/ingress/`) — Nginx and Kong configs

## Code Patterns

- **Async-first**: asyncpg, httpx, asyncio throughout the API layer
- **Streaming**: Server-Sent Events for chat responses (`routes/chat.py`)
- **Dependency injection**: FastAPI `Depends()` with override-able providers for testability
- **Auth**: JWT Bearer tokens (`auth/jwt.py`)
- **Config**: Pydantic Settings loading from `.env` file

## Commit Convention

Prefix with component: `api:`, `pipelines:`, `tool:`, `box:`, `gw:`, `model:`, `eval:`
Example: `api: Chat route`, `tool: Graph search`

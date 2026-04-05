# Reranker Python Server — Implementation Plan

> **Project:** OME RAG — Reranking Service
> **Purpose:** Single Python service hosting CodeRankEmbed (bi-encoder) + Qwen3-Reranker (cross-encoder)
> **Server:** RHEL 9.7, 128 cores, 1TB RAM (CPU-only for rerankers)
> **Effort:** ~3 days
> **Last Updated:** April 2026

-----

## Table of Contents

1. [Architecture Decision](#1-architecture-decision)
1. [Models to Download](#2-models-to-download)
1. [Single Server — Two Models](#3-single-server)
1. [API Contract](#4-api-contract)
1. [Server Implementation](#5-server-implementation)
1. [TypeScript Client](#6-typescript-client)
1. [Integration with hybrid_search.ts](#7-integration)
1. [Systemd Service](#8-systemd-service)
1. [Air-Gapped Setup](#9-air-gapped-setup)
1. [Health Monitoring](#10-health-monitoring)
1. [Implementation Schedule](#11-implementation-schedule)
1. [Configuration Reference](#12-configuration)

-----

## 1. Architecture Decision

### 1.1 Why One Server, Not Two

Both models are small (137M + 600M), both use SentenceTransformers, both load PyTorch. Running two separate Flask servers means two Python processes, two copies of PyTorch in memory (~2 GB wasted), two systemd services to manage.

One server, two endpoints:

```
OME Reranker Service (:8086)
  │
  ├── POST /v1/biencoder/rerank    → CodeRankEmbed (137M, ~5ms/pair)
  ├── POST /v1/crossencoder/rerank → Qwen3-Reranker (600M, ~60ms/pair)
  ├── GET  /health                 → Both models status
  └── GET  /metrics                → Latency, throughput stats
```

### 1.2 Resource Budget

```
Memory:
  PyTorch runtime         ~800 MB   (shared between both models)
  CodeRankEmbed 137M      ~550 MB
  Qwen3-Reranker 600M     ~1.2 GB
  Request buffers         ~200 MB
  ─────────────────────────────────
  Total:                  ~2.75 GB

CPU:
  Bi-encoder:    4 threads per request (batch encode, fast)
  Cross-encoder: 8 threads per request (sequential scoring, heavier)
  Total:         ~12 cores peak (of 128 available)
```

### 1.3 Why FastAPI Over Flask

FastAPI with uvicorn gives async request handling — while cross-encoder is scoring 20 pairs (~1.2s), bi-encoder requests can still be served on another worker. Flask is single-threaded per worker.

```
FastAPI + uvicorn (2 workers)
  Worker 1: handles bi-encoder requests
  Worker 2: handles cross-encoder requests (or second bi-encoder)
```

-----

## 2. Models to Download

### 2.1 Download List

|#|Model                        |HuggingFace ID                         |Size   |License   |
|-|-----------------------------|---------------------------------------|-------|----------|
|1|CodeRankEmbed                |`nomic-ai/CodeRankEmbed`               |~522 MB|MIT       |
|2|Qwen3-Reranker-0.6B (seq-cls)|`tomaarsen/Qwen3-Reranker-0.6B-seq-cls`|~1.2 GB|Apache 2.0|

### 2.2 Download Commands (Internet-Connected Machine)

```bash
# Install huggingface CLI
pip install huggingface_hub[cli]

# Download CodeRankEmbed
huggingface-cli download nomic-ai/CodeRankEmbed \
  --local-dir /tmp/reranker-models/CodeRankEmbed \
  --local-dir-use-symlinks False

# Download Qwen3-Reranker (pre-converted seq-cls variant)
# This variant works directly with SentenceTransformers CrossEncoder
# without needing the yes/no token extraction hack
huggingface-cli download tomaarsen/Qwen3-Reranker-0.6B-seq-cls \
  --local-dir /tmp/reranker-models/Qwen3-Reranker-0.6B-seq-cls \
  --local-dir-use-symlinks False

# Package for transfer
cd /tmp
tar czf reranker-models.tar.gz reranker-models/
# Total: ~1.8 GB compressed
```

### 2.3 Why `tomaarsen/Qwen3-Reranker-0.6B-seq-cls`

The official `Qwen/Qwen3-Reranker-0.6B` uses a causal LM architecture with yes/no token scoring — you’d need to extract `lm_head` weights, compute `yes - no` vectors, and load as `Qwen3ForSequenceClassification`. The `tomaarsen` variant has this conversion already done. It works directly with `CrossEncoder` class — no custom code needed.

-----

## 3. Single Server — Two Models

### 3.1 Model Loading Strategy

```python
# Load both models at startup — they share the PyTorch runtime

import torch
torch.set_num_threads(12)  # Cap total threads

# Bi-encoder: SentenceTransformer (encode independently, dot product)
biencoder = SentenceTransformer("/models/CodeRankEmbed", trust_remote_code=True)

# Cross-encoder: CrossEncoder (encode jointly, single score)
crossencoder = CrossEncoder(
    "/models/Qwen3-Reranker-0.6B-seq-cls",
    trust_remote_code=True,
    automodel_args={"torch_dtype": "auto"},
)
```

### 3.2 Request Isolation

Bi-encoder and cross-encoder never run simultaneously in the same worker. FastAPI with 2 uvicorn workers ensures one worker can handle bi-encoder while the other handles cross-encoder.

If cross-encoder is busy (1.2s scoring), bi-encoder requests route to the other worker (~275ms). No queuing.

-----

## 4. API Contract

### 4.1 Bi-Encoder Rerank

```
POST /v1/biencoder/rerank
Content-Type: application/json

Request:
{
  "query": "how does authentication work",
  "documents": [
    {"id": "chunk-001", "content": "public class AuthService { ... }"},
    {"id": "chunk-002", "content": "const LoginForm = () => { ... }"}
  ],
  "top_k": 20
}

Response:
{
  "results": [
    {"id": "chunk-001", "score": 0.847},
    {"id": "chunk-002", "score": 0.623}
  ],
  "model": "CodeRankEmbed",
  "latency_ms": 275,
  "input_count": 80
}
```

### 4.2 Cross-Encoder Rerank

```
POST /v1/crossencoder/rerank
Content-Type: application/json

Request:
{
  "query": "how does authentication work",
  "documents": [
    {"id": "chunk-001", "content": "public class AuthService { ... }"},
    {"id": "chunk-002", "content": "const LoginForm = () => { ... }"}
  ],
  "top_k": 5,
  "instruction": "Rank code relevant to authentication and security patterns."
}

Response:
{
  "results": [
    {"id": "chunk-001", "score": 0.934},
    {"id": "chunk-002", "score": 0.412}
  ],
  "model": "Qwen3-Reranker-0.6B",
  "latency_ms": 1180,
  "input_count": 20
}
```

### 4.3 Health Check

```
GET /health

Response:
{
  "status": "ok",
  "models": {
    "biencoder": {
      "name": "CodeRankEmbed",
      "parameters": "137M",
      "loaded": true,
      "memory_mb": 550
    },
    "crossencoder": {
      "name": "Qwen3-Reranker-0.6B",
      "parameters": "600M",
      "loaded": true,
      "memory_mb": 1200
    }
  },
  "uptime_seconds": 86400
}
```

### 4.4 Metrics

```
GET /metrics

Response:
{
  "biencoder": {
    "total_requests": 12450,
    "avg_latency_ms": 285,
    "p95_latency_ms": 420,
    "avg_input_count": 55,
    "errors": 3
  },
  "crossencoder": {
    "total_requests": 12447,
    "avg_latency_ms": 1150,
    "p95_latency_ms": 1680,
    "avg_input_count": 20,
    "errors": 1
  }
}
```

-----

## 5. Server Implementation

### 5.1 Project Structure

```
src/reranker/
├── server.py               # FastAPI app — main entry
├── models.py               # Model loading + inference
├── schemas.py              # Pydantic request/response models
├── metrics.py              # Latency tracking
├── config.py               # Environment config
├── requirements.txt        # Python dependencies
└── test_server.py          # Smoke tests
```

### 5.2 config.py

```python
# src/reranker/config.py

import os

class Config:
    # Server
    HOST = os.environ.get("RERANKER_HOST", "127.0.0.1")
    PORT = int(os.environ.get("RERANKER_PORT", "8086"))
    WORKERS = int(os.environ.get("RERANKER_WORKERS", "2"))

    # Models
    BIENCODER_PATH = os.environ.get(
        "BIENCODER_MODEL_PATH", "/models/CodeRankEmbed"
    )
    CROSSENCODER_PATH = os.environ.get(
        "CROSSENCODER_MODEL_PATH", "/models/Qwen3-Reranker-0.6B-seq-cls"
    )

    # Inference
    MAX_BIENCODER_DOCS = int(os.environ.get("MAX_BIENCODER_DOCS", "100"))
    MAX_CROSSENCODER_DOCS = int(os.environ.get("MAX_CROSSENCODER_DOCS", "30"))
    BIENCODER_BATCH_SIZE = int(os.environ.get("BIENCODER_BATCH_SIZE", "32"))
    BIENCODER_MAX_CHARS = int(os.environ.get("BIENCODER_MAX_CHARS", "2048"))
    CROSSENCODER_MAX_CHARS = int(os.environ.get("CROSSENCODER_MAX_CHARS", "6000"))
    TORCH_THREADS = int(os.environ.get("TORCH_THREADS", "12"))

    # Default cross-encoder instruction
    DEFAULT_INSTRUCTION = (
        "Given a code search query, determine if the code snippet is relevant. "
        "Consider function names, class names, business logic, API endpoints, "
        "architectural patterns, and call relationships."
    )
```

### 5.3 schemas.py

```python
# src/reranker/schemas.py

from pydantic import BaseModel, Field
from typing import Optional


class Document(BaseModel):
    id: str
    content: str


class RerankRequest(BaseModel):
    query: str
    documents: list[Document]
    top_k: int = 20
    instruction: Optional[str] = None


class ScoredResult(BaseModel):
    id: str
    score: float


class RerankResponse(BaseModel):
    results: list[ScoredResult]
    model: str
    latency_ms: int
    input_count: int


class ModelStatus(BaseModel):
    name: str
    parameters: str
    loaded: bool
    memory_mb: int


class HealthResponse(BaseModel):
    status: str
    models: dict[str, ModelStatus]
    uptime_seconds: int


class MetricsSummary(BaseModel):
    total_requests: int
    avg_latency_ms: float
    p95_latency_ms: float
    avg_input_count: float
    errors: int


class MetricsResponse(BaseModel):
    biencoder: MetricsSummary
    crossencoder: MetricsSummary
```

### 5.4 metrics.py

```python
# src/reranker/metrics.py

import time
import threading
from collections import deque
from dataclasses import dataclass, field


@dataclass
class RequestMetric:
    latency_ms: int
    input_count: int
    timestamp: float


class MetricsCollector:
    def __init__(self, max_history: int = 10000):
        self._lock = threading.Lock()
        self._history: deque[RequestMetric] = deque(maxlen=max_history)
        self._errors: int = 0
        self._total: int = 0

    def record(self, latency_ms: int, input_count: int) -> None:
        with self._lock:
            self._total += 1
            self._history.append(RequestMetric(
                latency_ms=latency_ms,
                input_count=input_count,
                timestamp=time.time(),
            ))

    def record_error(self) -> None:
        with self._lock:
            self._errors += 1
            self._total += 1

    def summary(self) -> dict:
        with self._lock:
            if not self._history:
                return {
                    "total_requests": self._total,
                    "avg_latency_ms": 0,
                    "p95_latency_ms": 0,
                    "avg_input_count": 0,
                    "errors": self._errors,
                }

            latencies = sorted(m.latency_ms for m in self._history)
            input_counts = [m.input_count for m in self._history]
            p95_idx = int(len(latencies) * 0.95)

            return {
                "total_requests": self._total,
                "avg_latency_ms": round(sum(latencies) / len(latencies), 1),
                "p95_latency_ms": latencies[min(p95_idx, len(latencies) - 1)],
                "avg_input_count": round(sum(input_counts) / len(input_counts), 1),
                "errors": self._errors,
            }
```

### 5.5 models.py

```python
# src/reranker/models.py

import time
import logging
import numpy as np
import torch
from sentence_transformers import SentenceTransformer, CrossEncoder

from config import Config

logger = logging.getLogger("reranker.models")


class RerankerModels:
    """Loads and serves both reranker models."""

    def __init__(self):
        self.biencoder = None
        self.crossencoder = None
        self._biencoder_loaded = False
        self._crossencoder_loaded = False

    def load_all(self) -> None:
        """Load both models at startup. Called once per worker."""

        torch.set_num_threads(Config.TORCH_THREADS)
        logger.info(f"PyTorch threads: {Config.TORCH_THREADS}")

        # ── Load bi-encoder ──
        logger.info(f"Loading CodeRankEmbed from {Config.BIENCODER_PATH}...")
        start = time.time()
        try:
            self.biencoder = SentenceTransformer(
                Config.BIENCODER_PATH,
                trust_remote_code=True,
            )
            self.biencoder.eval()
            self._biencoder_loaded = True
            logger.info(
                f"CodeRankEmbed loaded in {time.time() - start:.1f}s "
                f"(137M params, 768d output)"
            )
        except Exception as e:
            logger.error(f"Failed to load CodeRankEmbed: {e}")

        # ── Load cross-encoder ──
        logger.info(f"Loading Qwen3-Reranker from {Config.CROSSENCODER_PATH}...")
        start = time.time()
        try:
            self.crossencoder = CrossEncoder(
                Config.CROSSENCODER_PATH,
                trust_remote_code=True,
                automodel_args={"torch_dtype": "auto"},
            )
            self.crossencoder.model.eval()
            self._crossencoder_loaded = True
            logger.info(
                f"Qwen3-Reranker loaded in {time.time() - start:.1f}s "
                f"(600M params)"
            )
        except Exception as e:
            logger.error(f"Failed to load Qwen3-Reranker: {e}")

    # ─────────────────────────────────────────────────────────
    # Bi-encoder: encode independently, score via dot product
    # ─────────────────────────────────────────────────────────

    def biencoder_rerank(
        self,
        query: str,
        documents: list[dict],
        top_k: int = 20,
    ) -> list[dict]:
        """
        CodeRankEmbed bi-encoder reranking.
        Encodes query and documents independently, scores via dot product.
        ~5ms per document at batch_size=32.
        """
        if not self._biencoder_loaded:
            raise RuntimeError("CodeRankEmbed not loaded")

        # ── Prefix query (same convention as nomic-embed-code) ──
        prefixed_query = f"Represent this query for searching relevant code: {query}"

        # ── Encode query ──
        q_emb = self.biencoder.encode(
            [prefixed_query],
            normalize_embeddings=True,
            show_progress_bar=False,
        )

        # ── Truncate documents for bi-encoder ──
        # Bi-encoder doesn't need full content — first 50 lines is enough
        doc_texts = []
        for d in documents:
            content = d["content"]
            lines = content.split("\n")
            if len(lines) > 50:
                content = "\n".join(lines[:50])
            doc_texts.append(content[:Config.BIENCODER_MAX_CHARS])

        # ── Encode documents (batched) ──
        doc_embs = self.biencoder.encode(
            doc_texts,
            normalize_embeddings=True,
            batch_size=Config.BIENCODER_BATCH_SIZE,
            show_progress_bar=False,
        )

        # ── Score via dot product ──
        scores = (doc_embs @ q_emb.T).flatten().tolist()

        # ── Build sorted results ──
        results = [
            {"id": d["id"], "score": round(float(scores[i]), 6)}
            for i, d in enumerate(documents)
        ]
        results.sort(key=lambda x: x["score"], reverse=True)

        return results[:top_k]

    # ─────────────────────────────────────────────────────────
    # Cross-encoder: encode jointly, single relevance score
    # ─────────────────────────────────────────────────────────

    def crossencoder_rerank(
        self,
        query: str,
        documents: list[dict],
        top_k: int = 5,
        instruction: str | None = None,
    ) -> list[dict]:
        """
        Qwen3-Reranker cross-encoder reranking.
        Processes query+document jointly through full attention.
        ~60ms per pair on CPU.
        """
        if not self._crossencoder_loaded:
            raise RuntimeError("Qwen3-Reranker not loaded")

        # ── Build instructed query ──
        instr = instruction or Config.DEFAULT_INSTRUCTION
        instructed_query = f"Instruct: {instr}\nQuery: {query}"

        # ── Build query-document pairs ──
        pairs = []
        for d in documents:
            content = d["content"]
            # Cross-encoder benefits from more context than bi-encoder
            lines = content.split("\n")
            if len(lines) > 150:
                head = "\n".join(lines[:105])
                tail = "\n".join(lines[-30:])
                content = (
                    f"{head}\n\n"
                    f"// ... {len(lines) - 135} lines omitted ...\n\n"
                    f"{tail}"
                )
            content = content[:Config.CROSSENCODER_MAX_CHARS]
            pairs.append((instructed_query, content))

        # ── Score all pairs ──
        scores = self.crossencoder.predict(
            pairs,
            show_progress_bar=False,
        ).tolist()

        # ── Build sorted results ──
        results = [
            {"id": d["id"], "score": round(float(scores[i]), 6)}
            for i, d in enumerate(documents)
        ]
        results.sort(key=lambda x: x["score"], reverse=True)

        return results[:top_k]

    # ─────────────────────────────────────────────────────────
    # Health
    # ─────────────────────────────────────────────────────────

    def health(self) -> dict:
        return {
            "biencoder": {
                "name": "CodeRankEmbed",
                "parameters": "137M",
                "loaded": self._biencoder_loaded,
                "memory_mb": 550 if self._biencoder_loaded else 0,
            },
            "crossencoder": {
                "name": "Qwen3-Reranker-0.6B",
                "parameters": "600M",
                "loaded": self._crossencoder_loaded,
                "memory_mb": 1200 if self._crossencoder_loaded else 0,
            },
        }
```

### 5.6 server.py — Main Application

```python
# src/reranker/server.py

import os
import sys
import time
import logging
from contextlib import asynccontextmanager

from fastapi import FastAPI, HTTPException
from fastapi.responses import JSONResponse
import uvicorn

from config import Config
from models import RerankerModels
from schemas import (
    RerankRequest, RerankResponse, ScoredResult,
    HealthResponse, MetricsResponse,
)
from metrics import MetricsCollector

# ── Logging ──
logging.basicConfig(
    level=logging.INFO,
    format="%(asctime)s [%(name)s] %(levelname)s %(message)s",
    handlers=[logging.StreamHandler(sys.stdout)],
)
logger = logging.getLogger("reranker")

# ── Global state ──
models = RerankerModels()
biencoder_metrics = MetricsCollector()
crossencoder_metrics = MetricsCollector()
start_time = time.time()


# ── Lifecycle ──
@asynccontextmanager
async def lifespan(app: FastAPI):
    """Load models on startup, cleanup on shutdown."""
    logger.info("=" * 60)
    logger.info("OME Reranker Service starting...")
    logger.info(f"  Bi-encoder:    {Config.BIENCODER_PATH}")
    logger.info(f"  Cross-encoder: {Config.CROSSENCODER_PATH}")
    logger.info(f"  Workers:       {Config.WORKERS}")
    logger.info(f"  Torch threads: {Config.TORCH_THREADS}")
    logger.info("=" * 60)

    models.load_all()

    health = models.health()
    loaded = sum(1 for m in health.values() if m["loaded"])
    logger.info(f"Models loaded: {loaded}/2")

    yield

    logger.info("Reranker service shutting down")


# ── App ──
app = FastAPI(
    title="OME Reranker Service",
    description="CodeRankEmbed (bi-encoder) + Qwen3-Reranker (cross-encoder)",
    version="1.0.0",
    lifespan=lifespan,
)


# ────────────────────────────────────────────────────────────
# Endpoints
# ────────────────────────────────────────────────────────────

@app.get("/health")
async def health():
    model_health = models.health()
    any_loaded = any(m["loaded"] for m in model_health.values())
    return {
        "status": "ok" if any_loaded else "degraded",
        "models": model_health,
        "uptime_seconds": int(time.time() - start_time),
    }


@app.get("/metrics")
async def metrics():
    return {
        "biencoder": biencoder_metrics.summary(),
        "crossencoder": crossencoder_metrics.summary(),
    }


@app.post("/v1/biencoder/rerank")
async def biencoder_rerank(req: RerankRequest):
    """
    Stage 1: CodeRankEmbed bi-encoder.
    Fast filter — encodes query and documents independently.
    Use to reduce 80 candidates to 20 before cross-encoder.
    """
    if len(req.documents) > Config.MAX_BIENCODER_DOCS:
        raise HTTPException(
            status_code=400,
            detail=f"Max {Config.MAX_BIENCODER_DOCS} documents"
        )

    start_ms = time.time() * 1000

    try:
        docs = [{"id": d.id, "content": d.content} for d in req.documents]
        results = models.biencoder_rerank(req.query, docs, req.top_k)

        latency = int(time.time() * 1000 - start_ms)
        biencoder_metrics.record(latency, len(req.documents))

        logger.info(
            f"[bi-encoder] {len(req.documents)}→{len(results)} "
            f"in {latency}ms "
            f"(top: {results[0]['score']:.4f})" if results else ""
        )

        return RerankResponse(
            results=[ScoredResult(**r) for r in results],
            model="CodeRankEmbed",
            latency_ms=latency,
            input_count=len(req.documents),
        )

    except Exception as e:
        biencoder_metrics.record_error()
        logger.error(f"[bi-encoder] Error: {e}")
        raise HTTPException(status_code=500, detail=str(e))


@app.post("/v1/crossencoder/rerank")
async def crossencoder_rerank(req: RerankRequest):
    """
    Stage 2: Qwen3-Reranker cross-encoder.
    Accurate scoring — processes query+document jointly.
    Use on 20 pre-filtered candidates from bi-encoder.
    Supports instruction-tuned reranking via `instruction` field.
    """
    if len(req.documents) > Config.MAX_CROSSENCODER_DOCS:
        raise HTTPException(
            status_code=400,
            detail=f"Max {Config.MAX_CROSSENCODER_DOCS} documents"
        )

    start_ms = time.time() * 1000

    try:
        docs = [{"id": d.id, "content": d.content} for d in req.documents]
        results = models.crossencoder_rerank(
            req.query, docs, req.top_k, req.instruction
        )

        latency = int(time.time() * 1000 - start_ms)
        crossencoder_metrics.record(latency, len(req.documents))

        logger.info(
            f"[cross-encoder] {len(req.documents)}→{len(results)} "
            f"in {latency}ms "
            f"(top: {results[0]['score']:.4f})" if results else ""
        )

        return RerankResponse(
            results=[ScoredResult(**r) for r in results],
            model="Qwen3-Reranker-0.6B",
            latency_ms=latency,
            input_count=len(req.documents),
        )

    except Exception as e:
        crossencoder_metrics.record_error()
        logger.error(f"[cross-encoder] Error: {e}")
        raise HTTPException(status_code=500, detail=str(e))


# ────────────────────────────────────────────────────────────
# Entry point
# ────────────────────────────────────────────────────────────

if __name__ == "__main__":
    uvicorn.run(
        "server:app",
        host=Config.HOST,
        port=Config.PORT,
        workers=Config.WORKERS,
        log_level="info",
        access_log=False,  # FastAPI handles logging
    )
```

### 5.7 test_server.py — Smoke Tests

```python
# src/reranker/test_server.py

"""
Run after server is up:
  python test_server.py
"""

import json
import time
import requests

BASE = "http://127.0.0.1:8086"

SAMPLE_DOCS = [
    {"id": "1", "content": "public class AuthService {\n  public boolean verifyToken(String jwt) {\n    return jwtProvider.validate(jwt);\n  }\n}"},
    {"id": "2", "content": "const LoginForm = () => {\n  const [email, setEmail] = useState('');\n  return <form onSubmit={handleLogin}>...</form>;\n}"},
    {"id": "3", "content": "spring.datasource.url=jdbc:postgresql://localhost:5432/bonds\nspring.datasource.username=admin"},
    {"id": "4", "content": "public class OrderService {\n  @Transactional\n  public Order submitOrder(OrderRequest req) {\n    validate(req);\n    return orderRepo.save(req.toEntity());\n  }\n}"},
    {"id": "5", "content": "export function calculateSettlement(trade: Trade): Settlement {\n  const days = getSettlementDays(trade.type);\n  return { date: addBusinessDays(trade.date, days) };\n}"},
]


def test_health():
    r = requests.get(f"{BASE}/health")
    data = r.json()
    assert data["status"] == "ok", f"Health check failed: {data}"
    assert data["models"]["biencoder"]["loaded"], "Bi-encoder not loaded"
    assert data["models"]["crossencoder"]["loaded"], "Cross-encoder not loaded"
    print("✅ Health check passed")


def test_biencoder():
    start = time.time()
    r = requests.post(f"{BASE}/v1/biencoder/rerank", json={
        "query": "JWT token authentication",
        "documents": SAMPLE_DOCS,
        "top_k": 3,
    })
    elapsed = int((time.time() - start) * 1000)

    data = r.json()
    assert r.status_code == 200, f"Bi-encoder failed: {r.text}"
    assert len(data["results"]) == 3
    assert data["results"][0]["id"] == "1", (
        f"Expected AuthService first, got id={data['results'][0]['id']}"
    )
    assert data["model"] == "CodeRankEmbed"

    print(f"✅ Bi-encoder passed ({elapsed}ms)")
    for res in data["results"]:
        print(f"   {res['id']}: {res['score']:.4f}")


def test_crossencoder():
    start = time.time()
    r = requests.post(f"{BASE}/v1/crossencoder/rerank", json={
        "query": "JWT token authentication",
        "documents": SAMPLE_DOCS,
        "top_k": 2,
        "instruction": "Rank code files implementing authentication and security patterns.",
    })
    elapsed = int((time.time() - start) * 1000)

    data = r.json()
    assert r.status_code == 200, f"Cross-encoder failed: {r.text}"
    assert len(data["results"]) == 2
    assert data["model"] == "Qwen3-Reranker-0.6B"

    print(f"✅ Cross-encoder passed ({elapsed}ms)")
    for res in data["results"]:
        print(f"   {res['id']}: {res['score']:.4f}")


def test_crossencoder_no_instruction():
    """Cross-encoder should work without instruction (uses default)."""
    r = requests.post(f"{BASE}/v1/crossencoder/rerank", json={
        "query": "order submission",
        "documents": SAMPLE_DOCS,
        "top_k": 2,
    })
    data = r.json()
    assert r.status_code == 200
    assert data["results"][0]["id"] == "4", "Expected OrderService first"
    print("✅ Cross-encoder without instruction passed")


def test_metrics():
    r = requests.get(f"{BASE}/metrics")
    data = r.json()
    assert data["biencoder"]["total_requests"] >= 1
    assert data["crossencoder"]["total_requests"] >= 1
    print(f"✅ Metrics: bi={data['biencoder']['total_requests']} reqs, "
          f"cross={data['crossencoder']['total_requests']} reqs")


if __name__ == "__main__":
    print(f"\nTesting reranker server at {BASE}\n")
    test_health()
    test_biencoder()
    test_crossencoder()
    test_crossencoder_no_instruction()
    test_metrics()
    print("\n✅ All tests passed\n")
```

### 5.8 requirements.txt

```
# src/reranker/requirements.txt

# Core
fastapi>=0.115.0
uvicorn[standard]>=0.32.0
pydantic>=2.9.0

# ML
torch>=2.1.0
sentence-transformers>=3.3.0
transformers>=4.45.0
numpy>=1.24.0

# Optional: faster tokenization
tokenizers>=0.20.0
```

-----

## 6. TypeScript Client

Single client class hitting both endpoints on the same server.

```typescript
// src/lib/reranker_client.ts

export interface RerankerResult {
  id: string;
  score: number;
}

export interface RerankerResponse {
  results: RerankerResult[];
  model: string;
  latency_ms: number;
  input_count: number;
}

export class RerankerClient {
  private baseUrl: string;
  private timeoutMs: number;

  constructor(
    baseUrl: string = process.env.RERANKER_URL || 'http://127.0.0.1:8086',
    timeoutMs: number = 15000
  ) {
    this.baseUrl = baseUrl;
    this.timeoutMs = timeoutMs;
  }

  async healthCheck(): Promise<{
    status: string;
    biencoder: boolean;
    crossencoder: boolean;
  }> {
    try {
      const res = await fetch(`${this.baseUrl}/health`, {
        signal: AbortSignal.timeout(3000),
      });
      const data = await res.json();
      return {
        status: data.status,
        biencoder: data.models.biencoder.loaded,
        crossencoder: data.models.crossencoder.loaded,
      };
    } catch {
      return { status: 'down', biencoder: false, crossencoder: false };
    }
  }

  /**
   * Stage 1: CodeRankEmbed bi-encoder.
   * Fast approximate scoring. Use to filter 80→20 candidates.
   */
  async biencoderRerank(
    query: string,
    documents: Array<{ id: string; content: string }>,
    topK: number = 20
  ): Promise<RerankerResult[]> {
    return this._call('/v1/biencoder/rerank', query, documents, topK);
  }

  /**
   * Stage 2: Qwen3-Reranker cross-encoder.
   * Accurate joint scoring. Use on 20 pre-filtered candidates.
   * Supports instruction-tuned reranking.
   */
  async crossencoderRerank(
    query: string,
    documents: Array<{ id: string; content: string }>,
    topK: number = 5,
    instruction?: string
  ): Promise<RerankerResult[]> {
    return this._call('/v1/crossencoder/rerank', query, documents, topK, instruction);
  }

  private async _call(
    endpoint: string,
    query: string,
    documents: Array<{ id: string; content: string }>,
    topK: number,
    instruction?: string
  ): Promise<RerankerResult[]> {
    const controller = new AbortController();
    const timeoutId = setTimeout(() => controller.abort(), this.timeoutMs);

    try {
      const body: Record<string, any> = { query, documents, top_k: topK };
      if (instruction) body.instruction = instruction;

      const response = await fetch(`${this.baseUrl}${endpoint}`, {
        method: 'POST',
        headers: { 'Content-Type': 'application/json' },
        body: JSON.stringify(body),
        signal: controller.signal,
      });

      if (!response.ok) {
        const text = await response.text();
        throw new Error(`Reranker ${endpoint} failed (${response.status}): ${text}`);
      }

      const data: RerankerResponse = await response.json();
      return data.results;

    } finally {
      clearTimeout(timeoutId);
    }
  }
}
```

-----

## 7. Integration with hybrid_search.ts

```typescript
// In hybrid_search.ts — replace existing BGE reranking section

import { RerankerClient } from './reranker_client';
import { getRerankerInstruction } from './reranker_instructions';

const rerankerClient = new RerankerClient();

// Feature flag — switch between old BGE and new cascaded pipeline
const USE_CASCADED = process.env.USE_CASCADED_RERANKING !== 'false';

async function rerankCandidates(
  query: string,
  candidates: SearchCandidate[],
  queryType: string,
  topK: number = 5
): Promise<SearchCandidate[]> {

  if (!USE_CASCADED) {
    // Old path — BGE via llama-server
    return await bgeReranker.rerank(query, candidates, topK);
  }

  // ── Dominant-winner short-circuit ──
  if (candidates.length > 0 && candidates[0].score >= 0.95) {
    return candidates.slice(0, topK);
  }

  // ── Score-ratio gating ──
  const topScore = Math.max(...candidates.map(c => c.score));
  let gated = candidates.filter(c => c.score >= topScore * 0.75);
  if (gated.length < 10) gated = candidates.slice(0, 20);

  // ── Dedup ──
  const seen = new Set<string>();
  const deduped = gated.filter(c => {
    const hash = c.contentHash || md5(c.content.slice(0, 500));
    if (seen.has(hash)) return false;
    seen.add(hash);
    return true;
  });

  // ── Stage 1: bi-encoder (N→20) ──
  let stage1 = deduped;

  if (deduped.length > 20) {
    try {
      const biResults = await rerankerClient.biencoderRerank(
        query,
        deduped.map(c => ({ id: c.chunkId, content: c.content.slice(0, 2048) })),
        20
      );
      const scoreMap = new Map(biResults.map(r => [r.id, r.score]));
      const ids = new Set(biResults.map(r => r.id));

      stage1 = deduped
        .filter(c => ids.has(c.chunkId))
        .map(c => ({ ...c, biScore: scoreMap.get(c.chunkId) ?? 0 }))
        .sort((a, b) => (b.biScore ?? 0) - (a.biScore ?? 0));

    } catch (err) {
      console.warn('[Rerank] Bi-encoder failed, passing 30 to cross-encoder:', err);
      stage1 = deduped.slice(0, 30);
    }
  }

  // ── Katz centrality fusion ──
  const maxCentrality = Math.max(
    ...stage1.map(c => c.graphNode?.centrality_score ?? 0), 0.001
  );
  stage1 = stage1.map(c => {
    const cn = (c.graphNode?.centrality_score ?? 0) / maxCentrality;
    return { ...c, fusedScore: 0.7 * (c.biScore ?? c.score) + 0.3 * cn };
  }).sort((a, b) => (b.fusedScore ?? 0) - (a.fusedScore ?? 0));

  // ── Stage 2: cross-encoder (20→topK) ──
  try {
    const instruction = getRerankerInstruction(queryType);
    const crossResults = await rerankerClient.crossencoderRerank(
      query,
      stage1.map(c => ({ id: c.chunkId, content: c.content })),
      topK,
      instruction
    );
    const finalMap = new Map(crossResults.map(r => [r.id, r.score]));
    const finalIds = new Set(crossResults.map(r => r.id));

    return stage1
      .filter(c => finalIds.has(c.chunkId))
      .map(c => ({ ...c, score: finalMap.get(c.chunkId) ?? c.score }))
      .sort((a, b) => b.score - a.score);

  } catch (err) {
    console.warn('[Rerank] Cross-encoder failed, returning bi-encoder top:', err);
    return stage1.slice(0, topK);
  }
}
```

-----

## 8. Systemd Service

```ini
# /etc/systemd/system/ome-reranker.service

[Unit]
Description=OME Reranker Service (CodeRankEmbed + Qwen3-Reranker)
After=network.target postgresql.service

[Service]
Type=simple
User=ome
WorkingDirectory=/opt/ome-rag/src/reranker

ExecStart=/usr/bin/python3 -m uvicorn server:app \
  --host 127.0.0.1 \
  --port 8086 \
  --workers 2 \
  --log-level info \
  --no-access-log

Restart=on-failure
RestartSec=10

# ── Model paths ──
Environment=BIENCODER_MODEL_PATH=/models/CodeRankEmbed
Environment=CROSSENCODER_MODEL_PATH=/models/Qwen3-Reranker-0.6B-seq-cls
Environment=RERANKER_PORT=8086
Environment=RERANKER_WORKERS=2
Environment=TORCH_THREADS=12
Environment=TOKENIZERS_PARALLELISM=false
Environment=TRANSFORMERS_CACHE=/models/.cache

# ── Resource limits ──
MemoryMax=5G
CPUQuota=1600%

# ── Logging ──
StandardOutput=journal
StandardError=journal
SyslogIdentifier=ome-reranker

[Install]
WantedBy=multi-user.target
```

```bash
# Enable and start
sudo systemctl daemon-reload
sudo systemctl enable ome-reranker
sudo systemctl start ome-reranker

# Check status
sudo systemctl status ome-reranker
journalctl -u ome-reranker -f

# Verify
curl http://127.0.0.1:8086/health | jq
```

-----

## 9. Air-Gapped Setup

### 9.1 Python Dependencies (Offline)

```bash
# On internet-connected machine:
mkdir -p /tmp/reranker-deps

pip download -d /tmp/reranker-deps \
  fastapi uvicorn pydantic \
  torch sentence-transformers transformers \
  numpy tokenizers

# Package
cd /tmp
tar czf reranker-deps.tar.gz reranker-deps/
```

### 9.2 Transfer Checklist

```
Files to transfer to air-gapped server:
──────────────────────────────────────────────────
/tmp/reranker-models.tar.gz          ~1.8 GB   (CodeRankEmbed + Qwen3-Reranker)
/tmp/reranker-deps.tar.gz            ~2.5 GB   (Python packages including PyTorch)
src/reranker/                        ~15 KB    (server code — 6 files)
──────────────────────────────────────────────────
Total transfer:                      ~4.3 GB
```

### 9.3 Install on Air-Gapped Server

```bash
# 1. Extract models
cd /models
tar xzf /path/to/reranker-models.tar.gz --strip-components=1

# Verify:
ls /models/CodeRankEmbed/config.json
ls /models/Qwen3-Reranker-0.6B-seq-cls/config.json

# 2. Install Python deps
tar xzf /path/to/reranker-deps.tar.gz
pip install --no-index --find-links=./reranker-deps \
  fastapi uvicorn pydantic \
  torch sentence-transformers transformers \
  numpy tokenizers \
  --break-system-packages

# 3. Deploy server code
cp -r src/reranker/ /opt/ome-rag/src/reranker/

# 4. Test manually
cd /opt/ome-rag/src/reranker
python3 server.py
# In another terminal:
python3 test_server.py

# 5. Install systemd service
sudo cp ome-reranker.service /etc/systemd/system/
sudo systemctl daemon-reload
sudo systemctl enable --now ome-reranker
```

-----

## 10. Health Monitoring

### 10.1 Add to Bootstrap Status Dashboard

The reranker health should appear in your existing bootstrap status page:

```typescript
// In src/api/bootstrap_status.ts — add reranker section

async function getRerankerStatus(): Promise<any> {
  try {
    const res = await fetch('http://127.0.0.1:8086/health', {
      signal: AbortSignal.timeout(3000),
    });
    return await res.json();
  } catch {
    return { status: 'down', models: {} };
  }
}

async function getRerankerMetrics(): Promise<any> {
  try {
    const res = await fetch('http://127.0.0.1:8086/metrics', {
      signal: AbortSignal.timeout(3000),
    });
    return await res.json();
  } catch {
    return null;
  }
}
```

### 10.2 Add to Startup Health Checks

```typescript
// In bootstrap.ts or server startup — verify reranker is available

const rerankerHealth = await rerankerClient.healthCheck();
if (rerankerHealth.status === 'down') {
  console.warn('[Startup] Reranker service not available — falling back to BGE');
} else {
  console.log(
    `[Startup] Reranker: bi-encoder=${rerankerHealth.biencoder}, ` +
    `cross-encoder=${rerankerHealth.crossencoder}`
  );
}
```

-----

## 11. Implementation Schedule

```
Day 1: Setup + Server
  ├── Transfer models + deps to air-gapped server
  ├── Install Python dependencies
  ├── Create config.py, schemas.py, metrics.py
  ├── Create models.py (both model loading + inference)
  ├── Create server.py (FastAPI + both endpoints)
  ├── Manual test: curl both endpoints
  ├── Run test_server.py smoke tests
  └── Install systemd service

Day 2: TypeScript Integration
  ├── Create RerankerClient (single client, two methods)
  ├── Create reranker_instructions.ts (per-query-type instructions)
  ├── Create query_type_detector.ts
  ├── Wire into hybrid_search.ts (feature-flagged)
  ├── Test with USE_CASCADED_RERANKING=true
  ├── Verify graceful degradation (stop reranker → falls back)
  └── Add health check to server startup

Day 3: Testing + Monitoring
  ├── Run golden queries: old pipeline vs new pipeline
  ├── A/B latency comparison
  ├── Verify source balance (code vs prose in top 5)
  ├── Test camelCase queries (useState, getOrderById)
  ├── Add reranker metrics to status dashboard
  ├── Tune: bi-encoder top_k (15 vs 20 vs 25)
  ├── Tune: cross-encoder instructions per query type
  └── Retire BGE reranker service (keep model file as fallback)
```

-----

## 12. Configuration Reference

### 12.1 Environment Variables

```bash
# ═══ Reranker Server ═══
RERANKER_HOST=127.0.0.1
RERANKER_PORT=8086
RERANKER_WORKERS=2

# ═══ Model Paths ═══
BIENCODER_MODEL_PATH=/models/CodeRankEmbed
CROSSENCODER_MODEL_PATH=/models/Qwen3-Reranker-0.6B-seq-cls

# ═══ Inference Limits ═══
MAX_BIENCODER_DOCS=100          # Max docs per bi-encoder request
MAX_CROSSENCODER_DOCS=30        # Max docs per cross-encoder request
BIENCODER_BATCH_SIZE=32         # Encoding batch size
BIENCODER_MAX_CHARS=2048        # Truncate docs for bi-encoder
CROSSENCODER_MAX_CHARS=6000     # Truncate docs for cross-encoder
TORCH_THREADS=12                # PyTorch thread count

# ═══ TypeScript Client ═══
RERANKER_URL=http://127.0.0.1:8086
USE_CASCADED_RERANKING=true     # false = fall back to BGE
```

### 12.2 Port Map (Updated)

```
Port    Service                    Status
──────────────────────────────────────────
:8081   Arctic Embed (prose)       Existing
:8084   BGE Reranker (llama)       RETIRING → fallback only
:8085   nomic-embed-code (llama)   Existing
:8086   Reranker Service (Python)  NEW — CodeRankEmbed + Qwen3-Reranker
:8090   Coder-Next (GPU)           Phase 1
:8091   235B-A22B (GPU)            Phase 1
```

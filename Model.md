# Phase 3 — Model Optimization & Reranking Pipeline Upgrade

> **Project:** OME RAG + OME Agent
> **Server:** RHEL 9.7, 128 cores, 1TB RAM, NVIDIA H100 (93.6 GB VRAM)
> **Prerequisite:** Phase 1 (Agent GPU models deployed), Phase 2 (basic agent working)
> **Effort:** ~8 days
> **Last Updated:** April 2026

-----

## Table of Contents

1. [Phase Context — Where We Are](#1-phase-context)
1. [Models Inventory — Download Reference](#2-models-inventory)
1. [CodeRankEmbed — Bi-Encoder Reranker (CPU)](#3-coderankembed)
1. [Qwen3-Reranker-0.6B — Cross-Encoder Upgrade (CPU)](#4-qwen3-reranker)
1. [Reranking Pipeline Rewiring](#5-reranking-pipeline-rewiring)
1. [Benchmarking & A/B Testing](#6-benchmarking)
1. [Future GPU Migration Path](#7-future-gpu-migration)
1. [Implementation Schedule](#8-implementation-schedule)
1. [Configuration Reference](#9-configuration)
1. [File Structure](#10-file-structure)

-----

## 1. Phase Context

### 1.1 Phase Progression

```
Phase 1 (DONE): Agent Deployment
  ├── Qwen3-Coder-Next on GPU (:8090) — fast reasoning
  ├── Qwen3-235B-A22B on GPU (:8091) — deep code generation
  ├── Qwen2.5-7B retired — Coder-Next takes over headers, condensation, health score
  └── All 128 CPU cores freed for embeddings + application

Phase 2 (DONE): Agent Core Logic
  ├── ome_rag_guide working (understand/implement/diagnose)
  ├── PR vector embedding + hybrid search
  ├── Intelligence layer refactored to lightweight mode
  └── Agent consuming RAG via MCP client

Phase 3 (THIS PLAN): Model Optimization ← YOU ARE HERE
  ├── Add CodeRankEmbed bi-encoder reranker (CPU)
  ├── Replace BGE Reranker with Qwen3-Reranker-0.6B (CPU, then GPU)
  ├── Rewire reranking pipeline: 2-stage cascaded
  ├── Benchmark and A/B test against current pipeline
  └── Optional: move reranker to GPU for further latency reduction

Phase 4 (FUTURE): Advanced Optimizations
  ├── Evaluate Qwen3-Embedding-0.6B as Arctic replacement
  ├── nomic-embed-code GPU FP16 for query-time (if latency needs it)
  └── CodeRankLLM 7B as listwise reranker (if quality needs it)
```

### 1.2 Current Query Pipeline Latency

```
Step                              Time
──────────────────────────────────────
Query embedding (nomic CPU)       ~200ms
Composite vector expansion        ~80ms
Dense ANN (HNSW)                  ~80ms
BM25 FTS                          ~30ms
RRF fusion                        ~5ms
Graph expansion                   ~20-50ms
Centrality scoring                ~5ms
BGE cross-encoder rerank (80→5)   ~6s       ← BOTTLENECK
MMR diversification               ~5ms
──────────────────────────────────────
Total P95:                        ~6.5s
```

### 1.3 Target After Phase 3

```
Step                              Time
──────────────────────────────────────
Query embedding (nomic CPU)       ~200ms
Composite vector expansion        ~80ms
Dense ANN (HNSW)                  ~80ms
BM25 FTS                          ~30ms
RRF fusion                        ~5ms
Graph expansion                   ~20-50ms
Centrality scoring                ~5ms
CodeRankEmbed bi-encode (80→20)   ~400ms    ← NEW
Katz centrality fusion            ~5ms      ← NEW
Qwen3-Reranker cross-encode(20→5) ~1.2s    ← UPGRADED
MMR diversification               ~5ms
──────────────────────────────────────
Total P95:                        ~2.1s     (3x improvement)
```

-----

## 2. Models Inventory — Download Reference

### 2.1 Models to Download (Air-Gapped Transfer)

|#|Model                        |HuggingFace ID                         |Size          |License   |Purpose                                     |
|-|-----------------------------|---------------------------------------|--------------|----------|--------------------------------------------|
|1|CodeRankEmbed                |`nomic-ai/CodeRankEmbed`               |~522 MB       |MIT       |Bi-encoder reranker (Stage 1)               |
|2|Qwen3-Reranker-0.6B          |`Qwen/Qwen3-Reranker-0.6B`             |~1.2 GB (FP16)|Apache 2.0|Cross-encoder reranker (Stage 2)            |
|3|Qwen3-Reranker-0.6B (seq-cls)|`tomaarsen/Qwen3-Reranker-0.6B-seq-cls`|~1.2 GB       |Apache 2.0|Simplified SentenceTransformers CrossEncoder|
|4|Qwen3-Reranker-0.6B GGUF     |`Mungert/Qwen3-Reranker-0.6B-GGUF`     |396 MB–1.2 GB |Apache 2.0|llama-server deployment (GPU path)          |

### 2.2 Models Already In Use (No Change)

|Model                        |HuggingFace ID                           |Port |Purpose                            |
|-----------------------------|-----------------------------------------|-----|-----------------------------------|
|nomic-embed-code 7B GGUF     |`nomic-ai/nomic-embed-code-GGUF`         |:8085|Code embedding (overnight + query) |
|snowflake-arctic-embed-l-v2.0|`Snowflake/snowflake-arctic-embed-l-v2.0`|:8081|Prose embedding                    |
|BGE Reranker v2-m3           |`BAAI/bge-reranker-v2-m3`                |:8084|Cross-encoder (RETIRING in Phase 3)|

### 2.3 Models to Retire

|Model                       |Reason                                                                    |
|----------------------------|--------------------------------------------------------------------------|
|BGE Reranker v2-m3 (~560 MB)|Replaced by Qwen3-Reranker-0.6B (better quality, code-aware, multilingual)|

### 2.4 Future Models (Phase 4, Not Now)

|Model               |HuggingFace ID             |Size   |Purpose                                                 |
|--------------------|---------------------------|-------|--------------------------------------------------------|
|CodeRankLLM         |`nomic-ai/CodeRankLLM`     |~14 GB |7B listwise code reranker (evaluate if quality needs it)|
|Qwen3-Embedding-0.6B|`Qwen/Qwen3-Embedding-0.6B`|~1.2 GB|Potential Arctic replacement for prose                  |
|Qwen3-Reranker-4B   |`Qwen/Qwen3-Reranker-4B`   |~8 GB  |Higher quality reranker if 0.6B isn’t sufficient        |

### 2.5 Download Instructions (Air-Gapped)

```bash
# On internet-connected machine:

# 1. CodeRankEmbed (SentenceTransformers format)
pip install huggingface_hub
huggingface-cli download nomic-ai/CodeRankEmbed --local-dir ./CodeRankEmbed

# 2. Qwen3-Reranker-0.6B (for SentenceTransformers CrossEncoder)
#    Use the pre-converted seq-cls variant for simpler deployment
huggingface-cli download tomaarsen/Qwen3-Reranker-0.6B-seq-cls --local-dir ./Qwen3-Reranker-0.6B-seq-cls

# 3. Qwen3-Reranker-0.6B GGUF (for future GPU deployment via llama-server)
huggingface-cli download Mungert/Qwen3-Reranker-0.6B-GGUF \
  --include "Qwen3-Reranker-0.6B-F16.gguf" \
  --local-dir ./Qwen3-Reranker-0.6B-GGUF

# Transfer to air-gapped server:
tar czf phase3-models.tar.gz CodeRankEmbed/ Qwen3-Reranker-0.6B-seq-cls/ Qwen3-Reranker-0.6B-GGUF/
scp phase3-models.tar.gz user@airgapped-server:/models/

# On air-gapped server:
cd /models && tar xzf phase3-models.tar.gz
```

-----

## 3. CodeRankEmbed — Bi-Encoder Reranker (CPU)

### 3.1 What It Does

CodeRankEmbed is a 137M parameter bi-encoder from the nomic-ai family (same team that built your nomic-embed-code). It was trained on CoRNStack — 21 million code retrieval examples with progressive hard negative mining.

**Bi-encoder vs Cross-encoder:**

```
Bi-encoder (CodeRankEmbed):
  encode(query) → q_vec       }
  encode(doc)   → d_vec       } independent
  score = dot(q_vec, d_vec)   } fast: ~5ms per pair

Cross-encoder (BGE / Qwen3-Reranker):
  encode(query + doc) → score   } joint attention
                                } accurate: ~75ms per pair
```

The bi-encoder is 15x faster but less accurate. It acts as a fast filter before the expensive cross-encoder — throwing away the bottom 75% of candidates so the cross-encoder only scores the survivors.

### 3.2 Why CodeRankEmbed Specifically

- **Same family as your embedding model** — nomic-ai trained both nomic-embed-code and CodeRankEmbed on CoRNStack. Uses the same prefix convention: `"Represent this query for searching relevant code: "`
- **Code-specialized** — trained on code retrieval, not general text. Understands that `getOrderById` is more relevant to “order retrieval” than `OrderConstants.java`
- **137M parameters** — tiny footprint on CPU, ~522 MB disk, ~600 MB RAM
- **8192 token context** — handles long code chunks without truncation
- **Built on Arctic-Embed-M-Long** — same architecture family as your Arctic prose embedder

### 3.3 Python Server

```python
# src/reranker/coderank_server.py

import os
import json
import time
import logging
from flask import Flask, request, jsonify
from sentence_transformers import SentenceTransformer
import numpy as np

logging.basicConfig(level=logging.INFO, format='%(asctime)s [CodeRankEmbed] %(message)s')
logger = logging.getLogger(__name__)

app = Flask(__name__)

MODEL_PATH = os.environ.get('CODERANK_MODEL_PATH', '/models/CodeRankEmbed')
MAX_DOC_TOKENS = int(os.environ.get('CODERANK_MAX_TOKENS', '512'))

logger.info(f"Loading CodeRankEmbed from {MODEL_PATH}...")
model = SentenceTransformer(MODEL_PATH, trust_remote_code=True)
logger.info("CodeRankEmbed loaded successfully")


@app.route("/health", methods=["GET"])
def health():
    return jsonify({"status": "ok", "model": "CodeRankEmbed", "parameters": "137M"})


@app.route("/rerank", methods=["POST"])
def rerank():
    start_ms = time.time() * 1000
    data = request.json

    query = data.get("query", "")
    documents = data.get("documents", [])   # [{id, content}]
    top_k = data.get("top_k", 20)

    if not query or not documents:
        return jsonify({"error": "query and documents required"}), 400

    # ── Encode query with nomic prefix ──
    # CodeRankEmbed uses same prefix as nomic-embed-code
    prefixed_query = f"Represent this query for searching relevant code: {query}"
    q_emb = model.encode(
        [prefixed_query],
        normalize_embeddings=True,
        show_progress_bar=False
    )

    # ── Encode documents ──
    # No document prefix needed for CodeRankEmbed (unlike nomic-embed-code)
    doc_texts = []
    for d in documents:
        content = d.get("content", "")
        # Truncate to avoid memory issues — bi-encoder doesn't need full content
        lines = content.split('\n')
        if len(lines) > 50:
            content = '\n'.join(lines[:50])
        doc_texts.append(content[:2048])  # Hard char limit

    doc_embs = model.encode(
        doc_texts,
        normalize_embeddings=True,
        batch_size=32,
        show_progress_bar=False
    )

    # ── Compute scores via dot product ──
    scores = (doc_embs @ q_emb.T).flatten().tolist()

    # ── Build results sorted by score ──
    results = []
    for i, d in enumerate(documents):
        results.append({
            "id": d["id"],
            "score": round(scores[i], 6),
        })

    results.sort(key=lambda x: x["score"], reverse=True)
    results = results[:top_k]

    elapsed_ms = round(time.time() * 1000 - start_ms)
    logger.info(
        f"Reranked {len(documents)} docs → top {top_k} in {elapsed_ms}ms "
        f"(top score: {results[0]['score']:.4f})"
    )

    return jsonify({
        "results": results,
        "latency_ms": elapsed_ms,
        "input_count": len(documents),
    })


if __name__ == "__main__":
    port = int(os.environ.get("CODERANK_PORT", "8086"))
    logger.info(f"Starting CodeRankEmbed server on port {port}")
    app.run(host="127.0.0.1", port=port, threaded=True)
```

### 3.4 TypeScript Client

```typescript
// src/lib/coderank_client.ts

export interface CodeRankResult {
  id: string;
  score: number;
}

export class CodeRankClient {
  private baseUrl: string;
  private timeoutMs: number;

  constructor(baseUrl: string = 'http://127.0.0.1:8086', timeoutMs: number = 10000) {
    this.baseUrl = baseUrl;
    this.timeoutMs = timeoutMs;
  }

  async healthCheck(): Promise<boolean> {
    try {
      const res = await fetch(`${this.baseUrl}/health`, {
        signal: AbortSignal.timeout(3000),
      });
      return res.ok;
    } catch {
      return false;
    }
  }

  async rerank(
    query: string,
    documents: Array<{ id: string; content: string }>,
    topK: number = 20
  ): Promise<CodeRankResult[]> {
    const controller = new AbortController();
    const timeoutId = setTimeout(() => controller.abort(), this.timeoutMs);

    try {
      const response = await fetch(`${this.baseUrl}/rerank`, {
        method: 'POST',
        headers: { 'Content-Type': 'application/json' },
        body: JSON.stringify({
          query,
          documents,
          top_k: topK,
        }),
        signal: controller.signal,
      });

      if (!response.ok) {
        throw new Error(`CodeRankEmbed rerank failed: ${response.status}`);
      }

      const data = await response.json();
      return data.results as CodeRankResult[];

    } finally {
      clearTimeout(timeoutId);
    }
  }
}
```

### 3.5 Systemd Service

```ini
# /etc/systemd/system/ome-coderank.service

[Unit]
Description=CodeRankEmbed Bi-Encoder Reranker
After=network.target

[Service]
Type=simple
User=ome
WorkingDirectory=/opt/ome-rag
ExecStart=/usr/bin/python3 src/reranker/coderank_server.py
Restart=on-failure
RestartSec=5

Environment=CODERANK_MODEL_PATH=/models/CodeRankEmbed
Environment=CODERANK_PORT=8086
Environment=CODERANK_MAX_TOKENS=512
Environment=TRANSFORMERS_CACHE=/models/.cache
Environment=TOKENIZERS_PARALLELISM=false

# Resource limits
MemoryMax=2G
CPUQuota=400%

[Install]
WantedBy=multi-user.target
```

### 3.6 Key Implementation Notes

**Document prefix:** CodeRankEmbed does NOT need a document-side prefix. Only the query needs `"Represent this query for searching relevant code: "`. This is different from nomic-embed-code which prefixes both sides.

**Content truncation:** The bi-encoder doesn’t benefit from full file content. Truncate documents to ~50 lines or 2048 chars. The cross-encoder stage (Qwen3-Reranker) handles full context for the top 20 survivors.

**Output dimensionality:** CodeRankEmbed produces 768-dimensional vectors. These are NOT stored — they’re computed at query time and discarded after scoring. No schema changes needed.

-----

## 4. Qwen3-Reranker-0.6B — Cross-Encoder Upgrade (CPU)

### 4.1 Why Replace BGE Reranker

|Aspect               |BGE Reranker v2-m3      |Qwen3-Reranker-0.6B                                         |
|---------------------|------------------------|------------------------------------------------------------|
|Parameters           |~560M                   |~600M                                                       |
|Architecture         |BERT-based cross-encoder|Qwen3 decoder, sequence classification                      |
|Code awareness       |General text only       |Trained on code retrieval tasks (MTEB-Code)                 |
|Multilingual         |100+ languages          |100+ languages                                              |
|Context length       |8192 tokens             |32K tokens                                                  |
|Ranking approach     |Pairwise scoring        |Pairwise with instruction support                           |
|Instruction-tunable  |No                      |Yes — custom task instructions improve domain performance   |
|Speed (CPU, per pair)|~75ms                   |~80ms (slightly larger, but instruction quality compensates)|

The key advantage is **instruction-tunable reranking**. You can tell Qwen3-Reranker exactly what you’re looking for:

```
Instruction: "Rank code files relevant to this query about Spring Boot authentication, 
prioritizing controller and service layer implementations over configuration or test files."
```

BGE has no mechanism for this — it’s a fixed relevance scorer.

### 4.2 Python Server

```python
# src/reranker/qwen3_reranker_server.py

import os
import json
import time
import logging
from flask import Flask, request, jsonify
from sentence_transformers import CrossEncoder

logging.basicConfig(level=logging.INFO, format='%(asctime)s [Qwen3Reranker] %(message)s')
logger = logging.getLogger(__name__)

app = Flask(__name__)

# Use the pre-converted seq-cls variant for simpler deployment
MODEL_PATH = os.environ.get('QWEN3_RERANKER_PATH', '/models/Qwen3-Reranker-0.6B-seq-cls')

logger.info(f"Loading Qwen3-Reranker from {MODEL_PATH}...")
model = CrossEncoder(
    MODEL_PATH,
    trust_remote_code=True,
    automodel_args={"torch_dtype": "auto"},
)
model.model.eval()
logger.info("Qwen3-Reranker loaded successfully")

# ── Default instruction for code retrieval ──
DEFAULT_INSTRUCTION = (
    "Given a code search query, determine if the code snippet is relevant. "
    "Consider function names, class names, business logic, API endpoints, "
    "architectural patterns, and call relationships."
)


@app.route("/health", methods=["GET"])
def health():
    return jsonify({"status": "ok", "model": "Qwen3-Reranker-0.6B", "parameters": "600M"})


@app.route("/rerank", methods=["POST"])
def rerank():
    start_ms = time.time() * 1000
    data = request.json

    query = data.get("query", "")
    documents = data.get("documents", [])   # [{id, content}]
    top_k = data.get("top_k", 5)
    instruction = data.get("instruction", DEFAULT_INSTRUCTION)

    if not query or not documents:
        return jsonify({"error": "query and documents required"}), 400

    # ── Build query-document pairs ──
    pairs = []
    for d in documents:
        content = d.get("content", "")
        # Cross-encoder benefits from more context than bi-encoder
        lines = content.split('\n')
        if len(lines) > 150:
            head = '\n'.join(lines[:105])
            tail = '\n'.join(lines[-30:])
            content = f"{head}\n\n// ... {len(lines) - 135} lines omitted ...\n\n{tail}"
        pairs.append((query, content))

    # ── Score pairs with instruction ──
    # Qwen3-Reranker accepts instruction as part of the query
    instructed_query = f"Instruct: {instruction}\nQuery: {query}"
    instructed_pairs = [(instructed_query, doc) for _, doc in pairs]

    scores = model.predict(instructed_pairs, show_progress_bar=False).tolist()

    # ── Build sorted results ──
    results = []
    for i, d in enumerate(documents):
        results.append({
            "id": d["id"],
            "score": round(float(scores[i]), 6),
        })

    results.sort(key=lambda x: x["score"], reverse=True)
    results = results[:top_k]

    elapsed_ms = round(time.time() * 1000 - start_ms)
    logger.info(
        f"Reranked {len(documents)} docs → top {top_k} in {elapsed_ms}ms "
        f"(top score: {results[0]['score']:.4f})"
    )

    return jsonify({
        "results": results,
        "latency_ms": elapsed_ms,
        "input_count": len(documents),
    })


if __name__ == "__main__":
    port = int(os.environ.get("QWEN3_RERANKER_PORT", "8087"))
    logger.info(f"Starting Qwen3-Reranker server on port {port}")
    app.run(host="127.0.0.1", port=port, threaded=True)
```

### 4.3 TypeScript Client

```typescript
// src/lib/qwen3_reranker_client.ts

export interface Qwen3RerankerResult {
  id: string;
  score: number;
}

export class Qwen3RerankerClient {
  private baseUrl: string;
  private timeoutMs: number;

  constructor(baseUrl: string = 'http://127.0.0.1:8087', timeoutMs: number = 15000) {
    this.baseUrl = baseUrl;
    this.timeoutMs = timeoutMs;
  }

  async healthCheck(): Promise<boolean> {
    try {
      const res = await fetch(`${this.baseUrl}/health`, {
        signal: AbortSignal.timeout(3000),
      });
      return res.ok;
    } catch {
      return false;
    }
  }

  async rerank(
    query: string,
    documents: Array<{ id: string; content: string }>,
    topK: number = 5,
    instruction?: string
  ): Promise<Qwen3RerankerResult[]> {
    const controller = new AbortController();
    const timeoutId = setTimeout(() => controller.abort(), this.timeoutMs);

    try {
      const body: Record<string, any> = { query, documents, top_k: topK };
      if (instruction) body.instruction = instruction;

      const response = await fetch(`${this.baseUrl}/rerank`, {
        method: 'POST',
        headers: { 'Content-Type': 'application/json' },
        body: JSON.stringify(body),
        signal: controller.signal,
      });

      if (!response.ok) {
        throw new Error(`Qwen3-Reranker failed: ${response.status}`);
      }

      const data = await response.json();
      return data.results as Qwen3RerankerResult[];

    } finally {
      clearTimeout(timeoutId);
    }
  }
}
```

### 4.4 Dimension-Specific Instructions

The instruction-tunable nature of Qwen3-Reranker allows per-query-type optimization:

```typescript
// src/lib/reranker_instructions.ts

export function getRerankerInstruction(queryType: string): string {
  switch (queryType) {
    case 'identifier':
      return (
        'Rank code snippets by exact or near-exact match to the identifier name. ' +
        'Prioritize definitions over usages, and source files over test files.'
      );

    case 'architecture':
      return (
        'Rank code relevant to software architecture and design patterns. ' +
        'Prioritize controllers, services, configuration classes, and entry points ' +
        'over utilities, DTOs, and test files.'
      );

    case 'debugging':
      return (
        'Rank code relevant to diagnosing this error or bug. ' +
        'Prioritize exception handling, error paths, validation logic, ' +
        'and recently changed code.'
      );

    case 'implementation':
      return (
        'Rank code files that would need modification to implement this feature. ' +
        'Prioritize service layer, controller endpoints, and data access objects. ' +
        'Include test files that would need updating.'
      );

    default:
      return (
        'Given a code search query, determine if the code snippet is relevant. ' +
        'Consider function names, class names, business logic, API endpoints, ' +
        'architectural patterns, and call relationships.'
      );
  }
}
```

### 4.5 Systemd Service

```ini
# /etc/systemd/system/ome-qwen3-reranker.service

[Unit]
Description=Qwen3-Reranker-0.6B Cross-Encoder
After=network.target

[Service]
Type=simple
User=ome
WorkingDirectory=/opt/ome-rag
ExecStart=/usr/bin/python3 src/reranker/qwen3_reranker_server.py
Restart=on-failure
RestartSec=5

Environment=QWEN3_RERANKER_PATH=/models/Qwen3-Reranker-0.6B-seq-cls
Environment=QWEN3_RERANKER_PORT=8087
Environment=TRANSFORMERS_CACHE=/models/.cache
Environment=TOKENIZERS_PARALLELISM=false

# Resource limits — cross-encoder needs more memory than bi-encoder
MemoryMax=4G
CPUQuota=800%

[Install]
WantedBy=multi-user.target
```

-----

## 5. Reranking Pipeline Rewiring

### 5.1 Current Pipeline (Before Phase 3)

```
RRF fusion (80 candidates)
  │
  ├── Graph expansion → ~80 candidates
  ├── Score-ratio gating (<75% of top) → ~60 candidates
  ├── Content dedup → ~55 candidates
  │
  └── BGE cross-encoder (55→5)         ~55 × 75ms = ~4.1s
      └── MMR diversification → final 5
```

### 5.2 New Pipeline (After Phase 3)

```
RRF fusion (80 candidates)
  │
  ├── Graph expansion → ~80 candidates
  ├── Dominant-winner short-circuit (≥0.95 → skip reranking, return top 5)
  ├── Score-ratio gating (<75% of top) → ~60 candidates
  ├── Content dedup (MD5 hash) → ~55 candidates
  │
  ├── Stage 1: CodeRankEmbed bi-encoder (55→20)    ~55 × 5ms = ~275ms
  │     └── Katz centrality fusion (0.7 × bi + 0.3 × centrality)
  │
  └── Stage 2: Qwen3-Reranker cross-encoder (20→5) ~20 × 60ms = ~1.2s
        └── MMR diversification → final 5

Total reranking: ~1.5s (was ~4.1s)
```

### 5.3 Integration in hybrid_search.ts

```typescript
// src/lib/hybrid_search.ts — Phase 3 reranking section

import { CodeRankClient } from './coderank_client';
import { Qwen3RerankerClient } from './qwen3_reranker_client';
import { getRerankerInstruction } from './reranker_instructions';

// Initialize clients (in module scope or constructor)
const coderankClient = new CodeRankClient(
  process.env.CODERANK_URL || 'http://127.0.0.1:8086'
);
const qwen3RerankerClient = new Qwen3RerankerClient(
  process.env.QWEN3_RERANKER_URL || 'http://127.0.0.1:8087'
);

// Feature flag for gradual rollout
const USE_CASCADED_RERANKING = process.env.USE_CASCADED_RERANKING !== 'false';

async function rerankCandidates(
  query: string,
  candidates: SearchCandidate[],
  queryType: string,
  topK: number = 5
): Promise<SearchCandidate[]> {

  // ── Fallback path: use old BGE reranker if cascaded is disabled ──
  if (!USE_CASCADED_RERANKING) {
    return await bgeReranker.rerank(query, candidates, topK);
  }

  // ── Dominant-winner short-circuit ──
  if (candidates.length > 0 && candidates[0].score >= 0.95) {
    console.log('[Rerank] Dominant winner detected, skipping reranking');
    return candidates.slice(0, topK);
  }

  // ── Score-ratio gating ──
  const topScore = Math.max(...candidates.map(c => c.score));
  const gatingThreshold = topScore * 0.75;
  let gated = candidates.filter(c => c.score >= gatingThreshold);

  // Ensure minimum candidates survive
  if (gated.length < 10) {
    gated = candidates.slice(0, Math.min(candidates.length, 20));
  }

  // ── Content deduplication ──
  const seen = new Set<string>();
  const deduped = gated.filter(c => {
    const hash = c.contentHash || md5(c.content.slice(0, 500));
    if (seen.has(hash)) return false;
    seen.add(hash);
    return true;
  });

  // ── Stage 1: CodeRankEmbed bi-encoder (N → 20) ──
  let stage1Results = deduped;

  if (deduped.length > 20) {
    try {
      const coderankResults = await coderankClient.rerank(
        query,
        deduped.map(c => ({
          id: c.chunkId,
          content: c.content.slice(0, 2048),  // Bi-encoder: truncate aggressively
        })),
        20
      );

      const scoreMap = new Map(coderankResults.map(r => [r.id, r.score]));
      const rankedIds = new Set(coderankResults.map(r => r.id));

      stage1Results = deduped
        .filter(c => rankedIds.has(c.chunkId))
        .map(c => ({
          ...c,
          biEncoderScore: scoreMap.get(c.chunkId) ?? 0,
        }));

    } catch (err) {
      console.warn('[Rerank] CodeRankEmbed failed, passing all to cross-encoder:', err);
      // Graceful degradation — limit to 30 for cross-encoder
      stage1Results = deduped.slice(0, 30);
    }
  }

  // ── Katz centrality fusion (between bi-encoder and cross-encoder) ──
  if (stage1Results.length > 0 && stage1Results[0].biEncoderScore !== undefined) {
    const maxCentrality = Math.max(
      ...stage1Results.map(c => c.graphNode?.centrality_score ?? 0),
      0.001
    );

    stage1Results = stage1Results
      .map(c => {
        const centralityNorm = (c.graphNode?.centrality_score ?? 0) / maxCentrality;
        const fusedScore = 0.7 * (c.biEncoderScore ?? c.score) + 0.3 * centralityNorm;
        return { ...c, fusedScore };
      })
      .sort((a, b) => (b.fusedScore ?? 0) - (a.fusedScore ?? 0));
  }

  // ── Stage 2: Qwen3-Reranker cross-encoder (20 → topK) ──
  try {
    const instruction = getRerankerInstruction(queryType);

    const crossEncoderResults = await qwen3RerankerClient.rerank(
      query,
      stage1Results.map(c => ({
        id: c.chunkId,
        content: c.content,  // Cross-encoder: full content (up to 150 lines)
      })),
      topK,
      instruction
    );

    const finalScoreMap = new Map(crossEncoderResults.map(r => [r.id, r.score]));
    const finalIds = new Set(crossEncoderResults.map(r => r.id));

    return stage1Results
      .filter(c => finalIds.has(c.chunkId))
      .map(c => ({
        ...c,
        crossEncoderScore: finalScoreMap.get(c.chunkId) ?? 0,
        score: finalScoreMap.get(c.chunkId) ?? c.score,
      }))
      .sort((a, b) => b.score - a.score);

  } catch (err) {
    console.warn('[Rerank] Qwen3-Reranker failed, returning bi-encoder results:', err);
    // Graceful degradation — return CodeRankEmbed top-K
    return stage1Results.slice(0, topK);
  }
}
```

### 5.4 Graceful Degradation Matrix

```
CodeRankEmbed    Qwen3-Reranker    Behavior
────────────────────────────────────────────────────────
    ✅ UP           ✅ UP           Full cascaded pipeline (optimal)
    ✅ UP           ❌ DOWN         Bi-encoder only, return top-5 (good)
    ❌ DOWN         ✅ UP           Pass 30 candidates to cross-encoder (OK)
    ❌ DOWN         ❌ DOWN         Score-ratio gating only, no rerank (degraded)
```

### 5.5 Query Type Detection

The `queryType` is needed for Qwen3-Reranker instruction selection. Derive from query characteristics:

```typescript
// src/lib/query_type_detector.ts

export function detectQueryType(query: string): string {
  const lower = query.toLowerCase().trim();

  // Single identifier — exact match
  if (/^[a-zA-Z_]\w*(\.[a-zA-Z_]\w*)*$/.test(query.trim())) {
    return 'identifier';
  }

  // Error/debug patterns
  if (/\b(error|exception|fail|bug|crash|null|undefined|timeout|500|404|NPE)\b/i.test(lower)) {
    return 'debugging';
  }

  // Implementation patterns
  if (/^(add|create|implement|build|modify|refactor)\b/i.test(lower)) {
    return 'implementation';
  }

  // Architecture patterns
  if (/\b(architecture|pattern|design|structure|flow|entry.?point|how does)\b/i.test(lower)) {
    return 'architecture';
  }

  return 'default';
}
```

-----

## 6. Benchmarking & A/B Testing

### 6.1 Golden Query Test Suite

Expand your existing golden queries to cover reranking quality:

```typescript
// test_script/benchmark_reranking.ts

const GOLDEN_QUERIES = [
  // ── Identifier queries ──
  {
    query: 'getOrderById',
    expectedTop3Files: ['order/OrderService', 'order/OrderRepository'],
    queryType: 'identifier',
  },
  {
    query: 'verifyJwtToken',
    expectedTop3Files: ['auth/JwtTokenProvider', 'auth/TokenService'],
    queryType: 'identifier',
  },

  // ── Architecture queries ──
  {
    query: 'how does bond trading authentication work',
    expectedTop3Files: ['auth/', 'security/', 'config/SecurityConfig'],
    queryType: 'architecture',
  },
  {
    query: 'kafka consumer retry handling',
    expectedTop3Files: ['messaging/', 'consumer/', 'config/KafkaConfig'],
    queryType: 'architecture',
  },

  // ── Debugging queries ──
  {
    query: 'NullPointerException in SettlementService',
    expectedTop3Files: ['settlement/SettlementService'],
    queryType: 'debugging',
  },

  // ── Implementation queries ──
  {
    query: 'add pagination to bond search API',
    expectedTop3Files: ['bond/BondController', 'bond/BondService', 'bond/BondRepository'],
    queryType: 'implementation',
  },

  // ── camelCase edge cases ──
  {
    query: 'useState useEffect',
    expectedTop3Files: [],  // Should return code, not prose
    mustHaveCodeResults: true,
  },
  {
    query: 'ThreadPoolTaskExecutor configuration',
    expectedTop3Files: ['config/AsyncConfig', 'config/ThreadPoolConfig'],
    queryType: 'architecture',
  },
];
```

### 6.2 A/B Comparison Script

```typescript
// test_script/ab_reranking.ts

async function compareRerankers(query: string, queryType: string) {
  // Get candidates (same for both paths)
  const candidates = await getRRFCandidates(query);

  // ── Path A: Old pipeline (BGE only) ──
  const startA = performance.now();
  const pathA = await bgeReranker.rerank(query, candidates, 5);
  const timeA = Math.round(performance.now() - startA);

  // ── Path B: New pipeline (CodeRankEmbed → Qwen3-Reranker) ──
  const startB = performance.now();
  const stage1 = await coderankClient.rerank(query, candidates, 20);
  const stage1Ids = new Set(stage1.map(r => r.id));
  const stage1Candidates = candidates.filter(c => stage1Ids.has(c.chunkId));
  const pathB = await qwen3RerankerClient.rerank(query, stage1Candidates, 5);
  const timeB = Math.round(performance.now() - startB);

  return {
    query,
    pathA: { results: pathA, timeMs: timeA },
    pathB: { results: pathB, timeMs: timeB },
    speedup: `${(timeA / timeB).toFixed(1)}x`,
  };
}
```

### 6.3 Metrics to Track

```typescript
// src/lib/reranker_metrics.ts

interface RerankerMetrics {
  // Latency
  biEncoderLatencyMs: number;
  crossEncoderLatencyMs: number;
  totalRerankLatencyMs: number;

  // Counts
  inputCandidates: number;
  afterBiEncoder: number;
  afterCrossEncoder: number;

  // Quality signals
  topScoreBiEncoder: number;
  topScoreCrossEncoder: number;
  scoreSpread: number;  // max - min in top 5

  // Degradation
  biEncoderFailed: boolean;
  crossEncoderFailed: boolean;
  fallbackUsed: boolean;
}

// Store in ingestion_metrics or a new reranker_metrics table
```

-----

## 7. Future GPU Migration Path

### 7.1 When to Consider GPU Reranker

Move Qwen3-Reranker to GPU if:

- Query P95 latency still exceeds 2s after Phase 3
- Cross-encoder becomes the bottleneck after bi-encoder filters to 20
- Concurrent queries cause CPU contention between reranker and embedding

### 7.2 GPU Deployment via llama-server (GGUF)

```bash
# Qwen3-Reranker GGUF on GPU — only ~1.2 GB VRAM
# NOTE: Requires special llama.cpp build with reranker support
# See: https://github.com/ngxson/llama.cpp/tree/xsn/qwen3_embd_rerank

./llama-server \
  --model /models/Qwen3-Reranker-0.6B-GGUF/Qwen3-Reranker-0.6B-F16.gguf \
  --host 127.0.0.1 --port 8087 \
  --n-gpu-layers 99 \
  --ctx-size 4096 --parallel 4 \
  --threads 4 --flash-attn \
  --reranking
```

**VRAM impact:** ~1.2 GB for FP16. You have ~8 GB headroom after Agent models. This fits easily.

**Expected speedup:** ~5-10x over CPU. Cross-encoder on 20 pairs would drop from ~1.2s to ~150ms.

### 7.3 Updated Latency After GPU Reranker

```
Step                              CPU         GPU
──────────────────────────────────────────────────
CodeRankEmbed (55→20)             ~275ms      ~275ms (stays CPU)
Qwen3-Reranker (20→5)            ~1.2s       ~150ms
──────────────────────────────────────────────────
Total reranking                   ~1.5s       ~425ms
Total query P95                   ~2.1s       ~1.0s
```

### 7.4 VRAM Budget After GPU Reranker

```
H100 93.6 GB (Phase 3 GPU option)
──────────────────────────────────
Qwen3-Coder-Next Q5_K_M          ~30 GB
Qwen3-235B-A22B Q4_K_XL          ~35 GB
KV caches (agent models)         ~21 GB
Qwen3-Reranker-0.6B F16          ~1.2 GB     ← NEW
──────────────────────────────────
Used: ~87.2 GB | Headroom: ~6.4 GB ← SAFE
```

-----

## 8. Implementation Schedule

```
Day 1: CodeRankEmbed Setup
  ├── Transfer model files to air-gapped server
  ├── Python server (coderank_server.py)
  ├── Systemd service
  ├── TypeScript client
  ├── Health check integration
  └── Verify: curl test with sample query + 10 code snippets

Day 2: Qwen3-Reranker Setup
  ├── Transfer model files (seq-cls variant)
  ├── Python server (qwen3_reranker_server.py)
  ├── Systemd service
  ├── TypeScript client
  ├── Instruction templates per query type
  ├── Query type detector
  └── Verify: curl test with same sample, compare scores vs BGE

Day 3: Pipeline Integration
  ├── Wire CodeRankEmbed into hybrid_search.ts (after RRF, before cross-encoder)
  ├── Wire Qwen3-Reranker as cross-encoder replacement
  ├── Katz centrality fusion between stages
  ├── Feature flag: USE_CASCADED_RERANKING
  ├── Graceful degradation (model down → fallback)
  └── Content dedup before bi-encoder

Day 4: Query Type Integration
  ├── Query type detector → instruction selector
  ├── Per-dimension reranker instructions
  ├── Test with 5 query types
  └── Verify instruction quality vs no-instruction

Day 5: Benchmarking
  ├── Run golden queries against old pipeline (BGE only)
  ├── Run golden queries against new pipeline (cascaded)
  ├── A/B comparison: quality + latency
  ├── camelCase edge cases (useState, getOrderById)
  ├── Source balance verification (code vs prose in top 5)
  └── Latency breakdown logging

Day 6: Testing & Tuning
  ├── Run on all 47 repos with canary queries
  ├── Tune score-ratio gating threshold
  ├── Tune bi-encoder top-K (15 vs 20 vs 25)
  ├── Tune cross-encoder top-K (3 vs 5 vs 7)
  ├── Verify MMR still prevents same-file dominance
  └── Edge cases: empty results, single result, all same file

Day 7: BGE Retirement & Monitoring
  ├── Disable BGE reranker service (don't delete yet)
  ├── Remove BGE from health checks
  ├── Add reranker metrics to ingestion_metrics or new table
  ├── Add reranker latency to bootstrap status dashboard
  ├── Update diagnose_search.mjs with reranker diagnostics
  └── Update test_reranker_hyde_optimizations.mjs

Day 8: Documentation & GPU Prep
  ├── Update Semantic Search Best Practices doc
  ├── Update system architecture docs
  ├── Download Qwen3-Reranker GGUF for future GPU migration
  ├── Test GGUF reranker on GPU (if time permits)
  └── Team walkthrough
```

-----

## 9. Configuration Reference

### 9.1 New Environment Variables

```bash
# ═══ CodeRankEmbed (Bi-Encoder) ═══
CODERANK_URL=http://127.0.0.1:8086        # Server URL
CODERANK_MODEL_PATH=/models/CodeRankEmbed  # Model directory
CODERANK_PORT=8086                         # Server port
CODERANK_MAX_TOKENS=512                    # Max tokens per document
CODERANK_TIMEOUT_MS=10000                  # Client timeout

# ═══ Qwen3-Reranker (Cross-Encoder) ═══
QWEN3_RERANKER_URL=http://127.0.0.1:8087                        # Server URL
QWEN3_RERANKER_PATH=/models/Qwen3-Reranker-0.6B-seq-cls         # Model directory
QWEN3_RERANKER_PORT=8087                                         # Server port
QWEN3_RERANKER_TIMEOUT_MS=15000                                  # Client timeout

# ═══ Pipeline Control ═══
USE_CASCADED_RERANKING=true                # Enable cascaded pipeline (false = old BGE path)
BIENCODER_TOP_K=20                         # Candidates passed from bi-encoder to cross-encoder
CROSSENCODER_TOP_K=5                       # Final results from cross-encoder
USE_RERANKER_INSTRUCTIONS=true             # Enable instruction-tuned reranking
```

### 9.2 Updated Port Map

```
Port    Model                          Purpose                   Phase
─────────────────────────────────────────────────────────────────────────
:8081   snowflake-arctic-embed-l-v2.0  Prose embedding           Existing
:8082   (retired)                      Was Qwen2.5-7B            Phase 1
:8083   (retired)                      Was CodeRankEmbed plans    —
:8084   BAAI/bge-reranker-v2-m3       Cross-encoder (RETIRING)   Phase 3
:8085   nomic-embed-code 7B GGUF       Code embedding            Existing
:8086   nomic-ai/CodeRankEmbed         Bi-encoder reranker        Phase 3 NEW
:8087   Qwen3-Reranker-0.6B           Cross-encoder reranker     Phase 3 NEW
:8090   Qwen3-Coder-Next              Agent fast model           Phase 1
:8091   Qwen3-235B-A22B               Agent deep model           Phase 1
```

-----

## 10. File Structure

```
src/
├── reranker/
│   ├── coderank_server.py              # CodeRankEmbed Python server
│   ├── qwen3_reranker_server.py        # Qwen3-Reranker Python server
│   └── requirements.txt                # sentence-transformers, flask
│
├── lib/
│   ├── coderank_client.ts              # TypeScript client for CodeRankEmbed
│   ├── qwen3_reranker_client.ts        # TypeScript client for Qwen3-Reranker
│   ├── reranker_instructions.ts        # Per-query-type instruction templates
│   ├── query_type_detector.ts          # Classify query → instruction type
│   ├── hybrid_search.ts                # UPDATED — cascaded reranking integration
│   └── reranker_metrics.ts             # Latency and quality tracking
│
├── test_script/
│   ├── benchmark_reranking.ts          # Golden query benchmark
│   ├── ab_reranking.ts                 # A/B comparison (old vs new)
│   ├── diagnose_search.mjs            # UPDATED — add reranker diagnostics
│   └── test_reranker_hyde_optimizations.mjs  # UPDATED
│
└── systemd/
    ├── ome-coderank.service            # CodeRankEmbed systemd
    └── ome-qwen3-reranker.service      # Qwen3-Reranker systemd


/models/
├── CodeRankEmbed/                      # ~522 MB, nomic-ai/CodeRankEmbed
│   ├── config.json
│   ├── model.safetensors
│   ├── tokenizer.json
│   └── ...
│
├── Qwen3-Reranker-0.6B-seq-cls/       # ~1.2 GB, tomaarsen/Qwen3-Reranker-0.6B-seq-cls
│   ├── config.json
│   ├── model.safetensors
│   ├── tokenizer.json
│   └── ...
│
└── Qwen3-Reranker-0.6B-GGUF/         # ~1.2 GB, for future GPU deployment
    └── Qwen3-Reranker-0.6B-F16.gguf
```

### Python Dependencies

```
# src/reranker/requirements.txt

flask>=3.0.0
sentence-transformers>=3.0.0
torch>=2.1.0
transformers>=4.45.0
numpy>=1.24.0
```

Install on air-gapped server:

```bash
# On internet-connected machine:
pip download -d ./reranker-deps flask sentence-transformers torch transformers numpy

# Transfer and install:
pip install --no-index --find-links=./reranker-deps \
  flask sentence-transformers torch transformers numpy \
  --break-system-packages
```

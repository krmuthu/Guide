# OME Agent — Definitive Implementation Plan (Dual-Model Architecture)

> **Document Version:** 2.0 — 2026-03-28
> **Project:** OME Agent (separate from OME RAG)
> **GPU:** NVIDIA H100 — 95,830 MB (93.6 GB) VRAM
> **Server:** RHEL 9.7, 128 cores, 1TB RAM
> **Models:** Qwen3-Coder-Next (fast) + Qwen3-235B-A22B (deep) — dual-model routing
> **Purpose:** Agentic AI coding assistant consuming OME RAG via MCP, exposing `ome_rag_guide` tool
> **Effort:** ~18-20 days

-----

## Table of Contents

### Part A — Architecture & Model Strategy

1. [System Architecture — Two Projects, Dual Models](#1-system-architecture)
1. [GPU Memory Layout — 93.6 GB Plan](#2-gpu-memory-layout)
1. [Model Deployment](#3-model-deployment)
1. [Model Router — Fast vs Deep](#4-model-router)

### Part B — PR & Jira Embedding (OME RAG Changes)

1. [PR/Jira Embedding Strategy](#5-prjira-embedding)
1. [Hybrid Search Upgrade](#6-hybrid-search-upgrade)
1. [PR→Symbol Graph Edges](#7-prsymbol-graph-edges)

### Part C — OME Agent Project

1. [Project Setup & Stack](#8-project-setup)
1. [MCP Server — Exposing ome_rag_guide](#9-mcp-server)
1. [RAG Client — Consuming OME RAG’s 43 Tools](#10-rag-client)
1. [Agent Database Schema](#11-agent-database)

### Part D — ome_rag_guide Implementation

1. [Tool Definition](#12-tool-definition)
1. [Three Response Modes](#13-three-modes)
1. [Intent Classification](#14-intent-classification)
1. [Orchestration Engine — The Brain](#15-orchestration-engine)
1. [Gather Planner & Executor](#16-gather-planner)
1. [Mode 1: UNDERSTAND](#17-mode-understand)
1. [Mode 2: IMPLEMENT](#18-mode-implement)
1. [Mode 3: DIAGNOSE](#19-mode-diagnose)
1. [Qwen Context Builder](#20-qwen-context-builder)
1. [Code Generation Engine](#21-code-generation-engine)
1. [Hallucination Prevention](#22-hallucination-prevention)
1. [Confidence Model](#23-confidence-model)

### Part E — Agent Execution (Phase 2 Ready)

1. [Action Plan as Agent Contract](#24-agent-contract)
1. [Extension Path](#25-extension-path)

### Part F — Implementation

1. [Implementation Schedule](#26-implementation-schedule)
1. [Configuration](#27-configuration)
1. [File Structure](#28-file-structure)

-----

# Part A — Architecture & Model Strategy

## 1. System Architecture

```
┌──────────────────────────────────────────────────────────────────────┐
│  IDE  (Windsurf / Cursor / VS Code)                                  │
│  Developer writes code, asks questions, implements tickets            │
└────────┬──────────────────────────────────────────┬──────────────────┘
         │ MCP (stdio or HTTP)                      │ MCP (stdio or HTTP)
         │ Simple: search, browse, read             │ Complex: understand, implement, debug
         ▼                                          ▼
┌────────────────────────────┐   ┌──────────────────────────────────────┐
│  OME RAG  (existing)       │   │  OME Agent  (NEW project)            │
│  Knowledge Layer (CPU)     │   │  Action Layer (GPU)                  │
│                            │   │                                      │
│  43 MCP tools              │◄──│  ome_rag_guide tool (the brain)      │
│  hybrid_search.ts (16-stg) │   │                                      │
│  IGraphClient (AGE/SQL)    │   │  Consumes RAG via MCP client         │
│  Bootstrap pipeline        │   │  Reads REPOS_DIR directly            │
│  Intelligence layer        │   │  Reads otak schema (read-only)       │
│  Health scoring             │   │                                      │
│  SDLC pipeline tools       │   │  ┌──────────────────────────────┐   │
│                            │   │  │  DUAL MODEL ROUTER            │   │
│  Port: 3000 (HTTP)         │   │  │                               │   │
│  + stdio for Windsurf      │   │  │  :8090 Qwen3-Coder-Next      │   │
│                            │   │  │  (fast: questions, intent,    │   │
│  CPU models (unchanged):   │   │  │   diagnosis, conventions)     │   │
│  :8081 Arctic Embed        │   │  │                               │   │
│  :8082 Qwen2.5-Coder-7B   │   │  │  :8091 Qwen3-235B-A22B       │   │
│  :8083 BGE Reranker        │   │  │  (deep: code generation,     │   │
│  :8084 Qwen2.5-Coder-1.5B │   │  │   refactoring, complex impl) │   │
│  :8085 nomic-embed-code    │   │  │                               │   │
│                            │   │  └──────────────────────────────┘   │
│  DB: PostgreSQL (otak)     │   │  Port: 3100 (HTTP) + stdio          │
│                            │   │  DB: PostgreSQL (ome_agent)          │
└────────────────────────────┘   └──────────────────────────────────────┘
         │                                          │
         ▼                                          ▼
┌──────────────────────────────────────────────────────────────────────┐
│  SHARED INFRASTRUCTURE                                                │
│                                                                       │
│  PostgreSQL (shared instance)                                         │
│  • otak schema — owned by RAG (Agent has read-only access)            │
│  • ome_agent schema — owned by Agent                                  │
│                                                                       │
│  REPOS_DIR (/repos) — shared filesystem (Agent reads directly)        │
│                                                                       │
│  H100 93.6GB GPU — owned by OME Agent                                 │
│  • :8090 Qwen3-Coder-Next Q5_K_M (fast model, full on GPU)           │
│  • :8091 Qwen3-235B-A22B Q4_K_XL (deep model, MoE offload to RAM)   │
│                                                                       │
│  Bitbucket / Jira / Confluence APIs — shared credentials              │
└──────────────────────────────────────────────────────────────────────┘
```

-----

## 2. GPU Memory Layout — 93.6 GB

```
┌──────────────────────────────────────────────────────────┐
│  H100 GPU — 95,830 MB (93.6 GB) VRAM                     │
│                                                           │
│  ┌────────────────────────────────────┐                   │
│  │  Qwen3-Coder-Next Q5_K_M          │  ~30 GB           │
│  │  Full model on GPU                 │                   │
│  │  80B total, 3B active              │                   │
│  │  Port :8090                        │                   │
│  └────────────────────────────────────┘                   │
│                                                           │
│  ┌────────────────────────────────────┐                   │
│  │  Qwen3-235B-A22B Q4_K_XL          │  ~35 GB           │
│  │  Non-MoE layers on GPU             │                   │
│  │  (attention, routing, embeddings)  │                   │
│  │  Port :8091                        │                   │
│  │  MoE expert layers → CPU RAM       │                   │
│  └────────────────────────────────────┘                   │
│                                                           │
│  ┌────────────────────────────────────┐                   │
│  │  KV Cache                          │  ~21 GB           │
│  │  Coder-Next: 64K ctx × 3 slots    │  (~9 GB)          │
│  │  235B: 32K ctx × 1 slot           │  (~12 GB)         │
│  └────────────────────────────────────┘                   │
│                                                           │
│  Headroom: ~8 GB (safety margin)                          │
│                                                           │
│  TOTAL: ~86 GB / 93.6 GB  ✅                              │
└──────────────────────────────────────────────────────────┘

┌──────────────────────────────────────────────────────────┐
│  CPU RAM — 1 TB                                           │
│                                                           │
│  Qwen3-235B MoE expert layers:    ~82 GB                  │
│  OME RAG process + 5 CPU models:  ~40 GB                  │
│  OME Agent Node.js process:       ~2 GB                   │
│  PostgreSQL:                      ~30 GB                  │
│  OS + filesystem cache:           ~50 GB                  │
│                                                           │
│  Free: ~800 GB                                            │
└──────────────────────────────────────────────────────────┘
```

-----

## 3. Model Deployment

### 3.1 Qwen3-Coder-Next — Fast Model (:8090)

```bash
# ── Download ──
huggingface-cli download unsloth/Qwen3-Coder-Next-GGUF \
  --local-dir /models/qwen3-coder-next \
  --include "*UD-Q5_K_M*"

# ── Deploy ──
/opt/ome-agent/llama.cpp/build/bin/llama-server \
  --model /models/qwen3-coder-next/Qwen3-Coder-Next-UD-Q5_K_M.gguf \
  --host 127.0.0.1 \
  --port 8090 \
  --n-gpu-layers 99 \
  --ctx-size 65536 \
  --parallel 3 \
  --threads 8 \
  --flash-attn \
  --jinja \
  --temp 1.0 --top-p 0.95 --top-k 40 --min-p 0.01
```

**Characteristics:**

- Full model on GPU (~30GB VRAM)
- 3 parallel slots × 64K context each
- **50+ tok/s** generation speed
- Purpose-built for coding agents
- Native tool calling
- No thinking mode (non-reasoning, instant responses)

**Used for:** Intent classification, UNDERSTAND mode, simple DIAGNOSE, convention detection, follow-up suggestions — any task where speed matters more than depth.

### 3.2 Qwen3-235B-A22B — Deep Model (:8091)

```bash
# ── Download (Instruct-2507 is latest with 256K support) ──
huggingface-cli download unsloth/Qwen3-235B-A22B-GGUF \
  --local-dir /models/qwen3-235b \
  --include "*UD-Q4_K_XL*"

# ── Deploy with MoE offloading ──
/opt/ome-agent/llama.cpp/build/bin/llama-server \
  --model /models/qwen3-235b/Qwen3-235B-A22B-UD-Q4_K_XL-00001-of-00003.gguf \
  --host 127.0.0.1 \
  --port 8091 \
  --n-gpu-layers 99 \
  -ot ".ffn_.*_exps.=CPU" \
  --ctx-size 32768 \
  --parallel 1 \
  --threads 32 \
  --flash-attn \
  --jinja \
  --temp 0.6 --top-p 0.95 --top-k 20 --min-p 0.0
```

**Characteristics:**

- Non-MoE layers on GPU (~35GB VRAM), MoE experts on CPU RAM (~82GB)
- 1 parallel slot × 32K context
- **~15-25 tok/s** generation speed (CPU memory bandwidth limited)
- 22B active parameters — significantly more capable than Coder-Next’s 3B
- Thinking mode available for complex reasoning
- Trained on tool calling and agent tasks

**Used for:** IMPLEMENT mode (production code generation), complex refactoring, multi-file changes, architecture-level reasoning — any task where code quality matters more than speed.

### 3.3 Systemd Services

```ini
# /etc/systemd/system/ome-agent-fast.service
[Unit]
Description=OME Agent - Qwen3-Coder-Next (Fast Model)
After=network.target

[Service]
Type=simple
User=ome
ExecStart=/opt/ome-agent/llama.cpp/build/bin/llama-server \
  --model /models/qwen3-coder-next/Qwen3-Coder-Next-UD-Q5_K_M.gguf \
  --host 127.0.0.1 --port 8090 \
  --n-gpu-layers 99 --ctx-size 65536 --parallel 3 \
  --threads 8 --flash-attn --jinja \
  --temp 1.0 --top-p 0.95 --top-k 40 --min-p 0.01
Restart=always
RestartSec=10
Environment=CUDA_VISIBLE_DEVICES=0

[Install]
WantedBy=multi-user.target
```

```ini
# /etc/systemd/system/ome-agent-deep.service
[Unit]
Description=OME Agent - Qwen3-235B-A22B (Deep Model)
After=network.target ome-agent-fast.service

[Service]
Type=simple
User=ome
ExecStart=/opt/ome-agent/llama.cpp/build/bin/llama-server \
  --model /models/qwen3-235b/Qwen3-235B-A22B-UD-Q4_K_XL-00001-of-00003.gguf \
  --host 127.0.0.1 --port 8091 \
  --n-gpu-layers 99 -ot ".ffn_.*_exps.=CPU" \
  --ctx-size 32768 --parallel 1 \
  --threads 32 --flash-attn --jinja \
  --temp 0.6 --top-p 0.95 --top-k 20 --min-p 0.0
Restart=always
RestartSec=30
Environment=CUDA_VISIBLE_DEVICES=0

[Install]
WantedBy=multi-user.target
```

```ini
# /etc/systemd/system/ome-agent.service
[Unit]
Description=OME Agent - Node.js MCP Server
After=ome-agent-fast.service ome-agent-deep.service

[Service]
Type=simple
User=ome
WorkingDirectory=/opt/ome-agent
ExecStart=/usr/bin/node --experimental-strip-types dist/server/http_entry.js
Restart=always
RestartSec=5
Environment=NODE_ENV=production

[Install]
WantedBy=multi-user.target
```

### 3.4 Startup Order

```bash
systemctl start ome-agent-fast    # Coder-Next loads in ~30s
systemctl start ome-agent-deep    # 235B loads in ~2-3 min (MoE offload)
systemctl start ome-agent         # Node.js server starts, verifies both models healthy
```

-----

## 4. Model Router — Fast vs Deep

```typescript
// src/llm/model_router.ts

export type ModelTier = 'fast' | 'deep';

export interface RoutingDecision {
  tier: ModelTier;
  model: string;
  port: number;
  reason: string;
  maxTokens: number;
  temperature: number;
}

export function routeToModel(
  mode: GuideMode,
  task: ModelTask,
  complexity?: number
): RoutingDecision {

  // ── FAST MODEL (:8090 Qwen3-Coder-Next) ──
  // Speed-sensitive tasks, simple reasoning

  // Intent classification — always fast (10 tokens output)
  if (task === 'intent_classify') {
    return {
      tier: 'fast', model: 'qwen3-coder-next', port: 8090,
      reason: 'Intent classification is a simple routing decision',
      maxTokens: 20, temperature: 0.0,
    };
  }

  // Convention detection — fast (200 tokens output)
  if (task === 'convention_detect') {
    return {
      tier: 'fast', model: 'qwen3-coder-next', port: 8090,
      reason: 'Convention detection is pattern recognition, not generation',
      maxTokens: 400, temperature: 0.2,
    };
  }

  // Follow-up suggestions — fast (100 tokens)
  if (task === 'followup_suggest') {
    return {
      tier: 'fast', model: 'qwen3-coder-next', port: 8090,
      reason: 'Follow-up suggestions are creative but short',
      maxTokens: 200, temperature: 0.5,
    };
  }

  // UNDERSTAND mode — fast (explanation, not code generation)
  if (mode === 'understand') {
    return {
      tier: 'fast', model: 'qwen3-coder-next', port: 8090,
      reason: 'Explanations benefit from speed, context is pre-gathered',
      maxTokens: 1500, temperature: 0.3,
    };
  }

  // Simple DIAGNOSE — fast (error triage, not fix generation)
  if (mode === 'diagnose' && task === 'analyze') {
    return {
      tier: 'fast', model: 'qwen3-coder-next', port: 8090,
      reason: 'Error analysis is pattern matching, not code generation',
      maxTokens: 1000, temperature: 0.2,
    };
  }

  // ── DEEP MODEL (:8091 Qwen3-235B-A22B) ──
  // Quality-sensitive tasks, complex reasoning, code generation

  // IMPLEMENT mode — always deep (production code generation)
  if (mode === 'implement') {
    return {
      tier: 'deep', model: 'qwen3-235b-a22b', port: 8091,
      reason: 'Code generation requires deep reasoning for accuracy',
      maxTokens: 8000, temperature: 0.15,
    };
  }

  // DIAGNOSE fix generation — deep (writing fix code)
  if (mode === 'diagnose' && task === 'generate_fix') {
    return {
      tier: 'deep', model: 'qwen3-235b-a22b', port: 8091,
      reason: 'Fix generation needs accurate code output',
      maxTokens: 4000, temperature: 0.15,
    };
  }

  // Default to fast
  return {
    tier: 'fast', model: 'qwen3-coder-next', port: 8090,
    reason: 'Default routing',
    maxTokens: 1000, temperature: 0.3,
  };
}

export type ModelTask =
  | 'intent_classify'
  | 'convention_detect'
  | 'followup_suggest'
  | 'explain'           // UNDERSTAND answer generation
  | 'analyze'           // DIAGNOSE error analysis
  | 'generate_code'     // IMPLEMENT code generation
  | 'generate_fix';     // DIAGNOSE fix code generation
```

### 4.1 What the Developer Experiences

```
"How does JWT auth work?"
  → mode=UNDERSTAND → task=explain → FAST (:8090)
  → Response: ~2-3 seconds ✓

"What does submitOrder do?"
  → mode=UNDERSTAND → task=explain → FAST (:8090)
  → Response: ~1 second ✓

"Add pagination to bond search API"
  → mode=IMPLEMENT → task=generate_code → DEEP (:8091)
  → 235B thinks through the problem, generates accurate code
  → Response: ~45-50 seconds (worth the wait for correct code) ✓

"Refactor OrderValidationService to Strategy pattern"
  → mode=IMPLEMENT → task=generate_code → DEEP (:8091)
  → 235B reasons through interfaces, concrete strategies, factory
  → Response: ~60-90 seconds (complex multi-file, correct architecture) ✓

"NullPointerException in SettlementService"
  → mode=DIAGNOSE → task=analyze → FAST (:8090)
  → Response: ~3 seconds (error location + reverse call chain) ✓
  → If developer says "fix it":
    → task=generate_fix → DEEP (:8091)
    → Response: ~30 seconds (accurate fix code) ✓

"Implement BMT-1234"
  → mode=IMPLEMENT → task=generate_code → DEEP (:8091)
  → Gathers Jira context + code + PRs + blast radius
  → 235B generates full change plan with diffs
  → Response: ~50-60 seconds ✓
```

-----

## 4.2 Dual LLM Client

```typescript
// src/llm/dual_client.ts

export class DualLLMClient {
  private fastUrl: string;   // :8090 Coder-Next
  private deepUrl: string;   // :8091 235B

  constructor() {
    this.fastUrl = process.env.FAST_MODEL_URL || 'http://127.0.0.1:8090';
    this.deepUrl = process.env.DEEP_MODEL_URL || 'http://127.0.0.1:8091';
  }

  async chat(
    prompt: string,
    routing: RoutingDecision,
    options?: { systemPrompt?: string; tools?: any[] }
  ): Promise<LLMResponse> {
    const startMs = performance.now();
    const url = routing.tier === 'fast' ? this.fastUrl : this.deepUrl;

    const messages: any[] = [];
    if (options?.systemPrompt) {
      messages.push({ role: 'system', content: options.systemPrompt });
    }
    messages.push({ role: 'user', content: prompt });

    const body: any = {
      messages,
      max_tokens: routing.maxTokens,
      temperature: routing.temperature,
      top_p: 0.95,
      top_k: routing.tier === 'fast' ? 40 : 20,
      min_p: routing.tier === 'fast' ? 0.01 : 0.0,
      stream: false,
    };

    if (options?.tools) {
      body.tools = options.tools;
    }

    // Timeout: fast=30s, deep=120s (code generation takes time)
    const timeoutMs = routing.tier === 'fast'
      ? parseInt(process.env.FAST_MODEL_TIMEOUT_MS || '30000')
      : parseInt(process.env.DEEP_MODEL_TIMEOUT_MS || '120000');

    const response = await fetch(`${url}/v1/chat/completions`, {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify(body),
      signal: AbortSignal.timeout(timeoutMs),
    });

    if (!response.ok) {
      throw new Error(`${routing.model} error: ${response.status}`);
    }

    const data = await response.json();
    const latencyMs = Math.round(performance.now() - startMs);
    const content = data.choices?.[0]?.message?.content || '';

    return {
      content,
      model: routing.model,
      tier: routing.tier,
      promptTokens: data.usage?.prompt_tokens || 0,
      completionTokens: data.usage?.completion_tokens || 0,
      latencyMs,
    };
  }

  async healthCheck(): Promise<{ fast: boolean; deep: boolean }> {
    const [fastOk, deepOk] = await Promise.allSettled([
      fetch(`${this.fastUrl}/health`, { signal: AbortSignal.timeout(5000) }),
      fetch(`${this.deepUrl}/health`, { signal: AbortSignal.timeout(5000) }),
    ]);
    return {
      fast: fastOk.status === 'fulfilled' && (fastOk.value as Response).ok,
      deep: deepOk.status === 'fulfilled' && (deepOk.value as Response).ok,
    };
  }
}

export interface LLMResponse {
  content: string;
  model: string;
  tier: ModelTier;
  promptTokens: number;
  completionTokens: number;
  latencyMs: number;
}
```

### 4.3 Graceful Degradation

```typescript
// If deep model is down, fall back to fast model with a warning
async function chatWithFallback(
  prompt: string,
  routing: RoutingDecision,
  client: DualLLMClient,
  options?: any
): Promise<LLMResponse & { fellBack: boolean }> {
  try {
    const response = await client.chat(prompt, routing, options);
    return { ...response, fellBack: false };
  } catch (err) {
    if (routing.tier === 'deep') {
      console.warn('[ModelRouter] Deep model failed, falling back to fast:', err);
      const fallbackRouting: RoutingDecision = {
        ...routing,
        tier: 'fast',
        model: 'qwen3-coder-next',
        port: 8090,
        reason: `Fallback: ${routing.reason} (deep model unavailable)`,
      };
      const response = await client.chat(prompt, fallbackRouting, options);
      return { ...response, fellBack: true };
    }
    throw err;
  }
}
```

-----

# Part B — PR & Jira Embedding (OME RAG Changes)

## 5. PR/Jira Embedding Strategy

### 5.1 Why It’s Worth It

FTS on AI-condensed PR summaries misses ~30-40% of conceptual queries because vocabulary doesn’t overlap. The coding assistant needs “how was this done before?” to surface existing patterns. Vector embedding fixes this.

### 5.2 What to Embed

Embed the AI-condensed outputs from `pr_condenser.ts` and `jira_condenser.ts` — clean, 50-200 word prose. Use Arctic Embed (:8081, 1024d halfvec).

**PR:** `Core Problem: {core_problem}\nImplementation: {implementation_summary}\nKey Decisions: {key_decisions}`
**Jira:** `Business Goal: {business_goal}\nTask: {specific_task}\nAcceptance Criteria: {acceptance_criteria}`

### 5.3 Schema (In otak namespace — OME RAG project)

```sql
CREATE TABLE IF NOT EXISTS otak.vec_pr_summaries (
  pr_id        INTEGER PRIMARY KEY REFERENCES otak.pull_requests(id) ON DELETE CASCADE,
  repo_slug    TEXT NOT NULL,
  embedding    HALFVEC(1024) NOT NULL,
  content_hash TEXT NOT NULL
);
CREATE INDEX idx_vec_pr_hnsw ON otak.vec_pr_summaries
  USING hnsw (embedding halfvec_cosine_ops) WITH (m = 16, ef_construction = 100);
CREATE INDEX idx_vec_pr_repo ON otak.vec_pr_summaries (repo_slug);

CREATE TABLE IF NOT EXISTS otak.vec_jira_summaries (
  issue_key    TEXT PRIMARY KEY,
  embedding    HALFVEC(1024) NOT NULL,
  content_hash TEXT NOT NULL
);
CREATE INDEX idx_vec_jira_hnsw ON otak.vec_jira_summaries
  USING hnsw (embedding halfvec_cosine_ops) WITH (m = 16, ef_construction = 100);
```

### 5.4 Ingestion Hooks

In `pr_ingestion.ts` — after `pr_condenser.ts` generates summaries, embed via Arctic (:8081) and store with content-hash skip for unchanged summaries.

In `jira_ingestion.ts` — same pattern after `jira_condenser.ts`.

### 5.5 Backfill

```bash
npx tsx src/scripts/backfill_pr_vectors.ts     # ~10 min for ~2000 PRs
npx tsx src/scripts/backfill_jira_vectors.ts   # ~15 min for ~3000 tickets
```

-----

## 6. Hybrid Search Upgrade

### 6.1 Dual Query Embedding at Stage 2

```typescript
// In hybrid_search.ts stage 2 — add parallel prose embedding
const [codeEmbedding, proseEmbedding] = await Promise.all([
  llamaClient.embedCode(prefixedQuery),    // :8085, 3584d
  llamaClient.embed(query),                // :8081, 1024d
]);
```

### 6.2 Upgrade Stages 6-7 to Hybrid

PR summary search (stage 6) and Jira summary search (stage 7) become vector + FTS with mini-RRF. Parallel execution, prose embedding reused from stage 2.

-----

## 7. PR→Symbol Graph Edges

During PR ingestion, find `graph_nodes` overlapping with changed line ranges and create `modifies` edges. This enables the Agent to query: “Which PR last modified submitOrder?” → graph traversal via `modifies` relationship.

**OME RAG changes total effort: ~3 days**

-----

# Part C — OME Agent Project

## 8. Project Setup & Stack

```json
{
  "name": "ome-agent",
  "type": "module",
  "engines": { "node": ">=24.0.0" },
  "dependencies": {
    "@modelcontextprotocol/sdk": "^1.26.0",
    "express": "^5.2.1",
    "pg": "^8.19.0",
    "undici": "^7.22.0",
    "zod": "^3.23.0",
    "p-limit": "^6.0.0",
    "dotenv": "^16.4.5"
  },
  "devDependencies": {
    "typescript": "^5.5.0",
    "@types/node": "^22.0.0",
    "@types/pg": "^8.0.0"
  }
}
```

Runtime: Node.js LTS (Krypton) >= 24.0.0, TypeScript ^5.5.0, ESM modules, `--experimental-strip-types`.

-----

## 9. MCP Server — Exposing ome_rag_guide

```typescript
// src/server/mcp_server.ts

import { Server } from '@modelcontextprotocol/sdk/server/index.js';
import { OME_RAG_GUIDE } from '../guide/tool_definition.js';
import { orchestrate } from '../guide/orchestrator.js';

export function createMCPServer(ctx: AgentContext): Server {
  const server = new Server(
    { name: 'ome-agent', version: '1.0.0' },
    { capabilities: { tools: {} } }
  );

  // Register the single tool
  server.setRequestHandler('tools/list', async () => ({
    tools: [OME_RAG_GUIDE],
  }));

  server.setRequestHandler('tools/call', async (request) => {
    const { name, arguments: args } = request.params;

    if (name !== 'ome_rag_guide') {
      throw new Error(`Unknown tool: ${name}`);
    }

    const result = await orchestrate(args as GuideParams, ctx);

    return {
      content: [{
        type: 'text',
        text: JSON.stringify(result, null, 2),
      }],
    };
  });

  return server;
}
```

```typescript
// src/server/http_entry.ts

import express from 'express';
import { createMCPServer } from './mcp_server.js';
import { StreamableHTTPServerTransport } from '@modelcontextprotocol/sdk/server/streamableHttp.js';

const app = express();
app.use(express.json());

// Bearer token auth (same as RAG)
app.use('/mcp', (req, res, next) => {
  const token = req.headers.authorization?.replace('Bearer ', '');
  if (token !== process.env.AGENT_MCP_API_KEY) {
    return res.status(401).json({ error: 'Unauthorized' });
  }
  next();
});

// MCP endpoint
app.post('/mcp', async (req, res) => {
  const transport = new StreamableHTTPServerTransport('/mcp', res);
  const server = createMCPServer(agentContext);
  await server.connect(transport);
  await transport.handleRequest(req, res);
});

// Health endpoint
app.get('/health', async (req, res) => {
  const models = await llmClient.healthCheck();
  const ragConnected = await ragClient.isConnected();
  res.json({
    status: models.fast && ragConnected ? 'healthy' : 'degraded',
    models: {
      fast: { healthy: models.fast, port: 8090, model: 'qwen3-coder-next' },
      deep: { healthy: models.deep, port: 8091, model: 'qwen3-235b-a22b' },
    },
    rag: { connected: ragConnected },
  });
});

app.listen(parseInt(process.env.AGENT_PORT || '3100'), () => {
  console.log(`[OME Agent] MCP server on port ${process.env.AGENT_PORT || 3100}`);
});
```

-----

## 10. RAG Client — Consuming OME RAG’s 43 Tools

```typescript
// src/rag_client/mcp_client.ts

import { Client } from '@modelcontextprotocol/sdk/client/index.js';
import { StreamableHTTPClientTransport } from '@modelcontextprotocol/sdk/client/streamableHttp.js';

export class RAGClient {
  private client: Client;
  private connected = false;

  async connect(): Promise<void> {
    const transport = new StreamableHTTPClientTransport(
      new URL(process.env.OME_RAG_URL || 'http://localhost:3000/mcp'),
      { requestInit: { headers: { Authorization: `Bearer ${process.env.MCP_API_KEY}` } } }
    );
    this.client = new Client({ name: 'ome-agent', version: '1.0.0' });
    await this.client.connect(transport);
    this.connected = true;
  }

  isConnected(): boolean { return this.connected; }

  // ── Typed wrappers for the 15+ RAG tools the Agent uses ──

  async searchCode(query: string, topK = 10): Promise<SearchResult[]> {
    return this.call('search_semantic_code', { query, top_k: topK });
  }

  async searchExact(query: string, topK = 10): Promise<SearchResult[]> {
    return this.call('search_exact_match', { query, top_k: topK });
  }

  async findSymbol(symbol: string, repo?: string): Promise<SymbolResult[]> {
    return this.call('find_symbol', { symbol, repo_slug: repo });
  }

  async getCodeContext(symbol: string, repo?: string): Promise<CodeContext> {
    return this.call('get_code_context', { symbol, repo_slug: repo });
  }

  async readFullFile(filePath: string, repo: string): Promise<FileContent> {
    return this.call('read_full_file', { file_path: filePath, repo_slug: repo });
  }

  async batchReadFiles(files: Array<{ path: string; repo: string }>): Promise<FileContent[]> {
    return this.call('batch_read_files', {
      files: files.map(f => ({ file_path: f.path, repo_slug: f.repo })),
    });
  }

  async getFileDependencies(filePath: string, repo: string): Promise<Dependencies> {
    return this.call('get_file_dependencies', { file_path: filePath, repo_slug: repo });
  }

  async suggestRelatedFiles(filePath: string, repo: string): Promise<string[]> {
    return this.call('suggest_related_files', { file_path: filePath, repo_slug: repo });
  }

  async analyzeBlastRadius(symbol: string, repo: string): Promise<BlastRadius> {
    return this.call('analyze_blast_radius', { symbol, repo_slug: repo });
  }

  async getTicketContext(ticketKey: string): Promise<TicketContext> {
    return this.call('get_ticket_context', { ticket_key: ticketKey });
  }

  async getProjectInfo(repo: string): Promise<ProjectInfo> {
    return this.call('get_project_info', { repo_slug: repo });
  }

  async getRecentChanges(repo: string): Promise<RecentChange[]> {
    return this.call('get_recent_changes', { repo_slug: repo });
  }

  async getFileHistory(filePath: string, repo: string): Promise<FileHistory> {
    return this.call('get_file_history', { file_path: filePath, repo_slug: repo });
  }

  async getRepoConfig(repo: string): Promise<RepoConfig> {
    return this.call('get_repo_config', { repo_slug: repo });
  }

  async searchArchitecture(query: string): Promise<ConfluenceDoc[]> {
    return this.call('search_architecture', { query });
  }

  async analyzeGraphHealth(repo: string): Promise<GraphHealth> {
    return this.call('analyze_graph_health', { repo_slug: repo });
  }

  async queryKnowledgeGraph(query: string): Promise<GraphResult> {
    return this.call('query_knowledge_graph', { query });
  }

  async lookupGlossary(term: string): Promise<GlossaryEntry[]> {
    return this.call('lookup_glossary', { term });
  }

  async checkHallucination(filePath: string, repo: string): Promise<boolean> {
    return this.call('check_ai_hallucinations', { file_path: filePath, repo_slug: repo });
  }

  async compareImplementations(query: string): Promise<ComparisonResult> {
    return this.call('compare_implementations', { query });
  }

  // ── SDLC Tools (for Phase 2) ──

  async createPullRequest(params: any): Promise<any> {
    return this.call('create_pull_request', params);
  }

  async generatePRDescription(params: any): Promise<any> {
    return this.call('generate_pr_description', params);
  }

  async reviewPullRequest(params: any): Promise<any> {
    return this.call('review_pull_request', params);
  }

  // ── Generic call wrapper ──

  private async call(toolName: string, args: Record<string, any>): Promise<any> {
    const result = await this.client.callTool({ name: toolName, arguments: args });
    // Parse the text content from MCP response
    const text = result.content
      ?.filter((c: any) => c.type === 'text')
      .map((c: any) => c.text)
      .join('');
    try {
      return JSON.parse(text || '{}');
    } catch {
      return text;
    }
  }
}
```

### 10.1 Direct DB Access (Read-Only)

For complex queries the Agent needs but RAG MCP tools don’t expose:

```typescript
// src/rag_client/rag_db.ts

import pg from 'pg';
const { Pool } = pg;

// Read-only connection to otak schema
const ragPool = new Pool({
  connectionString: process.env.RAG_DATABASE_URL,
  max: 5,
  application_name: 'ome-agent-readonly',
});

ragPool.on('connect', (client) => {
  client.query('SET default_transaction_read_only = ON');
  client.query('SET search_path TO otak');
});

export { ragPool };

// ── Typed queries Agent needs ──

export async function getGlossaryTerms(): Promise<Set<string>> {
  const result = await ragPool.query(`SELECT UPPER(acronym) AS a FROM acronym_glossary`);
  return new Set(result.rows.map(r => r.a));
}

export async function searchPRsVector(
  proseEmbedding: number[],
  queryText: string,
  topK: number = 5
): Promise<PRSearchResult[]> {
  // Hybrid: vector + FTS with mini-RRF
  const [vec, fts] = await Promise.all([
    ragPool.query(`
      SELECT vps.pr_id, pr.repo_slug, pr.pr_number, pr.title,
             pr.core_problem, pr.implementation_summary, pr.key_decisions,
             1 - (vps.embedding <=> $1::halfvec) AS score
      FROM vec_pr_summaries vps
      JOIN pull_requests pr ON vps.pr_id = pr.id
      ORDER BY vps.embedding <=> $1::halfvec LIMIT $2
    `, [vectorToString(proseEmbedding), topK]),
    ragPool.query(`
      SELECT pr.id AS pr_id, pr.repo_slug, pr.pr_number, pr.title,
             pr.core_problem, pr.implementation_summary, pr.key_decisions,
             ts_rank_cd(fts.tsvector, plainto_tsquery('english', $1)) AS score
      FROM fts_pr_summaries fts
      JOIN pull_requests pr ON fts.pr_id = pr.id
      WHERE fts.tsvector @@ plainto_tsquery('english', $1)
      ORDER BY score DESC LIMIT $2
    `, [queryText, topK]),
  ]);
  return miniRRF(vec.rows, fts.rows, 'pr_id');
}

export async function searchJiraVector(
  proseEmbedding: number[],
  queryText: string,
  topK: number = 5
): Promise<JiraSearchResult[]> {
  // Same hybrid pattern for Jira
  // ...
}

export async function getHealthWarnings(
  filePaths: string[],
  repoSlugs: string[]
): Promise<HealthWarning[]> {
  const results = [];
  for (const repo of repoSlugs) {
    const health = await ragPool.query(
      `SELECT dimensions FROM health_scores WHERE repo_slug = $1 ORDER BY analyzed_at DESC LIMIT 1`,
      [repo]
    );
    if (health.rows[0]?.dimensions) {
      const dims = typeof health.rows[0].dimensions === 'string'
        ? JSON.parse(health.rows[0].dimensions) : health.rows[0].dimensions;
      // Extract findings for affected files
      for (const [dimName, dimData] of Object.entries(dims as Record<string, any>)) {
        for (const finding of (dimData as any)?.findings || []) {
          if (finding.filePath && filePaths.some(fp =>
            finding.filePath.includes(fp) || fp.includes(finding.filePath)
          )) {
            results.push({
              dimension: dimName,
              severity: finding.severity,
              message: finding.description || finding.evidence,
              filePath: finding.filePath,
              recommendation: finding.recommendation || finding.fix,
            });
          }
        }
      }
    }
  }
  return results.sort((a, b) =>
    ({ major: 0, moderate: 1, minor: 2 }[a.severity] || 2) -
    ({ major: 0, moderate: 1, minor: 2 }[b.severity] || 2)
  ).slice(0, 10);
}
```

### 10.2 Direct File System Access

```typescript
// src/rag_client/file_reader.ts

import { promises as fs } from 'fs';
import path from 'path';
import crypto from 'crypto';

const REPOS_DIR = process.env.REPOS_DIR || '/repos';

export interface FullFileContent {
  content: string;
  lineCount: number;
  truncated: boolean;
}

export async function readFileFromDisk(
  filePath: string,
  repoSlug: string,
  maxLines = 500
): Promise<FullFileContent | null> {
  const fullPath = path.join(REPOS_DIR, repoSlug.replace('/', path.sep), filePath);
  try {
    const raw = await fs.readFile(fullPath, 'utf-8');
    const lines = raw.split('\n');
    if (lines.length > maxLines) {
      const head = lines.slice(0, Math.floor(maxLines * 0.8)).join('\n');
      const tail = lines.slice(-Math.floor(maxLines * 0.15)).join('\n');
      return {
        content: `${head}\n\n// ... ${lines.length - maxLines} lines omitted ...\n\n${tail}`,
        lineCount: lines.length,
        truncated: true,
      };
    }
    return { content: raw, lineCount: lines.length, truncated: false };
  } catch {
    return null;
  }
}

export async function fileExistsOnDisk(filePath: string, repoSlug: string): Promise<boolean> {
  try {
    await fs.access(path.join(REPOS_DIR, repoSlug.replace('/', path.sep), filePath));
    return true;
  } catch {
    return false;
  }
}

export function md5(content: string): string {
  return crypto.createHash('md5').update(content).digest('hex');
}

export async function readLineRange(
  filePath: string,
  repoSlug: string,
  startLine: number,
  endLine: number
): Promise<string | null> {
  const file = await readFileFromDisk(filePath, repoSlug, 10000);
  if (!file) return null;
  const lines = file.content.split('\n');
  return lines.slice(startLine - 1, endLine).join('\n');
}
```

-----

## 11. Agent Database Schema

```sql
CREATE SCHEMA IF NOT EXISTS ome_agent;

-- Sessions
CREATE TABLE ome_agent.sessions (
  id           UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  session_type TEXT NOT NULL,
  created_at   TIMESTAMPTZ DEFAULT NOW(),
  last_active  TIMESTAMPTZ DEFAULT NOW(),
  metadata     JSONB
);

-- Every ome_rag_guide invocation
CREATE TABLE ome_agent.guide_invocations (
  id               UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  session_id       UUID REFERENCES ome_agent.sessions(id),
  query            TEXT NOT NULL,
  mode             TEXT NOT NULL,
  intent_confidence FLOAT,
  model_used       TEXT NOT NULL,
  model_tier       TEXT NOT NULL,
  rag_tools_called JSONB,
  total_ms         INTEGER,
  response_confidence TEXT,
  created_at       TIMESTAMPTZ DEFAULT NOW()
);

-- Change plans (for Phase 2)
CREATE TABLE ome_agent.change_plans (
  id             UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  invocation_id  UUID REFERENCES ome_agent.guide_invocations(id),
  ticket_key     TEXT,
  repo_slug      TEXT NOT NULL,
  plan_data      JSONB NOT NULL,
  status         TEXT DEFAULT 'generated',
  created_at     TIMESTAMPTZ DEFAULT NOW()
);

-- Model metrics
CREATE TABLE ome_agent.model_metrics (
  id               UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  model            TEXT NOT NULL,
  tier             TEXT NOT NULL,
  task             TEXT NOT NULL,
  prompt_tokens    INTEGER,
  completion_tokens INTEGER,
  latency_ms       INTEGER,
  temperature      FLOAT,
  created_at       TIMESTAMPTZ DEFAULT NOW()
);
CREATE INDEX idx_model_metrics_created ON ome_agent.model_metrics (created_at);
```

-----

# Part D — ome_rag_guide Implementation

## 12. Tool Definition

```typescript
// src/guide/tool_definition.ts

import { z } from 'zod';

export const GuideParamsSchema = z.object({
  query: z.string().describe('Natural language question or instruction'),
  context: z.string().optional().describe('Additional context: error messages, current code'),
  current_file: z.string().optional().describe('File currently open in IDE'),
  current_repo: z.string().optional().describe('Repository being worked in'),
  ticket_id: z.string().optional().describe('Jira ticket ID'),
  mode: z.enum(['auto', 'understand', 'implement', 'diagnose']).default('auto'),
});

export type GuideParams = z.infer<typeof GuideParamsSchema>;

export const OME_RAG_GUIDE = {
  name: 'ome_rag_guide',
  description: `AI coding assistant for the codebase. Call this FIRST for any coding task.

Returns codebase context + AI-generated code based on response mode:

UNDERSTAND: Explains how code works — entry points, execution flow, architecture patterns,
  dependency graphs, documentation. Uses fast model for instant responses.

IMPLEMENT: Generates production-ready code changes — exact files to modify, proposed code
  with line numbers, blast radius, tests to update, team conventions. Uses deep reasoning
  model (Qwen3-235B) for maximum accuracy.

DIAGNOSE: Analyzes errors — likely location, reverse call chain, recent changes,
  configuration context, suggested fixes with code.

Orchestrates 43 internal RAG tools + dual GPU models automatically.`,

  inputSchema: {
    type: 'object',
    properties: {
      query: { type: 'string', description: 'Natural language question or instruction' },
      context: { type: 'string', description: 'Additional context: error messages, current code' },
      current_file: { type: 'string', description: 'File currently open in IDE' },
      current_repo: { type: 'string', description: 'Repository being worked in' },
      ticket_id: { type: 'string', description: 'Jira ticket ID' },
      mode: { type: 'string', enum: ['auto', 'understand', 'implement', 'diagnose'], description: 'Force mode. Default auto.' },
    },
    required: ['query'],
  },
};
```

-----

## 13. Three Response Modes

|Mode          |Fast Model (:8090)     |Deep Model (:8091)       |What Developer Gets                                   |
|--------------|-----------------------|-------------------------|------------------------------------------------------|
|**UNDERSTAND**|✅ Generates explanation|❌                        |Code context + natural language explanation + evidence|
|**IMPLEMENT** |✅ Intent + conventions |✅ Generates code changes |Full change plan with production code + blast radius  |
|**DIAGNOSE**  |✅ Error analysis       |✅ Fix code (if requested)|Root cause analysis + suggested fix code              |

-----

## 14. Intent Classification

```typescript
// src/guide/intent_classifier.ts

export type GuideMode = 'understand' | 'implement' | 'diagnose';

export function classifyIntent(query: string, mode?: string, ticketId?: string): {
  mode: GuideMode;
  confidence: number;
} {
  if (mode && mode !== 'auto') return { mode: mode as GuideMode, confidence: 1.0 };
  if (ticketId) return { mode: 'implement', confidence: 0.95 };

  const lower = query.toLowerCase().trim();

  // IMPLEMENT
  if (/^(add|create|implement|build|write|update|change|modify|refactor|remove|delete|migrate|introduce|set up|move|rename|extract|split|merge)\b/i.test(lower))
    return { mode: 'implement', confidence: 0.92 };
  if (/implement\s+[A-Z][A-Z0-9_]+-\d+/i.test(lower))
    return { mode: 'implement', confidence: 0.95 };
  if (/\b(i need to|i want to|we should|let's|can you)\b.+(add|create|modify|fix|build|implement)\b/i.test(lower))
    return { mode: 'implement', confidence: 0.85 };

  // DIAGNOSE
  if (/\b(error|exception|fail|bug|broken|crash|not working|debug|null|undefined|timeout|500|404|stacktrace|NullPointerException)\b/i.test(lower))
    return { mode: 'diagnose', confidence: 0.85 };
  if (/^(fix|resolve|debug|troubleshoot|investigate|why is|why does|what caused)\b/i.test(lower))
    return { mode: 'diagnose', confidence: 0.85 };

  // UNDERSTAND (default)
  return { mode: 'understand', confidence: 0.75 };
}
```

For low-confidence cases, use Coder-Next (:8090) for quick resolution:

```typescript
async function resolveAmbiguousIntent(
  query: string,
  llm: DualLLMClient
): Promise<GuideMode> {
  const routing = routeToModel('understand', 'intent_classify');
  const response = await llm.chat(
    `Classify this developer request as exactly one word: "understand" (asking a question), "implement" (wanting code changes), or "diagnose" (debugging an error).\n\nRequest: "${query}"\n\nReply with ONE word:`,
    routing
  );
  const word = response.content.trim().toLowerCase();
  if (word.includes('implement')) return 'implement';
  if (word.includes('diagnose')) return 'diagnose';
  return 'understand';
}
```

-----

## 15. Orchestration Engine — The Brain

```typescript
// src/guide/orchestrator.ts

export async function orchestrate(
  params: GuideParams,
  ctx: AgentContext
): Promise<GuideResponse> {
  const startMs = performance.now();

  // ═══ Phase 1: CLASSIFY ═══
  let intent = classifyIntent(params.query, params.mode, params.ticket_id);

  // Resolve ambiguous intent via fast model
  if (intent.confidence < 0.7) {
    intent.mode = await resolveAmbiguousIntent(params.query, ctx.llm);
    intent.confidence = 0.8;
  }

  // Extract structured elements from query
  const extracted = extractQueryElements(params.query);
  // { symbols, filePaths, ticketIds, repoHints, glossaryTerms }

  // ═══ Phase 2: BUILD GATHER PLAN ═══
  const plan = buildGatherPlan(intent.mode, params, extracted);

  // ═══ Phase 3: PARALLEL GATHER (all RAG calls at once) ═══
  const gathered = await executeGatherPlan(plan, params, extracted, ctx);

  // ═══ Phase 4: READ FULL FILES FROM DISK ═══
  const targetFiles = selectTargetFiles(gathered, intent.mode, params);
  const fullFiles = await readTargetFilesFromDisk(targetFiles);

  // ═══ Phase 5: FIND SIMILAR PATTERNS (for IMPLEMENT mode) ═══
  let similarPRs: PRSearchResult[] = [];
  if (intent.mode === 'implement') {
    similarPRs = await searchPRsVector(
      gathered.proseEmbedding,
      params.query
    );
  }

  // ═══ Phase 6: BUILD QWEN CONTEXT ═══
  const qwenContext = buildQwenContext(
    params, intent, extracted, gathered, fullFiles, similarPRs
  );

  // ═══ Phase 7: GENERATE RESPONSE (model-routed) ═══
  let response: GuideResponse;

  switch (intent.mode) {
    case 'understand':
      response = await generateUnderstandResponse(params.query, qwenContext, ctx);
      break;
    case 'implement':
      response = await generateImplementResponse(params.query, qwenContext, ctx);
      break;
    case 'diagnose':
      response = await generateDiagnoseResponse(params.query, qwenContext, ctx);
      break;
  }

  // ═══ Phase 8: HALLUCINATION CHECK ═══
  response = await validateResponse(response, fullFiles);

  // ═══ Phase 9: LOG ═══
  const totalMs = Math.round(performance.now() - startMs);
  await ctx.db.logInvocation({
    query: params.query,
    mode: intent.mode,
    intentConfidence: intent.confidence,
    modelUsed: response._modelUsed || 'unknown',
    modelTier: response._modelTier || 'fast',
    ragToolsCalled: gathered._toolsCalled,
    totalMs,
    responseConfidence: response.confidence.level,
  });

  return response;
}
```

-----

## 16. Gather Planner & Executor

```typescript
// src/guide/gather_planner.ts

export interface GatherPlan {
  searchCode: boolean;
  searchExact: boolean;
  findSymbols: string[];
  getCodeContext: string[];
  analyzeBlastRadius: string | null;
  getTicketContext: string | null;
  getProjectInfo: string | null;
  getRecentChanges: string | null;
  searchArchitecture: boolean;
  getRepoConfig: string | null;
  analyzeGraphHealth: string | null;
  readCurrentFile: boolean;
  getFileDeps: boolean;
  suggestRelated: boolean;
  lookupGlossary: string[];
}

export function buildGatherPlan(
  mode: GuideMode,
  params: GuideParams,
  extracted: ExtractedElements
): GatherPlan {
  return {
    // Always
    searchCode: true,
    searchExact: extracted.symbols.length > 0,
    findSymbols: extracted.symbols.slice(0, 3),
    getCodeContext: extracted.symbols.slice(0, 2),
    lookupGlossary: extracted.glossaryTerms,

    // Current file context
    readCurrentFile: !!params.current_file,
    getFileDeps: !!params.current_file,
    suggestRelated: !!params.current_file,

    // Mode-dependent
    analyzeBlastRadius: mode === 'implement' && extracted.symbols[0]
      ? extracted.symbols[0] : null,
    getTicketContext: params.ticket_id || extracted.ticketIds[0] || null,
    getProjectInfo: params.current_repo || extracted.repoHints[0] || null,
    getRecentChanges: mode === 'diagnose' && params.current_repo
      ? params.current_repo : null,
    searchArchitecture: mode === 'understand' || mode === 'implement',
    getRepoConfig: (mode === 'implement' || mode === 'diagnose') && params.current_repo
      ? params.current_repo : null,
    analyzeGraphHealth: mode === 'implement' && params.current_repo
      ? params.current_repo : null,
  };
}
```

```typescript
// src/guide/gather_executor.ts

export async function executeGatherPlan(
  plan: GatherPlan,
  params: GuideParams,
  extracted: ExtractedElements,
  ctx: AgentContext
): Promise<GatheredContext> {

  const promises: Record<string, Promise<any>> = {};
  const toolsCalled: string[] = [];

  // ── Code search ──
  if (plan.searchCode) {
    promises.codeResults = ctx.rag.searchCode(params.query, 15);
    toolsCalled.push('search_semantic_code');
  }

  if (plan.searchExact) {
    promises.exactResults = ctx.rag.searchExact(extracted.symbols.join(' '), 10);
    toolsCalled.push('search_exact_match');
  }

  // ── Symbol resolution ──
  for (const sym of plan.findSymbols) {
    promises[`symbol_${sym}`] = ctx.rag.findSymbol(sym, params.current_repo);
    toolsCalled.push('find_symbol');
  }

  for (const sym of plan.getCodeContext) {
    promises[`context_${sym}`] = ctx.rag.getCodeContext(sym, params.current_repo);
    toolsCalled.push('get_code_context');
  }

  // ── Blast radius ──
  if (plan.analyzeBlastRadius) {
    promises.blastRadius = ctx.rag.analyzeBlastRadius(
      plan.analyzeBlastRadius, params.current_repo || ''
    );
    toolsCalled.push('analyze_blast_radius');
  }

  // ── Jira ──
  if (plan.getTicketContext) {
    promises.ticketContext = ctx.rag.getTicketContext(plan.getTicketContext);
    toolsCalled.push('get_ticket_context');
  }

  // ── Project info ──
  if (plan.getProjectInfo) {
    promises.projectInfo = ctx.rag.getProjectInfo(plan.getProjectInfo);
    toolsCalled.push('get_project_info');
  }

  // ── Recent changes ──
  if (plan.getRecentChanges) {
    promises.recentChanges = ctx.rag.getRecentChanges(plan.getRecentChanges);
    toolsCalled.push('get_recent_changes');
  }

  // ── Confluence ──
  if (plan.searchArchitecture) {
    promises.confluence = ctx.rag.searchArchitecture(params.query);
    toolsCalled.push('search_architecture');
  }

  // ── Config ──
  if (plan.getRepoConfig) {
    promises.repoConfig = ctx.rag.getRepoConfig(plan.getRepoConfig);
    toolsCalled.push('get_repo_config');
  }

  // ── Graph health ──
  if (plan.analyzeGraphHealth) {
    promises.graphHealth = ctx.rag.analyzeGraphHealth(plan.analyzeGraphHealth);
    toolsCalled.push('analyze_graph_health');
  }

  // ── Current file ──
  if (plan.readCurrentFile && params.current_file && params.current_repo) {
    promises.currentFile = ctx.rag.readFullFile(params.current_file, params.current_repo);
    toolsCalled.push('read_full_file');
  }

  if (plan.getFileDeps && params.current_file && params.current_repo) {
    promises.fileDeps = ctx.rag.getFileDependencies(params.current_file, params.current_repo);
    toolsCalled.push('get_file_dependencies');
  }

  if (plan.suggestRelated && params.current_file && params.current_repo) {
    promises.relatedFiles = ctx.rag.suggestRelatedFiles(params.current_file, params.current_repo);
    toolsCalled.push('suggest_related_files');
  }

  // ── Glossary ──
  for (const term of plan.lookupGlossary) {
    promises[`glossary_${term}`] = ctx.rag.lookupGlossary(term);
    toolsCalled.push('lookup_glossary');
  }

  // ── Execute all in parallel ──
  const results: Record<string, any> = {};
  const settled = await Promise.allSettled(
    Object.entries(promises).map(async ([key, promise]) => {
      results[key] = await promise;
    })
  );

  // Log failures but don't crash
  settled.forEach((s, i) => {
    if (s.status === 'rejected') {
      console.warn(`[Gather] ${Object.keys(promises)[i]} failed:`, (s as any).reason?.message);
    }
  });

  return { ...assembleGathered(results, extracted), _toolsCalled: toolsCalled };
}
```

-----

## 17. Mode 1: UNDERSTAND

Uses **fast model** (:8090 Coder-Next) — instant explanations.

```typescript
// src/guide/modes/understand.ts

export interface UnderstandResponse {
  mode: 'understand';
  explanation: string;                    // Coder-Next generated
  evidence: Evidence[];
  entryPoint: EntryPointInfo | null;
  executionFlow: FlowStep[] | null;
  architecturePattern: string | null;
  documentation: DocRef[];
  glossary: GlossaryEntry[];
  followUpSuggestions: string[];
  confidence: ConfidenceInfo;
  _modelUsed: string;
  _modelTier: string;
}

export async function generateUnderstandResponse(
  query: string,
  ctx: QwenContext,
  agentCtx: AgentContext
): Promise<UnderstandResponse> {

  // Route to FAST model
  const routing = routeToModel('understand', 'explain');

  const prompt = buildUnderstandPrompt(query, ctx);

  const response = await chatWithFallback(
    prompt, routing, agentCtx.llm,
    { systemPrompt: 'You are a senior engineer explaining code. Use only the facts provided. Reference specific files and functions. Be concise.' }
  );

  // Extract follow-ups via fast model
  const followUps = await generateFollowUps(query, ctx, agentCtx.llm);

  return {
    mode: 'understand',
    explanation: response.content,
    evidence: buildEvidence(ctx),
    entryPoint: ctx.entryPoint,
    executionFlow: ctx.executionFlow,
    architecturePattern: ctx.pattern?.name || null,
    documentation: ctx.confluence,
    glossary: ctx.glossary,
    followUpSuggestions: followUps,
    confidence: computeConfidence(ctx, response),
    _modelUsed: response.model,
    _modelTier: response.tier,
  };
}

function buildUnderstandPrompt(query: string, ctx: QwenContext): string {
  return `QUESTION: "${query}"

${ctx.entryPoint ? `ENTRY POINT: ${ctx.entryPoint.symbol} in ${ctx.entryPoint.file} (${ctx.entryPoint.repo})
Reason: ${ctx.entryPoint.reason}` : ''}

${ctx.executionFlow ? `EXECUTION FLOW:
${ctx.executionFlow.map((s, i) => `${i + 1}. ${s.symbol} (${s.file})${s.description ? ' — ' + s.description : ''}`).join('\n')}` : ''}

${ctx.pattern ? `ARCHITECTURE: ${ctx.pattern.name} — ${ctx.pattern.description}` : ''}

CODE EVIDENCE:
${ctx.files.slice(0, 5).map(f =>
  `=== ${f.path} (${f.repo}) ===\n${f.content.slice(0, 2500)}\n=== END ===`
).join('\n\n')}

DEPENDENCIES:
${ctx.relationships.slice(0, 10).map(r => `- ${r.from} ${r.type} ${r.to} (${r.toFile})`).join('\n')}

${ctx.glossary.length > 0 ? `GLOSSARY: ${ctx.glossary.map(g => `${g.acronym} = ${g.meaning}`).join(', ')}` : ''}

${ctx.healthWarnings.filter(w => w.severity === 'major').length > 0
  ? `⚠️ HEALTH WARNINGS:\n${ctx.healthWarnings.filter(w => w.severity === 'major').map(w => `- ${w.message}`).join('\n')}` : ''}

Write a clear technical explanation. Reference specific files and functions. Under 400 words.`;
}
```

-----

## 18. Mode 2: IMPLEMENT

Uses **deep model** (:8091 Qwen3-235B) — production code generation.

```typescript
// src/guide/modes/implement.ts

export interface ImplementResponse {
  mode: 'implement';
  summary: string;

  // ── Code Changes (235B generated) ──
  changes: CodeChange[];
  newFiles: NewFileSpec[];
  configChanges: ConfigChange[];

  // ── Context ──
  targetFiles: TargetFile[];
  referenceFiles: TargetFile[];
  similarPRs: PRSummary[];
  conventions: Convention[];

  // ── Impact ──
  blastRadius: BlastRadiusInfo;
  testsToUpdate: string[];
  healthWarnings: HealthWarning[];

  // ── Agent-Ready Metadata ──
  affectedSymbols: AffectedSymbol[];
  executionOrder: string[];
  preChecks: string[];
  postChecks: string[];

  confidence: ConfidenceInfo;
  _modelUsed: string;
  _modelTier: string;
}

export interface CodeChange {
  id: string;
  file: string;
  repo: string;
  action: 'modify' | 'insert' | 'replace' | 'delete';
  description: string;
  reason: string;
  currentCode: {
    startLine: number;
    endLine: number;
    content: string;
    contentHash: string;
  } | null;
  proposedCode: string;
  priority: number;
  dependencies: string[];
  affectedSymbols: string[];
}

export async function generateImplementResponse(
  query: string,
  ctx: QwenContext,
  agentCtx: AgentContext
): Promise<ImplementResponse> {

  // ── Step 1: Detect conventions via FAST model ──
  let conventions: Convention[] = [];
  if (ctx.files.length >= 2) {
    const convRouting = routeToModel('implement', 'convention_detect');
    const convPrompt = buildConventionPrompt(ctx.files.slice(0, 3));
    const convResponse = await chatWithFallback(convPrompt, convRouting, agentCtx.llm);
    conventions = parseConventions(convResponse.content);
  }

  // ── Step 2: Generate code changes via DEEP model ──
  const codeRouting = routeToModel('implement', 'generate_code');

  // Build numbered file contents for precise line references
  const fileContexts = ctx.files.slice(0, 5).map(f => {
    const numbered = f.content.split('\n')
      .map((line, i) => `${String(i + 1).padStart(4)}: ${line}`)
      .join('\n');
    return `=== ${f.path} (${f.repo}, role: ${f.role}) [${f.lineCount} lines] ===\n${numbered}\n=== END ===`;
  }).join('\n\n');

  const prPatterns = ctx.similarPRs.slice(0, 2).map(pr =>
    `PR-${pr.prNumber} "${pr.title}":\n  Problem: ${pr.coreProblem}\n  Implementation: ${pr.implementationSummary}\n  Decisions: ${pr.keyDecisions}`
  ).join('\n\n');

  const prompt = `You are a production software engineer writing code for a banking application.
Generate EXACT code changes as JSON. Reference EXACT line numbers from the source files below.
Only use files and symbols that exist in the evidence.

TASK: ${query}

${ctx.ticketContext ? `JIRA CONTEXT:
  Business Goal: ${ctx.ticketContext.businessGoal || ''}
  Task: ${ctx.ticketContext.specificTask || ''}
  Acceptance Criteria: ${ctx.ticketContext.acceptanceCriteria || ''}` : ''}

${ctx.pattern ? `ARCHITECTURE: ${ctx.pattern.name} — ${ctx.pattern.description}` : ''}

${conventions.length > 0 ? `TEAM CONVENTIONS (MUST follow):\n${conventions.map(c => `- ${c.convention}`).join('\n')}` : ''}

${prPatterns ? `SIMILAR PAST IMPLEMENTATIONS:\n${prPatterns}` : ''}

${ctx.healthWarnings.filter(w => w.severity === 'major').map(w =>
  `⚠️ ${w.message} → ${w.recommendation || 'Fix this too'}`
).join('\n')}

SOURCE FILES (line numbers matter):
${fileContexts}

DEPENDENCIES:
${ctx.relationships.slice(0, 10).map(r => `- ${r.from} ${r.type} ${r.to} (${r.toFile})`).join('\n')}

Respond with ONLY valid JSON:
{
  "summary": "1-2 sentence summary",
  "changes": [
    {
      "file": "exact/path.java",
      "repo": "repo-slug",
      "action": "modify",
      "description": "what",
      "startLine": 45,
      "endLine": 52,
      "currentCode": "exact text at those lines",
      "proposedCode": "new code",
      "reason": "why",
      "priority": 1,
      "affectedSymbols": ["symbolName"]
    }
  ],
  "newFiles": [
    {
      "file": "path/NewFile.java",
      "repo": "repo-slug",
      "description": "purpose",
      "content": "full file content",
      "reason": "why",
      "basedOn": "existing/Similar.java"
    }
  ],
  "configChanges": [
    {
      "file": "application.yml",
      "repo": "repo-slug",
      "description": "what",
      "proposedValue": "config block",
      "reason": "why"
    }
  ]
}`;

  const codeResponse = await chatWithFallback(prompt, codeRouting, agentCtx.llm);
  const plan = parseJsonResponse(codeResponse.content);

  // ── Step 3: Assemble response ──
  const changes = buildChanges(plan, ctx);
  const newFiles = buildNewFiles(plan);
  const configChanges = buildConfigChanges(plan);

  return {
    mode: 'implement',
    summary: plan?.summary || `${changes.length} changes planned`,
    changes,
    newFiles,
    configChanges,
    targetFiles: ctx.files.filter(f => f.role === 'modify').map(toTargetFile),
    referenceFiles: ctx.files.filter(f => f.role === 'reference').map(toTargetFile),
    similarPRs: ctx.similarPRs.map(toSummary),
    conventions,
    blastRadius: ctx.blastRadius || { riskLevel: 'unknown', totalFiles: 0, directConsumers: [], crossRepoConsumers: [] },
    testsToUpdate: identifyTests(ctx),
    healthWarnings: ctx.healthWarnings,
    affectedSymbols: extractAffectedSymbols(changes, ctx),
    executionOrder: changes.map(c => c.id),
    preChecks: buildPreChecks(ctx),
    postChecks: buildPostChecks(ctx),
    confidence: computeConfidence(ctx, codeResponse),
    _modelUsed: codeResponse.model,
    _modelTier: codeResponse.tier,
  };
}
```

-----

## 19. Mode 3: DIAGNOSE

Uses **fast model** for analysis, **deep model** for fix generation.

```typescript
// src/guide/modes/diagnose.ts

export interface DiagnoseResponse {
  mode: 'diagnose';
  analysis: string;                       // Fast model: root cause analysis
  likelyLocation: ErrorLocation[];
  reverseCallChain: FlowStep[];
  suggestedFix: CodeChange[] | null;      // Deep model: fix code (if confident)
  recentChanges: RecentChange[];
  configContext: ConfigFile[];
  evidence: Evidence[];
  confidence: ConfidenceInfo;
  _modelUsed: string;
  _modelTier: string;
}

export async function generateDiagnoseResponse(
  query: string,
  ctx: QwenContext,
  agentCtx: AgentContext
): Promise<DiagnoseResponse> {

  // ── Step 1: Analyze via FAST model ──
  const analyzeRouting = routeToModel('diagnose', 'analyze');
  const analysisPrompt = buildDiagnosePrompt(query, ctx);
  const analysisResponse = await chatWithFallback(analysisPrompt, analyzeRouting, agentCtx.llm);

  // ── Step 2: Generate fix via DEEP model (if high confidence) ──
  let suggestedFix: CodeChange[] | null = null;

  if (ctx.files.length > 0 && ctx.relationships.length > 0) {
    const fixRouting = routeToModel('diagnose', 'generate_fix');
    const fixPrompt = buildFixPrompt(query, ctx, analysisResponse.content);
    const fixResponse = await chatWithFallback(fixPrompt, fixRouting, agentCtx.llm);
    const fixPlan = parseJsonResponse(fixResponse.content);
    if (fixPlan?.changes) {
      suggestedFix = buildChanges(fixPlan, ctx);
    }
  }

  return {
    mode: 'diagnose',
    analysis: analysisResponse.content,
    likelyLocation: identifyErrorLocations(ctx),
    reverseCallChain: ctx.executionFlow || [],
    suggestedFix,
    recentChanges: ctx.recentChanges || [],
    configContext: ctx.configFiles || [],
    evidence: buildEvidence(ctx),
    confidence: computeConfidence(ctx, analysisResponse),
    _modelUsed: suggestedFix ? 'qwen3-235b + qwen3-coder-next' : 'qwen3-coder-next',
    _modelTier: suggestedFix ? 'deep + fast' : 'fast',
  };
}
```

-----

## 20-23. Supporting Modules

### 20. Qwen Context Builder

Assembles all gathered data into a structured context object for prompt building. Handles entry point detection (from `get_code_context` results), execution flow assembly (from `getCallChain` data), pattern detection (from `projectInfo.tech_stack`), and file role assignment (modify vs reference).

### 21. Code Generation Engine

The `parseJsonResponse()` function strips markdown fences, finds JSON bounds, validates structure. `buildChanges()` maps raw JSON to typed `CodeChange[]` with auto-generated IDs and dependency tracking.

### 22. Hallucination Prevention

Three-layer verification: (1) file exists on disk via `REPOS_DIR`, (2) line content at stated range matches via fuzzy comparison (>75% word overlap), (3) symbols exist in `graph_nodes` table. Content hash (MD5) generated for every `currentCode` block.

### 23. Confidence Model

```typescript
function computeConfidence(ctx: QwenContext, llmResponse: LLMResponse): ConfidenceInfo {
  const level =
    ctx.files.length >= 3 && ctx.relationships.length >= 2 ? 'high' :
    ctx.files.length >= 1 ? 'medium' : 'low';

  return {
    level,
    dataSources: ctx._toolsCalled || [],
    modelUsed: llmResponse.model,
    modelTier: llmResponse.tier,
    fellBack: (llmResponse as any).fellBack || false,
    graphCoverage: ctx.relationships.length > 0,
    filesVerified: true, // Set by hallucination guard
  };
}
```

-----

# Part E — Agent Execution

## 24. Action Plan as Agent Contract

The `ImplementResponse` is designed for a future agent to execute mechanically:

- `changes[].currentCode.contentHash` — verify file hasn’t changed before editing
- `executionOrder` — topologically sorted change sequence
- `preChecks/postChecks` — automated verification steps
- `affectedSymbols[].referenceCount` — blast radius awareness

## 25. Extension Path

```
Phase 1 (NOW):  Developer → Windsurf → ome_rag_guide → code plan → developer applies
Phase 2 (NEXT): Developer → Windsurf → ome_rag_guide → code plan → agent applies → PR
Phase 3 (LATER): Jira webhook → ome_rag_guide → agent applies → auto PR → reviewer
```

Phase 2 needs: `execute_plan` tool, `verify_hashes`, git worktree management, RAG’s `create_pull_request` integration.

-----

# Part F — Implementation

## 26. Implementation Schedule

```
Week 1: Infrastructure (Days 1-5)

  Day 1: GPU Setup — Dual Model Deployment
    ├── Build llama.cpp with CUDA for H100
    ├── Download Qwen3-Coder-Next Q5_K_M (~30GB)
    ├── Download Qwen3-235B-A22B Q4_K_XL (~120GB download, ~82GB MoE on RAM)
    ├── Deploy Coder-Next on :8090 (verify: 50+ tok/s, 64K ctx, 3 slots)
    ├── Deploy 235B on :8091 with MoE offload (verify: ~20 tok/s, 32K ctx)
    ├── Systemd services (ome-agent-fast, ome-agent-deep)
    └── Health check endpoint for both models

  Day 2: Project Setup + MCP Server
    ├── Initialize ome-agent project (package.json, tsconfig, .env)
    ├── DualLLMClient class with routing + fallback
    ├── MCP server (stdio + HTTP transport)
    ├── ome_rag_guide tool registration
    ├── Health endpoint
    └── Test: Windsurf sees ome_rag_guide tool

  Day 3: RAG Client (MCP Consumer)
    ├── RAGClient class — MCP SDK client to OME RAG
    ├── Typed wrappers for 20+ RAG tools
    ├── Connection management (reconnect, timeout)
    ├── Test: search_semantic_code, find_symbol, get_code_context via MCP
    └── Windsurf config (both RAG + Agent MCP servers)

  Day 4: Direct Access + Agent DB
    ├── Read-only otak connection (rag_db.ts)
    ├── PR vector search (searchPRsVector)
    ├── Jira vector search (searchJiraVector)
    ├── Health warnings query
    ├── Glossary loader
    ├── File reader (REPOS_DIR)
    ├── ome_agent schema + migrations
    └── Agent DB: sessions, invocations, model_metrics tables

  Day 5: PR/Jira Embedding (OME RAG changes)
    ├── vec_pr_summaries + vec_jira_summaries tables
    ├── pr_ingestion.ts embed hook
    ├── jira_ingestion.ts embed hook
    ├── Hybrid search upgrade (stages 6-7)
    ├── Backfill scripts
    ├── PR→symbol modifies graph edges
    └── Verify: "how was resilience implemented" finds circuit breaker PRs

Week 2: Core Logic (Days 6-10)

  Day 6: Intent Classifier + Gather Planner
    ├── intent_classifier.ts (understand/implement/diagnose)
    ├── Ambiguous intent resolution via fast model
    ├── Query element extractor (symbols, paths, tickets, repos)
    ├── gather_planner.ts (plan per mode)
    ├── gather_executor.ts (parallel RAG calls)
    └── 50+ intent classification tests

  Day 7: Model Router + Context Builder
    ├── model_router.ts (fast vs deep per task)
    ├── Graceful degradation (deep→fast fallback)
    ├── qwen_context_builder.ts
    ├── Entry point detection
    ├── Execution flow assembly
    ├── Pattern detection (CSR, CHoS, Event-Driven)
    └── File role assignment (modify vs reference)

  Day 8: UNDERSTAND Mode
    ├── understand.ts response builder
    ├── Explain prompt (fast model)
    ├── Evidence builder
    ├── Follow-up suggestion generator
    ├── Documentation integration (Confluence)
    └── Glossary expansion

  Day 9: IMPLEMENT Mode — Core
    ├── implement.ts response builder
    ├── Convention detection (fast model)
    ├── Code generation prompt (deep model)
    ├── Numbered file context builder
    ├── Similar PR injection
    └── JSON response parser

  Day 10: IMPLEMENT Mode — Assembly + DIAGNOSE
    ├── Change plan assembly (CodeChange[], NewFileSpec[])
    ├── Blast radius integration
    ├── Test identification
    ├── Pre/post checklist generation
    ├── diagnose.ts — analysis (fast) + fix generation (deep)
    └── Error location identification

Week 3: Quality + Testing (Days 11-15)

  Day 11: Hallucination Prevention
    ├── File existence verification (REPOS_DIR)
    ├── Line content fuzzy matching + correction
    ├── Symbol existence verification
    ├── Content hash generation (MD5)
    ├── Confidence computation
    └── Affected symbol extraction

  Day 12: Full Integration + Orchestrator
    ├── Wire orchestrator end-to-end
    ├── All 3 modes working through MCP
    ├── Metrics logging (invocations, model_metrics)
    ├── Error handling (models down, RAG down, files missing)
    └── Graceful degradation paths

  Day 13: Testing — UNDERSTAND + DIAGNOSE
    ├── 10+ UNDERSTAND golden queries
    ├── 5+ DIAGNOSE golden queries
    ├── Explanation quality
    ├── Evidence accuracy
    ├── Model routing correctness
    └── Response latency (UNDERSTAND <5s target)

  Day 14: Testing — IMPLEMENT
    ├── 10+ IMPLEMENT golden tasks
    ├── Code generation quality (compiles? follows patterns?)
    ├── Line number accuracy
    ├── Convention detection accuracy
    ├── Blast radius correctness
    ├── Similar PR relevance
    ├── Hallucination rate (target: <5%)
    └── Response latency (IMPLEMENT <90s target)

  Day 15: Polish + Documentation
    ├── Edge cases (huge files, no graph, empty repo, both models down)
    ├── Performance profiling
    ├── PROJECT_ARCHITECTURE.md for OME Agent
    ├── README with setup instructions
    ├── Systemd + deployment guide
    └── Team walkthrough

Week 4 (Optional): Extended Testing + Phase 2 Prep

  Days 16-18:
    ├── Cross-model consistency testing (same query → same facts)
    ├── A/B comparison: Agent output quality vs raw RAG output
    ├── Load testing (3 concurrent UNDERSTAND + 1 IMPLEMENT)
    ├── execute_plan tool skeleton (Phase 2)
    ├── Git worktree management prototype
    └── Integration with RAG's create_pull_request
```

-----

## 27. Configuration

```bash
# ═══ OME Agent Server ═══
AGENT_PORT=3100
AGENT_HOST=0.0.0.0
AGENT_MCP_API_KEY=<bearer-token>

# ═══ Dual Model ═══
FAST_MODEL_URL=http://127.0.0.1:8090      # Qwen3-Coder-Next
DEEP_MODEL_URL=http://127.0.0.1:8091      # Qwen3-235B-A22B
FAST_MODEL_TIMEOUT_MS=30000               # 30s for fast model
DEEP_MODEL_TIMEOUT_MS=120000              # 120s for deep model (code gen takes time)

# ═══ RAG Connection ═══
OME_RAG_URL=http://localhost:3000/mcp
MCP_API_KEY=<shared-bearer-token>

# ═══ Shared Resources ═══
REPOS_DIR=/repos
RAG_DATABASE_URL=postgresql://ome_agent_readonly:***@localhost:5432/omedb

# ═══ Agent DB ═══
AGENT_DATABASE_URL=postgresql://ome_agent:***@localhost:5432/omedb

# ═══ Limits ═══
GUIDE_MAX_TARGET_FILES=8
GUIDE_MAX_REFERENCE_FILES=4
GUIDE_MAX_SIMILAR_PRS=3
GUIDE_MAX_CHANGES=12
GUIDE_HALLUCINATION_CHECK=true
GUIDE_CONVENTION_DETECTION=true

# ═══ Feature Flags ═══
AGENT_EXECUTION_ENABLED=false              # Phase 2
EMBED_PR_SUMMARIES=true                    # OME RAG flag
EMBED_JIRA_SUMMARIES=true                  # OME RAG flag
```

-----

## 28. File Structure

```
ome-agent/
├── package.json
├── tsconfig.json
├── .env / .env.example
├── README.md
│
├── src/
│   ├── server/
│   │   ├── mcp_server.ts                 # MCP server (1 tool: ome_rag_guide)
│   │   ├── http_entry.ts                 # Express + Streamable HTTP
│   │   └── stdio_entry.ts               # Stdio for Windsurf direct
│   │
│   ├── llm/
│   │   ├── dual_client.ts                # DualLLMClient (fast + deep)
│   │   ├── model_router.ts               # Route tasks to fast/deep
│   │   ├── health.ts                     # Model health checks
│   │   └── types.ts                      # LLMResponse, RoutingDecision
│   │
│   ├── rag_client/
│   │   ├── mcp_client.ts                 # RAGClient — MCP SDK consumer (20+ typed wrappers)
│   │   ├── rag_db.ts                     # Read-only otak queries (PR vectors, health, glossary)
│   │   ├── file_reader.ts                # REPOS_DIR filesystem access
│   │   └── types.ts                      # RAG response types
│   │
│   ├── guide/
│   │   ├── tool_definition.ts            # ome_rag_guide schema + Zod
│   │   ├── orchestrator.ts               # 9-phase brain
│   │   ├── intent_classifier.ts          # understand / implement / diagnose
│   │   ├── gather_planner.ts             # Which RAG tools per mode
│   │   ├── gather_executor.ts            # Parallel RAG call execution
│   │   ├── query_extractor.ts            # Extract symbols, paths, tickets from query
│   │   ├── qwen_context_builder.ts       # Assemble LLM prompt context
│   │   │
│   │   ├── modes/
│   │   │   ├── understand.ts             # UNDERSTAND (fast model)
│   │   │   ├── implement.ts              # IMPLEMENT (deep model for code)
│   │   │   └── diagnose.ts               # DIAGNOSE (fast + deep)
│   │   │
│   │   ├── analyze/
│   │   │   ├── entry_point.ts            # Entry point detection
│   │   │   ├── flow_tracer.ts            # Execution flow assembly
│   │   │   ├── pattern_detector.ts       # CSR, CHoS, Event-Driven
│   │   │   ├── convention_extractor.ts   # Qwen fast convention detection
│   │   │   └── error_locator.ts          # Error area identification
│   │   │
│   │   ├── verify/
│   │   │   ├── hallucination_guard.ts    # 3-layer verification
│   │   │   ├── confidence.ts             # Confidence computation
│   │   │   └── content_hash.ts           # MD5 for agent verification
│   │   │
│   │   └── types.ts                      # All response types
│   │
│   ├── db/
│   │   ├── agent_pool.ts                 # ome_agent schema connection
│   │   ├── agent_db.ts                   # Typed queries (log invocations, etc.)
│   │   └── migrations/
│   │       └── 001_initial.sql           # sessions, invocations, change_plans, metrics
│   │
│   ├── execution/                        # Phase 2 (stubbed)
│   │   ├── plan_executor.ts
│   │   ├── worktree_manager.ts
│   │   └── pr_creator.ts
│   │
│   └── types/
│       └── shared.ts
│
├── test/
│   ├── intent_classifier.test.ts
│   ├── model_router.test.ts
│   ├── understand_golden.test.ts
│   ├── implement_golden.test.ts
│   ├── diagnose_golden.test.ts
│   ├── hallucination.test.ts
│   └── rag_client.test.ts
│
├── scripts/
│   ├── setup_gpu.sh                      # Build llama.cpp + download both models
│   ├── setup_db.sh                       # Create ome_agent schema
│   ├── test_fast_model.ts                # Smoke test Coder-Next
│   ├── test_deep_model.ts                # Smoke test 235B
│   └── benchmark_models.ts              # Compare fast vs deep latency/quality
│
└── systemd/
    ├── ome-agent.service                 # Node.js
    ├── ome-agent-fast.service            # Coder-Next :8090
    └── ome-agent-deep.service            # 235B :8091
```

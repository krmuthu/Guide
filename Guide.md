# OME Agent — Definitive Implementation Plan (Final)

> **Document Version:** 3.0 — 2026-03-28
> **Project:** OME Agent (separate from OME RAG)
> **GPU:** NVIDIA H100 — 95,830 MB (93.6 GB) VRAM
> **Server:** RHEL 9.7, 128 cores, 1TB RAM
> **Models:** Qwen3-Coder-Next (fast, :8090) + Qwen3-235B-A22B (deep, :8091)
> **Relationship:** OME Agent consumes OME RAG via MCP client
> **Changes to OME RAG:** PR vector embedding, simplified intelligence layer, PR→symbol graph edges
> **Effort:** ~18-20 days (Agent) + ~2 days (RAG changes)

-----

## Table of Contents

### Part A — Architecture

1. [System Architecture](#1-system-architecture)
1. [GPU Memory Layout — 93.6 GB](#2-gpu-memory-layout)
1. [Model Deployment](#3-model-deployment)
1. [Model Router — Fast vs Deep](#4-model-router)

### Part B — Changes to OME RAG

1. [Intelligence Layer Refactoring — Keep Enrichment, Move Reasoning](#5-intelligence-layer-refactoring)
1. [PR Vector Embedding (YES) + Jira Embedding (NO)](#6-pr-embedding)
1. [PR→Symbol Graph Edges](#7-prsymbol-graph-edges)

### Part C — OME Agent Project

1. [Project Setup](#8-project-setup)
1. [MCP Server — Exposing ome_rag_guide](#9-mcp-server)
1. [RAG Client — Consuming OME RAG](#10-rag-client)
1. [Agent Database Schema](#11-agent-database)

### Part D — ome_rag_guide Implementation

1. [Tool Definition](#12-tool-definition)
1. [Three Modes + Intent Classification](#13-intent-classification)
1. [Orchestration Engine](#14-orchestration-engine)
1. [Gather Planner & Executor](#15-gather-planner)
1. [Mode 1: UNDERSTAND (Fast Model)](#16-mode-understand)
1. [Mode 2: IMPLEMENT (Deep Model)](#17-mode-implement)
1. [Mode 3: DIAGNOSE (Fast + Deep)](#18-mode-diagnose)
1. [Sparse Ticket Enrichment](#19-sparse-ticket-enrichment)
1. [Similar PR Discovery (Vector Search)](#20-similar-pr-discovery)
1. [Hallucination Prevention](#21-hallucination-prevention)

### Part E — Agent-Ready Architecture

1. [Phase 2 Extension Path](#22-phase-2)

### Part F — Implementation

1. [Implementation Schedule](#23-implementation-schedule)
1. [Configuration](#24-configuration)
1. [File Structure](#25-file-structure)

-----

# Part A — Architecture

## 1. System Architecture

```
┌──────────────────────────────────────────────────────────────────────┐
│  IDE  (Windsurf / Cursor / VS Code)                                  │
└────────┬──────────────────────────────────────────┬──────────────────┘
         │ MCP                                      │ MCP
         │ Simple: search, browse, read             │ Complex: understand, implement, debug
         ▼                                          ▼
┌────────────────────────────┐   ┌──────────────────────────────────────┐
│  OME RAG  (existing)       │   │  OME Agent  (NEW project)            │
│  Knowledge Layer (CPU)     │   │  Action Layer (GPU)                  │
│                            │   │                                      │
│  43 MCP tools              │◄──│  ome_rag_guide (the brain)           │
│  hybrid_search.ts          │   │                                      │
│  IGraphClient              │   │  Dual GPU Models:                    │
│  Bootstrap pipeline        │   │  :8090 Coder-Next (fast, 3B active)  │
│  Lightweight enrichment    │   │  :8091 235B-A22B (deep, 22B active)  │
│  (graph context, glossary, │   │                                      │
│   health warnings)         │   │  Consumes RAG via MCP client         │
│                            │   │  Reads REPOS_DIR + otak (read-only)  │
│  NEW: vec_pr_summaries     │   │                                      │
│  NEW: PR→symbol edges      │   │  Agent DB: ome_agent schema          │
│                            │   │                                      │
│  CPU models (unchanged):   │   │  Port: 3100                          │
│  :8081 Arctic Embed        │   │                                      │
│  :8082 Qwen2.5-7B         │   │                                      │
│  :8083 BGE Reranker        │   │                                      │
│  :8084 Qwen2.5-1.5B       │   │                                      │
│  :8085 nomic-embed-code    │   │                                      │
│                            │   │                                      │
│  Port: 3000                │   │                                      │
└────────────────────────────┘   └──────────────────────────────────────┘
```

-----

## 2. GPU Memory Layout

```
H100 — 95,830 MB (93.6 GB)
──────────────────────────────
Qwen3-Coder-Next Q5_K_M       ~30 GB   (full on GPU)
Qwen3-235B-A22B Q4_K_XL       ~35 GB   (non-MoE on GPU, MoE→CPU RAM)
KV Cache:
  Coder-Next 64K ctx × 3 slots ~9 GB
  235B 32K ctx × 1 slot        ~12 GB
──────────────────────────────
Total:                         ~86 GB
Headroom:                      ~8 GB

CPU RAM (1 TB):
  235B MoE expert layers:      ~82 GB
  Everything else:             ~120 GB
  Free:                        ~800 GB
```

-----

## 3. Model Deployment

### 3.1 Fast Model — Qwen3-Coder-Next (:8090)

```bash
./llama-server \
  --model /models/qwen3-coder-next/Qwen3-Coder-Next-UD-Q5_K_M.gguf \
  --host 127.0.0.1 --port 8090 \
  --n-gpu-layers 99 --ctx-size 65536 --parallel 3 \
  --threads 8 --flash-attn --jinja \
  --temp 1.0 --top-p 0.95 --top-k 40 --min-p 0.01
```

80B MoE (3B active), 256K native context, **50+ tok/s**, full on GPU, 3 parallel slots.

### 3.2 Deep Model — Qwen3-235B-A22B (:8091)

```bash
./llama-server \
  --model /models/qwen3-235b/Qwen3-235B-A22B-UD-Q4_K_XL-00001-of-00003.gguf \
  --host 127.0.0.1 --port 8091 \
  --n-gpu-layers 99 -ot ".ffn_.*_exps.=CPU" \
  --ctx-size 32768 --parallel 1 \
  --threads 32 --flash-attn --jinja \
  --temp 0.6 --top-p 0.95 --top-k 20 --min-p 0.0
```

235B MoE (22B active), thinking mode, **~15-25 tok/s**, MoE on CPU RAM, 1 parallel slot.

### 3.3 Three Systemd Services

`ome-agent-fast.service` → Coder-Next :8090
`ome-agent-deep.service` → 235B :8091 (depends on fast)
`ome-agent.service` → Node.js MCP server (depends on both models)

-----

## 4. Model Router

```typescript
// src/llm/model_router.ts

export function routeToModel(mode: GuideMode, task: ModelTask): RoutingDecision {
  // ── FAST (:8090) — speed tasks ──
  if (task === 'intent_classify') return fast(20, 0.0);
  if (task === 'convention_detect') return fast(400, 0.2);
  if (task === 'followup_suggest') return fast(200, 0.5);
  if (task === 'sparse_ticket_enrich') return fast(300, 0.1);
  if (mode === 'understand') return fast(1500, 0.3);
  if (mode === 'diagnose' && task === 'analyze') return fast(1000, 0.2);

  // ── DEEP (:8091) — quality tasks ──
  if (mode === 'implement') return deep(8000, 0.15);
  if (mode === 'diagnose' && task === 'generate_fix') return deep(4000, 0.15);

  return fast(1000, 0.3); // Default
}
```

**What the developer experiences:**

|Scenario                       |Model          |Time       |
|-------------------------------|---------------|-----------|
|“How does auth work?”          |Fast           |~2-3s      |
|“What does submitOrder do?”    |Fast           |~1s        |
|“NPE in SettlementService”     |Fast (analysis)|~3s        |
|“Add pagination to bond search”|**Deep**       |**~45-50s**|
|“Refactor to Strategy pattern” |**Deep**       |**~60-90s**|
|“Fix the NPE” (after diagnosis)|**Deep**       |**~30s**   |
|“Implement BMT-1234”           |**Deep**       |**~50-60s**|

Graceful degradation: if deep model is down, falls back to fast with a warning.

-----

# Part B — Changes to OME RAG

## 5. Intelligence Layer Refactoring

### 5.1 What STAYS in RAG (Lightweight Enrichment)

These are cheap, benefit both direct Windsurf users AND the Agent:

```
KEEP (rename intelligence layer → graph_enrichment):
  ├── Graph context on search results (callers, callees, centrality)     ~50ms cached
  ├── Health warnings attached to file results                           ~10ms DB query
  ├── Glossary term expansion (acronym_glossary)                         ~5ms cached
  ├── Structured response format (typed JSON)                            ~0ms
  ├── Source-balanced minimum (camelCase BM25 fix)                       Already in pipeline
  └── PR/Jira hybrid search stages 6-7 (stage 6 upgraded to vector+FTS) In pipeline
```

### 5.2 What MOVES to Agent (Heavy Analysis)

```
MOVE TO AGENT (or delete from RAG):
  ├── Query classification (7 types)          → Agent has understand/implement/diagnose
  ├── Entry point detection                   → Agent does this with graph data + Coder-Next
  ├── Execution flow tracing                  → Agent traces with IGraphClient results
  ├── Architecture pattern detection          → Agent does in IMPLEMENT mode
  ├── Convention extraction                   → Agent uses Coder-Next
  ├── Server-side Qwen summary generation     → Agent has GPU models
  ├── Rendering instructions (_instructions)  → Agent controls presentation
  └── 7 analysis pipelines (symbol_lookup,    → All move to Agent's 3 modes
      code_understanding, implementation,
      debugging, architecture, comparison,
      sdlc_pipeline)
```

### 5.3 Feature Flag for Transition

```bash
# OME RAG .env
INTELLIGENCE_LAYER_MODE=lightweight   # Default — graph enrichment only
                                       # 'full' — old behavior (fallback if Agent down)
                                       # 'disabled' — raw chunks, no enrichment
```

### 5.4 Simplified RAG Search Response

```typescript
// What RAG returns after refactoring:
interface EnrichedSearchResult {
  file_path: string;
  repo_slug: string;
  content: string;
  score: number;
  // ── Lightweight enrichment (kept) ──
  graphContext: {
    callers: string[];
    callees: string[];
    centralityScore: number;
    centralityLabel: string;   // 'high-traffic' | 'moderate' | 'low-traffic'
    decorators: string | null; // @Controller, @Service, etc.
  } | null;
  healthWarnings: Array<{
    severity: string;
    message: string;
  }>;
  glossaryExpansions: Array<{
    acronym: string;
    meaning: string;
  }>;
}
```

Both consumers (direct Windsurf and OME Agent) get enriched data. The Agent does its own deeper analysis on top.

### 5.5 Files to Change in OME RAG

```
REFACTOR (don't delete — move to 'legacy' or feature-flag):
  src/mcp/intelligence/query_classifier.ts       → Flag-gated
  src/mcp/intelligence/pipeline_router.ts        → Flag-gated
  src/mcp/intelligence/rendering_instructions.ts → Flag-gated
  src/mcp/intelligence/server_side_reasoning.ts  → Flag-gated
  src/mcp/intelligence/pipelines/*.ts            → Flag-gated

KEEP + SIMPLIFY:
  src/lib/intelligence/graph_enrichment.ts       → Rename to src/lib/enrichment.ts
  src/lib/intelligence/glossary_loader.ts        → Keep as-is
  src/lib/intelligence/cache.ts                  → Keep as-is

EFFORT: ~1 day (add feature flag + simplify the dispatch path)
```

-----

## 6. PR Vector Embedding (YES) + Jira Embedding (NO)

### 6.1 Why PR Embedding — Yes

`pr_condenser.ts` works from **code diffs and commit messages** — always concrete and technical. The condensed output is high-quality prose perfect for Arctic embedding:

```
PR condensation output (typical quality):
  core_problem: "Order submission failed silently when Kafka broker unavailable"
  implementation_summary: "Added @CircuitBreaker with Resilience4j, 50% failure threshold"
  key_decisions: "Used Resilience4j over Spring Retry for consistency"
```

When a developer asks “how was resilience implemented?”, vector search finds this PR even though “resilience” never appears in the condensed text. FTS misses it entirely.

### 6.2 Why Jira Embedding — No

Developers write poor ticket descriptions. `jira_condenser.ts` can’t create meaning from nothing:

```
Actual ticket descriptions in your repos:
  "Implement as discussed in standup"
  "Fix the bond search thing"
  "See Confluence page"
  "AC: it should work correctly"
  "" (empty)
```

Embedding “fix the bond search thing” produces a vector that FTS already matches. Zero added value. The time is better spent on sparse ticket enrichment in the Agent (§19).

### 6.3 Schema (In otak namespace)

```sql
-- PR summary vectors only — no Jira vectors
CREATE TABLE IF NOT EXISTS otak.vec_pr_summaries (
  pr_id        INTEGER PRIMARY KEY REFERENCES otak.pull_requests(id) ON DELETE CASCADE,
  repo_slug    TEXT NOT NULL,
  embedding    HALFVEC(1024) NOT NULL,
  content_hash TEXT NOT NULL
);

CREATE INDEX idx_vec_pr_hnsw ON otak.vec_pr_summaries
  USING hnsw (embedding halfvec_cosine_ops) WITH (m = 16, ef_construction = 100);

CREATE INDEX idx_vec_pr_repo ON otak.vec_pr_summaries (repo_slug);
```

### 6.4 Ingestion Hook — In `pr_ingestion.ts`

After `pr_condenser.ts` generates summaries:

```typescript
if (process.env.EMBED_PR_SUMMARIES !== 'false') {
  const summaryText = [
    pr.core_problem ? `Core Problem: ${pr.core_problem}` : '',
    pr.implementation_summary ? `Implementation: ${pr.implementation_summary}` : '',
    pr.key_decisions ? `Key Decisions: ${pr.key_decisions}` : '',
  ].filter(Boolean).join('\n');

  if (summaryText.length > 30) {
    const hash = md5(summaryText);
    const existing = await pool.query(
      `SELECT content_hash FROM otak.vec_pr_summaries WHERE pr_id = $1`, [pr.id]
    );
    if (!existing.rows[0] || existing.rows[0].content_hash !== hash) {
      const emb = await llamaClient.embed(summaryText); // Arctic :8081
      await pool.query(`
        INSERT INTO otak.vec_pr_summaries (pr_id, repo_slug, embedding, content_hash)
        VALUES ($1, $2, $3::halfvec, $4)
        ON CONFLICT (pr_id) DO UPDATE SET embedding = $3::halfvec, content_hash = $4
      `, [pr.id, pr.repo_slug, vectorToString(normalizeVector(emb)), hash]);
    }
  }
}
```

### 6.5 Hybrid Search Upgrade — Stage 6 Only

Upgrade ONLY stage 6 (PR search) to hybrid vector+FTS. Stage 7 (Jira search) stays FTS-only.

```typescript
// Stage 6: PR summary search — NOW hybrid
async function searchPRSummaries(queryText: string, proseEmb: number[], topK = 10) {
  const [vecResults, ftsResults] = await Promise.all([
    pool.query(`
      SELECT vps.pr_id, pr.repo_slug, pr.title, pr.core_problem,
             pr.implementation_summary, pr.key_decisions,
             1 - (vps.embedding <=> $1::halfvec) AS score
      FROM otak.vec_pr_summaries vps
      JOIN otak.pull_requests pr ON vps.pr_id = pr.id
      ORDER BY vps.embedding <=> $1::halfvec LIMIT $2
    `, [vectorToString(proseEmb), topK]),

    pool.query(`
      SELECT pr.id AS pr_id, pr.repo_slug, pr.title, pr.core_problem,
             pr.implementation_summary, pr.key_decisions,
             ts_rank_cd(fts.tsvector, plainto_tsquery('english', $1)) AS score
      FROM otak.fts_pr_summaries fts
      JOIN otak.pull_requests pr ON fts.pr_id = pr.id
      WHERE fts.tsvector @@ plainto_tsquery('english', $1)
      ORDER BY score DESC LIMIT $2
    `, [queryText, topK]),
  ]);

  return miniRRF(vecResults.rows, ftsResults.rows, 'pr_id');
}

// Stage 7: Jira summary search — stays FTS-only (unchanged)
```

### 6.6 Dual Query Embedding at Stage 2

Add prose embedding alongside code embedding (reused by stage 6 and Confluence stage):

```typescript
// Stage 2: parallel embed
const [codeEmbedding, proseEmbedding] = await Promise.all([
  llamaClient.embedCode(prefixedQuery),    // :8085, 3584d
  llamaClient.embed(query),                // :8081, 1024d — for PR vectors + Confluence
]);
```

### 6.7 Backfill

```bash
npx tsx src/scripts/backfill_pr_vectors.ts   # ~10 min for ~2000 PRs
# No Jira backfill needed
```

-----

## 7. PR→Symbol Graph Edges

During PR ingestion, create `modifies` edges from PR graph nodes to the specific code symbols changed:

```typescript
// In pr_ingestion.ts — after processing diff hunks
for (const file of pr.filesChanged) {
  const affectedNodes = await pool.query(`
    SELECT id, name, type FROM otak.graph_nodes
    WHERE file_path = $1 AND repo_slug = $2
    AND start_line <= $4 AND end_line >= $3
  `, [file.path, pr.repo_slug, file.changedStartLine, file.changedEndLine]);

  for (const node of affectedNodes.rows) {
    await graphClient.insertEdges([{
      source_id: prNodeId,
      target_id: node.id,
      relationship: 'modifies',
      metadata: JSON.stringify({
        pr_number: pr.pr_number, author: pr.author, title: pr.title,
      }),
    }]);
  }
}
```

This enables: “Which PR last modified submitOrder?” → graph traversal via `modifies` relationship.

### OME RAG Total Changes: ~2 days

```
Day A: PR vector embedding + hybrid search stage 6 + backfill
Day B: Intelligence layer feature flag + PR→symbol graph edges
```

-----

# Part C — OME Agent Project

## 8. Project Setup

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
  }
}
```

Node.js LTS >= 24.0.0, TypeScript ^5.5.0, ESM, `--experimental-strip-types`.

-----

## 9. MCP Server

Exposes ONE tool: `ome_rag_guide`. Both stdio (Windsurf direct) and HTTP transport (bearer token auth).

Windsurf sees 43 tools from RAG + 1 tool from Agent = 44 tools total.

```json
// Developer's mcp_config.json
{
  "mcpServers": {
    "ome-rag": { "url": "http://localhost:3000/mcp" },
    "ome-agent": { "url": "http://localhost:3100/mcp" }
  }
}
```

-----

## 10. RAG Client

`RAGClient` class — MCP SDK client wrapping 20+ of RAG’s 43 tools with typed methods:

`searchCode`, `searchExact`, `findSymbol`, `getCodeContext`, `readFullFile`, `batchReadFiles`, `getFileDependencies`, `suggestRelatedFiles`, `analyzeBlastRadius`, `getTicketContext`, `getProjectInfo`, `getRecentChanges`, `getFileHistory`, `getRepoConfig`, `searchArchitecture`, `analyzeGraphHealth`, `queryKnowledgeGraph`, `lookupGlossary`, `checkHallucination`, `compareImplementations`

Plus SDLC tools for Phase 2: `createPullRequest`, `generatePRDescription`, `reviewPullRequest`

**Direct DB access (read-only)** for:

- `otak.vec_pr_summaries` — PR vector search (hybrid with FTS)
- `otak.acronym_glossary` — glossary loading
- `otak.health_scores` / `otak.health_dimensions` — health warnings
- `otak.jira_tickets` / `otak.fts_jira_summaries` — Jira context by ticket key
- `otak.pr_jira_links` — PR↔Jira linkage
- `otak.pull_requests` — PR details, condensed summaries

**Direct filesystem access:** `REPOS_DIR` for full file reading.

-----

## 11. Agent Database Schema

```sql
CREATE SCHEMA IF NOT EXISTS ome_agent;

CREATE TABLE ome_agent.sessions (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  session_type TEXT NOT NULL,
  created_at TIMESTAMPTZ DEFAULT NOW(),
  last_active TIMESTAMPTZ DEFAULT NOW()
);

CREATE TABLE ome_agent.guide_invocations (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  session_id UUID REFERENCES ome_agent.sessions(id),
  query TEXT NOT NULL,
  mode TEXT NOT NULL,                    -- understand | implement | diagnose
  model_used TEXT NOT NULL,              -- qwen3-coder-next | qwen3-235b
  model_tier TEXT NOT NULL,              -- fast | deep
  rag_tools_called JSONB,
  response_confidence TEXT,
  total_ms INTEGER,
  created_at TIMESTAMPTZ DEFAULT NOW()
);

CREATE TABLE ome_agent.change_plans (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  invocation_id UUID REFERENCES ome_agent.guide_invocations(id),
  ticket_key TEXT,
  repo_slug TEXT NOT NULL,
  plan_data JSONB NOT NULL,
  status TEXT DEFAULT 'generated',       -- generated | approved | executing | applied | failed
  created_at TIMESTAMPTZ DEFAULT NOW()
);

CREATE TABLE ome_agent.model_metrics (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  model TEXT NOT NULL,
  tier TEXT NOT NULL,
  task TEXT NOT NULL,
  prompt_tokens INTEGER,
  completion_tokens INTEGER,
  latency_ms INTEGER,
  created_at TIMESTAMPTZ DEFAULT NOW()
);
CREATE INDEX idx_model_metrics_ts ON ome_agent.model_metrics (created_at);
```

-----

# Part D — ome_rag_guide Implementation

## 12. Tool Definition

```typescript
export const OME_RAG_GUIDE = {
  name: 'ome_rag_guide',
  description: `AI coding assistant. Call FIRST for any coding task.

UNDERSTAND: Explains code — entry points, execution flow, architecture, docs. Fast model (~2s).
IMPLEMENT: Generates production code changes with exact line numbers. Deep model (~45-90s).
DIAGNOSE: Analyzes errors + generates fix code. Fast analysis (~3s) + deep fix (~30s).

Orchestrates 43 internal RAG tools + dual GPU models automatically.`,

  inputSchema: {
    type: 'object',
    properties: {
      query: { type: 'string', description: 'Natural language question or instruction' },
      context: { type: 'string', description: 'Error messages, current code, conversation history' },
      current_file: { type: 'string', description: 'File open in IDE' },
      current_repo: { type: 'string', description: 'Current repository' },
      ticket_id: { type: 'string', description: 'Jira ticket ID' },
      mode: { type: 'string', enum: ['auto', 'understand', 'implement', 'diagnose'] },
    },
    required: ['query'],
  },
};
```

-----

## 13. Three Modes + Intent Classification

```typescript
export type GuideMode = 'understand' | 'implement' | 'diagnose';

export function classifyIntent(query: string, mode?: string, ticketId?: string): {
  mode: GuideMode; confidence: number;
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

  return { mode: 'understand', confidence: 0.75 };
}
```

Ambiguous (< 0.7 confidence) → resolved by Coder-Next (:8090) in 1 call, 10 tokens.

-----

## 14. Orchestration Engine

```typescript
export async function orchestrate(
  params: GuideParams,
  ctx: AgentContext
): Promise<GuideResponse> {
  const startMs = performance.now();

  // 1. CLASSIFY
  let intent = classifyIntent(params.query, params.mode, params.ticket_id);
  if (intent.confidence < 0.7) {
    intent.mode = await resolveIntentViaFastModel(params.query, ctx.llm);
    intent.confidence = 0.8;
  }

  // 2. EXTRACT query elements (symbols, paths, tickets, repos, glossary)
  const extracted = extractQueryElements(params.query, ctx.glossary);

  // 3. BUILD GATHER PLAN (which RAG tools to call)
  const plan = buildGatherPlan(intent.mode, params, extracted);

  // 4. PARALLEL GATHER (all RAG MCP calls at once)
  const gathered = await executeGatherPlan(plan, params, extracted, ctx);

  // 5. READ FULL FILES (from REPOS_DIR directly)
  const targetFiles = selectTargetFiles(gathered, intent.mode, params);
  const fullFiles = await readFilesFromDisk(targetFiles);

  // 6. FIND SIMILAR PRs (vector search — for IMPLEMENT mode)
  let similarPRs: PRResult[] = [];
  if (intent.mode === 'implement') {
    similarPRs = await searchSimilarPRs(params.query, ctx);
  }

  // 7. SPARSE TICKET ENRICHMENT (if ticket has poor description)
  if (gathered.ticketContext && isSparseTicket(gathered.ticketContext)) {
    gathered.ticketContext = await enrichSparseTicket(gathered.ticketContext, ctx);
  }

  // 8. BUILD QWEN CONTEXT
  const qwenContext = buildQwenContext(
    params, intent, extracted, gathered, fullFiles, similarPRs
  );

  // 9. GENERATE (model-routed per mode)
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

  // 10. HALLUCINATION CHECK
  response = await validateResponse(response, fullFiles);

  // 11. LOG
  await ctx.db.logInvocation({
    query: params.query, mode: intent.mode,
    modelUsed: response._modelUsed, modelTier: response._modelTier,
    ragToolsCalled: gathered._toolsCalled,
    totalMs: Math.round(performance.now() - startMs),
    responseConfidence: response.confidence.level,
  });

  return response;
}
```

-----

## 15. Gather Planner & Executor

**What gets gathered per mode:**

|RAG Tool                       |UNDERSTAND      |IMPLEMENT       |DIAGNOSE        |
|-------------------------------|----------------|----------------|----------------|
|`search_semantic_code`         |✅               |✅               |✅               |
|`search_exact_match`           |If symbols found|If symbols found|If symbols found|
|`find_symbol` (per symbol)     |✅               |✅               |✅               |
|`get_code_context` (per symbol)|✅ top 2         |✅ top 2         |✅ top 1         |
|`analyze_blast_radius`         |—               |✅               |—               |
|`get_ticket_context`           |If ticket_id    |✅ if ticket_id  |—               |
|`get_project_info`             |If current_repo |✅               |If current_repo |
|`get_recent_changes`           |—               |—               |✅               |
|`search_architecture`          |✅               |✅               |—               |
|`get_repo_config`              |—               |✅               |✅               |
|`analyze_graph_health`         |—               |✅               |—               |
|`read_full_file` (current)     |If file_hint    |If file_hint    |If file_hint    |
|`get_file_dependencies`        |If file_hint    |If file_hint    |If file_hint    |
|`suggest_related_files`        |If file_hint    |If file_hint    |—               |
|`lookup_glossary`              |If terms found  |If terms found  |If terms found  |

All calls execute in **parallel** via `Promise.allSettled`. Failures are logged but don’t crash the pipeline.

-----

## 16. Mode 1: UNDERSTAND (Fast Model)

Uses **Coder-Next (:8090)** — instant explanations.

Prompt includes: entry point (from `get_code_context` graph data), execution flow (from `getCallChain`), architecture pattern (from `projectInfo.tech_stack` + decorators), code evidence (top 5 files, 2500 chars each), dependencies (from graph), glossary, health warnings.

Returns: `explanation` (Coder-Next generated text), `evidence[]`, `entryPoint`, `executionFlow`, `architecturePattern`, `documentation[]`, `glossary[]`, `followUpSuggestions[]`.

**Target latency: <5 seconds.**

-----

## 17. Mode 2: IMPLEMENT (Deep Model)

Two-step generation:

**Step 1 — Convention detection via Fast model (:8090):**
Reads 2-3 reference files, extracts team conventions (300 tokens, ~1s).

**Step 2 — Code generation via Deep model (:8091):**
Full files with line numbers, Jira context, similar PRs, conventions, health warnings → generates `CodeChange[]` with exact `startLine/endLine`, `currentCode`, `proposedCode`, content hash for agent verification.

Returns: `summary`, `changes[]` (CodeChange), `newFiles[]`, `configChanges[]`, `targetFiles[]`, `referenceFiles[]`, `similarPRs[]`, `conventions[]`, `blastRadius`, `testsToUpdate[]`, `healthWarnings[]`, `affectedSymbols[]`, `executionOrder[]`, `preChecks[]`, `postChecks[]`.

**Target latency: <90 seconds.**

### CodeChange Schema (Agent-Ready)

```typescript
export interface CodeChange {
  id: string;                         // "change-001"
  file: string;                       // Verified on disk
  repo: string;
  action: 'modify' | 'insert' | 'replace' | 'delete';
  description: string;
  reason: string;
  currentCode: {
    startLine: number;
    endLine: number;
    content: string;
    contentHash: string;              // MD5 — agent verifies before applying
  } | null;
  proposedCode: string;               // 235B generated production code
  priority: number;
  dependencies: string[];             // Change IDs that must happen first
  affectedSymbols: string[];
}
```

-----

## 18. Mode 3: DIAGNOSE (Fast + Deep)

**Step 1 — Analysis via Fast model (:8090):**
Error area identification, reverse call chain, recent changes, config context → root cause analysis (~3s).

**Step 2 — Fix generation via Deep model (:8091):**
If high-confidence diagnosis, generates fix code as `CodeChange[]` (~30s).

Returns: `analysis` (text), `likelyLocation[]`, `reverseCallChain[]`, `suggestedFix[]` (CodeChange, nullable), `recentChanges[]`, `configContext[]`, `evidence[]`.

-----

## 19. Sparse Ticket Enrichment

Instead of embedding Jira tickets (which have poor descriptions), the Agent detects sparse tickets and enriches them by climbing the Epic→Story hierarchy:

```typescript
// src/guide/analyze/ticket_enricher.ts

export function isSparseTicket(ticket: TicketContext): boolean {
  const descLength = (
    (ticket.businessGoal || '').length +
    (ticket.specificTask || '').length +
    (ticket.acceptanceCriteria || '').length
  );
  return descLength < 80; // Less than ~2 sentences of real content
}

export async function enrichSparseTicket(
  ticket: TicketContext,
  ctx: AgentContext
): Promise<TicketContext> {

  // ── Strategy 1: Read parent Epic/Story for context ──
  if (ticket.parentKey) {
    try {
      const parent = await ctx.rag.getTicketContext(ticket.parentKey);
      if (parent.businessGoal && parent.businessGoal.length > ticket.businessGoal?.length) {
        ticket.enrichedFromParent = true;
        ticket.parentContext = {
          key: ticket.parentKey,
          summary: parent.summary,
          businessGoal: parent.businessGoal,
          specificTask: parent.specificTask,
        };
      }
    } catch {}
  }

  if (ticket.epicKey && ticket.epicKey !== ticket.parentKey) {
    try {
      const epic = await ctx.rag.getTicketContext(ticket.epicKey);
      if (epic.businessGoal) {
        ticket.epicContext = {
          key: ticket.epicKey,
          summary: epic.summary,
          businessGoal: epic.businessGoal,
        };
      }
    } catch {}
  }

  // ── Strategy 2: Find past PRs for this ticket via pr_jira_links ──
  const linkedPRs = await ctx.ragDb.query(`
    SELECT pr.title, pr.core_problem, pr.implementation_summary
    FROM otak.pr_jira_links pjl
    JOIN otak.pull_requests pr ON pjl.pr_id = pr.id
    WHERE pjl.issue_key = $1
    ORDER BY pr.created_at DESC LIMIT 3
  `, [ticket.ticketId]);

  if (linkedPRs.rows.length > 0) {
    ticket.linkedPRContext = linkedPRs.rows.map(r => ({
      title: r.title,
      coreProblem: r.core_problem,
      implementationSummary: r.implementation_summary,
    }));
  }

  // ── Strategy 3: Find linked Confluence docs ──
  const linkedDocs = await ctx.ragDb.query(`
    SELECT cp.title, cp.breadcrumb, LEFT(cp.body, 500) AS snippet
    FROM otak.jira_confluence_links jcl
    JOIN otak.confluence_pages cp ON jcl.page_id = cp.page_id
    WHERE jcl.issue_key = $1
  `, [ticket.ticketId]);

  if (linkedDocs.rows.length > 0) {
    ticket.linkedDocumentation = linkedDocs.rows.map(r => ({
      title: r.title,
      breadcrumb: r.breadcrumb,
      snippet: r.snippet,
    }));
  }

  // ── Strategy 4: Use Coder-Next to infer intent from whatever context we have ──
  if (isSparseTicket(ticket) && (ticket.parentContext || ticket.linkedPRContext || ticket.linkedDocumentation)) {
    const routing = routeToModel('implement', 'sparse_ticket_enrich');
    const prompt = `A developer needs to work on Jira ticket "${ticket.ticketId}".
The ticket description is sparse: "${ticket.summary}"

Additional context:
${ticket.parentContext ? `Parent story (${ticket.parentContext.key}): ${ticket.parentContext.summary}\nBusiness Goal: ${ticket.parentContext.businessGoal}` : ''}
${ticket.epicContext ? `Epic (${ticket.epicContext.key}): ${ticket.epicContext.summary}\nGoal: ${ticket.epicContext.businessGoal}` : ''}
${ticket.linkedPRContext ? `Past PRs:\n${ticket.linkedPRContext.map(pr => `- "${pr.title}": ${pr.coreProblem}`).join('\n')}` : ''}
${ticket.linkedDocumentation ? `Docs:\n${ticket.linkedDocumentation.map(d => `- ${d.title}: ${d.snippet?.slice(0, 200)}`).join('\n')}` : ''}

Based on this context, write a clear 2-3 sentence description of what this ticket requires.`;

    try {
      const response = await ctx.llm.chat(prompt, routing);
      if (response.content.length > 30) {
        ticket.inferredDescription = response.content.trim();
      }
    } catch {}
  }

  return ticket;
}
```

**This is more valuable than Jira embedding** because:

- It only runs when needed (sparse ticket detected)
- It uses the hierarchy (`parent_key`, `epic_key`) — which embedding can’t do
- It uses the 3B-active Coder-Next to REASON about context from multiple sources
- It enriches a KNOWN ticket — no semantic search needed

-----

## 20. Similar PR Discovery (Vector Search)

The Agent directly queries `otak.vec_pr_summaries` for semantically similar PRs:

```typescript
// src/guide/analyze/similar_prs.ts

export async function searchSimilarPRs(
  query: string,
  ctx: AgentContext
): Promise<PRResult[]> {

  // Get prose embedding for the query via RAG's Arctic (:8081)
  // The Agent can call this directly or reuse from gather phase
  // For now, use RAG's embed endpoint or call search that returns PR results

  // Direct DB query against vec_pr_summaries + fts_pr_summaries
  const [vecResults, ftsResults] = await Promise.all([
    ctx.ragDb.query(`
      SELECT vps.pr_id, pr.repo_slug, pr.pr_number, pr.title,
             pr.core_problem, pr.implementation_summary, pr.key_decisions,
             1 - (vps.embedding <=> (
               SELECT embedding FROM otak.query_embedding_cache
               WHERE query_hash = md5($1) LIMIT 1
             )) AS score
      FROM otak.vec_pr_summaries vps
      JOIN otak.pull_requests pr ON vps.pr_id = pr.id
      WHERE vps.embedding <=> (
        SELECT embedding FROM otak.query_embedding_cache
        WHERE query_hash = md5($1) LIMIT 1
      ) < 0.7
      ORDER BY score DESC LIMIT 5
    `, [query]),

    ctx.ragDb.query(`
      SELECT pr.id AS pr_id, pr.repo_slug, pr.pr_number, pr.title,
             pr.core_problem, pr.implementation_summary, pr.key_decisions,
             ts_rank_cd(fts.tsvector, plainto_tsquery('english', $1)) AS score
      FROM otak.fts_pr_summaries fts
      JOIN otak.pull_requests pr ON fts.pr_id = pr.id
      WHERE fts.tsvector @@ plainto_tsquery('english', $1)
      ORDER BY score DESC LIMIT 5
    `, [query]),
  ]);

  return miniRRF(vecResults.rows, ftsResults.rows, 'pr_id').slice(0, 3);
}
```

**Note:** If the query embedding isn’t in `query_embedding_cache` (because RAG’s hybrid_search hasn’t been called for this exact query), the Agent falls back to FTS-only PR search. The Agent can also call RAG’s `search_semantic_code` which populates the cache as a side effect.

-----

## 21. Hallucination Prevention

Three-layer verification:

1. **File existence** — every `change.file` checked against `REPOS_DIR` filesystem
1. **Line content** — `change.currentCode` fuzzy-matched (>75% word overlap) against actual file content at stated lines; auto-corrects line numbers if content found elsewhere in file
1. **Symbol existence** — `change.affectedSymbols` checked against `otak.graph_nodes`

Content hash (MD5) generated for every verified `currentCode` block — used by future agent for safe application.

-----

# Part E — Agent-Ready Architecture

## 22. Phase 2 Extension Path

The `ImplementResponse` is designed for mechanical execution:

|Field                              |Agent Uses For                           |
|-----------------------------------|-----------------------------------------|
|`changes[].currentCode.contentHash`|Verify file hasn’t changed before editing|
|`changes[].proposedCode`           |Exact code to write                      |
|`executionOrder[]`                 |Apply changes in this sequence           |
|`preChecks[]`                      |Run before applying                      |
|`postChecks[]`                     |Run after applying                       |

```
Phase 1 (NOW):  Developer → Windsurf → ome_rag_guide → plan → developer applies
Phase 2 (NEXT): Developer → ome_rag_guide → agent applies → runs tests → creates PR
Phase 3 (FUTURE): Jira webhook → ome_rag_guide → agent → auto PR → reviewer
```

Phase 2 needs: `execute_plan` tool, git worktree management, RAG’s `create_pull_request` integration. The `execution/` directory is stubbed for this.

-----

# Part F — Implementation

## 23. Implementation Schedule

```
Week 1: Infrastructure (Days 1-5)

  Day 1: GPU Setup — Dual Model Deployment
    ├── Build llama.cpp with CUDA for H100
    ├── Download Qwen3-Coder-Next Q5_K_M
    ├── Download Qwen3-235B-A22B Q4_K_XL
    ├── Deploy Coder-Next :8090 (verify 50+ tok/s, 64K ctx, 3 slots)
    ├── Deploy 235B :8091 with MoE offload (verify ~20 tok/s, 32K ctx)
    ├── Systemd services (ome-agent-fast, ome-agent-deep, ome-agent)
    └── Health check both models

  Day 2: Agent Project Setup + MCP Server
    ├── Initialize ome-agent project
    ├── DualLLMClient with routing + graceful fallback
    ├── MCP server (stdio + HTTP, bearer auth)
    ├── ome_rag_guide tool registration
    └── Test: Windsurf sees ome_rag_guide

  Day 3: RAG Client (MCP Consumer) + Direct Access
    ├── RAGClient class — 20+ typed MCP tool wrappers
    ├── Read-only otak connection (PR vectors, health, glossary, Jira FTS)
    ├── File reader (REPOS_DIR)
    ├── Connection management + reconnect
    └── Test: call search_semantic_code, find_symbol via MCP

  Day 4: Agent DB + Intent Classifier
    ├── ome_agent schema (sessions, invocations, change_plans, model_metrics)
    ├── intent_classifier.ts (understand/implement/diagnose)
    ├── Query element extractor (symbols, paths, tickets, repos)
    ├── Ambiguous intent resolution via fast model
    └── 50+ intent tests

  Day 5: OME RAG Changes (PR Embedding + Intelligence Layer Refactor)
    ├── vec_pr_summaries table + HNSW index
    ├── pr_ingestion.ts embed hook
    ├── Backfill script for existing PRs
    ├── Hybrid search stage 6 upgrade (vector + FTS)
    ├── Dual query embedding at stage 2
    ├── PR→symbol modifies graph edges
    ├── Intelligence layer feature flag (lightweight mode)
    └── Verify: "how was resilience implemented" finds circuit breaker PRs

Week 2: Core Logic (Days 6-10)

  Day 6: Gather Planner + Executor + Context Builder
    ├── gather_planner.ts (plan per mode)
    ├── gather_executor.ts (parallel RAG MCP calls)
    ├── qwen_context_builder.ts
    ├── Entry point detection (from graph context)
    ├── Execution flow assembly
    └── Pattern detection

  Day 7: UNDERSTAND Mode
    ├── understand.ts (fast model)
    ├── Explanation prompt
    ├── Evidence builder
    ├── Follow-up generator
    ├── Glossary expansion
    └── Test: 5 golden UNDERSTAND queries

  Day 8: IMPLEMENT Mode — Core
    ├── implement.ts (deep model for code gen)
    ├── Convention detection (fast model)
    ├── Code generation prompt (numbered files + line refs)
    ├── JSON response parser
    └── Similar PR injection

  Day 9: IMPLEMENT Mode — Assembly + Ticket Enrichment
    ├── Change plan assembly (CodeChange[], NewFile[])
    ├── Blast radius integration
    ├── Test identification
    ├── Sparse ticket enrichment (§19)
    ├── Pre/post checklist generation
    └── Content hash generation

  Day 10: DIAGNOSE Mode
    ├── diagnose.ts (fast analysis + deep fix)
    ├── Error location identification
    ├── Reverse call chain integration
    ├── Recent changes + config context
    └── Fix code generation (deep model)

Week 3: Quality + Testing (Days 11-15)

  Day 11: Hallucination Prevention + Integration
    ├── 3-layer verification (file, lines, symbols)
    ├── Line number fuzzy matching + correction
    ├── Confidence computation
    ├── Wire orchestrator end-to-end
    └── Error handling (models down, RAG down)

  Day 12: Testing — UNDERSTAND + DIAGNOSE
    ├── 10+ UNDERSTAND golden queries
    ├── 5+ DIAGNOSE golden queries
    ├── Explanation quality assessment
    ├── Evidence accuracy
    ├── Model routing correctness
    └── Latency: UNDERSTAND <5s target

  Day 13: Testing — IMPLEMENT
    ├── 10+ IMPLEMENT golden tasks
    ├── Code generation quality (compiles? patterns followed?)
    ├── Line number accuracy
    ├── Convention detection accuracy
    ├── Blast radius correctness
    ├── Sparse ticket enrichment quality
    ├── Hallucination rate (<5% target)
    └── Latency: IMPLEMENT <90s target

  Day 14: Cross-Model Consistency
    ├── Same query to UNDERSTAND → same facts regardless of IDE model
    ├── Same task to IMPLEMENT → same code plan
    ├── Verify graceful degradation (deep model down → fast fallback)
    └── Load test (3 concurrent UNDERSTAND + 1 IMPLEMENT)

  Day 15: Polish + Documentation
    ├── Edge cases (huge files, no graph, empty repo, both models down)
    ├── Performance profiling
    ├── PROJECT_ARCHITECTURE.md for OME Agent
    ├── README + deployment guide
    └── Team walkthrough

Week 4 (Optional): Phase 2 Prep

  Days 16-18:
    ├── execute_plan tool skeleton
    ├── Content hash verification before apply
    ├── Git worktree management prototype
    ├── Integration with RAG's create_pull_request
    └── Pipeline state tracking via ome_agent.change_plans
```

-----

## 24. Configuration

```bash
# ═══ OME Agent Server ═══
AGENT_PORT=3100
AGENT_HOST=0.0.0.0
AGENT_MCP_API_KEY=<bearer-token>

# ═══ Dual Model ═══
FAST_MODEL_URL=http://127.0.0.1:8090      # Qwen3-Coder-Next
DEEP_MODEL_URL=http://127.0.0.1:8091      # Qwen3-235B-A22B
FAST_MODEL_TIMEOUT_MS=30000               # 30s
DEEP_MODEL_TIMEOUT_MS=120000              # 120s

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

# ═══ OME RAG Flags (set in RAG's .env) ═══
EMBED_PR_SUMMARIES=true                    # Enable PR vector embedding
INTELLIGENCE_LAYER_MODE=lightweight        # Simplified enrichment
```

-----

## 25. File Structure

```
ome-agent/
├── package.json
├── tsconfig.json
├── .env / .env.example
├── README.md
│
├── src/
│   ├── server/
│   │   ├── mcp_server.ts                 # MCP server (1 tool)
│   │   ├── http_entry.ts                 # Express + HTTP transport
│   │   └── stdio_entry.ts               # Stdio for Windsurf
│   │
│   ├── llm/
│   │   ├── dual_client.ts                # DualLLMClient (fast + deep)
│   │   ├── model_router.ts               # Route tasks to fast/deep
│   │   └── health.ts                     # Model health checks
│   │
│   ├── rag_client/
│   │   ├── mcp_client.ts                 # RAGClient (20+ typed wrappers)
│   │   ├── rag_db.ts                     # Read-only otak queries
│   │   ├── file_reader.ts                # REPOS_DIR filesystem access
│   │   └── types.ts                      # RAG response types
│   │
│   ├── guide/
│   │   ├── tool_definition.ts            # ome_rag_guide schema
│   │   ├── orchestrator.ts               # 11-phase brain
│   │   ├── intent_classifier.ts          # understand / implement / diagnose
│   │   ├── gather_planner.ts             # Which RAG tools per mode
│   │   ├── gather_executor.ts            # Parallel RAG call execution
│   │   ├── query_extractor.ts            # Extract symbols, paths, tickets
│   │   ├── qwen_context_builder.ts       # Assemble LLM prompt context
│   │   │
│   │   ├── modes/
│   │   │   ├── understand.ts             # Fast model explanation
│   │   │   ├── implement.ts              # Deep model code generation
│   │   │   └── diagnose.ts               # Fast analysis + deep fix
│   │   │
│   │   ├── analyze/
│   │   │   ├── entry_point.ts            # Entry point detection
│   │   │   ├── flow_tracer.ts            # Execution flow assembly
│   │   │   ├── pattern_detector.ts       # CSR, CHoS, Event-Driven
│   │   │   ├── convention_extractor.ts   # Fast model convention detection
│   │   │   ├── error_locator.ts          # Error area identification
│   │   │   ├── ticket_enricher.ts        # Sparse ticket enrichment
│   │   │   └── similar_prs.ts            # PR vector + FTS search
│   │   │
│   │   ├── verify/
│   │   │   ├── hallucination_guard.ts    # 3-layer verification
│   │   │   ├── confidence.ts             # Confidence computation
│   │   │   └── content_hash.ts           # MD5 for agent verification
│   │   │
│   │   └── types.ts                      # All response types
│   │
│   ├── db/
│   │   ├── agent_pool.ts                 # ome_agent schema
│   │   ├── agent_db.ts                   # Typed queries
│   │   └── migrations/001_initial.sql
│   │
│   ├── execution/                        # Phase 2 (stubbed)
│   │   ├── plan_executor.ts
│   │   ├── worktree_manager.ts
│   │   └── pr_creator.ts
│   │
│   └── types/shared.ts
│
├── test/
│   ├── intent_classifier.test.ts
│   ├── model_router.test.ts
│   ├── understand_golden.test.ts
│   ├── implement_golden.test.ts
│   ├── diagnose_golden.test.ts
│   ├── ticket_enricher.test.ts
│   ├── hallucination.test.ts
│   └── rag_client.test.ts
│
├── scripts/
│   ├── setup_gpu.sh                      # Build llama.cpp + download both models
│   ├── setup_db.sh                       # Create ome_agent schema
│   ├── test_fast_model.ts                # Smoke test Coder-Next
│   ├── test_deep_model.ts                # Smoke test 235B
│   └── benchmark_models.ts              # Compare fast vs deep
│
└── systemd/
    ├── ome-agent.service
    ├── ome-agent-fast.service
    └── ome-agent-deep.service
```

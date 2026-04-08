# OME Agent — Consolidated Implementation Plan (Final)

> **Version:** 4.0 — Incorporates all design discussions
> **Project:** OME Agent — Agentic coding assistant consuming OME RAG via MCP
> **Key Decisions:** Chunk+SCIP native retrieval (no disk reads), three-tier LLM routing, agentic retrieval loop, Plan→Execute→Verify pipeline, separate implementation and test phases, Agent communicates with RAG via MCP only (no direct DB), cross-repo via SCIP MCP tools
> **GPU:** NVIDIA H100 — 93.6 GB VRAM
> **Models:** Qwen2.5-1.5B (CPU) + Coder-Next (GPU) + Qwen3.5-122B (GPU). 235B not deployed.
> **Effort:** ~18 days

-----

## Table of Contents

### Part A — Architecture

1. [Three-Tier Model Architecture](#1-three-tier-model-architecture)
1. [GPU Memory Layout](#2-gpu-memory-layout)
1. [Model Deployment](#3-model-deployment)
1. [Task-to-Model Routing](#4-task-to-model-routing)

### Part B — New RAG MCP Tools

1. [New Chunk + SCIP MCP Tools](#5-new-chunk--scip-mcp-tools)
1. [Improved Tool Descriptions for Windsurf](#6-improved-tool-descriptions-for-windsurf)

### Part C — Agentic Retrieval Loop

1. [Retrieval Architecture — Chunk + SCIP Native](#7-retrieval-architecture)
1. [Retrieval Loop Mechanics](#8-retrieval-loop-mechanics)
1. [RAG MCP Tool Usage in Loop](#9-rag-mcp-tool-usage-in-loop)
1. [Response Processors — Signal Extraction](#10-response-processors)
1. [File Context Assembly via MCP](#11-file-context-assembly-via-mcp)
1. [Cross-Repo Retrieval](#12-cross-repo-retrieval)
1. [Context Accumulator](#13-context-accumulator)

### Part D — Plan → Execute → Verify Pipeline

1. [Pipeline Overview](#14-pipeline-overview)
1. [Phase 1: Blueprint Planning](#15-phase-1-blueprint-planning)
1. [Phase 2: Per-File Code Generation](#16-phase-2-per-file-code-generation)
1. [Phase 3: Test Generation (Separate Phase)](#17-phase-3-test-generation)
1. [Phase 4: Verification](#18-phase-4-verification)
1. [Phase 5: Self-Correction](#19-phase-5-self-correction)
1. [Handling 20+ File Modifications](#20-handling-20-file-modifications)

### Part E — Implementation

1. [ome_rag_guide Tool Definition](#21-ome_rag_guide-tool-definition)
1. [Full Orchestration Flow](#22-full-orchestration-flow)
1. [Configuration](#23-configuration)
1. [Implementation Schedule](#24-implementation-schedule)
1. [File Structure](#25-file-structure)

-----

# Part A — Architecture

## 1. Three-Tier Model Architecture

Every task in the Agent maps to exactly one model tier based on what the task ACTUALLY requires — classification, code-aware reasoning, or deep code generation.

```
┌─────────────────────────────────────────────────────────────────┐
│ TIER 1: Qwen2.5-1.5B (:8084, CPU, ~50-100ms)                   │
│                                                                 │
│ Classification and routing — no code understanding needed.      │
│ Outputs: 10-50 tokens. Latency: instant. Concurrency: unlimited.│
│                                                                 │
│ Tasks:                                                          │
│   • Intent classification (understand / implement / diagnose)   │
│   • Complexity routing (simple / medium / complex)              │
│   • Model routing (which tier handles the next task)            │
│   • Sufficiency checks (has enough context? yes/no)             │
│   • Sparse ticket detection (sparse / adequate)                 │
│   • Action validation (is this retrieval action redundant?)     │
│   • File role classification (target / reference / type / skip) │
└─────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────┐
│ TIER 2: Qwen3-Coder-Next (:8090, GPU, 3 slots, ~2-5s)          │
│                                                                 │
│ Code-aware reasoning — understands code but doesn't generate    │
│ production implementations. Fast, parallel.                     │
│                                                                 │
│ Tasks:                                                          │
│   • Retrieval reasoning (which MCP tools to call next)          │
│   • Convention detection (extract patterns from reference code) │
│   • Test generation (follow existing test patterns)             │
│   • Verification (cross-file contract checking)                 │
│   • Simple UNDERSTAND answers (single symbol/file explanation)  │
│   • Sparse ticket enrichment (infer requirements from context)  │
│   • File confirmation decisions (which files to assemble)       │
└─────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────┐
│ TIER 3: Qwen3.5-122B-A10B (:8091, GPU, 2 slots, ~8-15s)        │
│                                                                 │
│ Deep reasoning + quality code generation. Used when 3B active   │
│ parameters aren't enough for the task's complexity.             │
│                                                                 │
│ Tasks:                                                          │
│   • Blueprint planning (structural reasoning over all files)    │
│   • Per-file code generation (production-quality proposedCode)  │
│   • Complex UNDERSTAND (architecture, multi-repo, business flow)│
│   • DIAGNOSE root cause analysis (subtle bugs)                  │
│   • DIAGNOSE fix generation                                     │
│   • Retry on Tier 2 failure (fallback for specific files)       │
└─────────────────────────────────────────────────────────────────┘

NOT DEPLOYED:
  Qwen3-235B-A22B — 22B active is overkill when the architecture
  provides perfect context via chunk+SCIP. 122B with 10B active
  is sufficient. Saves ~32GB CPU RAM and gains a parallel slot.
```

### Why Three Tiers

|                       |1.5B (Tier 1)  |Coder-Next (Tier 2)|122B (Tier 3)   |
|-----------------------|---------------|-------------------|----------------|
|Active params          |1.5B           |3B                 |10B             |
|Speed                  |~100 tok/s     |~50 tok/s          |~30-40 tok/s    |
|Parallel slots         |Unlimited (CPU)|3                  |2               |
|Context window         |32K            |64K                |64K+            |
|Code understanding     |None           |Good               |Excellent       |
|Code generation quality|Poor           |Adequate           |Production-grade|
|Cost per call          |~0             |Low                |Medium          |
|Latency                |~50-100ms      |~2-5s              |~8-15s          |

**The principle:** Use the cheapest tier that can do the job. If 1.5B can classify intent in 50ms, don’t burn a Coder-Next slot for 2 seconds. If Coder-Next can detect conventions, don’t wait 10 seconds for 122B.

-----

## 2. GPU Memory Layout

```
H100 — 93.6 GB VRAM
──────────────────────────────────
Qwen3-Coder-Next Q5_K_M         ~30 GB  (full on GPU)
Qwen3.5-122B-A10B Q4_K_M        ~28 GB  (non-MoE on GPU, experts → CPU)
KV Cache:
  Coder-Next 64K ctx × 3 slots   ~9 GB
  122B 64K ctx × 2 slots          ~12 GB
──────────────────────────────────
Total:                           ~79 GB
Headroom:                        ~15 GB

CPU (1 TB RAM):
  Qwen2.5-1.5B:                  ~3 GB
  122B MoE expert layers:        ~50 GB
  OME RAG models:                ~25 GB (nomic, arctic, reranker, Qwen2.5-7B)
  Free:                          ~920 GB
```

-----

## 3. Model Deployment

### 3.1 Tier 1 — Qwen2.5-1.5B (:8084)

Already running on CPU. No changes.

```bash
# Existing deployment — unchanged
./llama-server \
  --model /models/qwen2.5-1.5b-instruct-q4_k_m.gguf \
  --host 127.0.0.1 --port 8084 \
  --ctx-size 32768 --parallel 4 \
  --threads 8
```

### 3.2 Tier 2 — Coder-Next (:8090)

```bash
./llama-server \
  --model /models/qwen3-coder-next/Qwen3-Coder-Next-UD-Q5_K_M.gguf \
  --host 127.0.0.1 --port 8090 \
  --n-gpu-layers 99 --ctx-size 65536 --parallel 3 \
  --threads 8 --flash-attn --jinja \
  --temp 0.3 --top-p 0.95 --top-k 40 --min-p 0.01
```

### 3.3 Tier 3 — Qwen3.5-122B (:8091)

```bash
./llama-server \
  --model /models/qwen3.5-122b/Qwen3.5-122B-A10B-UD-Q4_K_M.gguf \
  --host 127.0.0.1 --port 8091 \
  --n-gpu-layers 99 -ot ".ffn_.*_exps.=CPU" \
  --ctx-size 65536 --parallel 2 \
  --threads 32 --flash-attn --jinja \
  --temp 0.3 --top-p 0.95 --top-k 20 --min-p 0.0
```

### 3.4 Systemd Services

```
ome-agent-tier1.service  → 1.5B :8084 (already running)
ome-agent-tier2.service  → Coder-Next :8090
ome-agent-tier3.service  → 122B :8091 (depends on tier2)
ome-agent.service        → Node.js MCP server (depends on all models)
```

-----

## 4. Task-to-Model Routing

```typescript
// src/llm/model_router.ts

export type ModelTier = 'tier1' | 'tier2' | 'tier3';

export function routeToTier(task: AgentTask): ModelTier {
  switch (task) {
    // ── Tier 1: 1.5B — Classification, routing ──
    case 'intent_classify':        return 'tier1';
    case 'complexity_route':       return 'tier1';
    case 'model_route':            return 'tier1';
    case 'sufficiency_check':      return 'tier1';
    case 'sparse_ticket_detect':   return 'tier1';
    case 'action_validate':        return 'tier1';
    case 'file_role_classify':     return 'tier1';

    // ── Tier 2: Coder-Next — Code-aware reasoning ──
    case 'retrieval_reason':       return 'tier2';
    case 'convention_detect':      return 'tier2';
    case 'test_generate':          return 'tier2';
    case 'verify_contracts':       return 'tier2';
    case 'understand_simple':      return 'tier2';
    case 'ticket_enrich':          return 'tier2';
    case 'file_confirm':           return 'tier2';

    // ── Tier 3: 122B — Deep reasoning + code generation ──
    case 'blueprint_plan':         return 'tier3';
    case 'code_generate':          return 'tier3';
    case 'understand_complex':     return 'tier3';
    case 'diagnose_analyze':       return 'tier3';
    case 'diagnose_fix':           return 'tier3';
    case 'retry_failed_file':      return 'tier3';

    default:                       return 'tier2';
  }
}

export function getTierConfig(tier: ModelTier): TierConfig {
  switch (tier) {
    case 'tier1': return {
      url: 'http://127.0.0.1:8084',
      maxTokens: 50,
      temperature: 0.0,
      timeout: 5_000,
    };
    case 'tier2': return {
      url: 'http://127.0.0.1:8090',
      maxTokens: 2_000,
      temperature: 0.3,
      timeout: 30_000,
    };
    case 'tier3': return {
      url: 'http://127.0.0.1:8091',
      maxTokens: 8_000,
      temperature: 0.3,
      timeout: 120_000,
    };
  }
}
```

### UNDERSTAND Mode — Complexity-Based Routing

```typescript
// The 1.5B tier decides which model handles the explanation

function routeUnderstandComplexity(
  retrievalResult: RetrievalResult
): 'understand_simple' | 'understand_complex' {
  const filesFound = retrievalResult.filesDiscovered;
  const reposInvolved = new Set(retrievalResult.repoSlugs).size;

  // Simple: 1-2 files, single repo, specific symbol question
  if (filesFound <= 2 && reposInvolved === 1) return 'understand_simple';  // → Tier 2

  // Complex: multiple files, cross-repo, architecture question
  return 'understand_complex';  // → Tier 3
}
```

-----

# Part B — New RAG MCP Tools

## 5. New Chunk + SCIP MCP Tools

The Agent talks to RAG ONLY through MCP tools. These new tools expose the chunk+SCIP infrastructure so the Agent never queries the database directly.

### 5.1 New Tools to Add to OME RAG

```typescript
// ── Chunk Assembly Tools ──

{
  name: 'assemble_file_context',
  description: `Build rich code context for a file from indexed data. Returns:
- File skeleton (imports, class declaration, decorators, fields)
- Matched chunk bodies with exact line numbers (syntactically complete CAST units)
- Unmatched method signatures (one line each — shows what else is in the file)
- SCIP-resolved caller/callee signatures from OTHER files
- Type definitions for all referenced types
- Interface contracts the class must satisfy
- Cross-repo references

This gives MORE context than reading a raw file because it includes
caller signatures and type definitions from across the codebase.
Use this instead of reading files directly.`,
  inputSchema: {
    filePath: { type: 'string', description: 'File path within the repo' },
    repoSlug: { type: 'string', description: 'Repository slug' },
    matchedChunkIds: { type: 'array', items: { type: 'string' },
      description: 'Chunk IDs from search results to include in full (optional — if empty, includes highest-centrality chunks)' },
    role: { type: 'string', enum: ['target', 'reference', 'type'],
      description: 'target: include full matched chunks with line numbers. reference: include method signatures + first-8-line previews. type: include full type definition.' },
  },
}

{
  name: 'get_file_skeleton',
  description: `Get the structural outline of a file without method bodies.
Returns imports, class declaration with decorators, field declarations,
and all method signatures with line numbers and centrality scores.
Use when you need to understand a file's structure without reading its full content.
Much cheaper than assemble_file_context — use for reference files or initial exploration.`,
  inputSchema: {
    filePath: { type: 'string' },
    repoSlug: { type: 'string' },
  },
}

{
  name: 'get_chunks_for_symbol',
  description: `Get the CAST chunk(s) containing a specific symbol.
Returns syntactically complete code blocks with exact start/end line numbers
and contextual headers (LLM-generated descriptions of what the code does).
Use when you know the symbol name and want its implementation.`,
  inputSchema: {
    symbol: { type: 'string', description: 'Symbol name (function, class, method)' },
    repoSlug: { type: 'string', description: 'Repository (optional — searches all if omitted)' },
  },
}

// ── SCIP Tools ──

{
  name: 'get_scip_callers',
  description: `Find all functions/methods that call a given symbol, with their
full signatures. Includes callers from OTHER files and OTHER repos.
Use this to understand who depends on a symbol before modifying it.
Returns: symbol name, signature, file path, repo slug, relationship type.`,
  inputSchema: {
    symbol: { type: 'string' },
    repoSlug: { type: 'string', description: 'Optional — filter to specific repo' },
    limit: { type: 'number', description: 'Max results (default 15)' },
  },
}

{
  name: 'get_scip_callees',
  description: `Find all functions/methods that a given symbol calls, with their
full signatures. Shows what APIs and dependencies the symbol uses.
Use this to understand what a function depends on.
Returns: symbol name, signature, file path, repo slug.`,
  inputSchema: {
    symbol: { type: 'string' },
    repoSlug: { type: 'string', description: 'Optional' },
    limit: { type: 'number', description: 'Max results (default 15)' },
  },
}

{
  name: 'get_scip_type_definition',
  description: `Resolve a type, interface, or enum by name. Returns the full
definition (all fields, methods, extends/implements) from wherever it is
defined in the codebase — including other repos.
Use when code references a type like OrderDTO or BondEntity and you need
to know its shape.`,
  inputSchema: {
    typeName: { type: 'string', description: 'Type/interface/enum name (e.g., OrderDTO, BondEntity)' },
    repoSlug: { type: 'string', description: 'Optional — searches all repos if omitted' },
  },
}

{
  name: 'get_scip_implementations',
  description: `Find all classes/types that implement a given interface.
Returns each implementor's file path, repo, and method signatures.
Use when modifying an interface to understand what breaks,
or when looking for reference implementations of a pattern.`,
  inputSchema: {
    interfaceName: { type: 'string' },
    repoSlug: { type: 'string', description: 'Optional' },
  },
}

{
  name: 'get_scip_references',
  description: `Find every location where a symbol is referenced across the
entire codebase. Returns file path, line number, role (definition/reference/import),
and repo slug. Use for impact analysis — how widely used is this symbol?`,
  inputSchema: {
    symbol: { type: 'string' },
    repoSlug: { type: 'string', description: 'Optional — searches all repos if omitted' },
    limit: { type: 'number', description: 'Max results (default 30)' },
  },
}
```

### 5.2 Tools NOT Used by Guide (Removed from Guide Loop)

```
REMOVED from guide retrieval:
  get_health_score        — nightly report for architects, not useful for code gen
  get_health_dimension    — same
  diagnose_search         — internal diagnostic
  check_hallucination     — replaced by disk verification in verify phase
```

-----

## 6. Improved Tool Descriptions for Windsurf

Clear descriptions help Windsurf’s Claude choose the right tool. Updated descriptions for existing tools:

```typescript
// Existing tools — updated descriptions

{
  name: 'search_semantic_code',
  description: `Search for code using natural language. Finds relevant functions,
classes, and code blocks across all 47 repositories using semantic understanding
and exact identifier matching combined.

Best for: "how does authentication work", "bond trading order processing",
"kafka consumer retry handling", or any natural language code question.

Returns code snippets with file paths, line numbers, relevance scores,
and graph context (callers, callees, centrality).`,
}

{
  name: 'find_symbol',
  description: `Look up a specific function, class, or type by its exact name.
Searches across ALL repositories. Returns the symbol's file location,
line numbers, signature, decorators, and type (function/class/interface/type).

Best for: "find OrderService", "where is BondDTO defined", "locate submitOrder".
Use this when you know the exact name. Use search_semantic_code when you
don't know the name and need to search by description.`,
}

{
  name: 'get_code_context',
  description: `Get the structural relationships for a symbol — who calls it,
what it calls, how important it is (centrality score), what decorators
it has, and what it imports. Shows how a symbol fits in the architecture.

Best for: "what calls submitOrder", "what does AuthService depend on",
"how central is PaymentProcessor to the system".`,
}

{
  name: 'analyze_blast_radius',
  description: `Impact analysis — what would break if you change this symbol?
Returns direct dependents, transitive dependents (dependencies of dependencies),
affected test files, and cross-repo consumers.

Best for: "what's the impact of changing submitOrder",
"is it safe to modify BondValidator".
Use BEFORE making changes to shared code. Not needed for simple additions.`,
}

{
  name: 'suggest_related_files',
  description: `Find files that are architecturally coupled to a given file —
they always change together in git history (temporal coupling / co-change).
These files likely need to change together even if they don't import each other.

Best for: "what else needs to change if I modify OrderService",
"related files for bond-search-controller".`,
}

{
  name: 'read_full_file',
  description: `Read the raw content of a file from disk. Returns the complete
file as plain text.

Use ONLY when you need the exact current disk content (e.g., config files,
package.json, Dockerfile) or when you need to verify content that might
have changed since last indexing.

For CODE files, prefer assemble_file_context instead — it gives you the
relevant code sections PLUS caller/callee signatures, type definitions,
and structural context that raw file content doesn't include.`,
}
```

-----

# Part C — Agentic Retrieval Loop

## 7. Retrieval Architecture

### Core Principle: No Disk Reads During Retrieval

```
                    ┌──────────────────────┐
                    │    Agent Query        │
                    └──────────┬───────────┘
                               │
                    ┌──────────▼───────────┐
                    │  Agentic Loop        │
                    │                      │
                    │  Seed → Reason →     │
                    │  Fetch → Accumulate  │
                    │  → Check → Repeat    │
                    │                      │
                    │  ALL data comes from │
                    │  RAG MCP tools:      │
                    │  • search_semantic   │
                    │  • find_symbol       │
                    │  • assemble_file_ctx │
                    │  • get_scip_callers  │
                    │  • get_scip_type_def │
                    │  • etc.              │
                    │                      │
                    │  NO pool.query()     │
                    │  NO fs.readFile()    │
                    └──────────┬───────────┘
                               │
                    ┌──────────▼───────────┐
                    │  RAG MCP Server      │
                    │  (:3000)             │
                    │                      │
                    │  Queries:            │
                    │  • chunks table      │
                    │  • vec_chunks        │
                    │  • scip_* tables     │
                    │  • graph (IGraphClient)│
                    │  • file_imports      │
                    │  • projects          │
                    │  • REPOS_DIR (only   │
                    │    for read_full_file│
                    │    on config files)  │
                    └──────────────────────┘
```

### What the Agent Sees Per File (via `assemble_file_context`)

```
┌─────────────────────────────────────────────────────────┐
│ FILE SKELETON (from file_imports + GraphNode + SCIP)     │
│   imports (file_imports table)                           │
│   @Service @Transactional (GraphNode.decorators)         │
│   class OrderService { (GraphNode)                       │
│     private orderRepo: OrderRepository (GraphNode.fields)│
│                                                          │
│ MATCHED CHUNKS (from search → chunks table)              │
│   submitOrder() — full body with line numbers (CAST)     │
│   validateOrder() — full body with line numbers (CAST)   │
│                                                          │
│ UNMATCHED METHOD SIGNATURES (from scip_symbol_info)      │
│   cancelOrder(orderId: string): void  // L81-115         │
│   getOrderStatus(id: string): OrderDTO  // L116-140      │
│   mapToDTO(order: Order): OrderDTO  // L201-250          │
│                                                          │
│ SCIP ENRICHMENT                                          │
│   Called by:                                             │
│     OrderController.handleSubmit(...): ResponseEntity     │
│     BatchProcessor.processOrders(...): BatchResult        │
│   Calls into:                                            │
│     BondValidator.validate(bondId: string): void          │
│     OrderRepository.save(order: Order): Order             │
│   Types used:                                            │
│     OrderDTO { orderId, bondId, quantity, status, ... }   │
│   Implements:                                            │
│     IOrderProcessor.submitOrder(req): OrderDTO            │
│   Cross-repo:                                            │
│     ms-settlements/SettlementService.settleOrder(...)     │
└─────────────────────────────────────────────────────────┘
```

This is ~950 tokens. A full file read would be ~2,800 tokens. **And this version includes caller signatures and type definitions that a full file read can never provide.**

-----

## 8. Retrieval Loop Mechanics

```
PHASE 0: SEED (no LLM, parallel, ~300ms)
  │
  │  Tier 1 (1.5B): classify intent → understand/implement/diagnose
  │
  ├── search_semantic_code(query)              ← always
  ├── find_symbol(symbols from query)          ← if symbols detected
  ├── get_ticket_context(ticket_id)            ← if ticket provided
  ├── get_project_info(current_repo)           ← if repo known
  ├── lookup_glossary(domain terms)            ← if terms detected
  └── assemble_file_context(current_file)      ← if file open in IDE
  │
  ▼ Process responses → extract signals → populate accumulator

PHASE 1: AGENTIC LOOP (max 5 iterations)
  │
  │  ┌─────────────────────────────────────────────────────────┐
  │  │ Each iteration:                                         │
  │  │                                                         │
  │  │ 1. Tier 1 (1.5B): sufficiency check                    │
  │  │    → if clearly sufficient: EXIT                        │
  │  │    → if clearly insufficient: CONTINUE                  │
  │  │    → if ambiguous: ask Tier 2                           │
  │  │                                                         │
  │  │ 2. Tier 2 (Coder-Next): reasoning                      │
  │  │    Input:  accumulated context summary + signals        │
  │  │            + candidate files + available MCP tools      │
  │  │    Output: actions (MCP tool calls) + file decisions    │
  │  │            OR sufficient=true                           │
  │  │                                                         │
  │  │ 3. Execute MCP tool calls (parallel)                    │
  │  │                                                         │
  │  │ 4. Process responses → extract signals                  │
  │  │    (programmatic — no LLM needed for most tools)        │
  │  │                                                         │
  │  │ 5. Accumulate results                                   │
  │  │                                                         │
  │  │ 6. Check hard limits → continue or exit                 │
  │  └─────────────────────────────────────────────────────────┘
  │
  ▼ When sufficient or limits reached

PHASE 2: POST-LOOP (Tier 2, ~2-3s)
  ├── Convention detection (from reference file chunks)
  ├── Cross-repo context assembly (from accumulated signals)
  ├── Tier 1: route complexity → decide Tier 2 or Tier 3 for generation
  └── Output: accumulated context → pipeline input
```

-----

## 9. RAG MCP Tool Usage in Loop

### What the Reasoning Model (Tier 2) Can Call

The reasoning prompt presents these MCP tools as available actions:

```
DISCOVERY:
  search_semantic_code { query, source? }  — natural language code search
  search_exact_match { query }             — exact identifier match
  search_architecture { query }            — find architectural patterns
  search_confluence { query }              — search documentation

SYMBOL + GRAPH:
  find_symbol { symbol }                   — locate symbol definition
  get_code_context { symbol }              — callers, callees, centrality
  get_blast_radius { symbol }              — impact analysis
  get_file_dependencies { filePath }       — imports and importedBy
  get_related_files { filePath }           — co-change temporal coupling
  get_cross_repo_dependencies { symbol }   — cross-repo consumers/providers

CONTEXT ASSEMBLY (replaces file reading):
  assemble_file_context { filePath, repoSlug, role }
    — builds skeleton + matched chunks + SCIP enrichment from database
  get_file_skeleton { filePath, repoSlug }
    — imports + class + fields + method signatures only (cheaper)
  get_chunks_for_symbol { symbol }
    — get specific chunk containing a symbol

SCIP:
  get_scip_callers { symbol }              — who calls this (with signatures)
  get_scip_callees { symbol }              — what this calls (with signatures)
  get_scip_type_definition { typeName }    — resolve type/interface definition
  get_scip_implementations { interfaceName } — find all implementations
  get_scip_references { symbol }           — every reference location

SDLC:
  get_ticket_context { ticketId }          — Jira ticket details
  search_similar_prs { query }             — past PRs with similar changes
  get_recent_changes { filePath }          — recent commits

PROJECT:
  get_project_info { repo }                — tech stack, framework
  get_repo_config { repo }                 — application.yml, Dockerfile
  lookup_glossary { terms }                — domain term expansion

CONTENT (rare — config files only):
  read_full_file { filePath }              — raw file from disk
    Use ONLY for config files (application.yml, Dockerfile, package.json).
    For code files, ALWAYS use assemble_file_context instead.

CONTROL:
  sufficient                               — stop gathering
```

### Mode-Specific Tool Relevance

```
IMPLEMENT:
  Primary:   search_semantic_code → assemble_file_context (target)
             → get_scip_type_definition → get_scip_callers
  Secondary: search_similar_prs → get_blast_radius → get_related_files
  Rare:      get_repo_config (if config change) → search_confluence (business rules)

UNDERSTAND:
  Primary:   search_semantic_code → assemble_file_context or get_file_skeleton
             → get_code_context → get_scip_callers/callees
  Secondary: search_confluence → search_architecture
  Rare:      get_cross_repo_dependencies (if cross-repo question)

DIAGNOSE:
  Primary:   search_semantic_code → assemble_file_context (error location)
             → get_scip_callers (trace to error) → get_recent_changes
  Secondary: get_repo_config (if config-related) → find_symbol (error symbols)
  Rare:      search_similar_prs (past fix patterns)
```

-----

## 10. Response Processors

Every MCP tool response goes through a processor that extracts structured signals. No LLM needed for signal extraction — it’s programmatic.

```typescript
// src/guide/retrieval/response_processors.ts

export interface ProcessedResponse {
  discoveredFiles: DiscoveredFile[];
  resolvedSymbols: ResolvedSymbol[];
  crossRepoRefs: CrossRepoRef[];
  signals: RetrievalSignal[];
  tokenEstimate: number;
}

export type RetrievalSignal =
  | { type: 'cross_repo_consumer'; consumerRepo: string; symbol: string }
  | { type: 'shared_type_referenced'; typeName: string; inFiles: string[] }
  | { type: 'high_centrality_symbol'; symbol: string; score: number }
  | { type: 'recent_change_detected'; filePath: string; daysAgo: number }
  | { type: 'temporal_coupling'; fileA: string; fileB: string; coChangeCount: number }
  | { type: 'sparse_ticket'; ticketId: string }
  | { type: 'interface_implementors'; interfaceName: string; count: number }
  | { type: 'no_results'; query: string; suggestion: string }
  | { type: 'convention_pattern'; pattern: string; exampleFile: string }
  | { type: 'test_coverage_gap'; filePath: string };
```

### Signal Extraction Per Tool (Programmatic, No LLM)

|Tool                   |Signals Extracted                                                                                                                     |
|-----------------------|--------------------------------------------------------------------------------------------------------------------------------------|
|`search_semantic_code` |`shared_type_referenced` (from content), `high_centrality_symbol`, `no_results` (if poor scores), `cross_repo` (if results span repos)|
|`find_symbol`          |`cross_repo` (if symbol in multiple repos), `interface_implementors` (if type is interface)                                           |
|`get_code_context`     |`high_centrality_symbol`, `cross_repo_consumer` (if callers from other repos)                                                         |
|`get_ticket_context`   |`sparse_ticket` (if description < 80 chars)                                                                                           |
|`suggest_related_files`|`temporal_coupling` (if coChangeCount ≥ 5)                                                                                            |
|`analyze_blast_radius` |`cross_repo_consumer`, `test_coverage_gap` (if no affected tests)                                                                     |
|`get_recent_changes`   |`recent_change_detected` (if change within 7 days)                                                                                    |
|`search_similar_prs`   |`convention_pattern` (from implementation summaries)                                                                                  |
|`get_file_dependencies`|`shared_type_referenced` (from imported types), `cross_repo` (if cross-repo imports)                                                  |

-----

## 11. File Context Assembly via MCP

When the reasoning model decides a file needs context, it calls `assemble_file_context` MCP tool. RAG handles the assembly internally — querying chunks, SCIP, graph, file_imports — and returns the assembled result.

### Decision Flow for File Context

```
Reasoning model sees candidate file from search results
  │
  ├── Is it a type/interface definition?
  │   YES → get_scip_type_definition (cheapest — just the definition)
  │
  ├── Do I need to understand its structure?
  │   YES → get_file_skeleton (imports + signatures, no method bodies)
  │
  ├── Do I need to modify this file? (IMPLEMENT target)
  │   YES → assemble_file_context(role='target')
  │          (skeleton + matched chunks with line numbers + SCIP enrichment)
  │
  ├── Do I need to follow its patterns? (reference)
  │   YES → assemble_file_context(role='reference')
  │          (skeleton + method previews, less detail than target)
  │
  └── Not relevant → skip
```

-----

## 12. Cross-Repo Retrieval

Cross-repo awareness comes naturally through SCIP MCP tools. No special cross-repo mode needed.

```
Developer: "Add pagination to bond search"

Loop iteration 1:
  search_semantic_code → finds BondSearchService in ms-bonds-trading

Loop iteration 2:
  assemble_file_context(BondSearchService) → SCIP enrichment shows:
    Called by: BondSearchController (same repo) ← expected
    Called by: ms-portfolio/PortfolioService.getBondHoldings() ← CROSS-REPO
    Type: BondSearchResult defined in ms-shared-types ← CROSS-REPO

  Signals emitted:
    cross_repo_consumer { consumerRepo: 'ms-portfolio', symbol: 'getBondHoldings' }
    shared_type_referenced { typeName: 'BondSearchResult', inFiles: [...] }

Loop iteration 3 (reasoning model sees signals):
  "Cross-repo consumer detected. If I add pagination, ms-portfolio needs
   to handle it too. Let me check the type definition."
  → get_scip_type_definition('BondSearchResult')      ← resolves from ms-shared-types
  → get_file_skeleton(ms-portfolio/PortfolioService)   ← understand the consumer
  → SUFFICIENT
```

Cross-repo works because:

- `find_symbol` searches ALL repos by default
- `get_scip_callers` returns cross-repo callers
- `assemble_file_context` SCIP enrichment includes cross-repo references
- Signals flag cross-repo dependencies for the reasoning model

-----

## 13. Context Accumulator

Tracks everything gathered across iterations. Provides compact summaries for reasoning and full data for generation.

```typescript
// src/guide/retrieval/context_accumulator.ts

export class ContextAccumulator {
  /** Assembled file contexts — ready for execution prompt */
  readonly assembledFiles: Map<string, AssembledFileContext> = new Map();

  /** File skeletons (for files not fully assembled) */
  readonly skeletons: Map<string, FileSkeleton> = new Map();

  /** Resolved symbols */
  readonly symbols: Map<string, ResolvedSymbol> = new Map();

  /** Resolved type definitions */
  readonly types: Map<string, TypeDefinition> = new Map();

  /** Search queries already executed (dedup) */
  readonly searchQueries: Set<string> = new Set();

  /** Signals from all tool responses */
  readonly signals: RetrievalSignal[] = [];

  /** Cross-repo references detected */
  readonly crossRepoRefs: CrossRepoRef[] = [];

  /** Ticket context */
  ticketContext: TicketContext | null = null;

  /** Similar PRs */
  similarPRs: PRResult[] = [];

  /** Conventions (detected from reference files) */
  conventions: string | null = null;

  /** Project info per repo */
  projectInfo: Map<string, any> = new Map();

  /** Tool call count */
  totalToolCalls: number = 0;

  /** Estimated tokens accumulated */
  totalTokens: number = 0;

  /** Context window budget */
  readonly contextBudget: number;

  /** Action log */
  readonly actionLog: ActionLogEntry[] = [];

  get remainingBudget(): number {
    return Math.max(0, this.contextBudget - this.totalTokens);
  }

  /** Is a specific action redundant? */
  isRedundant(action: RetrievalAction): boolean { ... }

  /** Absorb a processed MCP tool response */
  absorb(toolName: string, processed: ProcessedResponse): void { ... }

  /** Compact summary for Tier 2 reasoning prompt */
  toReasoningSummary(): string { ... }

  /** Full context for generation pipeline */
  toGenerationInput(query: string, mode: GuideMode): GenerationInput { ... }
}
```

-----

# Part D — Plan → Execute → Verify Pipeline

## 14. Pipeline Overview

After the retrieval loop completes, the accumulated context feeds into a structured generation pipeline.

```
Retrieval Loop Output
  │
  ▼
┌──────────────────────────────────────────────────────────────┐
│ PHASE 1: BLUEPRINT (Tier 3, single call, ~10-15s)            │
│                                                              │
│ Input:  Query + ticket + ALL file skeletons (SCIP signatures)│
│         + conventions + similar PRs + cross-repo context     │
│ Output: Per-file change instructions, execution order,       │
│         shared contracts, cross-file rules                   │
│                                                              │
│ Uses skeletons (not full content) — fits in one call even    │
│ for 40+ files. For 60+ files, hierarchical two-level plan.   │
└────────────────────────┬─────────────────────────────────────┘
                         │ Blueprint
                         ▼
┌──────────────────────────────────────────────────────────────┐
│ PHASE 2: CODE GENERATION (Tier 3, per-file, 2 parallel)      │
│                                                              │
│ One call per target file. Each call receives:                │
│   • Blueprint instruction for THIS file                      │
│   • Assembled file context (matched chunks + SCIP)           │
│   • Completed changes from dependency files (actual sigs)    │
│   • Conventions + ticket                                     │
│                                                              │
│ Processes files in dependency order:                          │
│   Shared types first → services → controllers → tests        │
│                                                              │
│ Output per file: CodeChange[]                                │
│                                                              │
│ 20 files ÷ 2 parallel slots = ~80 seconds                   │
└────────────────────────┬─────────────────────────────────────┘
                         │ All CodeChange[]
                         ▼
┌──────────────────────────────────────────────────────────────┐
│ PHASE 3: TEST GENERATION (Tier 2, per-file, 3 parallel)      │
│                                                              │
│ Separate phase — runs AFTER all implementation completes.    │
│                                                              │
│ Each call receives:                                          │
│   • The ACTUAL proposedCode from Phase 2 (not the plan)      │
│   • Existing test patterns (from test file chunks)           │
│   • Test framework config (jest/vitest/JUnit from project)   │
│   • What's already tested (test_coverage table via MCP)      │
│                                                              │
│ Output: TestChange[] (new test methods or updated assertions)│
│                                                              │
│ 20 test files ÷ 3 parallel slots = ~35 seconds              │
└────────────────────────┬─────────────────────────────────────┘
                         │ All CodeChange[] + TestChange[]
                         ▼
┌──────────────────────────────────────────────────────────────┐
│ PHASE 4: VERIFICATION (Tier 2 + deterministic + disk read)   │
│                                                              │
│ 4a. Deterministic checks (no LLM):                           │
│     • Line numbers match disk content (IMPLEMENT only)       │
│     • No overlapping changes in same file                    │
│     • Blueprint adherence (every planned file has changes)   │
│     • Auto-fix line number shifts                            │
│                                                              │
│ 4b. Contract verification (Tier 2):                          │
│     • Type compatibility across changes                      │
│     • Import consistency                                     │
│     • Interface contract satisfaction                        │
│     • No duplicate logic across files                        │
│                                                              │
│ 4c. Disk verification (IMPLEMENT only):                      │
│     • Read target files from REPOS_DIR                       │
│     • Verify content hash matches                            │
│     • Correct line numbers if shifted                        │
│     THIS IS THE ONLY DISK READ IN THE ENTIRE PIPELINE.       │
│                                                              │
│ Output: verified changes OR correction instructions          │
└────────────────────────┬─────────────────────────────────────┘
                         │ If corrections needed
                         ▼
┌──────────────────────────────────────────────────────────────┐
│ PHASE 5: SELF-CORRECTION (Tier 3, targeted, max 2 loops)     │
│                                                              │
│ Re-generate ONLY the specific files that failed verification.│
│ Each call receives the original assembled context +          │
│ the specific issue + the failed change.                      │
│                                                              │
│ If still fails after 2 attempts → return with warning.       │
└──────────────────────────────────────────────────────────────┘
```

-----

## 15. Phase 1: Blueprint Planning

### Single Call for Normal Tasks (≤40 files)

```
Input to Tier 3 (122B):
  • Query + ticket + conventions
  • File skeletons for ALL target + reference files
    (one-line method signatures from SCIP — ~15-30 lines per file)
  • Cross-repo context
  • Similar PRs

Output:
  {
    summary: "...",
    fileChangePlans: [
      { filePath, changeDescription, methodsToModify, methodsToAdd,
        importsToAdd, dependsOnFiles, contractsToHonor }
    ],
    newFiles: [
      { path, purpose, exports, consumedBy }
    ],
    sharedContracts: [
      { name: "OrderDTO", change: "Add status field", affectedFiles: [...] }
    ],
    executionOrder: ["shared/types.ts", "order/service.ts", "order/controller.ts"],
    crossFileRules: ["Import from @shared, not relative paths"]
  }
```

### Hierarchical Planning for Large Tasks (40+ files)

```
Level 1 (Tier 3, 1 call):
  Input: 60 file paths + contextual_header (one sentence each) ~1,500 tokens
  Output: Group into 4-5 work packages by dependency cluster
          + cross-package contracts

Level 2 (Tier 3, 4-5 calls parallel on 2 slots):
  Input per package: 12-15 file skeletons + cross-package contracts
  Output: Per-file change instructions within package
```

-----

## 16. Phase 2: Per-File Code Generation

### One File Per Call, Dependency Order

```typescript
// Process files in blueprint's executionOrder

for (const filePath of blueprint.executionOrder) {
  // Wait for dependencies to complete
  const deps = blueprint.fileChangePlans
    .find(p => p.filePath === filePath)?.dependsOnFiles ?? [];
  await waitForCompletion(deps);

  // Build prompt for this file
  const prompt = buildFileGenerationPrompt({
    blueprintInstruction: blueprint.fileChangePlans.find(p => p.filePath === filePath),
    assembledContext: accumulator.assembledFiles.get(filePath),
    completedDependencySignatures: getCompletedSignatures(deps),
    sharedContracts: blueprint.sharedContracts,
    crossFileRules: blueprint.crossFileRules,
    conventions: accumulator.conventions,
    ticket: accumulator.ticketContext,
  });

  // Generate on Tier 3 (122B)
  const result = await tier3.generate(prompt);
  completedChanges.set(filePath, parseCodeChanges(result));
}
```

### What Each Call Receives

```
1. Blueprint instruction for THIS file:
   "Add pagination support to findBonds(). Accept Pageable param,
    return Page<BondDTO> instead of List<BondDTO>."

2. Assembled file context (from assemble_file_context MCP):
   • Matched chunks with line numbers (the actual code to modify)
   • SCIP caller signatures (so generated code is compatible)
   • SCIP callee signatures (so generated code uses correct APIs)
   • Type definitions (so generated code uses correct types)

3. Completed dependency changes (actual generated signatures):
   "BondDTO now has these fields: ... (generated in previous file)"
   "BondSearchRequest now accepts pageSize and pageNumber"

4. Conventions + ticket + cross-file rules
```

### Parallel Execution with Dependency Ordering

```
Blueprint executionOrder: [types.ts, service.ts, controller.ts, test.ts]
                          [dto.ts]  [repository.ts]

Dependency graph:
  types.ts → no deps         → SLOT 1 (immediate)
  dto.ts → no deps           → SLOT 2 (immediate)
  service.ts → types, dto    → SLOT 1 (after types + dto complete)
  repository.ts → types, dto → SLOT 2 (after types + dto complete)
  controller.ts → service    → SLOT 1 (after service)

Timeline:
  T=0s:   Slot 1: types.ts  |  Slot 2: dto.ts
  T=8s:   Slot 1: service.ts | Slot 2: repository.ts
  T=16s:  Slot 1: controller.ts | Slot 2: (idle)
  T=24s:  Complete (5 files in 24s, not 40s sequential)
```

-----

## 17. Phase 3: Test Generation (Separate Phase)

Runs AFTER all implementation completes. Uses Tier 2 (Coder-Next, 3 parallel slots) because test generation follows patterns — it’s mechanical.

### Why Separate

1. **Different context** — needs the actual `proposedCode` from Phase 2, not the blueprint
1. **Different patterns** — needs existing test file chunks, not production code patterns
1. **Different model** — Tier 2 is sufficient (follow existing test patterns)
1. **Independence** — implementation failures don’t block test generation for other files
1. **Parallelism** — uses Coder-Next’s 3 slots while 122B sits idle

### What Each Test Generation Call Receives

```
1. The ACTUAL proposedCode for the source file (from Phase 2)

2. Existing test patterns:
   • Chunks from the existing test file (if it exists)
   • Test framework config (from get_project_info MCP)

3. What's already tested:
   • test_coverage mappings (from get_code_context MCP)

4. The new method signatures being added/modified
```

### Output

```typescript
interface TestChange {
  testFilePath: string;          // e.g., src/test/OrderServiceTest.java
  sourceFilePath: string;        // e.g., src/order/OrderService.java
  action: 'modify' | 'create';
  changes: CodeChange[];         // Same format as implementation changes
}
```

-----

## 18. Phase 4: Verification

### 4a. Deterministic Checks (No LLM)

```
• Line numbers: chunk content_hash vs disk content at stated lines
• Overlapping changes: no two changes touch same lines in same file
• Blueprint adherence: every fileChangePlan has corresponding CodeChange
• Content hash: MD5 of current disk content for agent verification
```

### 4b. Contract Verification (Tier 2)

```
• Type compatibility: caller param types match callee signatures
• Import consistency: same import style across all changes
• Interface contracts: modified class still satisfies interface
• No duplicate logic: hash-normalized method bodies across files
```

### 4c. Disk Verification (IMPLEMENT Only)

**The ONLY disk read in the entire pipeline.** After code generation, before returning response:

```
For each target file in CodeChange[]:
  1. Read file from REPOS_DIR
  2. Compare stated line content with disk content
  3. If match: generate contentHash from disk (authoritative)
  4. If shifted: fuzzy-find content, correct line numbers
  5. If missing: flag warning (file changed significantly since bootstrap)
```

-----

## 19. Phase 5: Self-Correction

If verification finds errors:

```
1. Collect failed changes with specific issue descriptions
2. Re-generate ONLY those files on Tier 3 (122B)
   Input: assembled context + failed change + issue + other passed changes
3. Re-verify
4. Max 2 correction loops
5. If still failing: return with warning, let developer decide
```

-----

## 20. Handling 20+ File Modifications

### Complete Timeline for 20-File IMPLEMENT

```
Tier 1 (1.5B):      intent classify                        50ms
Tier 1 (1.5B):      complexity route → complex              50ms

RETRIEVAL LOOP (~6-8s):
  Seed:              4 MCP calls parallel                    300ms
  Iteration 1:       Tier 2 reasoning + 3 MCP calls          3s
  Iteration 2:       Tier 2 reasoning + 4 MCP calls          3s
  Iteration 3:       Tier 1 sufficiency → sufficient          50ms
  Post-loop:         Tier 2 convention detection              2s

BLUEPRINT (Tier 3, 1 call):
  Input: 20 skeletons + ticket + conventions                 ~12s

CODE GENERATION (Tier 3, 2 parallel slots):
  20 files in dependency order                               ~80s
  (10 rounds × ~8s per file)

TEST GENERATION (Tier 2, 3 parallel slots):
  20 test files parallel                                     ~35s
  (7 rounds × ~5s per file)

VERIFICATION:
  Deterministic checks                                       500ms
  Tier 2 contract verification                               3s
  Disk verification (20 files)                               500ms

SELF-CORRECTION (if needed, ~10% of requests):
  2 files × Tier 3                                           ~16s

────────────────────────────────────────────────────
TOTAL:  ~140s without correction (~2.3 minutes)
        ~156s with correction (~2.6 minutes)

vs OLD approach: ~350s+ (~6+ minutes)
```

### For 40+ Files (Hierarchical Planning)

```
Level 1 plan: Tier 3, 1 call                                ~10s
Level 2 plan: Tier 3, 4 calls on 2 slots                    ~20s
Code gen: 40 files on 2 slots                                ~160s
Tests: 40 files on 3 slots                                   ~70s
Verify + correct                                             ~20s
────────────────────────────────────────────────────
TOTAL: ~285s (~4.7 minutes)
```

-----

# Part E — Implementation

## 21. ome_rag_guide Tool Definition

```typescript
export const OME_RAG_GUIDE = {
  name: 'ome_rag_guide',
  description: `AI coding assistant. Call for any coding task involving the codebase.

UNDERSTAND: Explains code — entry points, execution flow, architecture, business logic.
  Simple questions (~2s). Complex architecture (~15s).

IMPLEMENT: Generates production code changes with exact line numbers.
  Small changes (~30s). Large multi-file (~2-3 min). Includes test generation.

DIAGNOSE: Analyzes errors, finds root cause, generates fix code.
  Analysis (~5s). Fix generation (~30s).

Automatically orchestrates retrieval from 47 repositories, understands
cross-repo dependencies, follows team conventions, and generates
tests alongside implementation.`,

  inputSchema: {
    type: 'object',
    properties: {
      query: { type: 'string', description: 'Natural language question or instruction' },
      context: { type: 'string', description: 'Error messages, stacktraces, conversation context' },
      current_file: { type: 'string', description: 'File currently open in IDE' },
      current_repo: { type: 'string', description: 'Current repository slug' },
      ticket_id: { type: 'string', description: 'Jira ticket ID (e.g., BMT-1234)' },
      mode: { type: 'string', enum: ['auto', 'understand', 'implement', 'diagnose'] },
    },
    required: ['query'],
  },
};
```

-----

## 22. Full Orchestration Flow

```typescript
// src/guide/orchestrator.ts

export async function orchestrate(
  params: GuideParams,
  ctx: AgentContext
): Promise<GuideResponse> {

  // ── 1. CLASSIFY (Tier 1) ──
  const intent = await classifyIntent(params, ctx);       // 1.5B, ~50ms

  // ── 2. AGENTIC RETRIEVAL LOOP ──
  const retrieval = await runRetrievalLoop(               // Tier 1 + Tier 2
    params, intent.mode, ctx
  );

  // ── 3. ROUTE COMPLEXITY (Tier 1) ──
  const complexity = await routeComplexity(               // 1.5B, ~50ms
    intent.mode, retrieval
  );

  // ── 4. GENERATE based on mode + complexity ──
  let response: GuideResponse;

  switch (intent.mode) {
    case 'understand':
      if (complexity === 'simple') {
        response = await generateSimpleUnderstand(retrieval, ctx);   // Tier 2
      } else {
        response = await generateComplexUnderstand(retrieval, ctx);  // Tier 3
      }
      break;

    case 'implement':
      response = await runImplementPipeline(retrieval, ctx);
      // Pipeline internally handles:
      // Phase 1: Blueprint (Tier 3)
      // Phase 2: Per-file code gen (Tier 3, parallel)
      // Phase 3: Test gen (Tier 2, parallel)
      // Phase 4: Verification (Tier 2 + deterministic + disk)
      // Phase 5: Self-correction (Tier 3, if needed)
      break;

    case 'diagnose':
      response = await runDiagnosePipeline(retrieval, ctx);
      // Phase 1: Root cause analysis (Tier 3)
      // Phase 2: Fix generation (Tier 3)
      // Phase 3: Verification (Tier 2 + disk)
      break;
  }

  // ── 5. LOG ──
  await logInvocation(params, intent, retrieval, response, ctx);

  return response;
}
```

-----

## 23. Configuration

```bash
# ═══ Model Endpoints ═══
TIER1_URL=http://127.0.0.1:8084          # Qwen2.5-1.5B
TIER2_URL=http://127.0.0.1:8090          # Coder-Next
TIER3_URL=http://127.0.0.1:8091          # Qwen3.5-122B
TIER1_TIMEOUT_MS=5000
TIER2_TIMEOUT_MS=30000
TIER3_TIMEOUT_MS=120000

# ═══ RAG MCP ═══
OME_RAG_URL=http://localhost:3000/mcp
MCP_API_KEY=<bearer-token>

# ═══ Agent Server ═══
AGENT_PORT=3100
AGENT_HOST=0.0.0.0
AGENT_MCP_API_KEY=<bearer-token>

# ═══ Retrieval Loop ═══
RETRIEVAL_MAX_ITERATIONS=5
RETRIEVAL_MAX_TOOL_CALLS=25
RETRIEVAL_MAX_ACTIONS_PER_ITERATION=4
RETRIEVAL_MIN_REMAINING_BUDGET=5000

# ═══ Pipeline ═══
IMPLEMENT_HIERARCHICAL_THRESHOLD=40     # Files above this → two-level planning
MAX_CORRECTION_ATTEMPTS=2
ENABLE_TEST_GENERATION=true

# ═══ Verification ═══
DISK_VERIFY_ENABLED=true                # Verify line numbers against disk
REPOS_DIR=/repos

# ═══ Agent DB ═══
AGENT_DATABASE_URL=postgresql://ome_agent:***@localhost:5432/omedb
```

-----

## 24. Implementation Schedule

```
Week 1: Infrastructure + RAG Tools (Days 1-5)
═══════════════════════════════════════════════

Day 1: GPU Setup — Dual Model Deployment
  ├── Build llama.cpp with CUDA for H100
  ├── Download Qwen3-Coder-Next Q5_K_M
  ├── Download Qwen3.5-122B-A10B Q4_K_M
  ├── Deploy Coder-Next :8090 (verify 50+ tok/s, 64K ctx, 3 slots)
  ├── Deploy 122B :8091 with MoE offload (verify ~30-40 tok/s, 64K ctx, 2 slots)
  ├── Systemd services
  └── Health check all 3 tiers

Day 2: New RAG MCP Tools — Chunk Assembly
  ├── assemble_file_context tool (skeleton + chunks + SCIP)
  ├── get_file_skeleton tool (imports + class + method sigs)
  ├── get_chunks_for_symbol tool
  ├── Internal ChunkAssembler class in RAG
  │   ├── Query chunks table for file content
  │   ├── Query file_imports for imports
  │   ├── Query GraphNode for class/decorators/fields
  │   ├── Query scip_symbol_info for method signatures
  │   └── Format assembled result
  └── Test: assemble context for 5 real files, compare with full file read

Day 3: New RAG MCP Tools — SCIP
  ├── get_scip_callers tool
  ├── get_scip_callees tool
  ├── get_scip_type_definition tool
  ├── get_scip_implementations tool
  ├── get_scip_references tool
  ├── Internal SCIP query functions in RAG
  └── Test: verify SCIP results match IGraphClient for known symbols

Day 4: Updated Tool Descriptions + Agent Project Setup
  ├── Update descriptions for existing 43 MCP tools (clearer, with examples)
  ├── Remove health score tools from guide-accessible list
  ├── Update read_full_file description (prefer assemble_file_context for code)
  ├── Initialize ome-agent project (package.json, tsconfig, .env)
  ├── Three-tier LLM client (DualLLMClient → TripleLLMClient)
  ├── Model router (task → tier mapping)
  └── MCP server skeleton (stdio + HTTP, 1 tool: ome_rag_guide)

Day 5: RAG Client + Agent DB
  ├── RAGClient class — typed MCP tool wrappers for all tools
  │   (including new chunk/SCIP tools)
  ├── Agent DB schema (sessions, invocations, change_plans, metrics)
  ├── Intent classifier (Tier 1)
  ├── Complexity router (Tier 1)
  └── Test: call assemble_file_context + get_scip_callers via MCP from Agent

Week 2: Agentic Retrieval Loop (Days 6-9)
═══════════════════════════════════════════

Day 6: Response Processors + Signal System
  ├── ProcessedResponse type + RetrievalSignal types
  ├── Per-tool processors:
  │   processSemanticSearch, processFindSymbol, processCodeContext,
  │   processTicketContext, processRelatedFiles, processBlastRadius,
  │   processFileHistory, processPRSearch, processFileDependencies,
  │   processAssembledContext
  ├── Signal extraction (programmatic, no LLM)
  └── Test: process mock responses, verify signals extracted correctly

Day 7: Context Accumulator + Reasoning Prompt
  ├── ContextAccumulator class
  │   ├── Store assembled contexts, symbols, types, signals
  │   ├── Dedup guard (isRedundant)
  │   ├── toReasoningSummary() for Tier 2
  │   └── toGenerationInput() for pipeline
  ├── Reasoning prompt builder
  │   ├── Accumulated context summary
  │   ├── Candidate files (from search signals)
  │   ├── Available MCP tools as actions
  │   ├── Cross-repo context section
  │   ├── Sufficiency criteria per mode
  │   └── Output format (actions + file decisions)
  └── Reasoning response parser

Day 8: Loop Controller
  ├── runRetrievalLoop()
  │   ├── Seed phase (parallel MCP calls)
  │   ├── Iteration loop (Tier 1 sufficiency → Tier 2 reasoning → MCP calls)
  │   ├── Parallel action execution via pLimit
  │   ├── Signal-based cross-repo detection
  │   ├── Hard limits (max iterations, max tools, token budget)
  │   └── Post-loop enrichment (conventions via Tier 2)
  ├── Integration with orchestrator.ts
  └── Test: run loop on 5 golden queries, verify iteration count + tool calls

Day 9: Retrieval Testing + Tuning
  ├── Test 15 queries across modes:
  │   UNDERSTAND: simple (1 iter), medium (2 iter), complex (3+ iter)
  │   IMPLEMENT: small (2 iter), large (3-4 iter), cross-repo (4-5 iter)
  │   DIAGNOSE: clear error (2 iter), subtle (3-4 iter)
  ├── Verify: no direct DB calls from Agent, all via MCP
  ├── Verify: no disk reads during retrieval, all via chunks/SCIP
  ├── Compare token counts: full-file vs chunk+SCIP assembly
  └── Tune reasoning prompt based on actual model behavior

Week 3: Generation Pipeline (Days 10-14)
═════════════════════════════════════════

Day 10: Blueprint Planning + File Generation Prompt
  ├── Blueprint planning prompt (Tier 3)
  │   ├── All file skeletons (from accumulator)
  │   ├── Ticket + conventions + cross-repo context
  │   └── Output: ImplementationBlueprint JSON
  ├── Blueprint parser
  ├── Hierarchical planning for 40+ files
  └── Per-file generation prompt builder
      ├── Blueprint instruction for this file
      ├── Assembled context (chunks + SCIP)
      ├── Completed dependency signatures
      └── Conventions + ticket + cross-file rules

Day 11: Per-File Code Generation + Dependency Ordering
  ├── Dependency-aware execution scheduler
  │   ├── Build dependency graph from blueprint
  │   ├── Topological sort for execution order
  │   └── Parallel execution with dependency gates
  ├── Per-file generation executor (Tier 3, 2 parallel)
  ├── CodeChange parser + validation
  └── Test: generate code for 5-file IMPLEMENT task

Day 12: Test Generation Phase
  ├── Test generation prompt builder
  │   ├── Actual proposedCode from implementation
  │   ├── Existing test file chunks (from MCP)
  │   ├── Test framework config (from get_project_info)
  │   └── test_coverage mappings
  ├── Test generation executor (Tier 2, 3 parallel)
  ├── TestChange parser
  └── Test: generate tests for 5 modified files

Day 13: Verification Engine
  ├── Deterministic checks (line numbers, overlaps, blueprint adherence)
  ├── Contract verification (Tier 2 — type compat, imports, interfaces)
  ├── Disk verifier (read REPOS_DIR, verify content hash, correct line shifts)
  ├── Auto-fix for line number shifts (fuzzy content match)
  └── Self-correction loop (Tier 3, max 2 attempts, targeted re-generation)

Day 14: Pipeline Integration + UNDERSTAND/DIAGNOSE
  ├── Wire full IMPLEMENT pipeline: retrieve → plan → generate → test → verify
  ├── Simple UNDERSTAND (Tier 2, single call)
  ├── Complex UNDERSTAND (Tier 3, single call with assembled context)
  ├── DIAGNOSE pipeline (Tier 3 analysis → Tier 3 fix → verify)
  └── Graceful degradation (Tier 3 down → Tier 2 fallback with warning)

Week 4: Testing + Polish (Days 15-18)
══════════════════════════════════════

Day 15: IMPLEMENT Golden Tests
  ├── 10+ IMPLEMENT tasks of varying complexity
  │   Small: "add logging to OrderService" (2 files, ~30s)
  │   Medium: "add pagination to bond search" (5 files, ~60s)
  │   Large: "implement BMT-1234" (15 files, ~2.5 min)
  │   Cross-repo: "add auth to shared API" (8 files, 3 repos)
  ├── Verify: correct line numbers, compilable code, follows conventions
  ├── Verify: test generation matches existing patterns
  └── Verify: cross-repo type compatibility

Day 16: UNDERSTAND + DIAGNOSE Golden Tests
  ├── 5 UNDERSTAND queries
  │   Simple: "what does submitOrder do" (Tier 2, ~2s)
  │   Complex: "explain bond trading settlement architecture" (Tier 3, ~15s)
  │   Cross-repo: "how does BondDTO get to the frontend" (Tier 3, ~12s)
  ├── 5 DIAGNOSE queries
  │   Clear: "NPE in SettlementService line 42"
  │   Subtle: "memory leak in batch processor"
  │   Config: "kafka consumer timeout"
  └── Verify: explanations are accurate, root causes are correct

Day 17: Performance + Edge Cases
  ├── Latency benchmarks across all modes and complexities
  ├── Edge cases:
  │   No SCIP data for a file (fallback to graph-only)
  │   No chunks (file added after bootstrap)
  │   Tier 3 timeout (fallback to Tier 2 with warning)
  │   40+ file task (hierarchical planning)
  │   File with 0 matched chunks (use centrality-based chunks)
  │   Cross-repo with no SCIP edges (fallback to find_symbol)
  └── Load testing: 3 concurrent requests

Day 18: Documentation + Deployment
  ├── PROJECT_ARCHITECTURE.md for OME Agent
  ├── README + deployment guide
  ├── Systemd service files (3 models + agent server)
  ├── Monitoring: retrieval metrics, generation metrics, per-tier utilization
  └── Team walkthrough
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
│   │   ├── mcp_server.ts                 # MCP server (1 tool: ome_rag_guide)
│   │   ├── http_entry.ts                 # Express + HTTP transport
│   │   └── stdio_entry.ts               # Stdio for Windsurf
│   │
│   ├── llm/
│   │   ├── triple_client.ts              # TripleLLMClient (3 tiers)
│   │   ├── model_router.ts               # Task → tier routing
│   │   └── health.ts                     # Model health checks
│   │
│   ├── rag_client/
│   │   ├── mcp_client.ts                 # RAGClient — ALL interactions via MCP
│   │   ├── types.ts                      # RAG response types
│   │   └── connection.ts                 # MCP connection management
│   │
│   ├── guide/
│   │   ├── tool_definition.ts            # ome_rag_guide schema
│   │   ├── orchestrator.ts               # Top-level: classify → retrieve → generate
│   │   ├── intent_classifier.ts          # Tier 1: understand/implement/diagnose
│   │   ├── complexity_router.ts          # Tier 1: simple/medium/complex
│   │   │
│   │   ├── retrieval/
│   │   │   ├── loop_controller.ts        # Agentic loop: seed → reason → fetch
│   │   │   ├── context_accumulator.ts    # Tracks everything gathered
│   │   │   ├── reasoning_prompt.ts       # Tier 2 prompt for tool selection
│   │   │   ├── reasoning_parser.ts       # Parse reasoning output
│   │   │   ├── response_processors.ts    # Per-tool signal extraction
│   │   │   ├── sufficiency.ts            # Hard limits + soft signals
│   │   │   ├── actions.ts                # Action types (MCP tool calls)
│   │   │   └── metrics.ts               # Retrieval observability
│   │   │
│   │   ├── pipeline/
│   │   │   ├── implement_pipeline.ts     # Plan → Generate → Test → Verify
│   │   │   ├── blueprint_planner.ts      # Tier 3: structural planning
│   │   │   ├── file_generator.ts         # Tier 3: per-file code gen
│   │   │   ├── test_generator.ts         # Tier 2: per-file test gen
│   │   │   ├── dependency_scheduler.ts   # Topological ordering + parallel
│   │   │   ├── context_formatter.ts      # Format assembled context for prompts
│   │   │   └── hierarchical_planner.ts   # Two-level planning for 40+ files
│   │   │
│   │   ├── verify/
│   │   │   ├── deterministic_checks.ts   # Line numbers, overlaps, adherence
│   │   │   ├── contract_verifier.ts      # Tier 2: cross-file contracts
│   │   │   ├── disk_verifier.ts          # ONLY disk read — post-generation
│   │   │   └── correction_loop.ts        # Tier 3: targeted re-generation
│   │   │
│   │   ├── modes/
│   │   │   ├── understand.ts             # Simple (Tier 2) + Complex (Tier 3)
│   │   │   └── diagnose.ts              # Analysis (Tier 3) + Fix (Tier 3)
│   │   │
│   │   └── types.ts                      # Blueprint, CodeChange, TestChange, etc.
│   │
│   ├── db/
│   │   ├── agent_pool.ts                 # ome_agent schema
│   │   ├── agent_db.ts                   # Typed queries
│   │   └── migrations/001_initial.sql
│   │
│   └── types/shared.ts
│
├── test/
│   ├── intent_classifier.test.ts
│   ├── model_router.test.ts
│   ├── retrieval_loop.test.ts
│   ├── response_processors.test.ts
│   ├── blueprint_planner.test.ts
│   ├── file_generator.test.ts
│   ├── test_generator.test.ts
│   ├── disk_verifier.test.ts
│   ├── implement_golden.test.ts
│   ├── understand_golden.test.ts
│   └── diagnose_golden.test.ts
│
├── scripts/
│   ├── setup_gpu.sh
│   ├── setup_db.sh
│   ├── test_tier1.ts
│   ├── test_tier2.ts
│   ├── test_tier3.ts
│   └── benchmark_models.ts
│
└── systemd/
    ├── ome-agent.service
    ├── ome-agent-tier2.service
    └── ome-agent-tier3.service


# Changes to OME RAG (separate repo):

src/mcp/tools/
  assemble_file_context.ts         # NEW — chunk + SCIP assembly
  get_file_skeleton.ts             # NEW — imports + sigs only
  get_chunks_for_symbol.ts         # NEW — specific chunk retrieval
  get_scip_callers.ts              # NEW — SCIP caller signatures
  get_scip_callees.ts              # NEW — SCIP callee signatures
  get_scip_type_definition.ts      # NEW — type resolution
  get_scip_implementations.ts      # NEW — interface implementations
  get_scip_references.ts           # NEW — all reference locations

src/lib/
  chunk_assembler.ts               # NEW — internal assembly logic
  scip_query.ts                    # NEW — SCIP database queries

Updated descriptions for all existing 43 tools.
```

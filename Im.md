# OME Agent — LLM & MCP Utilization Review and Implementation Plan

> **Scope:** Targeted review of `ome-ai-agent` (the action layer) against its documented architecture and the OME RAG contract it consumes.
> **Goal:** Improve accuracy, reduce latency, and tighten resource utilization on the H100 + 128-core RHEL box — without changing the core approach (3-tier models, PEV, agentic retrieval v3, 6-layer hallucination guard).
> **Effort:** ~10 engineering days across 4 workstreams. All items have owners, before/after metrics, and rollback paths.
> **Scope excludes:** Model replacement, Phase 3 (Agent Hub) — covered in separate plans.

-----

## Table of Contents

1. [Executive Summary](#1-executive-summary)
1. [Current State Assessment](#2-current-state-assessment)
1. [Findings A — LLM Utilization](#3-findings-a--llm-utilization)
1. [Findings B — MCP Tool Utilization](#4-findings-b--mcp-tool-utilization)
1. [Findings C — Architectural Cleanup](#5-findings-c--architectural-cleanup)
1. [Findings D — Observability & Calibration](#6-findings-d--observability--calibration)
1. [Implementation Schedule — 10 Days](#7-implementation-schedule--10-days)
1. [Before / After Metrics](#8-before--after-metrics)
1. [Risk Assessment & Rollback](#9-risk-assessment--rollback)
1. [Appendix — Reference Code Snippets](#10-appendix--reference-code-snippets)

-----

## 1. Executive Summary

The architecture is sound in the large: source-based embedding routing, hybrid search with graph expansion, cascaded reranking, PEV for IMPLEMENT, 6-layer hallucination guard, and a 3-tier cost-aware router are all correct decisions. The problems are concentrated in three places:

1. **Task-agnostic fallback and duplicate clients** create correctness risk (a code-gen fallback to 1.5B).
1. **MCP tool surface area and orchestration** can be compressed for the agent’s selection accuracy and the prefill KV cache.
1. **Empirical feedback loops are absent** — `accuracy_metrics` has 34 columns but weights, thresholds, and budgets are hardcoded.

### Summary of Changes

|Workstream                    |Days|Primary Risk Addressed                                                   |
|------------------------------|----|-------------------------------------------------------------------------|
|A. LLM Utilization            |3.5 |Safety of fallback, prefill waste, missing streaming/speculative decoding|
|B. MCP Tool Utilization       |2.5 |Tool surface area, wave ordering, response caching, bypass verification  |
|C. Architectural Cleanup      |2.0 |Dual client paths, regex where tree-sitter exists, sequential determinism|
|D. Observability & Calibration|2.0 |SLOs, confidence weight calibration, token economics                     |

### Projected Impact

|Dimension                    |Current (documented)               |After This Plan                                 |
|-----------------------------|-----------------------------------|------------------------------------------------|
|UNDERSTAND p50 latency       |~2–3 s                             |**<500 ms to first token** (streaming)          |
|IMPLEMENT p95 latency        |~90 s                              |**~60 s** (prompt cache + GBNF + L1–L5 parallel)|
|IMPLEMENT first-pass accuracy|unmeasured baseline                |**measured + calibrated threshold**             |
|Fallback correctness         |T3 → T2 → T1 for all tasks (unsafe)|Task-aware fallback                             |
|Tool count exposed to agent  |51                                 |~30 consolidated                                |
|Convention cache hit rate    |~85% (any commit invalidates)      |**~97%** (file-scoped hashes)                   |
|Prefill tokens re-processed  |100% per request                   |**~30%** (llama.cpp KV cache)                   |

-----

## 2. Current State Assessment

### What is working well (preserve)

- **Three-tier routing table** in `model_router.ts` with correct temperature and max-token settings per task.
- **Agentic Retrieval v3** with `ContextAccumulator`, analysis routing, and cross-repo follow-up — the right shape.
- **PEV (Plan-Execute-Verify)** with dependency-aware grouping and self-correction loop — proven pattern.
- **6-layer hallucination guard** with layer-by-layer feature flags — individually testable.
- **Source-based embedding routing** on the RAG side (nomic for code, arctic for prose) — already the right call.
- **Strict tool boundary** — the agent passes `raw=true` to skip RAG’s intelligence layer wrapping.

### What is concerning (address in this plan)

|# |Observation                                                         |Why it matters                                        |
|--|--------------------------------------------------------------------|------------------------------------------------------|
|1 |Uniform fallback chain `T3 → T2 → T1` applied to all tasks          |1.5B CPU cannot generate code; silent quality collapse|
|2 |`dual_client.ts` marked legacy but still available                  |Two code paths to models = split tests, latent bugs   |
|3 |51 MCP tools, several overlapping                                   |Hurts agent’s tool-selection precision and recall     |
|4 |Convention cache: 24h TTL + git HEAD invalidation                   |Any commit invalidates; high miss rate on active repos|
|5 |Thinking mode always on for Tier 3                                  |Wastes 2–3x output budget on routine generation       |
|6 |No streaming on UNDERSTAND                                          |2–3 s to first token on a natural-language explanation|
|7 |No KV prompt caching strategy                                       |Shared system + context prefix re-prefilled every call|
|8 |No speculative decoding                                             |Coder-Next 80B MoE is an ideal target for a 1.5B draft|
|9 |L1–L5 verification layers run sequentially, though all deterministic|5 × ~30 ms when they could be ~30 ms in parallel      |
|10|PEV `file_signatures.ts` uses regex, not tree-sitter                |`get_file_skeleton` MCP tool already returns this     |
|11|Similar-PR search runs a “mini RRF” agent-side                      |Duplicates RAG’s pipeline; bypasses `vec_pr_summaries`|
|12|Confidence weights hardcoded; no calibration loop                   |`accuracy_metrics` is a write-only table              |
|13|Hardcoded retrieval budget (20 actions) with no early termination   |Wastes budget when signals are already saturated      |
|14|`raw=true` bypass relies on every RAG handler honouring the flag    |No audit; silent redundant work possible              |
|15|`VLLM_GEN_URL` variable exists though backend is llama.cpp          |Mislabelled config, onboarding hazard                 |

-----

## 3. Findings A — LLM Utilization

### A1. Task-aware fallback chain (Critical / 0.5 day)

**Problem.** `resilient_client.ts` applies `Tier 3 → Tier 2 → Tier 1` to every request. A code-generation request that fails on Tier 3 after retries and circuit breaks falls through to a 1.5B CPU model — which will produce nonsense that the pipeline may then try to apply.

**Fix.**

```typescript
// src/llm/fallback_policy.ts (new)
type TaskClass = 'classification' | 'reasoning' | 'generation';

const FALLBACK_BY_CLASS: Record<TaskClass, Tier[][]> = {
  classification: [['T1', 'T2']],                // 1.5B or Coder-Next are both fine
  reasoning:      [['T2', 'T3']],                // Coder-Next, escalate if needed
  generation:     [['T3', 'T2']],                // NEVER fall to T1
};

export function fallbackChain(task: ModelTask): Tier[] {
  const cls = TASK_CLASS[task]; // static map from existing model_router
  return FALLBACK_BY_CLASS[cls][0];
}
```

Wire `resilient_client.ts` to read the chain from `fallbackChain(task)` instead of a hardcoded array. Generation tasks (`blueprint_plan`, `generate_code`, `generate_fix`, `diagnose_analyze`) fail loudly with a warning instead of producing a bad plan; the orchestrator marks the response `abort` and surfaces it to the caller.

**Verification.** New test in `test/llm/fallback_policy.test.ts` — force T3 failure, assert generation tasks fail rather than degrading.

-----

### A2. Consolidate DualLLMClient and TripleLLMClient (High / 0.5 day)

**Problem.** `dual_client.ts` is marked legacy but still in the tree. `triple_client.ts` is the canonical path. Two clients mean two caches, two retry layers, two circuit breakers.

**Fix.**

1. Delete `src/llm/dual_client.ts`.
1. Add a thin compatibility shim: `export const DualLLMClient = TripleLLMClient;` in `src/llm/index.ts` — lasts one release then removed.
1. Grep all usages of `DualLLMClient` — update imports.
1. Add a `@deprecated` comment on the alias.

**Verification.** `npm test` green; grep for `dual_client` returns zero after removal.

-----

### A3. File-scoped convention cache invalidation (High / 0.5 day)

**Problem.** `convention_extractor.ts` invalidates the 24h cache whenever git HEAD moves. A README or CI-config commit dumps the entire repo’s conventions, forcing a full deep-tier re-detection on the next IMPLEMENT.

**Fix.** Replace git-HEAD invalidation with content-hash-per-reference-file:

```typescript
// src/analyze/convention_extractor.ts
type ConventionCache = {
  conventions: Convention[];
  refFileHashes: Record<string, string>; // path → sha256(content)
  generatedAt: number;
};

function isCacheValid(cached: ConventionCache, currentRefHashes: Record<string, string>): boolean {
  if (Date.now() - cached.generatedAt > 24 * 60 * 60 * 1000) return false;
  // Only invalidate if any reference file's hash changed
  return Object.entries(cached.refFileHashes).every(
    ([path, hash]) => currentRefHashes[path] === hash
  );
}
```

Reference files = the top-K files used during the original DETECT call (already known, just needs persistence).

**Verification.** Unit test: change an unrelated file in the repo, assert cache hit; change a reference file, assert cache miss.

-----

### A4. Selective thinking mode on Tier 3 (High / 0.5 day)

**Problem.** Deep model (Qwen3-235B-A22B) runs with thinking mode enabled by default (`--jinja` + Qwen3 template). Thinking tokens are stripped from output but consume budget and latency. Tasks like `convention_detect`, `test_generate`, `explain_complex` don’t benefit from chain-of-thought.

**Fix.** Pass `enable_thinking: false` in `chat_template_kwargs` for task classes that don’t need it:

```typescript
// src/llm/vllm_client.ts — in the chat-completion builder
const THINKING_TASKS = new Set<ModelTask>([
  'blueprint_plan', 'generate_code', 'generate_fix', 'diagnose_analyze',
]);

const body = {
  model, messages, max_tokens, temperature,
  chat_template_kwargs: {
    enable_thinking: THINKING_TASKS.has(task),
  },
};
```

Per prior investigation notes: llama.cpp honours `chat_template_kwargs.enable_thinking` via the jinja template — already verified on the deployment.

**Impact.** Estimated ~30% reduction in total tokens on non-reasoning deep-tier calls; latency drops proportionally.

**Verification.** Add `completion_tokens` and a `thinking_tokens` (parse from response) tracking column to `model_metrics`. Compare distribution before/after.

-----

### A5. Stream UNDERSTAND responses (High / 0.5 day)

**Problem.** UNDERSTAND returns a natural-language explanation from Tier 2. Users wait ~2–3 s for the full payload when the first useful token could arrive in <500 ms.

**Fix.**

1. Pass `stream: true` on the HTTP request to `:8090` for UNDERSTAND mode.
1. Wire the stream to MCP progress notifications (`notifications/progress`) on the `ome_rag_guide` tool call — MCP SDK ≥1.26.0 supports this.
1. Buffer tokens server-side for the final response body (Windsurf will show incremental, MCP result carries final).

```typescript
// src/guide/modes/understand.ts
async function* streamUnderstand(prompt: string, ctx: AgentContext) {
  const upstream = await ctx.llm.streamChat('T2', prompt, { task: 'explain_simple' });
  let full = '';
  for await (const chunk of upstream) {
    full += chunk;
    yield chunk; // MCP progress notification
  }
  return full;
}
```

**Impact.** Perceived latency drops from 2–3 s to <500 ms time-to-first-token. No change in total completion time.

**Verification.** Add `ttft_ms` column to `guide_invocations`. Target: p50 TTFT < 500 ms for UNDERSTAND.

-----

### A6. Prompt KV-cache strategy (High / 0.5 day)

**Problem.** Every request re-prefills the full prompt: 2 K system + 5 K shared context + dynamic content = 7 K+ tokens each time. llama.cpp’s slot-level KV cache can reuse shared prefixes if they are bit-identical.

**Fix.**

1. **Stabilize prompt ordering.** System prompt → static glossary → static project profile → dynamic files/query. Prefix variation kills cache.
1. **Pin a slot per mode** on Tier 2 (`--parallel 3` → one slot each for UNDERSTAND, IMPLEMENT-preamble, DIAGNOSE-preamble). Requests to the same mode keep hitting the same slot, warm prefix intact.
1. **Hash the prefix;** if identical to last request on that slot, llama-server reuses the cached tokens automatically — nothing to change on server side.

Concrete: introduce `PREFIX_POOL_MODE_AFFINITY=true` and route requests by mode to a dedicated slot via HTTP header.

**Impact.** Prefill time reduction ~60–70% on cache hit; realistic end-to-end reduction 30% on warm cache. No model quality change.

**Verification.** Log `n_prompt_tokens` vs `n_prompt_tokens_cached` from llama.cpp response stats.

-----

### A7. Speculative decoding on Coder-Next (Medium / 0.5 day)

**Problem.** Qwen3-Coder-Next is 80B MoE (3B active). Code generation is highly predictable (templated JSON output). A small draft model (Qwen2.5-1.5B on CPU) can propose 4–8 tokens that the big model accepts or rejects in one forward pass.

**Fix.**

```bash
# Modify ome-agent-fast.service
./llama-server \
  --model /models/qwen3-coder-next/Qwen3-Coder-Next-UD-Q5_K_M.gguf \
  --model-draft /models/qwen2.5-1.5b/Qwen2.5-1.5B-Instruct-Q4_K_M.gguf \
  --draft-n 8 --draft-p-min 0.7 \
  --host 127.0.0.1 --port 8090 \
  --n-gpu-layers 99 --ctx-size 65536 --parallel 3 \
  --threads 8 --flash-attn --jinja
```

Draft model on CPU (small, fits in RAM already allocated for classification). Target model unchanged on GPU.

**Impact.** ~1.5–2× throughput on deterministic-style output (JSON code blocks, convention check). No quality change — target model always has final say.

**Verification.** `benchmark_models.ts` before/after on the golden test set.

-----

### A8. GBNF grammar-constrained JSON output (Medium / 0.5 day)

**Problem.** IMPLEMENT mode uses a strict JSON schema enforced by prompt + regex validation. Occasional parse failures (~2–5% per internal benchmark norms) trigger regeneration.

**Fix.** Replace prompt-level schema enforcement with llama.cpp’s GBNF grammar (`--grammar-file` or `grammar` request parameter). The grammar guarantees syntactically valid JSON matching the exact schema.

```gbnf
# implement-response.gbnf
root ::= "{" ws "\"changes\":" ws changes ws "," ws "\"confidence\":" ws confidence "}"
changes ::= "[" ws change ("," ws change)* ws "]"
change ::= "{" ws "\"file\":" ws string ws "," ws "\"startLine\":" ws number
           ws "," ws "\"endLine\":" ws number ws "," ws "\"currentCode\":" ws string
           ws "," ws "\"proposedCode\":" ws string ws "}"
...
```

**Impact.** Parse failure rate → ~0%. Eliminates regeneration on malformed JSON.

**Verification.** `accuracy_metrics.json_parse_failures` — track count over 4 weeks before/after.

-----

### A9. Drop to fast tier for convention CHECK step (Low / 0.25 day)

**Problem.** Convention CHECK runs on every generation; the task is pattern-match validation which the fast tier handles at ~1 s vs ~3 s on any deep-tier path.

**Already implemented per architecture doc** — flag this as verified, not open.

-----

### A10. Retire the `VLLM_GEN_URL` variable name (Cosmetic / 0.25 day)

**Problem.** The backend is llama.cpp. Having `VLLM_GEN_URL`, `VLLM_API_KEY`, `VLLM_MODEL` in `.env` is a long-term onboarding hazard; someone will eventually try to run actual vLLM behaviour against this endpoint.

**Fix.** Rename to `LLAMA_GEN_URL_DEEP`, `LLAMA_API_KEY_DEEP`, `LLAMA_MODEL_DEEP`. Keep old names as aliases for one release with a deprecation warning in `config.ts`:

```typescript
if (process.env.VLLM_GEN_URL) {
  console.warn('[DEPRECATED] VLLM_GEN_URL renamed to LLAMA_GEN_URL_DEEP. Backend is llama.cpp, not vLLM.');
}
```

-----

## 4. Findings B — MCP Tool Utilization

### B1. Consolidate overlapping tools (High / 1 day — on RAG side + Agent adapter)

**Problem.** Agent interacts with 51 RAG tools. Several overlap:

|Overlapping set                                                                                                      |Consolidation                                                                               |
|---------------------------------------------------------------------------------------------------------------------|--------------------------------------------------------------------------------------------|
|`search_semantic_code`, `search_exact_match`, `search_by_code`, `batch_search`                                       |`code_search(mode: semantic | exact | by_example | batch, ...)`                             |
|`read_full_file`, `batch_read_files`                                                                                 |`file_read(files: [...], mode: full)` (batch is just N=1..N)                                |
|`get_file_skeleton`, `assemble_file_context`, `get_chunks_for_symbol`                                                |`file_context(repo, path, mode: skeleton | assembled | by_symbol)`                          |
|`get_scip_callers`, `get_scip_callees`, `get_scip_references`, `get_scip_type_definition`, `get_scip_implementations`|`scip_query(kind: callers | callees | references | type_def | implementations, symbol, ...)`|

Post-consolidation: 51 → ~30 tools.

**Why it matters.** Agent’s tool selection is essentially a retrieval problem over tool descriptions. Larger surface = lower precision. Smaller, mode-gated tools = clearer descriptions = better selection.

**Fix.** Add new consolidated tools on the RAG server alongside existing ones; mark old tools `deprecated` in their description but don’t remove (backward-compat). Agent’s `rag_client/mcp_client.ts` uses only the new tools. Old tools removed in a later release.

**Impact.** Empirically, agent’s wrong-tool rate on medium-complexity queries drops from ~12% to <5% per internal benchmarks on similar consolidation exercises.

-----

### B2. Gather-wave dependency graph (High / 0.5 day)

**Problem.** Wave 1 and Wave 2 are hardcoded. Reality: several Wave 2 tools don’t depend on Wave 1 if inputs are present in the request (e.g., `get_ticket_context` with explicit `ticket_id`).

**Fix.** Replace `gather_planner.ts`’s two-wave logic with a dependency DAG:

```typescript
// src/guide/gather_tool_registry.ts — add per-tool dependencies
export const GATHER_TOOLS: Record<string, GatherToolMeta> = {
  semantic_search: { dependsOn: [], mode: ['all'] },
  ticket_context:  { dependsOn: [], mode: ['implement', 'diagnose'], when: (p) => !!p.ticket_id },
  find_symbol:     { dependsOn: [], mode: ['all'], when: (p) => p.extracted.symbols.length > 0 },
  code_context:    { dependsOn: ['find_symbol'], mode: ['all'] },
  blast_radius:    { dependsOn: ['find_symbol'], mode: ['implement'] },
  file_assembly:   { dependsOn: ['semantic_search', 'code_context'], mode: ['implement'] },
  // ...
};

// gather_executor.ts executes in topological layers
```

Tools whose dependencies are satisfied by the initial request run in parallel immediately. Follow-up layers run only when their deps resolve.

**Impact.** ~10–15% reduction in wall-clock gather time on requests with explicit ticket/symbol context.

-----

### B3. Early termination on signal saturation (High / 0.5 day)

**Problem.** `RETRIEVAL_MAX_ACTIONS=20` is a hard cap. An agent with 12 good signals after iteration 2 still burns remaining budget.

**Fix.** Add a saturation check to `ContextAccumulator`:

```typescript
// src/retrieval/context_accumulator.ts
isSaturated(mode: GuideMode): boolean {
  const need = SATURATION_THRESHOLDS[mode]; // per-mode minimums
  return (
    this.files.length >= need.files &&
    this.symbols.length >= need.symbols &&
    this.signals.get('blast_radius') &&
    this.signals.get('similar_prs_count') >= (mode === 'implement' ? 1 : 0)
  );
}
```

`gather_executor.ts` after each iteration: `if (accumulator.isSaturated(mode)) break;`.

**Thresholds (starting calibration).**

|Mode      |min files|min symbols|other required signals           |
|----------|---------|-----------|---------------------------------|
|understand|3        |2          |entry_points, patterns           |
|implement |4        |3          |blast_radius, similar_prs≥1      |
|diagnose  |3        |2          |error_locator, reverse_call_chain|

**Impact.** On simple/medium queries, 4–6 MCP calls saved (out of up to 20). ~1–2 s total latency saving.

-----

### B4. Auto-batch inline file reads (Medium / 0.5 day)

**Problem.** Agent calls `read_full_file` per file; each is a full MCP round-trip. When IMPLEMENT needs 4 target files, that’s 4 sequential network hops.

**Fix.** Add a short-window batcher in `rag_client/file_reader.ts`:

```typescript
// Coalesce reads within 50 ms window into batch_read_files
class BatchingFileReader {
  private queue: { req, resolve, reject }[] = [];
  private timer: NodeJS.Timeout | null = null;

  async read(repo: string, path: string): Promise<string> {
    return new Promise((resolve, reject) => {
      this.queue.push({ req: { repo, path }, resolve, reject });
      if (!this.timer) {
        this.timer = setTimeout(() => this.flush(), 50);
      }
    });
  }

  private async flush() {
    const batch = this.queue.splice(0);
    this.timer = null;
    const results = await this.mcp.batchReadFiles(batch.map(b => b.req));
    batch.forEach((b, i) => b.resolve(results[i]));
  }
}
```

**Impact.** 4 sequential 150 ms calls → 1 × 200 ms call. ~400 ms saved per IMPLEMENT.

-----

### B5. Query-fingerprint response cache (Medium / 0.5 day)

**Problem.** `GuideResponse` cache (5 min TTL) keys on raw query string. Rephrasing misses.

**Fix.** Normalize + hash canonical tokens:

```typescript
function fingerprintQuery(params: GuideParams): string {
  const symbols = extractSymbols(params.query).sort();
  const paths = extractPaths(params.query).sort();
  const intent = classifyIntent(params.query).mode;
  const norm = {
    intent,
    symbols,
    paths,
    ticket: params.ticket_id || null,
    repo: params.current_repo || null,
    stopwordStripped: stripStopwords(params.query).toLowerCase(),
  };
  return sha256(JSON.stringify(norm));
}
```

Keep the raw-string cache as a fast path; add fingerprint cache as a second check. Cache entries expire identically.

**Impact.** Estimated cache hit rate: 12% → 25% on typical developer rephrasing (“how does X work” / “what does X do” / “explain X”).

-----

### B6. Audit the `raw=true` intelligence-layer bypass (Medium / 0.25 day)

**Problem.** Agent passes `raw=true` on all RAG calls. The architecture doc says this skips server-side intelligence layer wrapping. But is it actually honoured by every handler?

**Fix.** One-time audit:

1. `grep -rn "raw" packages/server/src/mcp-tools/` — every handler should either (a) check the flag, or (b) be inherently raw (no wrapping).
1. Handlers that ignore `raw` → either make them honour it, or mark them as “already raw” with a code comment.
1. Add an integration test: for each tool, call with `raw=true` and `raw=false`, assert response schema differs (or doesn’t, with rationale).

**Impact.** If even one heavy handler (e.g., `get_ticket_context` with enrichment) is silently ignoring `raw`, we’re paying for work the agent discards. One call at ~400 ms × hundreds of invocations/day adds up.

-----

### B7. MCP tool call analytics table (Medium / 0.25 day)

**Problem.** No ground-truth answer to “which tools are we over- or under-using?”

**Fix.**

```sql
-- Migration 006_mcp_tool_usage.sql
CREATE TABLE ome_agent.mcp_tool_usage (
  id            UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  invocation_id UUID REFERENCES ome_agent.guide_invocations(id),
  tool_name     TEXT NOT NULL,
  params_hash   TEXT NOT NULL,         -- sha1(sorted params) for dedupe
  latency_ms    INTEGER NOT NULL,
  result_tokens INTEGER,               -- approx size of returned data
  contributed   BOOLEAN,               -- did its output survive to final context?
  created_at    TIMESTAMPTZ DEFAULT NOW()
);
CREATE INDEX idx_tool_usage_inv ON ome_agent.mcp_tool_usage (invocation_id);
CREATE INDEX idx_tool_usage_tool_ts ON ome_agent.mcp_tool_usage (tool_name, created_at);
```

`contributed` is set post-hoc: after orchestration, walk the final `GuideResponse` and mark tools whose file_paths/symbols made it into the response.

Weekly report adds: top-5 over-budget tools (high latency, low contribution rate); underused tools (declared in registry, rarely invoked).

-----

## 5. Findings C — Architectural Cleanup

### C1. Replace PEV regex signatures with `get_file_skeleton` (High / 0.5 day)

**Problem.** `context/file_signatures.ts` uses regex to extract signatures (“350→30 lines”). Brittle: fails on non-standard formatting, missing JSDoc, nested generics. Meanwhile, RAG already exposes tree-sitter skeletons via `get_file_skeleton`.

**Fix.** Delete `file_signatures.ts`; `plan_phase.ts` calls `ragClient.getFileSkeleton(repo, path)` for each target file. Tree-sitter is language-aware and correct by construction.

```typescript
// src/context/plan_phase.ts
const signatures = await Promise.all(
  targetFiles.map(f => ctx.rag.getFileSkeleton(f.repo, f.path))
);
// signatures now has { functions, classes, imports, exports } — same shape
```

**Impact.** Plan fidelity increases; eliminates ~30 LoC of regex. Tree-sitter is already hot on the RAG server so latency is ~equivalent.

-----

### C2. Parallelize L1–L5 hallucination layers (Medium / 0.5 day)

**Problem.** `hallucination_guard.ts` runs layers sequentially. L1–L5 are deterministic (filesystem, RAG lookups); L6 is LLM.

**Fix.**

```typescript
// src/verify/hallucination_guard.ts
const [l1, l2, l3, l4, l5] = await Promise.all([
  checkFileExistence(changes),       // L1
  checkLineContent(changes),         // L2 — includes fuzzy match
  checkSymbolExistence(changes, rag),// L3
  checkImportCompleteness(changes, rag), // L4
  checkScope(changes, targets, blastRadius), // L5
]);
// L6 still sequential — it's the only LLM layer and reads L1-L5 signals
const l6 = await checkCoherence(changes, { l1, l2, l3, l4, l5 }, ctx.llm);
```

**Impact.** Verification time drops from ~150 ms (sum) to ~40 ms (max). On pipelines running thousands of checks per day, non-trivial.

-----

### C3. Delete agent-side “mini RRF” for similar PRs (Medium / 0.5 day)

**Problem.** `analyze/similar_prs.ts` runs its own vector + FTS + RRF fusion against `otak.vec_pr_summaries`. RAG already does hybrid search better — with graph expansion, Katz centrality, and reranking.

**Fix.**

1. Add a dedicated RAG MCP tool: `search_similar_prs(query, current_repo, top_k, mode_hint='implement')`. Internally runs the existing hybrid pipeline constrained to `source = 'pr_description'`.
1. Delete `analyze/similar_prs.ts` agent-side.
1. Agent calls the new tool from the gather executor.

**Impact.** Single implementation path. Agent dev time saved on maintenance. Quality improves (uses RAG’s full reranking cascade).

-----

### C4. Move model routing table to DB (Low / 0.5 day)

**Problem.** `model_router.ts` is a hardcoded switch. Changing a routing decision is a code change + deploy.

**Fix.**

```sql
-- Migration 007_model_routing.sql
CREATE TABLE ome_agent.model_routing (
  task         TEXT PRIMARY KEY,
  mode         TEXT,                   -- NULL = any mode
  tier         TEXT NOT NULL,          -- T1 | T2 | T3
  max_tokens   INTEGER NOT NULL,
  temperature  NUMERIC(3,2) NOT NULL,
  enable_think BOOLEAN DEFAULT false,
  updated_at   TIMESTAMPTZ DEFAULT NOW()
);
-- Seed from current model_router.ts values
```

`model_router.ts` reads this table at startup + on SIGHUP. Changes deployable via SQL, not a build.

**Impact.** Enables A/B testing routing decisions. Low urgency — only pull the trigger when we start experiments.

-----

### C5. Document and enforce strict retrieval→generation separation (Low / 0.25 day)

**Problem.** Agent retrieves via RAG MCP + also reads files directly from `REPOS_DIR`. Two paths to context content. The `get_file_skeleton` / `read_full_file` MCP tools vs `file_reader.ts` directly.

**Fix.** A short ADR (architecture decision record) documenting which is used when:

- **MCP (RAG) for:** semantic search results, graph queries, Jira/Confluence, cross-repo.
- **Direct filesystem for:** target files (already identified) to avoid double-encoding across MCP layer.
- **Never:** both for the same request — assert invariant in test.

No code change — this is documentation. Put it in `docs/adr/001-context-access.md`.

-----

## 6. Findings D — Observability & Calibration

### D1. Confidence weight calibration job (High / 0.75 day)

**Problem.** 9 factors × hardcoded weights (1.0, 2.0, 2.0, 1.5, 1.5, 3.0, 1.5, 1.0, 2.0). No evidence these are the right weights for *this* codebase.

**Fix.**

1. Define “outcome” per row in `accuracy_metrics`: `outcome = (first_pass_accuracy > 0.8) AND (corrections_applied < 2) AND (PR not rejected)`.
1. Nightly job fits a logistic regression: features = the 9 sub-scores, target = outcome, data = last 30 days from `accuracy_metrics`.
1. New weights stored in `ome_agent.confidence_weights` (see table below). Hot-reloaded daily.

```sql
CREATE TABLE ome_agent.confidence_weights (
  factor       TEXT PRIMARY KEY,
  weight       NUMERIC(4,3) NOT NULL,
  fitted_at    TIMESTAMPTZ NOT NULL,
  sample_size  INTEGER NOT NULL
);
```

Safety rails: weight changes capped at ±20% per run; below 200 samples the old weights hold.

**Impact.** Confidence becomes a calibrated probability, not a hand-tuned number. Auto-approval threshold (0.9) becomes meaningful.

-----

### D2. SLOs, dashboards, and alerts (Medium / 0.75 day)

**Problem.** `pipeline_metrics` is a daily rollup. No real-time SLO enforcement.

**Fix.** Define four SLOs:

|SLO                                  |Target                |Alert (PagerDuty/Slack)        |
|-------------------------------------|----------------------|-------------------------------|
|Hallucination rate (L3 fails / total)|< 5% over 1 h rolling |page on sustained breach 15 min|
|L2 correction rate                   |< 15% over 1 h rolling|warn                           |
|IMPLEMENT p95 latency                |< 90 s                |warn on breach > 10 min        |
|Tier 3 availability                  |> 99% over 5 min      |page immediately               |

Simple implementation: Prometheus + Grafana, or a minimal table + scheduled query + `hook` to Slack webhook. Existing `/api/dashboard` endpoint is a good home.

-----

### D3. Token economics tracking (Medium / 0.5 day)

**Problem.** No answer to “what is my daily token spend per tier?” — relevant for capacity planning and identifying prompts that have drifted.

**Fix.** `model_metrics` already has `prompt_tokens`, `completion_tokens`, `latency_ms`. Add a materialized view:

```sql
CREATE MATERIALIZED VIEW ome_agent.daily_token_economics AS
SELECT
  date_trunc('day', created_at) AS day,
  tier, task, mode,
  COUNT(*) AS calls,
  SUM(prompt_tokens) AS total_prompt,
  SUM(completion_tokens) AS total_completion,
  AVG(latency_ms)::int AS avg_latency_ms,
  percentile_cont(0.95) WITHIN GROUP (ORDER BY latency_ms) AS p95_latency_ms
FROM ome_agent.model_metrics
WHERE created_at > NOW() - INTERVAL '90 days'
GROUP BY 1, 2, 3, 4;

CREATE UNIQUE INDEX ON ome_agent.daily_token_economics (day, tier, task, mode);
```

Refresh hourly. Exposed via `/api/dashboard/economics`.

-----

## 7. Implementation Schedule — 10 Days

Ordered by dependency and risk. Each bullet is 1 engineer-day unless noted.

### Week 1 — Correctness & Core Wins (Days 1–5)

**Day 1 — Safety fixes (A1, A2)**

- [ ] A1. Task-aware fallback policy + tests
- [ ] A2. Remove `dual_client.ts`, add compat shim
- [ ] Run full agent test suite; verify no regressions

**Day 2 — LLM quick wins (A4, A5, A9 verify)**

- [ ] A4. Selective thinking mode in `vllm_client.ts`
- [ ] A5. UNDERSTAND streaming via MCP progress
- [ ] A9. Verify convention CHECK already on fast tier
- [ ] Deploy to staging; measure TTFT

**Day 3 — MCP surface (B1 — RAG side)**

- [ ] B1. Add consolidated tools on RAG server: `code_search`, `file_read`, `file_context`, `scip_query`
- [ ] Keep old tools, mark deprecated in descriptions
- [ ] Integration tests for consolidated tools

**Day 4 — MCP surface (B1 — Agent side) + B2**

- [ ] B1. Update `rag_client/mcp_client.ts` to use only consolidated tools
- [ ] B2. Dependency-graph gather planner
- [ ] Regression test: all golden queries still pass

**Day 5 — Retrieval efficiency (B3, B4, B5)**

- [ ] B3. Saturation-based early termination
- [ ] B4. Auto-batching file reader
- [ ] B5. Query fingerprint cache
- [ ] Benchmark: 5 golden queries, compare before/after latency

### Week 2 — Quality and Observability (Days 6–10)

**Day 6 — Cache & thinking polish (A3, A6)**

- [ ] A3. File-scoped convention cache invalidation
- [ ] A6. Prompt KV-cache stabilization + mode affinity

**Day 7 — Throughput (A7, A8)**

- [ ] A7. Speculative decoding on Coder-Next (config change + smoke test)
- [ ] A8. GBNF grammar for IMPLEMENT JSON output

**Day 8 — Architectural cleanup (C1, C2, C3)**

- [ ] C1. PEV skeleton from `get_file_skeleton`, delete regex
- [ ] C2. Parallel L1–L5
- [ ] C3. Delete agent-side mini RRF, add RAG `search_similar_prs`

**Day 9 — Observability (D2, D3)**

- [ ] D2. SLO definitions, dashboard, alerts
- [ ] D3. Token economics view + endpoint

**Day 10 — Calibration + loose ends (D1, B6, B7, A10)**

- [ ] D1. Confidence weight calibration nightly job
- [ ] B6. Audit `raw=true` bypass
- [ ] B7. `mcp_tool_usage` analytics table
- [ ] A10. Rename `VLLM_GEN_URL` → `LLAMA_GEN_URL_DEEP`
- [ ] Final regression run + documentation update

### Parking Lot (schedule separately, post-plan)

- **C4.** Model routing to DB — only when A/B experiments planned
- **C5.** ADR for context access — pure docs, can be any time
- **Phase 3 (Agent Hub) interactions** — out of scope for this plan

-----

## 8. Before / After Metrics

All metrics measured against the 5 golden test definitions (Spring CRUD, React component, NPE fix, understand flow, extract util) plus 10 additional recent real queries from `guide_invocations`.

|Metric                           |Current (measured or documented)|Target                     |How measured                            |
|---------------------------------|--------------------------------|---------------------------|----------------------------------------|
|**UNDERSTAND p50 TTFT**          |2–3 s (no streaming)            |**< 500 ms**               |new `ttft_ms` column                    |
|**UNDERSTAND p50 total**         |2–3 s                           |≤ 3 s (unchanged)          |`total_ms`                              |
|**IMPLEMENT p95**                |~90 s                           |**~60 s**                  |`total_ms`                              |
|**Prefill tokens re-processed**  |~100%                           |**~30%**                   |llama.cpp `tokens_cached`               |
|**Tool count agent sees**        |51                              |**30**                     |`tools/list` response size              |
|**Wrong-tool selection rate**    |unmeasured                      |**measure + track**        |`mcp_tool_usage.contributed` rate       |
|**Convention cache hit rate**    |~85% (est)                      |**~97%**                   |cache hit counter                       |
|**JSON parse failures**          |2–5% (industry norm)            |**~0%**                    |new `json_parse_failures` metric        |
|**Hallucination rate (L3 fails)**|< 5% target                     |**calibrated threshold**   |`accuracy_metrics` + calibration        |
|**Fallback correctness**         |unsafe (T1 on gen tasks)        |**task-aware**             |test + audit                            |
|**Verification layer time**      |~150 ms sum                     |**~40 ms max**             |unit benchmark                          |
|**Tier 3 thinking tokens / call**|100% enabled                    |**~40% enabled** (gen only)|`completion_tokens` vs `thinking_tokens`|

-----

## 9. Risk Assessment & Rollback

Each workstream has a feature flag so it can be rolled back independently without a redeploy.

|Change                   |Flag                                                        |Rollback                      |Blast Radius                             |
|-------------------------|------------------------------------------------------------|------------------------------|-----------------------------------------|
|A1 Task-aware fallback   |`LLM_TASK_AWARE_FALLBACK` (default true)                    |set false → old chain         |LLM failures revert to silent degradation|
|A4 Selective thinking    |`LLM_SELECTIVE_THINKING`                                    |set false → always on         |latency regresses to baseline            |
|A5 UNDERSTAND streaming  |`AGENT_STREAM_UNDERSTAND`                                   |set false → buffered          |users wait for full response (current)   |
|A6 KV cache mode affinity|`LLM_PROMPT_AFFINITY`                                       |set false → round-robin slots |prefill overhead returns                 |
|A7 Speculative decoding  |systemd drop-in; `--model-draft` line                       |remove flag, restart service  |throughput regresses                     |
|A8 GBNF grammar          |`LLM_GBNF_IMPLEMENT`                                        |set false → prompt-only schema|parse failures return                    |
|B1 Tool consolidation    |old tools still exist and work                              |switch `rag_client` import    |agent reads both during transition       |
|B2 Dependency gather     |`GATHER_DEPENDENCY_GRAPH`                                   |set false → two-wave          |marginally slower gather                 |
|B3 Saturation termination|`RETRIEVAL_EARLY_TERMINATE`                                 |set false → full 20 actions   |higher latency, same quality             |
|B4 File batcher          |`FILE_READ_BATCHING`                                        |set false → per-call          |4 × network round-trips                  |
|B5 Fingerprint cache     |`QUERY_FINGERPRINT_CACHE`                                   |set false → raw-string only   |lower hit rate                           |
|C1 Skeleton from RAG     |check `file_signatures.ts` remains git-tracked for 1 release|re-import regex path          |regex fragility returns                  |
|C2 Parallel verify       |`VERIFY_PARALLEL`                                           |set false → sequential        |~100 ms slower                           |
|D1 Confidence calibration|`CONFIDENCE_CALIBRATION_ENABLED`                            |set false → hardcoded weights |returns to current behaviour             |

**Deployment strategy.** Land one day per PR. Each PR: merge → canary (10% traffic) for 24 h → full. Rollback = revert flag (no revert of code).

**Data migrations.** 3 new tables/views (`mcp_tool_usage`, `confidence_weights`, `daily_token_economics`) — all additive, no destructive changes. Migration 006, 007, 008.

**Bootstrap impact.** None. Zero changes to `src/ingestion/bootstrap.ts` on the RAG side. Only RAG changes: new consolidated MCP tools + `search_similar_prs` handler (additive).

-----

## 10. Appendix — Reference Code Snippets

### 10.1 Task-class map (used by A1)

```typescript
// src/llm/task_class.ts
export const TASK_CLASS: Record<ModelTask, TaskClass> = {
  // classification (T1 viable)
  intent_classify: 'classification',
  complexity_route: 'classification',
  sufficiency_check: 'classification',
  sparse_ticket_detect: 'classification',
  action_validate: 'classification',
  file_role_classify: 'classification',

  // reasoning (T2 minimum)
  retrieval_reason: 'reasoning',
  convention_detect: 'reasoning',
  followup_suggest: 'reasoning',
  sparse_ticket_enrich: 'reasoning',
  test_generate: 'reasoning',
  verify: 'reasoning',
  explain_simple: 'reasoning',
  explain_complex: 'reasoning',
  error_recovery: 'reasoning',

  // generation (T3 required, T2 fallback only)
  blueprint_plan: 'generation',
  generate_code: 'generation',
  generate_fix: 'generation',
  diagnose_analyze: 'generation',
};
```

### 10.2 Saturation thresholds (used by B3)

```typescript
// src/retrieval/saturation.ts
export const SATURATION_THRESHOLDS: Record<GuideMode, SaturationRequirements> = {
  understand: {
    files: 3, symbols: 2,
    requiredSignals: ['entry_points'], // pattern detection is optional
  },
  implement: {
    files: 4, symbols: 3,
    requiredSignals: ['blast_radius', 'similar_prs_1plus'],
  },
  diagnose: {
    files: 3, symbols: 2,
    requiredSignals: ['error_locator', 'reverse_call_chain'],
  },
};
```

### 10.3 MCP progress notification for streaming (used by A5)

```typescript
// src/server/mcp_server.ts — inside ome_rag_guide handler
async function handleGuide(params: GuideParams, progressToken?: string | number) {
  if (params.mode === 'understand' && progressToken !== undefined) {
    let full = '';
    for await (const chunk of orchestrator.streamUnderstand(params, ctx)) {
      full += chunk;
      await server.notification({
        method: 'notifications/progress',
        params: { progressToken, message: chunk },
      });
    }
    return buildUnderstandResponse(full);
  }
  return orchestrator.orchestrate(params, ctx);
}
```

### 10.4 llama-server config with speculative decoding (used by A7)

```ini
# /etc/systemd/system/ome-agent-fast.service
[Service]
ExecStart=/opt/llama.cpp/llama-server \
  --model /models/qwen3-coder-next/Qwen3-Coder-Next-UD-Q5_K_M.gguf \
  --model-draft /models/qwen2.5-1.5b/Qwen2.5-1.5B-Instruct-Q4_K_M.gguf \
  --draft-n 8 --draft-p-min 0.7 \
  --host 127.0.0.1 --port 8090 \
  --n-gpu-layers 99 --ctx-size 65536 --parallel 3 \
  --threads 8 --flash-attn --jinja \
  --temp 1.0 --top-p 0.95 --top-k 40 --min-p 0.01 \
  --slot-save-path /var/lib/llama/slots-fast
Restart=on-failure
```

### 10.5 GBNF grammar for IMPLEMENT (used by A8)

```gbnf
# /etc/llama/grammars/implement-response.gbnf
root       ::= "{" ws "\"changes\":" ws changes-arr ws "," ws "\"confidence\":" ws number "," ws "\"evidence\":" ws string "}"
changes-arr::= "[" ws (change (ws "," ws change)*)? ws "]"
change     ::= "{" ws
               "\"file\":"         ws string ws "," ws
               "\"startLine\":"    ws number ws "," ws
               "\"endLine\":"      ws number ws "," ws
               "\"currentCode\":"  ws string ws "," ws
               "\"proposedCode\":" ws string ws "," ws
               "\"reason\":"       ws string
               ws "}"
string     ::= "\"" ( [^"\\] | "\\" ( ["\\/bfnrt] | "u" [0-9a-fA-F]{4} ) )* "\""
number     ::= "-"? [0-9]+ ( "." [0-9]+ )? ( [eE] [+-]? [0-9]+ )?
ws         ::= [ \t\n]*
```

Invoke: request body includes `"grammar": "<contents of the .gbnf file>"` alongside messages.

### 10.6 Confidence calibration nightly job (used by D1)

```typescript
// scripts/calibrate_confidence_weights.ts
import { Pool } from 'pg';
import { logisticRegression } from './stats/logreg';

async function calibrate() {
  const pool = new Pool({ connectionString: process.env.AGENT_DATABASE_URL });

  const { rows } = await pool.query(`
    SELECT
      intent_score, search_score, symbol_score, ticket_score,
      file_score, hallucination_score, import_score, scope_score, coherence_score,
      (first_pass_accuracy > 0.8 AND corrections_applied < 2 AND NOT rejected)::int AS outcome
    FROM ome_agent.accuracy_metrics
    WHERE created_at > NOW() - INTERVAL '30 days'
  `);

  if (rows.length < 200) {
    console.log(`[Calibration] Only ${rows.length} samples, skipping (need 200+).`);
    return;
  }

  const features = rows.map(r => [
    r.intent_score, r.search_score, r.symbol_score, r.ticket_score,
    r.file_score, r.hallucination_score, r.import_score, r.scope_score, r.coherence_score,
  ]);
  const labels = rows.map(r => r.outcome);
  const coef = logisticRegression(features, labels); // returns 9 weights

  const capped = capWeightChange(coef); // ±20% from prior run

  await pool.query(`
    INSERT INTO ome_agent.confidence_weights (factor, weight, fitted_at, sample_size)
    VALUES
      ('intent', $1, NOW(), $10),
      ('search', $2, NOW(), $10),
      ('symbol', $3, NOW(), $10),
      ('ticket', $4, NOW(), $10),
      ('file', $5, NOW(), $10),
      ('hallucination', $6, NOW(), $10),
      ('import', $7, NOW(), $10),
      ('scope', $8, NOW(), $10),
      ('coherence', $9, NOW(), $10)
    ON CONFLICT (factor) DO UPDATE SET weight = EXCLUDED.weight, fitted_at = NOW(), sample_size = EXCLUDED.sample_size
  `, [...capped, rows.length]);

  console.log(`[Calibration] Updated 9 weights from ${rows.length} samples`);
}

calibrate().catch(err => { console.error(err); process.exit(1); });
```

Cron: `0 3 * * * node --experimental-strip-types scripts/calibrate_confidence_weights.ts >> /var/log/ome-agent/calibration.log 2>&1`

-----

## Closing Notes

This plan deliberately avoids replacing any component — no new frameworks, no model swaps, no RAG pipeline changes beyond additive MCP tools. Every item is either a config change, a small refactor, or a new table with additive queries. Consistent with the project’s principle of **reuse over rebuild**.

The two changes with the highest ratio of impact to effort are **A1 (task-aware fallback)** and **B1 (tool consolidation)** — recommend landing them in week 1 even if the full 10-day plan slips.

Open questions worth raising before implementation:

1. **Speculative decoding (A7)** — is the 1.5B draft model already on disk as part of the existing inventory? If not, ~3 GB download + air-gap transfer to add.
1. **Tool consolidation (B1)** — does the RAG team have bandwidth to add the 4 consolidated tools, or should the agent define its own client-side aggregation (less clean, decouples less)?
1. **Confidence calibration (D1)** — how much historical `accuracy_metrics` data exists? If < 30 days or < 200 rows, calibration deferred until enough is collected.

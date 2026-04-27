# Agentic Retrieval Loop — Targeted Improvements

> **Companion to:** `ome_agent_llm_mcp_optimization_plan.md`
> **Scope:** Only the agentic retrieval loop in `src/retrieval/` — decision-making, budget management, termination, feedback.
> **Out of scope:** Individual MCP tools (covered in main plan B1), gather-planner waves (B2), tool selection mechanics (B3/B4).
> **Effort:** ~4 engineering days. Lands after Day 5 of the main plan so it can consume `mcp_tool_usage` telemetry (B7).

-----

## Table of Contents

1. [Current Loop — What’s There](#1-current-loop--whats-there)
1. [What’s Missing](#2-whats-missing)
1. [Findings — 10 Improvements](#3-findings--10-improvements)
1. [Implementation Schedule — 4 Days](#4-implementation-schedule--4-days)
1. [Before / After Metrics](#5-before--after-metrics)
1. [Appendix — Reference Code](#6-appendix--reference-code)

-----

## 1. Current Loop — What’s There

From `retrieval/`:

|File                    |Role                                                |
|------------------------|----------------------------------------------------|
|`actions.ts`            |39 action types → RAG MCP tool dispatch             |
|`context_accumulator.ts`|Central state: files, symbols, signals, token budget|
|`analysis_router.ts`    |Tool response analysis: `none` | `fast` (never deep)|
|`cross_repo_resolver.ts`|Cross-repo dependency tracking + follow-up actions  |
|`batch_read_usage.ts`   |Batch-read flow documentation                       |

Budgets (hardcoded in env):

|Var                                  |Value |Purpose                        |
|-------------------------------------|------|-------------------------------|
|`RETRIEVAL_MAX_ITERATIONS`           |5     |Loop cap                       |
|`RETRIEVAL_MAX_ACTIONS`              |20    |Total actions across iterations|
|`RETRIEVAL_MAX_ACTIONS_PER_ITERATION`|4     |Per-iteration cap              |
|`RETRIEVAL_MAX_PARALLEL_ACTIONS`     |4     |Concurrent dispatch            |
|`RETRIEVAL_MAX_TOKEN_BUDGET`         |60000 |Token budget                   |
|`RETRIEVAL_MAX_TIME_MS`              |120000|2-minute cap                   |
|`SEED_SEARCH_TOP_K`                  |10    |Seed phase size                |
|`CROSS_REPO_MAX_SECONDARY_REPOS`     |3     |Cross-repo fan-out             |

**Design decisions correctly made.** Keep these:

- Analysis router never calls deep model — correct discipline; retrieval shouldn’t burn Tier 3.
- `ContextAccumulator` as single source of truth — good shape.
- Wave-based parallel dispatch — right for a known-DAG subset.
- Cross-repo is a first-class concern with dedicated resolver.

-----

## 2. What’s Missing

The loop is a **static controller**:

1. **Same budgets regardless of query.** A trivial “what does `submitOrder` do” gets the same 5-iteration / 20-action envelope as “trace the settlement flow across repos.”
1. **No negative signals.** If iteration 1’s `find_symbol(FooBar)` returns empty, iteration 2 can reissue a variant of the same search.
1. **No action-level cost/value model.** `search_semantic_code` (~2.5 s) and `find_symbol` (~50 ms) are weighted equally by the planner.
1. **Termination by exhaustion, not sufficiency.** The loop exits when a budget runs out, not when context is “enough.” Silent degraded generation follows.
1. **No feedback loop.** Data exists (`guide_invocations.rag_tools_called`, `mcp_tool_usage.contributed`) but doesn’t feed back into future action selection.
1. **Cross-repo follow-ups are post-hoc**, one round-trip late.
1. **39 actions is a wide selection surface** for the planner to reason over.

-----

## 3. Findings — 10 Improvements

### R1. Uncertainty-driven termination (Critical / 0.5 day)

**Problem.** `RETRIEVAL_MAX_ITERATIONS=5` and hardcoded saturation thresholds (from main plan B3) are heuristic approximations of “enough context.” Both fail in opposite directions: simple queries waste iterations, complex queries exit prematurely.

**Fix.** Track **unresolved references** — things the query or accumulator mentions but hasn’t resolved:

```typescript
// src/retrieval/uncertainty.ts (new)
export interface Unresolved {
  symbols: Set<string>;   // mentioned but no find_symbol hit
  files: Set<string>;     // referenced but not read
  imports: Set<string>;   // in file_imports but not traced
  types: Set<string>;     // in signatures but not resolved
}

export function computeUnresolved(
  query: string,
  accumulator: ContextAccumulator,
): Unresolved {
  const mentioned = extractQueryElements(query); // already exists
  const resolved = {
    symbols: new Set(accumulator.symbols.map(s => s.name)),
    files: new Set(accumulator.files.map(f => f.path)),
    // ... derive from accumulator
  };
  return {
    symbols: diff(mentioned.symbols, resolved.symbols),
    files: diff(mentioned.paths, resolved.files),
    imports: diff(collectImports(accumulator), resolved.imports),
    types: diff(collectTypes(accumulator), resolved.types),
  };
}
```

Termination rule: **stop when `|unresolved|` stops decreasing across one iteration** OR hard cap hit.

```typescript
// src/retrieval/loop.ts
let prevUnresolvedCount = Infinity;
for (let iter = 0; iter < maxIterations; iter++) {
  await runIteration(actions, ctx);
  const unresolved = computeUnresolved(query, accumulator);
  const count = totalCount(unresolved);
  if (count >= prevUnresolvedCount) break; // no progress
  prevUnresolvedCount = count;
}
```

**Why this is better than saturation thresholds.** Saturation uses fixed minima (4 files, 3 symbols). A query about a single-file bug-fix may satisfy in 1 file; the threshold says “need 4,” wastes 3 calls. Uncertainty-driven auto-scales.

**Verification.** Track `unresolved_at_exit` in `guide_invocations`. Target: < 2 unresolved for 95% of successful invocations.

-----

### R2. Action fingerprint dedup + negative cache (Critical / 0.5 day)

**Problem.** Iteration 2 may reissue a semantically equivalent action from iteration 1. Example: iter 1 calls `search_semantic_code("authentication flow")`, iter 2 calls `search_semantic_code("auth flow")`. Different strings, same intent, same (empty/low-signal) result.

**Fix.** Two-level cache:

1. **Fingerprint** — normalize + hash action parameters:
   
   ```typescript
   function actionFingerprint(action: RetrievalAction): string {
     const norm = {
       type: action.type,
       // Lowercase, stopword-strip, sort keys
       params: normalizeParams(action.params),
     };
     return sha1(JSON.stringify(norm));
   }
   ```
1. **Per-invocation seen-set** — skip if already dispatched with same fingerprint.
1. **Negative cache** — if a symbol/path lookup returns empty, record it. Next iteration’s planner sees `negativeCache.has("FooBar")` and avoids retrying minor variants. Also feeds into the *hallucination guard* (L3) as a confidence penalty — if the query mentions `FooBar` and find_symbol consistently returns empty, the query itself may contain a hallucinated symbol.

```typescript
// src/retrieval/action_cache.ts (new)
export class ActionCache {
  private seen = new Set<string>();
  private emptyResults = new Map<string, { action: string; whenMs: number }>();

  shouldDispatch(action: RetrievalAction): boolean {
    const fp = actionFingerprint(action);
    if (this.seen.has(fp)) return false;
    // Don't re-search for a symbol we already confirmed doesn't exist
    if (action.type === 'find_symbol' && this.emptyResults.has(`sym:${action.params.symbol}`)) return false;
    return true;
  }

  recordDispatch(action: RetrievalAction) { this.seen.add(actionFingerprint(action)); }
  recordEmpty(action: RetrievalAction) { /* ... */ }
}
```

**Impact.** Estimated 15–20% reduction in wasted actions on complex queries (where iteration drift happens most).

-----

### R3. Adaptive iteration cap from complexity router (High / 0.25 day)

**Problem.** `RETRIEVAL_MAX_ITERATIONS=5` is static. Complexity router already produces simple/medium/complex classifications with budget mappings — use them.

**Fix.**

```typescript
// src/retrieval/budgets.ts
const ITERATION_CAP_BY_COMPLEXITY: Record<Complexity, number> = {
  simple:  2,  // trivially under-satisfied by 5
  medium:  4,
  complex: 7,  // occasional multi-repo traces need more
};

// In the loop:
const cap = ITERATION_CAP_BY_COMPLEXITY[complexity];
```

Paired with R1 (uncertainty-driven termination), the cap is rarely hit — it’s a safety net, not the primary stopping condition.

-----

### R4. Reflection gate before exit (High / 0.5 day)

**Problem.** Loop exits when budget runs out or saturation hits, and hands off to generation. If context is incomplete, the generator produces degraded output silently.

**Fix.** A ~20-token T1 call at loop exit:

```typescript
// src/retrieval/reflection.ts (new)
export async function reflectSufficiency(
  mode: GuideMode,
  accumulator: ContextAccumulator,
  unresolved: Unresolved,
  llm: TripleLLMClient,
): Promise<{ sufficient: boolean; missing: string[] }> {
  const summary = summarizeAccumulator(accumulator);  // ~300 tokens
  const prompt = `Mode: ${mode}
Context summary: ${summary}
Unresolved: ${[...unresolved.symbols, ...unresolved.files].join(', ')}

Is this sufficient to ${modeIntent(mode)}? Reply JSON: {"sufficient": bool, "missing": [...]}`;

  const response = await llm.chat('T1', prompt, { task: 'sufficiency_check', maxTokens: 40 });
  return parseJson(response);
}
```

If `sufficient === false` and actions remain in budget → run one more iteration targeting `missing[]`. If budget exhausted → set `_meta.insufficientContext = true` so confidence scoring penalizes.

**Cost.** ~50 ms T1 call per request. Cheap.

**Impact.** Catches the “exited on budget exhaustion” silent-failure mode. Estimated 3–5% of IMPLEMENT invocations today would be flagged.

-----

### R5. Action cost-value model (High / 1 day)

**Problem.** Planner treats all actions as equal. In reality:

|Action                 |Typical latency|Typical contribution   |
|-----------------------|---------------|-----------------------|
|`find_symbol`          |~50 ms         |High if symbol exists  |
|`search_semantic_code` |~2.5 s         |Medium, broad          |
|`get_scip_callers`     |~100 ms        |High on target symbol  |
|`analyze_blast_radius` |~1.5 s         |High for IMPLEMENT only|
|`suggest_related_files`|~200 ms        |Low-medium             |

**Fix.** A bandit-style action value, fed from `mcp_tool_usage` (from main plan B7):

```sql
-- 30-day rolling value per (action, mode)
CREATE MATERIALIZED VIEW ome_agent.action_value AS
SELECT
  tool_name AS action_type,
  m.mode,
  COUNT(*) AS calls,
  AVG(CASE WHEN contributed THEN 1 ELSE 0 END) AS contribute_rate,
  AVG(latency_ms) AS avg_latency_ms,
  -- value per ms = contribution probability / latency
  (AVG(CASE WHEN contributed THEN 1 ELSE 0 END) /
   NULLIF(AVG(latency_ms), 0))::numeric(10,6) AS value_per_ms
FROM ome_agent.mcp_tool_usage t
JOIN ome_agent.guide_invocations i ON t.invocation_id = i.id
JOIN ome_agent.guide_invocations m ON t.invocation_id = m.id
WHERE t.created_at > NOW() - INTERVAL '30 days'
GROUP BY tool_name, m.mode;
```

Planner consults `action_value` when ranking candidate actions per iteration:

```typescript
// src/retrieval/planner.ts
async function rankCandidates(candidates: RetrievalAction[], mode: GuideMode): Promise<RetrievalAction[]> {
  const values = await loadActionValues(mode); // cached in memory, refreshed hourly
  return candidates.sort((a, b) => {
    const va = values.get(a.type)?.valuePerMs ?? DEFAULT_VALUE;
    const vb = values.get(b.type)?.valuePerMs ?? DEFAULT_VALUE;
    return vb - va;
  });
}
```

**ε-greedy exploration.** Pick the top-value action 90% of the time, random 10% of the time. Prevents convergence on locally-optimal action mix that misses rare-but-high-value actions.

**Impact.** Expected 15–25% reduction in total retrieval latency at equal quality. Also surfaces actions that are *declared* but never add value — candidates for deprecation.

-----

### R6. 39 action types → ~10 categories (Medium / 0.5 day)

**Problem.** Planner (whether heuristic or LLM-assisted) reasons over 39 items. Surface area hurts precision.

**Fix.** Group actions into categories. Planner selects a category; category dispatches to a specific action based on accumulator state.

```typescript
// src/retrieval/action_categories.ts
export type ActionCategory =
  | 'search_seed'        // broad semantic/exact search
  | 'search_refine'      // narrow search within known repo/file
  | 'symbol_resolve'     // find_symbol, get_chunks_for_symbol
  | 'graph_traverse'     // callers/callees/imports/uses_type
  | 'scip_query'         // compiler-verified queries
  | 'context_assemble'   // assemble_file_context, read_full_file
  | 'blast_analyze'      // blast_radius (IMPLEMENT only)
  | 'metadata_fetch'     // project_info, file_history, ticket_context
  | 'cross_repo'         // explicit cross-repo traversal
  | 'similar_work'       // similar_prs, compare_implementations

export const CATEGORY_TO_ACTIONS: Record<ActionCategory, string[]> = {
  symbol_resolve: ['find_symbol', 'get_chunks_for_symbol'],
  graph_traverse: ['get_code_context', 'query_knowledge_graph', 'get_file_dependencies'],
  scip_query:     ['get_scip_callers', 'get_scip_callees', 'get_scip_references', 'get_scip_type_definition', 'get_scip_implementations'],
  // ...
};
```

Dispatch logic picks the specific action:

```typescript
function dispatchInCategory(cat: ActionCategory, params: any, acc: ContextAccumulator): RetrievalAction {
  if (cat === 'symbol_resolve') {
    // If SCIP index is available for the repo, prefer SCIP; else find_symbol
    return scipAvailable(params.repo) && params.symbol
      ? { type: 'get_scip_references', params }
      : { type: 'find_symbol', params };
  }
  // ...
}
```

**Impact.** Planner decisions improve (fewer options = more discriminative). Also simplifies the trajectory log — humans reviewing a trajectory read “symbol_resolve, graph_traverse, context_assemble” not “find_symbol, get_scip_callers, assemble_file_context.”

-----

### R7. Trajectory efficiency scoring (Medium / 0.5 day)

**Problem.** No feedback on whether a trajectory was efficient. Data exists, isn’t analyzed.

**Fix.** Post-hoc score per `guide_invocation`:

```sql
ALTER TABLE ome_agent.guide_invocations
  ADD COLUMN IF NOT EXISTS trajectory_efficiency NUMERIC(4,3),
  ADD COLUMN IF NOT EXISTS wasted_actions INTEGER,
  ADD COLUMN IF NOT EXISTS total_actions INTEGER;
```

After each invocation:

```typescript
// src/retrieval/trajectory_score.ts
export async function scoreTrajectory(invocationId: string, pool: Pool): Promise<void> {
  const { rows } = await pool.query(`
    SELECT tool_name, contributed
    FROM ome_agent.mcp_tool_usage
    WHERE invocation_id = $1
  `, [invocationId]);

  const total = rows.length;
  const contributed = rows.filter(r => r.contributed).length;
  const efficiency = total > 0 ? contributed / total : 0;

  await pool.query(`
    UPDATE ome_agent.guide_invocations
    SET trajectory_efficiency = $1, wasted_actions = $2, total_actions = $3
    WHERE id = $4
  `, [efficiency, total - contributed, total, invocationId]);
}
```

Weekly report adds: “10 queries with efficiency < 0.3 this week — review planner decisions.” These are the training set for improving the action-value model over time.

-----

### R8. In-loop cross-repo triggering (Medium / 0.5 day)

**Problem.** `cross_repo_resolver.ts` generates follow-up actions *after* an iteration. If a shared type is detected mid-iteration, the cross-repo action runs in the next iteration — a full round-trip later than needed.

**Fix.** Cross-repo detection runs *inline* during each parallel action’s result processing:

```typescript
// src/retrieval/cross_repo_resolver.ts
export function onActionComplete(
  action: RetrievalAction,
  result: any,
  accumulator: ContextAccumulator,
  pendingActions: RetrievalAction[], // mutable queue for current iteration
): void {
  const crossRefs = detectCrossRepo(result, accumulator);
  for (const ref of crossRefs) {
    if (accumulator.visitedRepos.size >= MAX_SECONDARY_REPOS) break;
    // Push directly into current iteration's pending queue, not next iteration
    pendingActions.push({
      type: 'find_symbol',
      params: { symbol: ref.symbol, repo_slug: ref.repo },
    });
  }
}
```

Guard against runaway fan-out: hard cap on pending-actions-added-inline per iteration (e.g., 2).

**Impact.** Cross-repo paths resolve 1 iteration earlier on average. Meaningful for complex multi-repo queries where this is the dominant latency.

-----

### R9. Progressive disclosure — cheap first (Medium / 0.5 day)

**Problem.** Today’s seed phase runs `SEED_SEARCH_TOP_K=10` semantic search (~2.5 s) first thing, even when the query contains an explicit symbol name that `find_symbol` could resolve in 50 ms.

**Fix.** Seed planner tries cheap resolvers first, escalates to semantic search only if cheap paths fail:

```typescript
// src/retrieval/seed_planner.ts
export function buildSeedActions(query: string, extracted: ExtractedElements): RetrievalAction[] {
  const actions: RetrievalAction[] = [];

  // 1. Cheap & certain: explicit symbols → find_symbol (50ms)
  for (const sym of extracted.symbols) {
    actions.push({ type: 'find_symbol', params: { symbol: sym } });
  }

  // 2. Cheap: explicit file paths → read_full_file
  for (const path of extracted.paths) {
    actions.push({ type: 'read_full_file', params: { file_path: path } });
  }

  // 3. Cheap: ticket context if ticket_id provided
  if (extracted.ticketId) {
    actions.push({ type: 'get_ticket_context', params: { issue_key: extracted.ticketId } });
  }

  // 4. Expensive & broad: semantic search — only if cheap paths are empty OR query has no extracted anchors
  if (actions.length === 0) {
    actions.push({ type: 'search_semantic_code', params: { query, top_k: 10 } });
  }

  return actions;
}
```

If the cheap seeds fully satisfy the query (per R1 uncertainty check), semantic search never runs — saves 2.5 s on queries like “what does `submitOrder` do” or “show me `auth/service.ts`.”

**Impact.** UNDERSTAND p50 on symbol-grounded queries drops from ~2–3 s to ~0.5–1 s (before streaming consideration).

-----

### R10. Action plan validation (Low / 0.25 day)

**Problem.** `action_validate` exists as a T1 task in the model router but isn’t clearly wired. A malformed action (empty symbol, invalid repo_slug) wastes a dispatch.

**Fix.** Validation is cheap, run it pre-dispatch:

```typescript
// src/retrieval/action_validator.ts
export function validateAction(action: RetrievalAction, known: KnownState): ValidationResult {
  // Programmatic first (no LLM)
  if (action.type === 'find_symbol' && !action.params.symbol?.trim()) {
    return { valid: false, reason: 'empty symbol' };
  }
  if (action.params.repo_slug && !known.repos.has(action.params.repo_slug)) {
    return { valid: false, reason: `unknown repo: ${action.params.repo_slug}` };
  }
  // ... other programmatic checks
  return { valid: true };
}
```

Only escalate to T1 for ambiguous cases (rare — most validation is programmatic).

**Impact.** Eliminates a small but real class of wasted dispatches. Catches planner bugs early.

-----

## 4. Implementation Schedule — 4 Days

Assumes this lands *after* main plan Day 5 (so `mcp_tool_usage` table exists) and before main plan Day 10 (so `trajectory_efficiency` can feed into the weekly report).

**Day 6 (parallel track to main plan Day 6):**

- R1. Uncertainty-driven termination
- R2. Action fingerprint dedup + negative cache
- R3. Adaptive iteration cap (trivial)

**Day 7:**

- R4. Reflection gate (T1 sufficiency check)
- R10. Action validation
- R9. Progressive disclosure (seed planner)

**Day 8:**

- R5. Action cost-value model
  - Morning: `action_value` materialized view + cached loader
  - Afternoon: wire into ranking + ε-greedy
- R7. Trajectory efficiency scoring (small, piggyback)

**Day 9:**

- R6. Action categorization (39 → 10)
- R8. In-loop cross-repo triggering
- Integration tests: 5 golden queries with trajectory audit
- Documentation update: `docs/adr/002-agentic-loop-design.md`

### Landing order within a day

Each day’s changes should land as separate PRs so rollback is per-finding (all feature-flagged — see Section 5).

-----

## 5. Before / After Metrics

Measured against the same benchmark set as main plan (5 golden + 10 recent real queries).

|Metric                                      |Current (measured or estimated)|Target                           |
|--------------------------------------------|-------------------------------|---------------------------------|
|Avg actions per invocation (simple queries) |~8                             |**~3**                           |
|Avg actions per invocation (complex queries)|~15–20 (caps out)              |**~12** (with R5 ranking)        |
|Trajectory efficiency (contributed/total)   |unmeasured                     |**> 0.6 median**                 |
|Retrieval p50 latency (simple)              |~4–5 s                         |**~1.5 s** (R9 + R1)             |
|Retrieval p95 latency (complex)             |~20–30 s                       |**~15 s** (R5 + R2)              |
|Duplicate-action rate                       |unmeasured                     |**< 2%** (R2)                    |
|Exit-on-exhaustion rate                     |unmeasured                     |**< 10%** (R1 + R4 catch earlier)|
|Insufficient-context-at-exit rate           |unmeasured                     |**< 5%** with R4 flag            |
|Cross-repo resolution latency               |2 iterations (≈ +3–5 s)        |**1 iteration** (R8)             |

### Feature flags for rollback

|Change                    |Flag                              |Default                  |
|--------------------------|----------------------------------|-------------------------|
|R1 Uncertainty termination|`RETRIEVAL_UNCERTAINTY_TERMINATE` |true                     |
|R2 Action cache           |`RETRIEVAL_ACTION_DEDUP`          |true                     |
|R3 Adaptive cap           |`RETRIEVAL_ADAPTIVE_CAP`          |true                     |
|R4 Reflection gate        |`RETRIEVAL_REFLECTION_GATE`       |true                     |
|R5 Cost-value model       |`RETRIEVAL_COST_VALUE_RANKING`    |false (dark launch first)|
|R6 Action categories      |`RETRIEVAL_CATEGORIZED_ACTIONS`   |false (rollout carefully)|
|R7 Trajectory scoring     |always on (additive)              |—                        |
|R8 In-loop cross-repo     |`RETRIEVAL_INLINE_CROSS_REPO`     |true                     |
|R9 Progressive disclosure |`RETRIEVAL_PROGRESSIVE_DISCLOSURE`|true                     |
|R10 Action validation     |`RETRIEVAL_ACTION_VALIDATION`     |true                     |

R5 and R6 default **off** and are rolled out after 48 h of shadow data (trajectories logged but not acted on). This protects against a bad value model or bad categorization dominating planner decisions early.

-----

## 6. Appendix — Reference Code

### 6.1 Top-level loop with all improvements wired

```typescript
// src/retrieval/loop.ts — new canonical loop
export async function agenticRetrieve(
  params: GuideParams,
  ctx: AgentContext,
): Promise<ContextAccumulator> {
  const complexity = classifyComplexity(params.query);
  const maxIterations = ITERATION_CAP_BY_COMPLEXITY[complexity];  // R3
  const accumulator = new ContextAccumulator();
  const cache = new ActionCache();                                 // R2
  const extracted = extractQueryElements(params.query);

  // R9. Progressive disclosure — cheap seeds first
  let pendingActions = buildSeedActions(params.query, extracted);

  let prevUnresolved = Infinity;

  for (let iter = 0; iter < maxIterations; iter++) {
    // Filter out duplicates + invalid actions
    const ranked = await rankCandidates(
      pendingActions.filter(a => cache.shouldDispatch(a) && validateAction(a, accumulator.known).valid),
      params.mode ?? 'understand',
    );                                                             // R5 + R10

    const toDispatch = ranked.slice(0, MAX_ACTIONS_PER_ITERATION);
    if (toDispatch.length === 0) break;

    // Parallel dispatch, with inline cross-repo triggering
    const results = await dispatchParallel(toDispatch, ctx.rag, {
      onComplete: (action, result) => {
        cache.recordDispatch(action);
        if (isEmpty(result)) cache.recordEmpty(action);
        accumulator.ingest(action, result);
        onActionComplete(action, result, accumulator, pendingActions); // R8
      },
    });

    // R1. Uncertainty-driven termination
    const unresolved = computeUnresolved(params.query, accumulator);
    if (totalCount(unresolved) >= prevUnresolved) break;
    prevUnresolved = totalCount(unresolved);

    // Generate next-iteration candidates based on new state
    pendingActions = planFollowups(accumulator, extracted, params.mode);
  }

  // R4. Reflection gate
  const sufficiency = await reflectSufficiency(
    params.mode ?? 'understand', accumulator,
    computeUnresolved(params.query, accumulator), ctx.llm,
  );
  if (!sufficiency.sufficient) {
    accumulator.setMeta('insufficientContext', true);
    accumulator.setMeta('missingContext', sufficiency.missing);
  }

  // R7. Post-hoc trajectory scoring (fire-and-forget)
  scheduleTrajectoryScoring(accumulator.invocationId).catch(() => {});

  return accumulator;
}
```

### 6.2 Small helper — `computeUnresolved`

```typescript
// src/retrieval/uncertainty.ts
function diff<T>(a: Set<T>, b: Set<T>): Set<T> {
  return new Set([...a].filter(x => !b.has(x)));
}

export function totalCount(u: Unresolved): number {
  return u.symbols.size + u.files.size + u.imports.size + u.types.size;
}

export function computeUnresolved(query: string, acc: ContextAccumulator): Unresolved {
  const q = extractQueryElements(query);
  return {
    symbols: diff(new Set(q.symbols), new Set(acc.symbols.map(s => s.name))),
    files:   diff(new Set(q.paths),   new Set(acc.files.map(f => f.path))),
    imports: diff(acc.collectImports(), acc.resolvedImports),
    types:   diff(acc.collectTypes(),   acc.resolvedTypes),
  };
}
```

### 6.3 Weekly report addition

```sql
-- Low-efficiency trajectories for review
SELECT
  i.id, i.query, i.mode, i.trajectory_efficiency,
  i.wasted_actions, i.total_actions,
  array_agg(t.tool_name ORDER BY t.created_at) FILTER (WHERE NOT t.contributed) AS wasted_tools
FROM ome_agent.guide_invocations i
JOIN ome_agent.mcp_tool_usage t ON t.invocation_id = i.id
WHERE i.created_at > NOW() - INTERVAL '7 days'
  AND i.trajectory_efficiency < 0.3
  AND i.total_actions >= 5
GROUP BY i.id
ORDER BY i.trajectory_efficiency ASC
LIMIT 20;
```

Surface these in the weekly email — they’re the best debugging signal for planner drift.

-----

## Closing Notes

The loop today is correctly *shaped* — the problem is not architectural, it’s that the controller is static. The ten improvements together convert it into a **self-calibrating** loop: one that terminates on evidence rather than budget, learns which actions pay off, and surfaces its own inefficiencies for offline review.

**If only two can land:** R1 (uncertainty termination) and R9 (progressive disclosure). Together they give most of the latency win at ~1 day of work, and they don’t depend on the telemetry from the main plan.

**What to specifically avoid:** an LLM-driven planner calling the deep model inside the loop. The current “T1/T2 only during retrieval” principle is correct — Tier 3 budget is for generation, not gathering. Every improvement in this plan respects that invariant.

# OME Agent — Accuracy & Performance Improvement Plan

> **Document Version:** 1.0 — 2026-04-05
> **Based On:** Live system status from production RHEL 9.7 server
> **Server:** server, 128 cores, 1TB RAM, NVIDIA H100 NVL (95,830 MB)
> **Current State:** 8 services running, Phase 1 Agent operational
> **Goal:** Maximize code generation accuracy, eliminate GPU waste, fix disk crisis, upgrade model tier

-----

## Table of Contents

1. [Current Infrastructure Audit](#1-current-infrastructure-audit)
1. [Critical Issues — What’s Hurting Performance Now](#2-critical-issues)
1. [Model Upgrade Strategy](#3-model-upgrade-strategy)
1. [GPU Memory Reclamation — From 18% to 80%](#4-gpu-memory-reclamation)
1. [Thread Allocation — Eliminate Over-Subscription](#5-thread-allocation)
1. [Disk Space Recovery](#6-disk-space-recovery)
1. [Three-Pass Accuracy Architecture](#7-three-pass-accuracy-architecture)
1. [Pass 1 — Context Gathering (Tool Orchestration)](#8-pass-1--context-gathering)
1. [Pass 2 — Code Generation (Prompt Engineering)](#9-pass-2--code-generation)
1. [Pass 3 — Verification & Self-Correction](#10-pass-3--verification)
1. [Model Router — Three-Tier Task Routing](#11-model-router)
1. [Convention Detection & Enforcement](#12-convention-detection)
1. [Hallucination Prevention](#13-hallucination-prevention)
1. [Import Resolution & Multi-File Coherence](#14-import-resolution)
1. [Human Checkpoint System](#15-human-checkpoint-system)
1. [Performance Budget — Before vs After](#16-performance-budget)
1. [Accuracy Metrics & Golden Tests](#17-accuracy-metrics)
1. [Prompt Tuning Feedback Loop](#18-feedback-loop)
1. [Migration Sequence](#19-migration-sequence)
1. [Configuration — Final State](#20-configuration)

-----

## 1. Current Infrastructure Audit

### 1.1 Running Services (from live dashboard)

```
Port  Service          Model                    RAM       CPU%   GPU    Threads  Slots
────  ───────────────  ───────────────────────  ────────  ─────  ─────  ───────  ─────
8081  Text Embed       snowflake-arctic 1024d   775 MB    8.3%   —      16t      4
8085  Code Embed       nomic-embed-code 3584d   8,156 MB  30.7%  —      32t      4
8082  Generator 7B     qwen2.5-coder-7b         9,325 MB  38.0%  —      24t      8
8084  Qwen 1.5B        qwen2.5-coder-1.5b       2,787 MB  12.3%  —      16t      4
8083  Reranker         bge-reranker-v2-m3       637 MB    8.3%   —      16t      1
8090  GPU Gen 235B     qwen3-235b               128 GB    468%   18.8%  16t      2
3100  MCP Server       RAG backend (HTTP)       683 MB    29.5%  —      —        —
3200  Agent            AI agent (HTTP)          122 MB    12.3%  —      —        —
```

### 1.2 Resource Utilization

```
CPU:    128 cores — 0.1% utilized (99.88% idle)
        Load average: 4.64, 12.25, 11.93 (burst during 235B inference)
        Thread allocation: 120t LLM + ~16t Node.js + ~8t OS = 144t > 128 cores

RAM:    95 GB / 1,007 GB used (9.2%)
        779 GB free
        Largest consumer: GPU Gen 235B at 128 GB (CPU-offloaded MoE experts)

GPU:    NVIDIA H100 NVL — 95,830 MB VRAM
        17,995 MB / 95,830 MB used (18.8%)
        0% utilization at idle, 70°C, 122.98W
        Only non-MoE layers of 235B loaded on GPU

Disk:   / — 14G / 15G (94% FULL — CRITICAL)
        /apps — 463G / 500G (93% FULL — CRITICAL)
```

### 1.3 What This Tells Us

|Finding                               |Impact                                                                                                                              |
|--------------------------------------|------------------------------------------------------------------------------------------------------------------------------------|
|GPU at 18.8%                          |**77 GB of H100 VRAM sitting idle.** The entire H100 investment is wasted — 235B runs mostly on CPU via PCIe bus.                   |
|235B at 468% CPU, 128 GB RAM          |MoE expert routing hits PCIe bottleneck every token. Inference is ~15-25 tok/s when it should be ~60 tok/s on full GPU.             |
|Thread over-subscription (144t > 128c)|Context switches under load. When 235B + nomic + qwen2.5-coder run simultaneously, they fight for cores.                            |
|Disk at 93-94%                        |Cannot download new models (~73 GB for 122B). Must clean up first or pipeline cannot be upgraded.                                   |
|qwen2.5-coder-7b at 9.3 GB RAM        |Good model for headers, but Qwen3.5-9B is a generation newer with better code understanding and same-family compatibility with 122B.|
|qwen2.5-coder-1.5b at 2.8 GB RAM      |Very small — useful for ultra-fast classification but limited code understanding. May be replaceable.                               |
|CPU 99.88% idle                       |Models are waiting for requests, not saturated. The bottleneck is inference speed per request (latency), not throughput.            |

-----

## 2. Critical Issues — What’s Hurting Performance Now

### 2.1 Issue #1: GPU Waste → Slow Deep Inference

**Current:** Qwen3-235B uses 18 GB GPU + 128 GB CPU RAM. Every token generation requires PCIe round-trips to fetch MoE expert weights from CPU.

**Effect:** ~15-25 tok/s. A 2,000-token IMPLEMENT response takes 80-130 seconds. Developers wait over a minute for code suggestions.

**Fix:** Replace with Qwen3.5-122B-A10B Q4_K_M (~73 GB). Fits entirely on GPU. No PCIe bottleneck. ~60 tok/s. Same response in 33 seconds. Better benchmarks (SWE-bench 72.4 vs ~65, BFCL-V4 72.2 vs ~55).

### 2.2 Issue #2: Disk Full → Cannot Upgrade

**Current:** /apps at 463G/500G (93%). Qwen3-235B GGUF files are likely ~140 GB on disk.

**Fix:** Delete 235B model files after 122B is validated. Net savings: ~140 GB (235B) - 73 GB (122B) = ~67 GB freed. Also clean old bootstrap artifacts, git pack files, and log files.

### 2.3 Issue #3: Thread Over-Subscription

**Current:** 120 threads allocated to LLM processes + ~24 for Node.js/OS = 144 threads on 128 cores. Under concurrent load, all processes slow down.

**Fix:** Reallocate threads. With 235B gone, reclaim 16 threads. With Qwen3.5-9B replacing qwen2.5-coder-7b (same threads), total stays under 128.

### 2.4 Issue #4: Two Prompt Template Families

**Current:** Qwen2.5-coder uses ChatML template. Qwen3-235B uses Qwen3 template. Two different prompt formats maintained in code.

**Fix:** Qwen3.5-9B + Qwen3.5-122B share the same Qwen3.5 chat template and tokenizer. Single prompt format across all models.

### 2.5 Issue #5: No Structured Verification Pass

**Current:** Model generates code → immediate delivery to developer. No line number verification, no import checking, no convention validation.

**Fix:** Three-pass architecture — gather → generate → verify. Catches errors before they reach the developer.

-----

## 3. Model Upgrade Strategy

### 3.1 Before vs After

```
BEFORE (current):                           AFTER (target):
──────────────────────────                   ──────────────────────────
:8081  snowflake-arctic   775 MB   CPU       :8081  snowflake-arctic   775 MB   CPU  ← KEEP
:8085  nomic-embed-code   8,156 MB CPU       :8085  nomic-embed-code   8,156 MB CPU  ← KEEP
:8082  qwen2.5-coder-7b   9,325 MB CPU       :8082  qwen3.5-9b         ~7 GB   CPU  ← UPGRADE
:8084  qwen2.5-coder-1.5b 2,787 MB CPU       :8084  qwen2.5-coder-1.5b 2,787 MB CPU  ← KEEP (fast classification)
:8083  bge-reranker-v2-m3 637 MB   CPU       :8083  bge-reranker-v2-m3 637 MB   CPU  ← KEEP
:8090  qwen3-235b         128 GB   CPU+GPU   :8090  qwen3.5-122b       ~73 GB   GPU  ← UPGRADE
:3100  MCP Server         683 MB             :3100  MCP Server         683 MB        ← KEEP
:3200  Agent              122 MB             :3200  Agent              122 MB        ← KEEP
```

### 3.2 Why Each Decision

|Model                     |Decision                       |Reasoning                                                                                                                                                                                  |
|--------------------------|-------------------------------|-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
|snowflake-arctic (:8081)  |**Keep**                       |Prose embedding. Works well, no upgrade needed.                                                                                                                                            |
|nomic-embed-code (:8085)  |**Keep**                       |Code embedding. Best-in-class for code vectors. Future: GPU FP16 migration.                                                                                                                |
|qwen2.5-coder-7b (:8082)  |**Upgrade → Qwen3.5-9B**       |Same family as 122B (shared template). Better instruction following (91.2% IFEval). Better tool calling (66.1 BFCL). 262K context. Replaces contextual header generation + gains new tasks.|
|qwen2.5-coder-1.5b (:8084)|**Keep**                       |Ultra-fast (~40 tok/s CPU) for intent classification, scope check, simple yes/no. 2.8 GB RAM is negligible. Remove only if Qwen3.5-9B handles everything fast enough.                      |
|bge-reranker-v2-m3 (:8083)|**Keep**                       |Cross-encoder reranker. Critical for search accuracy. No better alternative at this size on CPU.                                                                                           |
|qwen3-235b (:8090)        |**Upgrade → Qwen3.5-122B-A10B**|Fits fully on GPU (73 GB < 95.8 GB). 3x faster inference. Better benchmarks across the board. Eliminates 128 GB CPU RAM usage.                                                             |

### 3.3 Resource Impact

```
                    BEFORE          AFTER           DELTA
                    ──────          ─────           ─────
GPU VRAM:           18 GB (18.8%)   ~79 GB (82%)    +61 GB utilized
CPU RAM (LLM):      ~150 GB         ~20 GB          -130 GB freed
Disk (/apps):       ~140 GB (235B)  ~73 GB (122B)   -67 GB freed
                    + ~5 GB (7B)    + ~6 GB (9B)    +1 GB
Total threads:      120t            104t            -16t (under 128c limit)
Inference speed:    ~15-25 tok/s    ~60 tok/s       3-4x faster
Prompt formats:     2 families      1 family        Simplified
```

-----

## 4. GPU Memory Reclamation — From 18% to 82%

### 4.1 Current GPU Layout

```
H100 NVL — 95,830 MB
──────────────────────────────
qwen3-235b non-MoE layers:     ~18 GB
KV Cache (32K × 2 slots):     ~0 GB (unused at idle)
──────────────────────────────
Used:                          ~18 GB (18.8%)
Wasted:                        ~78 GB
```

### 4.2 Target GPU Layout

```
H100 NVL — 95,830 MB
──────────────────────────────
qwen3.5-122b Q4_K_M (full):   ~73 GB   (ALL layers on GPU)
KV Cache (32K × 3 slots):     ~6 GB
──────────────────────────────
Used:                          ~79 GB (82%)
Headroom:                      ~17 GB (for burst/overhead)
```

### 4.3 Deployment Command — Qwen3.5-122B

```bash
# /etc/systemd/system/ome-gpu-gen.service
[Service]
ExecStart=/opt/llama-server-cuda \
  --model /apps/models/qwen3.5-122b-a10b/Qwen3.5-122B-A10B-Q4_K_M.gguf \
  --host 127.0.0.1 --port 8090 \
  --n-gpu-layers 99 \
  --ctx-size 32768 \
  --parallel 3 \
  --threads 8 \
  --flash-attn \
  --jinja \
  --reasoning-format deepseek \
  --temp 0.6 --top-p 0.95 --top-k 20 --min-p 0.0
```

Key differences from current 235B deployment:

- `--n-gpu-layers 99` — all layers on GPU (no `-ot ".ffn_.*_exps.=CPU"`)
- `--parallel 3` — up from 2 (model fits, so more concurrent slots)
- `--threads 8` — down from 16 (GPU does compute, CPU only does I/O)
- No CPU offload → no PCIe bottleneck → ~60 tok/s

-----

## 5. Thread Allocation — Eliminate Over-Subscription

### 5.1 Current: 144 Threads > 128 Cores (Over-Subscribed)

```
Text Embed:   16t
Code Embed:   32t
Gen 7B:       24t    ← heavy
Qwen 1.5B:    16t
Reranker:     16t
GPU Gen:      16t    ← unnecessary on GPU model
──────────────────
LLM total:    120t
Node.js:      ~16t
OS:           ~8t
──────────────────
TOTAL:        ~144t  > 128 cores ← PROBLEM
```

### 5.2 Target: 108 Threads ≤ 128 Cores (Clean)

```
Text Embed:   16t    ← keep (light model, occasional use)
Code Embed:   32t    ← keep (heavy model, critical path)
Qwen3.5-9B:  24t    ← same as current 7B slot
Qwen 1.5B:   16t    ← keep (fast classifier)
Reranker:    16t    ← keep (search critical path)
GPU Gen 122B: 8t     ← reduced (GPU does all compute, CPU only for I/O)
──────────────────
LLM total:   112t
Node.js:     ~12t    ← tune down (async I/O, doesn't need many)
OS:          ~4t
──────────────────
TOTAL:       ~128t  = 128 cores ← CLEAN FIT
```

### 5.3 Thread Tuning Per Model

```bash
# Text Embed (snowflake-arctic, :8081) — 16t, 4 slots
# Light model, infrequent queries. 16t is fine.
--threads 16 --parallel 4

# Code Embed (nomic-embed-code, :8085) — 32t, 4 slots
# Heavy model (7B GGUF), critical path for every search query.
# 32t justified by batch embedding throughput.
--threads 32 --parallel 4

# Qwen3.5-9B (:8082) — 24t, 4 slots (up from 8 on qwen2.5-coder-7b)
# Handles contextual headers (high volume during bootstrap)
# + convention checks + ticket enrichment during pipeline
--threads 24 --parallel 4

# Qwen2.5-coder-1.5B (:8084) — 16t, 4 slots
# Ultra-fast intent classification. Keep as-is.
--threads 16 --parallel 4

# Reranker (bge-reranker-v2-m3, :8083) — 16t
# Cross-encoder, sequential by nature. 16t is appropriate.
--threads 16

# GPU Gen (qwen3.5-122b, :8090) — 8t, 3 slots
# GPU does ALL matrix math. CPU threads only handle:
#   - HTTP request/response
#   - Tokenization (fast)
#   - JSON parsing
# 8 threads is sufficient. More would waste cores.
--threads 8 --parallel 3
```

-----

## 6. Disk Space Recovery

### 6.1 Current Crisis

```
/     — 14G / 15G (94%)   ← system partition, dangerously full
/apps — 463G / 500G (93%)  ← models + repos + data, dangerously full
```

### 6.2 Recovery Plan

```bash
# Step 1: Identify large files
du -sh /apps/models/* | sort -rh | head -20
du -sh /apps/ome_rag/repos/*/.git | sort -rh | head -20
find /apps -name "*.log" -size +100M -exec ls -lh {} \;

# Step 2: After 122B is validated, remove 235B model files
# qwen3-235b GGUF is likely 3 shard files × ~47 GB each = ~140 GB
rm -f /apps/models/qwen3-235b/*.gguf
# Expected savings: ~140 GB

# Step 3: Git garbage collection on repos
for repo in /apps/ome_rag/repos/*/*; do
  git -C "$repo" gc --aggressive --prune=now 2>/dev/null
done
# Expected savings: ~5-15 GB

# Step 4: Clean old logs and temp files
find /apps -name "*.log" -mtime +30 -delete
find /tmp -name "ome-*" -mtime +7 -delete
# Expected savings: ~2-5 GB

# Step 5: Clean / partition
sudo journalctl --vacuum-size=500M
sudo dnf clean all
# Expected savings: ~1-3 GB

# Net after migration:
# /apps: 463 - 140 (235B) + 73 (122B) + 6 (9B) - 5 (7B) - 10 (cleanup) = ~387 GB
# /apps: 387G / 500G (77%) ← healthy
```

### 6.3 Download Sequence (Must Free Space First)

```bash
# Cannot download 122B (73 GB) when /apps is at 93%
# Must delete 235B files first, OR use /tmp as staging

# Option A: Download to /tmp (if /tmp has space on separate partition)
# Option B: Download 122B first (if 37 GB free is enough for streaming download)
# Option C: Delete 235B, download 122B (brief downtime for deep model)

# Recommended: Option C — 10-minute planned downtime
systemctl stop ome-gpu-gen
rm -f /apps/models/qwen3-235b/*.gguf           # +140 GB free
aria2c -x 16 -s 16 --continue=true \
  -d /apps/models/qwen3.5-122b/ \
  "<nexus-proxy-url>/Qwen3.5-122B-A10B-Q4_K_M.gguf"  # -73 GB
# Update systemd service to point to new model
systemctl start ome-gpu-gen
```

-----

## 7. Three-Pass Accuracy Architecture

### 7.1 Why Single-Pass Fails

Current flow: Developer asks → Agent calls ome_rag_guide → 235B generates code in one shot → return.

Problems with one-shot generation:

- Model guesses line numbers from truncated chunk context (not full files)
- No verification that generated code compiles
- No check that referenced symbols actually exist
- No convention enforcement (model uses training patterns, not team patterns)
- Missing imports not caught until developer tries to compile

### 7.2 Three-Pass Design

```
Pass 1: GATHER (Qwen3.5-9B + RAG Tools, ~5s)
  ├── Parallel RAG MCP calls (8-12 tools)
  ├── Read FULL target files with line numbers from REPOS_DIR
  ├── Convention samples from sibling files
  ├── Graph context (callers, callees, signatures)
  ├── Similar PRs (vector + FTS hybrid search)
  ├── Blast radius traversal
  └── Output: ContextBundle (~20K tokens, assembled for 122B)

Pass 2: GENERATE (Qwen3.5-122B on GPU, ~25s)
  ├── Single inference call with layered prompt
  ├── System prompt (rules + constraints)
  ├── Framework context (Spring Boot / React specifics)
  ├── Convention injection (team's actual patterns)
  ├── Full target files with numbered lines
  ├── Graph context (compiler-verified signatures)
  ├── Strict JSON output schema
  └── Output: Raw CodeChange[] + NewFile[] + ImportChange[]

Pass 3: VERIFY (Qwen3.5-9B + filesystem + SCIP, ~8s)
  ├── File existence check (every changed file on disk)
  ├── Line content verification (currentCode matches actual file)
  ├── Fuzzy auto-correction (if content shifted lines)
  ├── Import completeness (extract used symbols → verify imports)
  ├── Symbol existence (graph + SCIP verification)
  ├── Convention violation check (9B compares against detected rules)
  ├── Scope boundary check (changes within blast radius)
  ├── Content hash generation (MD5 for execution-time re-verification)
  └── Output: Verified CodeChange[] with per-change confidence scores

Total: ~38 seconds (vs ~80-130 seconds current single-pass on 235B)
```

-----

## 8. Pass 1 — Context Gathering (Tool Orchestration)

### 8.1 Three-Wave Parallel Execution

```typescript
// src/guide/gather/gather_executor.ts

// ── Wave 1: Independent queries (all parallel, ~2s) ──
const wave1 = await Promise.allSettled([
  rag.searchCode(query, { topK: 10, repo }),       // Semantic search
  rag.getProjectInfo(repo),                         // Tech stack, deps
  rag.getTicketContext(ticketKey),                   // Jira enrichment
  searchSimilarPRs(query, ctx),                     // PR vector search
]);

// ── Wave 2: Depends on Wave 1 (parallel within wave, ~3s) ──
const targetFiles = extractFilePaths(wave1.searchResults);
const wave2 = await Promise.allSettled([
  batchReadFilesWithLineNumbers(targetFiles, repo),  // FULL files, numbered
  ...symbols.map(s => rag.getCodeContext(s, 2)),     // Call chains
  ...symbols.map(s => rag.analyzeBlastRadius(s)),    // Impact analysis
  ...targetFiles.map(f => rag.getFileDependencies(f)), // Imports/imported-by
]);

// ── Wave 3: Convention sampling (~1s) ──
const conventionFiles = await gatherConventionSamples(
  targetFiles, framework, repo
);
```

### 8.2 Full File Reading — The #1 Accuracy Decision

```typescript
// CRITICAL: Always read full files for targets, never truncated chunks

export async function readFileWithLineNumbers(
  filePath: string, repoSlug: string
): Promise<NumberedFile> {
  const fullPath = path.join(REPOS_DIR, repoSlug, filePath);
  const content = await fs.readFile(fullPath, 'utf-8');
  const lines = content.split('\n');

  // Line-numbered format for the model
  const numbered = lines
    .map((line, i) => `${String(i + 1).padStart(4)}│ ${line}`)
    .join('\n');

  return {
    path: filePath,
    content,                    // Raw for hash verification
    numberedContent: numbered,  // Numbered for model prompt
    lineCount: lines.length,
    contentHash: md5(content),
  };
}
```

Why this matters: When the model sees `  42│ public void submitOrder(OrderRequest req) {`, it generates `"startLine": 42` — exact, not guessed from a truncated chunk.

### 8.3 Convention Sampling

```typescript
// Read 3-5 sibling files from the same directory as target files
// These show the team's actual patterns (naming, error handling, DI style)

async function gatherConventionSamples(
  targetFiles: string[], framework: Framework, repo: string
): Promise<ConventionFile[]> {
  const samples: ConventionFile[] = [];

  for (const target of targetFiles.slice(0, 3)) {
    const dir = path.dirname(target);
    const siblings = await listDirectory(dir, repo);

    // Same-type files in same directory = strongest convention signal
    const sameType = siblings
      .filter(f => f !== path.basename(target))
      .filter(f => matchesExtension(f, target))
      .slice(0, 2);

    for (const sibling of sameType) {
      const content = await readFileWithLineNumbers(
        path.join(dir, sibling), repo
      );
      if (content.lineCount <= 200) {
        samples.push({
          path: path.join(dir, sibling),
          content: content.numberedContent,
          relationship: 'same_directory',
        });
      }
    }
  }

  return samples.slice(0, 5);
}
```

### 8.4 Smart Tool Selection by Ticket Type

```typescript
export function buildSmartGatherPlan(
  ticketType: string,
  complexity: string,
  elements: ExtractedElements
): GatherPlan {
  const plan = baseGatherPlan();

  if (ticketType === 'Bug') {
    // Bug: narrow scope, need recent changes + error context
    plan.add('get_recent_changes', { repo: elements.repo, limit: 5 });
    plan.semanticSearch.topK = 5;
    plan.blastRadius.maxDepth = 1;
    plan.conventionSample.maxFiles = 2;
  }

  if (ticketType === 'Story') {
    // Feature: broad scope, need more convention + historical context
    plan.semanticSearch.topK = 10;
    plan.blastRadius.maxDepth = 2;
    plan.conventionSample.maxFiles = 5;
    plan.similarPRs.maxResults = 3;
  }

  if (complexity === 'small') {
    plan.fileReads.maxFiles = 3;
    plan.conventionSample.maxFiles = 2;
  }

  if (complexity === 'large') {
    plan.fileReads.maxFiles = 8;
    plan.add('search_architecture', { repo: elements.repo });
  }

  return plan;
}
```

-----

## 9. Pass 2 — Code Generation (Prompt Engineering)

### 9.1 Prompt Structure — Layered

```
┌─────────────────────────────────────────────┐
│  SYSTEM PROMPT (role + constraints + rules)  │  400 tokens
├─────────────────────────────────────────────┤
│  FRAMEWORK CONTEXT (Spring Boot / React)     │  200 tokens
├─────────────────────────────────────────────┤
│  CONVENTIONS (from team's actual code)       │  1,500 tokens
├─────────────────────────────────────────────┤
│  TICKET REQUIREMENT (what to implement)      │  500 tokens
├─────────────────────────────────────────────┤
│  TARGET FILES (full, with line numbers)      │  12,000 tokens
├─────────────────────────────────────────────┤
│  REFERENCE FILES (similar patterns)          │  3,000 tokens
├─────────────────────────────────────────────┤
│  GRAPH CONTEXT (callers, callees, types)     │  1,000 tokens
├─────────────────────────────────────────────┤
│  SIMILAR PRs (how team solved this before)   │  800 tokens
├─────────────────────────────────────────────┤
│  BLAST RADIUS (what breaks if wrong)         │  300 tokens
├─────────────────────────────────────────────┤
│  OUTPUT FORMAT (strict JSON schema)          │  800 tokens
└─────────────────────────────────────────────┘
                                        Total: ~20,500 tokens
                                        Leaves ~11K for generation in 32K ctx
```

### 9.2 System Prompt — Accuracy Rules

```typescript
export function buildSystemPrompt(framework: Framework): string {
  return `You are a senior ${frameworkLabel(framework)} engineer implementing precise code changes.

RULES — violating any makes your output unusable:

1. LINE NUMBERS MUST BE EXACT. Use the EXACT line numbers from the numbered files below. Never guess.

2. currentCode MUST BE VERBATIM. Copy the exact text between startLine and endLine. Byte-for-byte.

3. IMPORTS MUST BE COMPLETE. If proposedCode uses any symbol not imported in the file, add it to importChanges.

4. SCOPE MUST BE MINIMAL. Only change what the ticket requires. Do not refactor, rename, or "improve" unrelated code.

5. ONE CHANGE PER BLOCK. Each change modifies one continuous block of lines.

6. MATCH CONVENTIONS. The <conventions> section shows your team's actual style. Follow it exactly.

7. RESPOND WITH JSON ONLY. No markdown, no explanation, no preamble.`;
}
```

### 9.3 Framework-Specific Context

```typescript
const FRAMEWORK_CONTEXTS = {
  spring_boot: `<framework>
Spring Boot microservice.
Pattern: Controller → Service → Repository.
DI: Constructor injection only.
Validation: @Valid on request bodies.
Transactions: @Transactional on write methods.
Errors: @ControllerAdvice with domain exceptions.
Tests: @SpringBootTest + @MockBean.
</framework>`,

  react: `<framework>
React micro-frontend (Module Federation).
Pattern: Component → Hook → Service/API.
State: React hooks only. No class components.
API: Custom hooks wrapping fetch with error handling.
Routing: React Router v6+ with lazy loading.
Tests: Vitest or Jest + @testing-library/react.
Error boundaries: Required at MFE root.
</framework>`,
};
```

### 9.4 Convention Injection

```typescript
export function buildConventionSection(conventions: ConventionSnapshot): string {
  if (conventions.files.length === 0) return '';

  return `<conventions>
These files show your team's patterns. Match them exactly.

${conventions.files.map(f =>
  `--- ${f.path} (${f.relationship}) ---\n${f.content}`
).join('\n\n')}

${conventions.detectedPatterns.length > 0
  ? `Detected patterns:\n${conventions.detectedPatterns.map(p => `- ${p}`).join('\n')}`
  : ''}
</conventions>`;
}
```

### 9.5 Target Files with Line Numbers

```typescript
export function buildTargetFileSection(files: NumberedFile[]): string {
  return `<target_files>
Files to modify. Use EXACT line numbers from the left column.

${files.map(f =>
  `=== ${f.path} (${f.lineCount} lines, hash: ${f.contentHash}) ===\n${f.numberedContent}`
).join('\n\n')}
</target_files>`;
}
```

### 9.6 Output Format — Strict JSON

```typescript
export function buildOutputFormat(): string {
  return `<output_format>
Respond with a single JSON object. No markdown. No explanation.

{
  "summary": "One paragraph describing all changes",
  "changes": [
    {
      "id": "change-001",
      "file": "exact/path/from/target_files.ts",
      "action": "modify",
      "description": "What this change does",
      "reason": "Which ticket requirement this satisfies",
      "currentCode": {
        "startLine": 42,
        "endLine": 48,
        "content": "VERBATIM text from numbered file at these lines"
      },
      "proposedCode": "Replacement code with matching indentation",
      "affectedSymbols": ["functionName"]
    }
  ],
  "newFiles": [
    { "path": "src/new/file.ts", "content": "Complete content", "reason": "Why needed" }
  ],
  "importChanges": [
    { "file": "path.ts", "action": "add", "importStatement": "import { X } from './x'", "insertAfterLine": 3 }
  ],
  "executionOrder": ["change-001", "change-002"]
}

RULES:
- file must match a path from <target_files>
- startLine/endLine must match numbered lines exactly
- currentCode.content must be byte-for-byte match
- proposedCode must maintain same indentation
- importChanges must list ALL new imports needed
</output_format>`;
}
```

-----

## 10. Pass 3 — Verification & Self-Correction

### 10.1 Six Parallel Verification Checks

```typescript
// src/guide/verify/verification_pipeline.ts

export async function verifyChanges(
  output: GeneratedOutput,
  context: ContextBundle,
  ctx: AgentContext
): Promise<VerifiedOutput> {

  // All checks run in parallel — they're independent
  const [fileCheck, lineCheck, importCheck, symbolCheck, scopeCheck, conventionCheck] =
    await Promise.allSettled([
      verifyFileExistence(output, ctx),
      verifyLineContent(output, context),
      verifyImportCompleteness(output, context, ctx),
      verifySymbolExistence(output, ctx),
      checkScopeBoundary(output, context),
      checkConventionViolations(output, context.conventions, ctx),
    ]);

  // Merge results
  const results = [
    ...extractResults(fileCheck),
    ...extractResults(lineCheck),
    ...extractResults(importCheck),
    ...extractResults(symbolCheck),
    ...extractResults(scopeCheck),
    ...extractResults(conventionCheck),
  ];

  // Auto-correct where possible (line shifts, missing imports)
  const corrected = applyCorrections(output, results);

  // Compute confidence
  const fatal = results.filter(r => r.severity === 'fatal').length;
  const warnings = results.filter(r => r.severity === 'warning').length;
  const correctedCount = results.filter(r => r.severity === 'corrected').length;

  const score = Math.max(0, 1.0 - (fatal * 0.2) - (warnings * 0.05) - (correctedCount * 0.02));

  return {
    changes: corrected,
    confidence: {
      score,
      level: score >= 0.85 ? 'high' : score >= 0.6 ? 'medium' : 'low',
      fatalIssues: fatal,
      warnings,
      autoCorrected: correctedCount,
    },
    verificationResults: results,
  };
}
```

### 10.2 Line Content Verification with Auto-Correction

```typescript
async function verifyLineContent(
  output: GeneratedOutput,
  context: ContextBundle
): Promise<VerificationResult[]> {
  const results: VerificationResult[] = [];

  for (const change of output.changes) {
    if (!change.currentCode) continue;

    const file = context.targetFiles.find(f => f.path === change.file);
    if (!file) continue;

    const actualLines = file.content.split('\n')
      .slice(change.currentCode.startLine - 1, change.currentCode.endLine)
      .join('\n');

    if (actualLines.trim() === change.currentCode.content.trim()) {
      results.push({ changeId: change.id, check: 'line_content', passed: true, severity: 'ok' });
      continue;
    }

    // Content doesn't match — try fuzzy find (content may have shifted)
    const fuzzy = fuzzyFindInFile(file.content, change.currentCode.content, 0.85);

    if (fuzzy) {
      // Auto-correct line numbers
      change.currentCode.startLine = fuzzy.startLine;
      change.currentCode.endLine = fuzzy.endLine;
      change.currentCode.content = fuzzy.matchedContent;
      results.push({
        changeId: change.id,
        check: 'line_content',
        passed: true,
        severity: 'corrected',
        warning: `Lines shifted: now ${fuzzy.startLine}-${fuzzy.endLine}`,
      });
    } else {
      results.push({
        changeId: change.id,
        check: 'line_content',
        passed: false,
        severity: 'fatal',
        error: `Content at L${change.currentCode.startLine}-${change.currentCode.endLine} does not match`,
      });
    }
  }

  return results;
}
```

### 10.3 Import Auto-Resolution

```typescript
async function verifyImportCompleteness(
  output: GeneratedOutput,
  context: ContextBundle,
  ctx: AgentContext
): Promise<VerificationResult[]> {
  const results: VerificationResult[] = [];

  for (const change of output.changes) {
    if (!change.proposedCode) continue;

    // Extract symbols used in proposed code
    const usedSymbols = extractUsedSymbols(change.proposedCode, context.framework);
    const file = context.targetFiles.find(f => f.path === change.file);
    if (!file) continue;

    const existingImports = parseImports(file.content, context.framework);

    for (const symbol of usedSymbols) {
      if (isBuiltinType(symbol, context.framework)) continue;
      if (isDefinedInFile(symbol, file.content)) continue;

      const imported = existingImports.some(imp =>
        imp.symbols.includes(symbol) || imp.default === symbol
      );
      const addedImport = output.importChanges?.some(ic =>
        ic.file === change.file && ic.importStatement.includes(symbol)
      );

      if (!imported && !addedImport) {
        // Try to find source via knowledge graph
        const symbolInfo = await ctx.rag.findSymbol(symbol);

        if (symbolInfo?.length > 0) {
          const relativePath = computeRelativePath(change.file, symbolInfo[0].file_path);
          const importStmt = buildImportStatement(symbol, relativePath, context.framework);

          // Auto-add missing import
          if (!output.importChanges) output.importChanges = [];
          output.importChanges.push({
            file: change.file,
            action: 'add',
            importStatement: importStmt,
            insertAfterLine: findLastImportLine(file.content),
          });

          results.push({
            changeId: change.id, check: 'import',
            passed: true, severity: 'corrected',
            warning: `Auto-added: ${importStmt}`,
          });
        } else {
          results.push({
            changeId: change.id, check: 'import',
            passed: false, severity: 'warning',
            error: `'${symbol}' used but not imported, not found in graph`,
          });
        }
      }
    }
  }

  return results;
}
```

-----

## 11. Model Router — Three-Tier Task Routing

### 11.1 Final Architecture with Current Port Layout

```
┌─────────────────────────────────────────────────────────┐
│  TIER 1 — GPU (Qwen3.5-122B-A10B, :8090)                │
│  ~60 tok/s, 3 parallel slots, 32K context                │
│                                                          │
│  Tasks:                                                  │
│  • Code generation (IMPLEMENT mode)                      │
│  • Convention DETECTION (extract patterns from code)     │
│  • DIAGNOSE analysis + fix generation                    │
│  • UNDERSTAND explanations                               │
│  • Multi-file coherence prompts                          │
│  • PR description generation (code-aware summary)        │
└─────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────┐
│  TIER 2 — CPU (Qwen3.5-9B, :8082)                        │
│  ~15 tok/s, 4 parallel slots, 32K context                │
│                                                          │
│  Tasks:                                                  │
│  • Contextual header generation (bootstrap, existing)    │
│  • Convention violation CHECK (compare vs rules)         │
│  • Ticket enrichment inference (sparse ticket → intent)  │
│  • Import verification prompts (symbol lookup assist)    │
│  • Feedback categorization (classify rejection reasons)  │
│  • Follow-up suggestion generation                       │
│  • Scope creep detection prompts                         │
└─────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────┐
│  TIER 3 — CPU (Qwen2.5-coder-1.5B, :8084)               │
│  ~40 tok/s, 4 parallel slots, 8K context                 │
│                                                          │
│  Tasks:                                                  │
│  • Intent classification (10-20 tokens, <0.5s)           │
│  • Scope yes/no classification (10 tokens)               │
│  • Risk level classification (10 tokens)                 │
│  • Ambiguous intent resolution                           │
│  • Glossary term matching                                │
└─────────────────────────────────────────────────────────┘
```

### 11.2 Routing Logic

```typescript
// src/llm/model_router.ts

export type ModelTier = 'gpu_deep' | 'cpu_mid' | 'cpu_fast';

export function routeTask(mode: GuideMode, task: ModelTask): RoutingDecision {
  // ── TIER 3: cpu_fast (:8084, 1.5B) — <0.5s, classification only ──
  if (task === 'intent_classify') return tier3(20, 0.0);
  if (task === 'scope_check_binary') return tier3(10, 0.0);
  if (task === 'risk_classify') return tier3(10, 0.0);
  if (task === 'ambiguity_resolve') return tier3(20, 0.0);

  // ── TIER 2: cpu_mid (:8082, 9B) — 1-5s, code-aware lightweight ──
  if (task === 'contextual_header') return tier2(300, 0.2);
  if (task === 'convention_violation_check') return tier2(200, 0.1);
  if (task === 'ticket_enrich') return tier2(300, 0.1);
  if (task === 'import_verify') return tier2(100, 0.0);
  if (task === 'feedback_categorize') return tier2(10, 0.0);
  if (task === 'followup_suggest') return tier2(200, 0.5);
  if (task === 'scope_creep_analyze') return tier2(100, 0.1);

  // ── TIER 1: gpu_deep (:8090, 122B) — 15-30s, precision tasks ──
  if (mode === 'implement') return tier1(8000, 0.15);
  if (mode === 'understand') return tier1(1500, 0.3);
  if (mode === 'diagnose' && task === 'analyze') return tier1(1000, 0.2);
  if (mode === 'diagnose' && task === 'generate_fix') return tier1(4000, 0.15);
  if (task === 'convention_detect') return tier1(400, 0.2);
  if (task === 'coherence_check') return tier1(500, 0.1);
  if (task === 'pr_description') return tier1(800, 0.3);

  return tier2(500, 0.3); // Default to mid-tier
}

function tier1(maxTokens: number, temp: number): RoutingDecision {
  return { model: 'gpu_deep', port: 8090, maxTokens, temperature: temp };
}
function tier2(maxTokens: number, temp: number): RoutingDecision {
  return { model: 'cpu_mid', port: 8082, maxTokens, temperature: temp };
}
function tier3(maxTokens: number, temp: number): RoutingDecision {
  return { model: 'cpu_fast', port: 8084, maxTokens, temperature: temp };
}
```

### 11.3 Graceful Degradation

```typescript
// If a tier is unhealthy, fall back to next tier up

async function routeWithFallback(task: ModelTask, mode: GuideMode): Promise<RoutingDecision> {
  const primary = routeTask(mode, task);

  if (await isHealthy(primary.port)) return primary;

  // Fallback chain: tier3 → tier2 → tier1
  if (primary.model === 'cpu_fast') {
    if (await isHealthy(8082)) return { ...primary, model: 'cpu_mid', port: 8082 };
    if (await isHealthy(8090)) return { ...primary, model: 'gpu_deep', port: 8090 };
  }
  if (primary.model === 'cpu_mid') {
    if (await isHealthy(8090)) return { ...primary, model: 'gpu_deep', port: 8090 };
  }

  throw new Error(`No healthy model available for task ${task}`);
}
```

-----

## 12. Convention Detection & Enforcement

### 12.1 Two-Stage Process

```
Stage 1: DETECT (122B on GPU, once per repo/directory, cached 24h)
  Input: 3-5 sibling files
  Output: JSON array of convention strings
  Why 122B: Needs deep code understanding to extract patterns
  Cache key: {repo}:{directory}:{HEAD_hash}

Stage 2: CHECK (9B on CPU, every generation)
  Input: Generated code + detected conventions
  Output: Violation list (if any)
  Why 9B: Comparing output against explicit rules is lightweight
  Action: If violations found → single retry with violation feedback to 122B
```

### 12.2 Convention Detection Prompt (122B)

```typescript
export function buildConventionDetectionPrompt(
  files: ConventionFile[], framework: Framework
): string {
  return `Analyze these ${frameworkLabel(framework)} files and extract coding conventions.

${files.map(f => `--- ${f.path} ---\n${f.content}`).join('\n\n')}

Extract ONLY conventions visible in the code. Do not invent patterns.
Respond as a JSON array of strings.

Example: ["Constructor injection only, never @Autowired on fields",
 "Service methods use @Transactional(readOnly=true) for queries",
 "Exceptions extend BaseApplicationException"]`;
}
```

### 12.3 Convention Violation Check Prompt (9B)

```typescript
export function buildViolationCheckPrompt(
  conventions: string[],
  generatedCode: string
): string {
  return `Check if this code follows all conventions. List ONLY actual violations.
If no violations, respond with [].

Conventions:
${conventions.map((c, i) => `${i + 1}. ${c}`).join('\n')}

Generated code:
${generatedCode}

Respond with a JSON array of violation strings, or [].`;
}
```

### 12.4 Convention Cache

```typescript
const CONVENTION_CACHE = new Map<string, {
  conventions: string[];
  headHash: string;
  detectedAt: number;
}>();

const CACHE_TTL = 24 * 60 * 60 * 1000; // 24 hours

export async function getConventions(
  repo: string, directory: string, framework: Framework, ctx: AgentContext
): Promise<string[]> {
  const headHash = await getRepoHeadHash(repo);
  const key = `${repo}:${directory}`;
  const cached = CONVENTION_CACHE.get(key);

  // Valid if same HEAD and within TTL
  if (cached && cached.headHash === headHash && Date.now() - cached.detectedAt < CACHE_TTL) {
    return cached.conventions;
  }

  // Detect via 122B (GPU)
  const samples = await gatherConventionSamples([directory + '/*'], framework, repo);
  const prompt = buildConventionDetectionPrompt(samples, framework);
  const response = await llmCall(prompt, routeTask('implement', 'convention_detect'));
  const conventions = parseJSONArray(response);

  CONVENTION_CACHE.set(key, { conventions, headHash, detectedAt: Date.now() });
  return conventions;
}
```

-----

## 13. Hallucination Prevention

### 13.1 Four Verification Layers

```
Layer 1: Filesystem
  ├── Every change.file checked at REPOS_DIR/{repo}/{file}
  ├── Every newFile.path directory exists or follows existing structure
  └── Severity: FATAL if file missing (except for new files)

Layer 2: Content Hash
  ├── change.currentCode.content matched against actual file bytes
  ├── MD5 hash computed for execution-time re-verification
  ├── Auto-correct: fuzzy match if content shifted (>85% word overlap)
  └── Severity: FATAL if no match found

Layer 3: Symbol Graph
  ├── change.affectedSymbols checked via IGraphClient.findSymbol()
  ├── Signatures compared against GraphNode.signature
  └── Severity: WARNING (could be new symbol)

Layer 4: SCIP (when operational)
  ├── Symbol resolution: does definition exist?
  ├── Call chain: does this call path exist?
  ├── Type hierarchy: does type implement claimed interface?
  ├── Content freshness: is SCIP index current with HEAD?
  └── Severity: Gate decision (proceed / caution / confirm / abort)
```

### 13.2 Fuzzy Line Matching

```typescript
export function fuzzyFindInFile(
  fileContent: string,
  targetContent: string,
  threshold: number = 0.85
): { startLine: number; endLine: number; matchedContent: string } | null {
  const fileLines = fileContent.split('\n');
  const targetLines = targetContent.trim().split('\n');
  const targetLen = targetLines.length;

  // Sliding window search
  for (let i = 0; i <= fileLines.length - targetLen; i++) {
    const window = fileLines.slice(i, i + targetLen).join('\n');
    const similarity = computeWordOverlap(window.trim(), targetContent.trim());

    if (similarity >= threshold) {
      return {
        startLine: i + 1,
        endLine: i + targetLen,
        matchedContent: window,
      };
    }
  }

  return null;
}
```

-----

## 14. Import Resolution & Multi-File Coherence

### 14.1 Import Resolution Pipeline

```typescript
// After generation, before returning to developer:
// 1. Extract all symbols used in proposedCode
// 2. Check if each symbol is imported or defined locally
// 3. If missing: query findSymbol() → auto-add import
// 4. If not found in graph: flag as warning

// Symbol extraction is framework-aware:
// TypeScript: import { X } from, import X from, require()
// Java: import com.bmt.*, @Autowired Type name
```

### 14.2 Multi-File Coherence Checks

```typescript
export async function checkCoherence(output: GeneratedOutput, context: ContextBundle) {
  const issues: string[] = [];

  // 1. Type changes: if interface changes in file A, usages in B must match
  const typeChanges = extractTypeChanges(output);
  for (const [type, newDef] of typeChanges) {
    for (const change of output.changes) {
      if (change.proposedCode?.includes(type) && change.file !== newDef.file) {
        const consistent = validateTypeUsage(change.proposedCode, type, newDef);
        if (!consistent) issues.push(`Type '${type}' inconsistent in ${change.file}`);
      }
    }
  }

  // 2. Signature changes: callers must pass new parameters
  const sigChanges = extractSignatureChanges(output);
  for (const [func, newSig] of sigChanges) {
    const callers = context.graphContext.get(func)?.callers || [];
    for (const caller of callers) {
      const callerUpdated = output.changes.some(c => c.affectedSymbols.includes(caller));
      if (!callerUpdated && context.blastRadius.directlyAffected.includes(caller)) {
        issues.push(`'${func}' signature changed but caller '${caller}' not updated`);
      }
    }
  }

  // 3. Export check: imports must reference actual exports
  for (const ic of output.importChanges || []) {
    const symbol = extractImportedSymbol(ic.importStatement);
    const sourceFile = resolveImportPath(ic.file, ic.importStatement);
    if (sourceFile) {
      const source = context.targetFiles.find(f => f.path === sourceFile);
      if (source && !extractExports(source.content).includes(symbol)) {
        issues.push(`Import '${symbol}' from '${sourceFile}' but not exported`);
      }
    }
  }

  return { coherent: issues.length === 0, issues };
}
```

-----

## 15. Human Checkpoint System

### 15.1 Confidence-Gated Actions

```
Confidence ≥ 0.9 AND risk = low:
  → Auto-approve eligible (if ALLOW_AUTO_APPROVE=true)
  → Still creates PR for human code review

Confidence 0.7 - 0.89:
  → Requires human plan approval
  → Normal flow

Confidence 0.5 - 0.69:
  → Requires human approval
  → WARNING banner + specific scrutiny areas

Confidence < 0.5:
  → Pipeline ABORTS
  → Jira comment: "Too complex for automation"
  → Suggest using ome_rag_guide in IDE manually
```

### 15.2 Plan Review UI

```
GET /api/pipeline/review/:runId

Shows:
├── Ticket context (enriched description, parent epic)
├── Change plan with syntax-highlighted diffs
├── Per-change confidence scores
├── Blast radius visualization
├── Convention compliance status
├── Verification results (what was auto-corrected)
├── Similar PRs referenced
└── Actions: APPROVE | REQUEST CHANGES | REJECT
```

### 15.3 Safety Invariants

1. No execution without plan approval (unless auto-approve criteria met)
1. Content hash re-verification at execution time (not just at plan time)
1. 48-hour plan expiry (prevents stale plans executing on changed code)
1. Single auto-fix retry on test failure, then abort
1. No force push, no direct push to main/develop
1. Full audit trail — every state transition logged
1. Kill switch: `PIPELINE_ENABLED=false` stops all new runs

-----

## 16. Performance Budget — Before vs After

### 16.1 IMPLEMENT Mode Latency

```
BEFORE (current 235B on CPU+GPU):
  Intent classification:    ~1.0s  (1.5B on :8084)
  RAG search + gather:      ~3.0s  (MCP tools)
  Prompt assembly:          ~0.1s
  235B inference (2K out):  ~80-130s  (15-25 tok/s, PCIe bottleneck)
  No verification pass:     0s
  ─────────────────────────────────
  Total:                    ~85-135 seconds

AFTER (122B full GPU + three-pass):
  Pass 1 — Gather:
    Intent (1.5B :8084):    ~0.3s
    Wave 1 (4 parallel):    ~2.0s
    Wave 2 (6 parallel):    ~3.0s
    Wave 3 (convention):    ~1.0s
    Subtotal:               ~6.3s

  Pass 2 — Generate:
    Prompt assembly:        ~0.1s
    122B input (20K):       ~10s   (prompt processing)
    122B output (2K):       ~33s   (60 tok/s)
    JSON parse:             ~0.1s
    Subtotal:               ~43s

  Pass 3 — Verify:
    6 parallel checks:      ~3.0s
    Convention check (9B):  ~3.0s
    Import resolution:      ~2.0s
    Subtotal:               ~8.0s

  ─────────────────────────────────
  Total:                    ~57 seconds
  Improvement:              ~40-55% faster WITH verification
```

### 16.2 UNDERSTAND Mode Latency

```
BEFORE:  ~15-30s (235B generating 1.5K tokens)
AFTER:   ~12-18s (122B at 60 tok/s)
         Improvement: ~30% faster
```

### 16.3 Resource Utilization

```
                      BEFORE          AFTER
GPU VRAM:             18.8%           ~82%          ← 4x better utilization
CPU for deep model:   468% (5 cores)  ~0%           ← freed entirely
Deep model speed:     15-25 tok/s     ~60 tok/s     ← 3x faster
Thread contention:    144t > 128c     ~128t = 128c  ← eliminated
RAM for deep model:   128 GB          ~0 GB         ← freed entirely
Disk usage:           93%             ~77%          ← healthy
```

-----

## 17. Accuracy Metrics & Golden Tests

### 17.1 Metrics to Track

```typescript
interface AccuracyMetrics {
  lineNumberAccuracy: number;     // % of changes with correct line numbers
  contentHashMatch: number;       // % of changes where currentCode matched
  importCompleteness: number;     // % of runs with no missing imports
  compilationSuccess: number;     // % where generated code compiles
  testPassRate: number;           // % where tests pass first try
  conventionCompliance: number;   // % of changes matching team style
  scopeAccuracy: number;          // % of changes within blast radius
  symbolExistence: number;        // % of symbols found in graph
  planApprovalRate: number;       // % approved by humans
  prMergeRate: number;            // % of PRs actually merged
}
```

### 17.2 Golden Test Suite

```typescript
const GOLDEN_TESTS = [
  {
    name: 'Add pagination to bond search (Spring Boot)',
    repo: 'ms-bonds-trading',
    expectedChanges: [
      { file: '**/BondSearchController.java', mustContain: ['Pageable', 'Page<'] },
      { file: '**/BondSearchService.java', mustContain: ['Pageable'] },
    ],
    maxChanges: 5,
    mustCompile: true,
  },
  {
    name: 'Add error boundary to equity MFE (React)',
    repo: 'mfe-equity-dashboard',
    expectedNewFiles: [
      { path: '**/ErrorBoundary.tsx', mustContain: ['componentDidCatch'] },
    ],
    maxChanges: 3,
  },
  {
    name: 'Fix NPE in settlement processor (Bug)',
    repo: 'ms-settlement',
    expectedChanges: [
      { file: '**/SettlementProcessor.java', mustContain: ['null'] },
    ],
    maxChanges: 2,
  },
];
```

### 17.3 Accuracy Targets

```
Phase 2 Launch targets (Month 1):
  Line number accuracy:      ≥ 85%
  Compilation success:       ≥ 80%
  Import completeness:       ≥ 90%  (with auto-resolution)
  Test pass (first try):     ≥ 70%
  Convention compliance:     ≥ 75%
  Plan approval rate:        ≥ 60%

Mature targets (Month 3):
  Line number accuracy:      ≥ 95%
  Compilation success:       ≥ 95%
  Import completeness:       ≥ 98%
  Test pass (first try):     ≥ 85%
  Convention compliance:     ≥ 90%
  Plan approval rate:        ≥ 80%
```

-----

## 18. Prompt Tuning Feedback Loop

### 18.1 Learn from Rejections

```typescript
// When plan is rejected, classify the reason via 1.5B
async function recordRejection(runId: string, notes: string) {
  const prompt = `Classify this plan rejection into one category:
- line_number_error
- missing_import
- wrong_pattern
- scope_creep
- convention_violation
- incomplete_changes
- incorrect_logic
- other

"${notes}"

Category:`;

  const category = await llmCall(prompt, routeTask('implement', 'risk_classify'));
  await db.recordFeedback(runId, category.trim(), notes);
}
```

### 18.2 Weekly Report

```
Monday 9 AM Slack digest:
  Pipelines run: 23
  Plans approved: 15 (65%)
  Plans rejected: 6 (26%) ← breakdown by category
  Plans expired: 2 (9%)
  PRs merged: 12 (80% of approved)
  Top rejection reason: "missing_import" (3 occurrences)
  Recommendation: Increase import verification aggressiveness
```

-----

## 19. Migration Sequence

### 19.1 Phase 1 — Disk Cleanup (Day 0, 30 minutes)

```bash
# Free space before downloading new models
# 1. Clean logs and temp files
find /apps -name "*.log" -mtime +30 -delete
find /tmp -name "ome-*" -mtime +7 -delete
sudo journalctl --vacuum-size=500M
sudo dnf clean all
# 2. Git gc on repos
for repo in /apps/ome_rag/repos/*/*; do
  git -C "$repo" gc --prune=now 2>/dev/null
done
# 3. Verify space freed
df -h /apps
```

### 19.2 Phase 2 — Download New Models (Day 0, 2-3 hours)

```bash
# Download Qwen3.5-122B-A10B Q4_K_M (~73 GB)
aria2c -x 16 -s 16 --continue=true --max-tries=0 \
  -d /apps/models/qwen3.5-122b/ \
  "<source>/Qwen3.5-122B-A10B-Q4_K_M.gguf"

# Download Qwen3.5-9B Q4_K_M (~6 GB)
aria2c -x 16 -s 16 --continue=true \
  -d /apps/models/qwen3.5-9b/ \
  "<source>/Qwen3.5-9B-Q4_K_M.gguf"

# Verify checksums
sha256sum /apps/models/qwen3.5-122b/*.gguf
sha256sum /apps/models/qwen3.5-9b/*.gguf
```

### 19.3 Phase 3 — Swap GPU Model (Day 1, 15 min downtime)

```bash
# 1. Stop 235B
systemctl stop ome-gpu-gen

# 2. Update service file to point to 122B
# /etc/systemd/system/ome-gpu-gen.service
# --model /apps/models/qwen3.5-122b/Qwen3.5-122B-A10B-Q4_K_M.gguf
# --n-gpu-layers 99 --parallel 3 --threads 8

# 3. Start 122B
systemctl daemon-reload
systemctl start ome-gpu-gen

# 4. Verify health
curl -s http://127.0.0.1:8090/health | jq .

# 5. Smoke test
curl -s http://127.0.0.1:8090/v1/chat/completions \
  -H "Content-Type: application/json" \
  -d '{"model":"qwen3.5","messages":[{"role":"user","content":"Write a hello world in Java"}],"max_tokens":200}'

# 6. After validation: delete 235B to free ~67 GB
rm -f /apps/models/qwen3-235b/*.gguf
```

### 19.4 Phase 4 — Swap CPU Mid-Tier (Day 1, 5 min downtime)

```bash
# 1. Stop qwen2.5-coder-7b
systemctl stop ome-gen-7b

# 2. Update service to Qwen3.5-9B
# --model /apps/models/qwen3.5-9b/Qwen3.5-9B-Q4_K_M.gguf
# --ctx-size 32768 --parallel 4 --threads 24 --jinja --reasoning-format deepseek

# 3. Start
systemctl daemon-reload
systemctl start ome-gen-7b  # or rename service

# 4. Verify + smoke test
curl -s http://127.0.0.1:8082/health | jq .

# 5. After validation: delete qwen2.5-coder-7b to free ~5 GB
rm -f /apps/models/qwen2.5-coder-7b/*.gguf
```

### 19.5 Phase 5 — Update Prompt Templates (Day 2)

```
- Update all prompts to Qwen3.5 chat template (shared by 9B and 122B)
- Remove Qwen2.5 ChatML template support
- Update model_router.ts with three-tier routing
- Test: 5 golden queries through each tier
```

### 19.6 Phase 6 — Three-Pass Architecture (Days 3-8)

```
Day 3: Pass 1 — Gather
  Parallel executor, full file reader, convention sampler
Day 4: Pass 2 — Generate
  Layered prompt builder, JSON output parser
Day 5: Pass 3 — Verify
  6 parallel checks, auto-correction, import resolution
Day 6: Wire three passes together + performance test
Day 7: Convention detection + cache + violation check
Day 8: Integration testing on 5 golden tickets
```

### 19.7 Phase 7 — Human Checkpoints + Dashboard (Days 9-12)

```
Day 9:  Checkpoint state machine + review API
Day 10: Review Web UI + Slack notifications
Day 11: Execution engine (worktree, apply, test)
Day 12: End-to-end pipeline test on real ticket
```

### 19.8 Phase 8 — Calibration (Days 13-15)

```
Day 13: Run on 3 MFE repos, tune thresholds
Day 14: Run on 3 MS repos, tune thresholds
Day 15: Accuracy measurement, feedback loop, documentation
```

-----

## 20. Configuration — Final State

```bash
# ═══ Models ═══
# GPU — Qwen3.5-122B-A10B (:8090)
GPU_MODEL_PATH=/apps/models/qwen3.5-122b/Qwen3.5-122B-A10B-Q4_K_M.gguf
GPU_MODEL_PORT=8090
GPU_MODEL_CTX=32768
GPU_MODEL_PARALLEL=3
GPU_MODEL_THREADS=8

# CPU Mid — Qwen3.5-9B (:8082)
MID_MODEL_PATH=/apps/models/qwen3.5-9b/Qwen3.5-9B-Q4_K_M.gguf
MID_MODEL_PORT=8082
MID_MODEL_CTX=32768
MID_MODEL_PARALLEL=4
MID_MODEL_THREADS=24

# CPU Fast — Qwen2.5-coder-1.5B (:8084) — unchanged
FAST_MODEL_PORT=8084

# ═══ Embedding + Reranker — unchanged ═══
TEXT_EMBED_PORT=8081    # snowflake-arctic 1024d
CODE_EMBED_PORT=8085    # nomic-embed-code 3584d
RERANKER_PORT=8083      # bge-reranker-v2-m3

# ═══ Services — unchanged ═══
MCP_SERVER_PORT=3100
AGENT_PORT=3200

# ═══ Three-Pass Architecture ═══
GATHER_MAX_RAG_CALLS=12
GATHER_MAX_TARGET_FILES=8
GATHER_MAX_REFERENCE_FILES=5
GATHER_MAX_SIMILAR_PRS=3
GENERATE_TOKEN_BUDGET=20500
VERIFY_FUZZY_THRESHOLD=0.85
CONVENTION_CACHE_TTL_HOURS=24

# ═══ Pipeline ═══
PIPELINE_ENABLED=true
ALLOW_AUTO_APPROVE=false
PLAN_EXPIRY_HOURS=48
MIN_CONFIDENCE_FOR_EXECUTION=0.5
MIN_CONFIDENCE_FOR_AUTO_APPROVE=0.9
AUTO_FIX_MAX_RETRIES=1

# ═══ Accuracy Targets ═══
TARGET_LINE_ACCURACY=0.85
TARGET_COMPILATION_SUCCESS=0.80
TARGET_IMPORT_COMPLETENESS=0.90
TARGET_TEST_PASS_RATE=0.70
```

-----

## Summary — Impact of Each Change

|Change                       |Accuracy Impact                                               |Performance Impact                           |
|-----------------------------|--------------------------------------------------------------|---------------------------------------------|
|235B → 122B on GPU           |Better benchmarks (+7 SWE-bench, +30% tool calling)           |**3-4x faster** inference (60 vs 15-25 tok/s)|
|qwen2.5-coder-7b → qwen3.5-9b|Better instruction following (+16% IFEval), same prompt family|Marginal (similar speed on CPU)              |
|Full files with line numbers |**Eliminates line number guessing** — #1 accuracy win         |+2s gather time (negligible)                 |
|Three-pass architecture      |Catches errors BEFORE delivery (imports, symbols, conventions)|+8s verify time, net faster overall          |
|Convention injection         |**Matches team patterns** instead of training patterns        |+1s convention sample gather                 |
|Import auto-resolution       |**Catches missing imports** via knowledge graph               |+2s verification                             |
|Content hash verification    |**Prevents stale code changes**                               |+0.1s per change                             |
|Thread de-subscription       |No more context switching under load                          |More predictable latency                     |
|GPU utilization 18% → 82%    |—                                                             |**$250K GPU investment finally utilized**    |

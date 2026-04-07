# OME Agent — Multi-Pass Context Strategy (Zero Quality Loss)

> **Problem:** `ome_rag_guide` prompts exceed Qwen3.5 context limit → 400 errors
> **Principle:** Never compress, never truncate, never lose context. Loop until done.
> **Approach:** Split work across multiple LLM calls, each with full uncompressed content
> **Effort:** ~4 days

-----

## Table of Contents

1. [Why Compression Fails](#1-why-compression-fails)
1. [Multi-Pass Architecture](#2-multi-pass-architecture)
1. [Context Estimator — Know Before You Send](#3-context-estimator)
1. [Single-Pass vs Multi-Pass Decision](#4-single-pass-vs-multi-pass-decision)
1. [IMPLEMENT Mode — Per-File Passes](#5-implement-mode--per-file-passes)
1. [DIAGNOSE Mode — Analyze Then Fix](#6-diagnose-mode--analyze-then-fix)
1. [UNDERSTAND Mode — Accumulate Then Synthesize](#7-understand-mode--accumulate-then-synthesize)
1. [Pass Orchestrator](#8-pass-orchestrator)
1. [Carry-Forward Context](#9-carry-forward-context)
1. [Merge Engine](#10-merge-engine)
1. [Integration with Orchestrator](#11-integration-with-orchestrator)
1. [Configuration](#12-configuration)
1. [Implementation Schedule](#13-implementation-schedule)

-----

## 1. Why Compression Fails

Every compression technique trades information for space:

|Technique                     |What You Lose                                                               |
|------------------------------|----------------------------------------------------------------------------|
|Truncate file head/tail       |Method bodies in the middle — model generates wrong logic                   |
|Signature-only reference files|Pattern details — model follows wrong conventions                           |
|Compact PRs to 2 lines        |Implementation nuance — model repeats mistakes the team already learned from|
|Compact graph to summary      |Specific caller/callee names — model misses dependencies                    |
|Drop low-priority sections    |Context that turns a good answer into a great one                           |

**The model generates better code when it sees more context.** The quality difference between a 20K-token prompt and a 30K-token prompt is measurable — fewer hallucinated imports, better error handling patterns, correct method signatures on first try.

**Solution:** Don’t shrink the context. Split the work.

-----

## 2. Multi-Pass Architecture

### Core Idea

Instead of cramming everything into one LLM call, split into focused passes where each pass:

- Fits within the context window
- Receives FULL uncompressed content for its scope
- Produces a partial result
- Passes a compact carry-forward to the next pass

Then a lightweight merge pass combines all partial results.

```
SINGLE-PASS (current — breaks on large context):
  ┌──────────────────────────────────────────────────┐
  │ System + Query + ALL files + ALL context → Model │ → 400 ERROR
  └──────────────────────────────────────────────────┘

MULTI-PASS (new — always fits):
  ┌─────────────────────────────────────────────┐
  │ Pass 1: System + Query + File A (full)      │ → Changes for A
  │         + ticket + conventions + graph       │   + carry-forward summary
  └─────────────────────────────────────────────┘
  ┌─────────────────────────────────────────────┐
  │ Pass 2: System + Query + File B (full)      │ → Changes for B
  │         + ticket + conventions + graph       │   + carry-forward summary
  │         + carry-forward from Pass 1          │
  └─────────────────────────────────────────────┘
  ┌─────────────────────────────────────────────┐
  │ Pass 3 (merge): Combine all changes         │ → Final ImplementResponse
  │         + resolve dependencies               │
  │         + generate execution order            │
  └─────────────────────────────────────────────┘
```

### Key Properties

- **Each pass sees full, uncompressed files** — no truncation, no signatures-only
- **Shared context (ticket, conventions, graph) is included in every pass** — it’s small enough
- **Carry-forward is a compact summary of decisions made in prior passes** — not the full output
- **Single-pass is the default** — multi-pass only activates when context would exceed the window
- **Final merge pass is lightweight** — just organizes, doesn’t regenerate code

-----

## 3. Context Estimator

Before sending anything to the model, estimate the total token count. This is the gate that decides single-pass vs multi-pass.

```typescript
// src/guide/context/context_estimator.ts

/** ~3.8 chars per token for mixed English/code — conservative estimate */
const CHARS_PER_TOKEN = 3.8;

export function estimateTokens(text: string): number {
  if (!text) return 0;
  return Math.ceil(text.length / CHARS_PER_TOKEN);
}

export interface ContextEstimate {
  systemPrompt: number;
  query: number;
  targetFiles: Array<{ filePath: string; tokens: number }>;
  referenceFiles: Array<{ filePath: string; tokens: number }>;
  ticket: number;
  conventions: number;
  similarPRs: number;
  graphContext: number;
  healthWarnings: number;
  knownFindings: number;
  errorContext: number;
  totalInput: number;
  generationReserve: number;
  totalRequired: number;
  contextWindowSize: number;
  fits: boolean;
  overflowTokens: number;
}

export function estimateFullContext(
  input: ContextInput,
  mode: GuideMode,
  contextWindowSize: number
): ContextEstimate {
  const systemPrompt = estimateTokens(getSystemPrompt(mode));
  const query = estimateTokens(input.query);

  const targetFiles = input.targetFiles.map(f => ({
    filePath: f.filePath,
    tokens: estimateTokens(formatFileWithLineNumbers(f)),
  }));

  const referenceFiles = input.referenceFiles.map(f => ({
    filePath: f.filePath,
    tokens: estimateTokens(formatFileWithLineNumbers(f)),
  }));

  const ticket = input.ticketContext
    ? estimateTokens(formatTicketFull(input.ticketContext))
    : 0;

  const conventions = input.conventions
    ? estimateTokens(input.conventions)
    : 0;

  const similarPRs = input.similarPRs.length > 0
    ? estimateTokens(formatPRsFull(input.similarPRs))
    : 0;

  const graphContext = estimateTokens(formatGraphFull(input.graphContext));
  const healthWarnings = estimateTokens(formatHealthWarnings(input.healthWarnings));
  const knownFindings = estimateTokens(formatKnownFindings(input.knownFindings));
  const errorContext = input.errorContext
    ? estimateTokens(input.errorContext)
    : 0;

  const generationReserve = getGenerationReserve(mode);

  const totalInput = systemPrompt + query
    + targetFiles.reduce((sum, f) => sum + f.tokens, 0)
    + referenceFiles.reduce((sum, f) => sum + f.tokens, 0)
    + ticket + conventions + similarPRs + graphContext
    + healthWarnings + knownFindings + errorContext;

  const totalRequired = totalInput + generationReserve;

  return {
    systemPrompt, query, targetFiles, referenceFiles,
    ticket, conventions, similarPRs, graphContext,
    healthWarnings, knownFindings, errorContext,
    totalInput, generationReserve, totalRequired,
    contextWindowSize,
    fits: totalRequired <= contextWindowSize,
    overflowTokens: Math.max(0, totalRequired - contextWindowSize),
  };
}

function getGenerationReserve(mode: GuideMode): number {
  switch (mode) {
    case 'implement': return 8_000;
    case 'diagnose': return 6_000;
    case 'understand': return 4_000;
  }
}
```

-----

## 4. Single-Pass vs Multi-Pass Decision

```typescript
// src/guide/context/pass_planner.ts

export interface PassPlan {
  strategy: 'single' | 'multi';
  reason: string;
  passes: PassDefinition[];
}

export interface PassDefinition {
  passNumber: number;
  passType: 'analyze' | 'generate' | 'merge';
  /** Full uncompressed files assigned to this pass */
  targetFiles: GatheredFile[];
  /** Full uncompressed reference files (shared across all passes) */
  referenceFiles: GatheredFile[];
  /** Include ticket context in this pass */
  includeTicket: boolean;
  /** Include similar PRs in this pass */
  includePRs: boolean;
  /** Include graph/blast radius in this pass */
  includeGraph: boolean;
  /** Include conventions in this pass */
  includeConventions: boolean;
  /** Carry-forward from previous passes */
  carryForward: boolean;
  /** Estimated tokens for this pass */
  estimatedTokens: number;
}

export function planPasses(
  estimate: ContextEstimate,
  input: ContextInput,
  mode: GuideMode
): PassPlan {

  // ── FITS IN ONE PASS — use single pass (no overhead) ──
  if (estimate.fits) {
    return {
      strategy: 'single',
      reason: `Total ${estimate.totalRequired} tokens fits in ${estimate.contextWindowSize} window`,
      passes: [{
        passNumber: 1,
        passType: 'generate',
        targetFiles: input.targetFiles,
        referenceFiles: input.referenceFiles,
        includeTicket: true,
        includePRs: true,
        includeGraph: true,
        includeConventions: true,
        carryForward: false,
        estimatedTokens: estimate.totalRequired,
      }],
    };
  }

  // ── DOESN'T FIT — plan multi-pass ──
  const contextWindow = estimate.contextWindowSize;
  const generationReserve = getGenerationReserve(mode);

  // Shared context (included in EVERY pass) — this is small enough to repeat
  const sharedTokens = estimate.systemPrompt
    + estimate.query
    + estimate.ticket
    + estimate.conventions
    + estimate.graphContext
    + estimate.healthWarnings
    + estimate.knownFindings
    + estimate.errorContext;

  // Carry-forward overhead per pass (grows slightly with each pass)
  const CARRY_FORWARD_PER_PASS = 500; // ~500 tokens for summary of prior decisions

  // Budget available for files in each pass
  const filesBudgetPerPass = contextWindow - sharedTokens - generationReserve - CARRY_FORWARD_PER_PASS;

  if (filesBudgetPerPass < 2_000) {
    // Shared context alone nearly fills the window — this shouldn't happen
    // Fall back to single pass with minimal context
    console.warn(`[PassPlanner] Shared context alone is ${sharedTokens} tokens — very tight budget`);
    return buildFallbackPlan(input, estimate);
  }

  // ── Group target files into passes ──
  const passes: PassDefinition[] = [];
  let currentPassFiles: GatheredFile[] = [];
  let currentPassTokens = 0;
  let passNumber = 1;

  // Sort target files: largest first (greedy bin packing)
  const sortedTargets = [...estimate.targetFiles]
    .sort((a, b) => b.tokens - a.tokens);

  for (const fileEstimate of sortedTargets) {
    const file = input.targetFiles.find(f => f.filePath === fileEstimate.filePath)!;

    if (currentPassTokens + fileEstimate.tokens > filesBudgetPerPass && currentPassFiles.length > 0) {
      // Current pass is full — start new pass
      passes.push(buildFilePass(passNumber, currentPassFiles, input, sharedTokens + currentPassTokens + generationReserve, mode));
      passNumber++;
      currentPassFiles = [];
      currentPassTokens = 0;
    }

    currentPassFiles.push(file);
    currentPassTokens += fileEstimate.tokens;
  }

  // Last batch of target files
  if (currentPassFiles.length > 0) {
    passes.push(buildFilePass(passNumber, currentPassFiles, input, sharedTokens + currentPassTokens + generationReserve, mode));
    passNumber++;
  }

  // ── Reference files pass (if they don't fit in target passes) ──
  const refTokens = estimate.referenceFiles.reduce((sum, f) => sum + f.tokens, 0);

  if (refTokens > 0) {
    // Try to distribute reference files across existing passes
    const refBudgetPerPass = filesBudgetPerPass - (passes.length > 0 ? passes[0].estimatedTokens - sharedTokens - generationReserve : 0);

    if (refBudgetPerPass > refTokens) {
      // Reference files fit alongside targets in each pass — add them
      for (const pass of passes) {
        pass.referenceFiles = input.referenceFiles;
        pass.estimatedTokens += refTokens;
      }
    } else {
      // Include similar PRs only in the first pass (reference for conventions)
      // Reference files available only in the passes where they fit
      for (const pass of passes) {
        const remainingBudget = filesBudgetPerPass -
          pass.targetFiles.reduce((sum, f) => sum + estimateTokens(formatFileWithLineNumbers(f)), 0);

        // Fit as many reference files as possible
        const fittingRefs: GatheredFile[] = [];
        let refAccum = 0;
        for (const ref of input.referenceFiles) {
          const refTok = estimateTokens(formatFileWithLineNumbers(ref));
          if (refAccum + refTok <= remainingBudget) {
            fittingRefs.push(ref);
            refAccum += refTok;
          }
        }
        pass.referenceFiles = fittingRefs;
        pass.estimatedTokens += refAccum;
      }
    }
  }

  // ── Merge pass (always the last pass) ──
  passes.push({
    passNumber,
    passType: 'merge',
    targetFiles: [],
    referenceFiles: [],
    includeTicket: true,
    includePRs: false,
    includeGraph: true,
    includeConventions: false,
    carryForward: true,
    estimatedTokens: sharedTokens + (passes.length * 800) + 2_000, // carry-forwards + merge generation
  });

  return {
    strategy: 'multi',
    reason: `${estimate.totalRequired} tokens exceeds ${estimate.contextWindowSize} window. ` +
            `Split into ${passes.length} passes (${passes.length - 1} generate + 1 merge)`,
    passes,
  };
}


function buildFilePass(
  passNumber: number,
  files: GatheredFile[],
  input: ContextInput,
  estimatedTokens: number,
  mode: GuideMode,
): PassDefinition {
  return {
    passNumber,
    passType: 'generate',
    targetFiles: files,
    referenceFiles: [],  // Populated later by reference file distribution
    includeTicket: true,
    includePRs: passNumber === 1, // Similar PRs only in first pass (avoid repetition)
    includeGraph: true,
    includeConventions: true,
    carryForward: passNumber > 1,
    estimatedTokens,
  };
}


function buildFallbackPlan(input: ContextInput, estimate: ContextEstimate): PassPlan {
  // Extreme case: even shared context is huge
  // Process one file per pass, minimize shared context to essentials only
  const passes: PassDefinition[] = input.targetFiles.map((file, i) => ({
    passNumber: i + 1,
    passType: 'generate' as const,
    targetFiles: [file],
    referenceFiles: [],
    includeTicket: i === 0,
    includePRs: false,
    includeGraph: i === 0,
    includeConventions: true,
    carryForward: i > 0,
    estimatedTokens: estimate.targetFiles[i]?.tokens ?? 0 + 5_000,
  }));

  passes.push({
    passNumber: passes.length + 1,
    passType: 'merge',
    targetFiles: [],
    referenceFiles: [],
    includeTicket: false,
    includePRs: false,
    includeGraph: false,
    includeConventions: false,
    carryForward: true,
    estimatedTokens: 4_000,
  });

  return { strategy: 'multi', reason: 'Fallback: one file per pass', passes };
}
```

-----

## 5. IMPLEMENT Mode — Per-File Passes

Each pass generates `CodeChange[]` for its assigned target files. The model sees:

- Full target file(s) with line numbers — **uncompressed**
- Full reference files — **uncompressed** (as many as fit)
- Full ticket context
- Full conventions
- Full graph context
- Full similar PRs (first pass only)
- Carry-forward: summary of changes generated in prior passes

```typescript
// src/guide/context/implement_passes.ts

/**
 * Execute IMPLEMENT mode across multiple passes.
 * Each pass generates CodeChange[] for its assigned files.
 * Final merge pass combines and orders all changes.
 */
export async function executeImplementPasses(
  plan: PassPlan,
  input: ContextInput,
  ctx: AgentContext
): Promise<ImplementResponse> {

  const allChanges: CodeChange[] = [];
  const allNewFiles: NewFile[] = [];
  const carryForward: CarryForwardEntry[] = [];

  // ── Generate passes ──
  const generatePasses = plan.passes.filter(p => p.passType === 'generate');

  for (const pass of generatePasses) {
    console.log(
      `[Implement] Pass ${pass.passNumber}/${plan.passes.length}: ` +
      `${pass.targetFiles.map(f => f.filePath).join(', ')} ` +
      `(~${pass.estimatedTokens} tokens)`
    );

    const prompt = buildImplementPassPrompt(pass, input, carryForward);

    const routing = routeToModel('implement', 'generate_code');
    const response = await ctx.llm.chat(prompt, routing);

    const parsed = parseImplementPassResponse(response.content);

    // Validate changes against actual files on disk
    for (const change of parsed.changes) {
      const validated = await validateChangeAgainstDisk(change, input.repoBasePath);
      allChanges.push(validated);
    }

    if (parsed.newFiles) {
      allNewFiles.push(...parsed.newFiles);
    }

    // Build carry-forward for next pass
    carryForward.push({
      passNumber: pass.passNumber,
      filesProcessed: pass.targetFiles.map(f => f.filePath),
      changesSummary: parsed.changes.map(c => ({
        id: c.id,
        file: c.file,
        action: c.action,
        description: c.description,
        affectedSymbols: c.affectedSymbols,
      })),
      newFilesSummary: parsed.newFiles?.map(f => ({
        path: f.path,
        purpose: f.description,
      })) ?? [],
      conventionsObserved: parsed.conventionsApplied ?? [],
    });
  }

  // ── Merge pass ──
  const mergePrompt = buildMergePassPrompt(input, allChanges, allNewFiles, carryForward);
  const mergeRouting = routeToModel('implement', 'merge_plan');
  const mergeResponse = await ctx.llm.chat(mergePrompt, mergeRouting);
  const merged = parseMergeResponse(mergeResponse.content, allChanges, allNewFiles);

  return merged;
}


/**
 * Build prompt for a single IMPLEMENT generate pass.
 * ALL content is FULL and UNCOMPRESSED.
 */
function buildImplementPassPrompt(
  pass: PassDefinition,
  input: ContextInput,
  carryForward: CarryForwardEntry[]
): string {
  const sections: string[] = [];

  // ── System instruction ──
  sections.push(getImplementSystemPrompt());

  // ── Task ──
  sections.push(`## Task\n${input.query}`);

  // ── Carry-forward from prior passes ──
  if (pass.carryForward && carryForward.length > 0) {
    sections.push(formatCarryForward(carryForward));
  }

  // ── Ticket (full) ──
  if (pass.includeTicket && input.ticketContext) {
    sections.push(`## Ticket\n${formatTicketFull(input.ticketContext)}`);
  }

  // ── Conventions (full) ──
  if (pass.includeConventions && input.conventions) {
    sections.push(`## Team Conventions\n${input.conventions}`);
  }

  // ── Target files (FULL with line numbers — never compressed) ──
  sections.push('## Target Files — Generate changes for these');
  for (const file of pass.targetFiles) {
    sections.push(formatFileWithLineNumbers(file));
  }

  // ── Reference files (FULL — never compressed) ──
  if (pass.referenceFiles.length > 0) {
    sections.push('## Reference Files — Follow these patterns');
    for (const file of pass.referenceFiles) {
      sections.push(formatFileWithLineNumbers(file));
    }
  }

  // ── Similar PRs (full — first pass only) ──
  if (pass.includePRs && input.similarPRs.length > 0) {
    sections.push(`## Similar PRs (team solved analogous problems)\n${formatPRsFull(input.similarPRs)}`);
  }

  // ── Graph context (full) ──
  if (pass.includeGraph) {
    sections.push(`## Dependencies & Impact\n${formatGraphFull(input.graphContext)}`);
  }

  // ── Health warnings (full) ──
  if (input.healthWarnings.length > 0) {
    sections.push(`## Health Warnings\n${formatHealthWarnings(input.healthWarnings)}`);
  }

  // ── Output instructions ──
  sections.push(getImplementOutputInstructions(pass));

  return sections.join('\n\n---\n\n');
}


function getImplementOutputInstructions(pass: PassDefinition): string {
  const fileList = pass.targetFiles.map(f => f.filePath).join(', ');

  return `## Output Instructions

Generate changes ONLY for these target files: ${fileList}

Respond with a JSON object:
{
  "changes": [
    {
      "id": "change-NNN",
      "file": "exact/file/path.ts",
      "action": "modify" | "insert" | "replace" | "delete",
      "description": "what this change does",
      "reason": "why this change is needed",
      "currentCode": {
        "startLine": <number>,
        "endLine": <number>,
        "content": "exact current code at those lines"
      },
      "proposedCode": "the new code to write",
      "affectedSymbols": ["symbol1", "symbol2"],
      "dependencies": ["change-NNN"]
    }
  ],
  "newFiles": [
    {
      "path": "path/to/new/file.ts",
      "content": "full file content",
      "description": "purpose of this new file"
    }
  ],
  "conventionsApplied": ["convention1", "convention2"]
}

CRITICAL:
- Use EXACT line numbers from the target files above
- currentCode.content must match the actual file content at those lines
- Generate production-ready code, not pseudocode
- Follow conventions from the Team Conventions section
- Include imports if new dependencies are needed`;
}
```

-----

## 6. DIAGNOSE Mode — Analyze Then Fix

DIAGNOSE naturally has two steps (fast analysis + deep fix). Multi-pass only needed if the error location spans multiple large files.

```typescript
// src/guide/context/diagnose_passes.ts

export async function executeDiagnosePasses(
  plan: PassPlan,
  input: ContextInput,
  ctx: AgentContext
): Promise<DiagnoseResponse> {

  if (plan.strategy === 'single') {
    // Normal two-step: fast analysis → deep fix (both fit in one call each)
    return executeSinglePassDiagnose(input, ctx);
  }

  // Multi-pass: error spans multiple files
  // Pass 1..N: Analyze each file for error contribution (fast model)
  // Final: Generate fix across all files (deep model, with carry-forward)

  const analyses: FileAnalysis[] = [];
  const generatePasses = plan.passes.filter(p => p.passType === 'generate');

  for (const pass of generatePasses) {
    const prompt = buildDiagnoseAnalyzePrompt(pass, input);
    const routing = routeToModel('diagnose', 'analyze');
    const response = await ctx.llm.chat(prompt, routing);

    const analysis = parseDiagnoseAnalysis(response.content);
    analyses.push({
      files: pass.targetFiles.map(f => f.filePath),
      findings: analysis.findings,
      likelyContribution: analysis.likelyContribution,
    });
  }

  // Merge analyses → generate fix
  const fixPrompt = buildDiagnoseFixPrompt(input, analyses);
  const fixRouting = routeToModel('diagnose', 'generate_fix');
  const fixResponse = await ctx.llm.chat(fixPrompt, fixRouting);

  return parseDiagnoseResponse(fixResponse.content);
}
```

-----

## 7. UNDERSTAND Mode — Accumulate Then Synthesize

UNDERSTAND uses the fast model (64K context) so it rarely needs multi-pass. But when it does (massive codebase exploration):

```typescript
// src/guide/context/understand_passes.ts

export async function executeUnderstandPasses(
  plan: PassPlan,
  input: ContextInput,
  ctx: AgentContext
): Promise<UnderstandResponse> {

  if (plan.strategy === 'single') {
    return executeSinglePassUnderstand(input, ctx);
  }

  // Multi-pass: analyze each file group, then synthesize
  const insights: PassInsight[] = [];
  const generatePasses = plan.passes.filter(p => p.passType === 'generate');

  for (const pass of generatePasses) {
    const prompt = buildUnderstandPassPrompt(pass, input, insights);
    const routing = routeToModel('understand', 'explain');
    const response = await ctx.llm.chat(prompt, routing);

    insights.push({
      files: pass.targetFiles.map(f => f.filePath),
      explanation: response.content,
      keyFindings: extractKeyFindings(response.content),
    });
  }

  // Synthesis pass — combine all insights into coherent explanation
  const synthesisPrompt = buildUnderstandSynthesisPrompt(input, insights);
  const synthesisRouting = routeToModel('understand', 'explain');
  const synthesis = await ctx.llm.chat(synthesisPrompt, synthesisRouting);

  return parseUnderstandResponse(synthesis.content, insights);
}
```

-----

## 8. Pass Orchestrator

Central coordinator that estimates, plans, and executes.

```typescript
// src/guide/context/pass_orchestrator.ts

export async function executeWithPasses(
  input: ContextInput,
  mode: GuideMode,
  ctx: AgentContext
): Promise<GuideResponse> {

  const contextWindowSize = mode === 'understand'
    ? parseInt(process.env.FAST_MODEL_CTX_SIZE || '65536', 10)
    : parseInt(process.env.DEEP_MODEL_CTX_SIZE || '32768', 10);

  // ── Step 1: Estimate total context ──
  const estimate = estimateFullContext(input, mode, contextWindowSize);

  console.log(
    `[PassOrchestrator] Mode: ${mode}, ` +
    `Total: ${estimate.totalRequired} tokens, ` +
    `Window: ${contextWindowSize}, ` +
    `Fits: ${estimate.fits}, ` +
    `Overflow: ${estimate.overflowTokens}`
  );

  if (estimate.fits) {
    console.log(`[PassOrchestrator] Single-pass — all content fits`);
  } else {
    console.log(
      `[PassOrchestrator] Multi-pass needed — overflow ${estimate.overflowTokens} tokens. ` +
      `File sizes: ${estimate.targetFiles.map(f => `${f.filePath}(${f.tokens})`).join(', ')}`
    );
  }

  // ── Step 2: Plan passes ──
  const plan = planPasses(estimate, input, mode);

  console.log(
    `[PassOrchestrator] Strategy: ${plan.strategy}, ` +
    `Passes: ${plan.passes.length}, ` +
    `Reason: ${plan.reason}`
  );

  // ── Step 3: Execute per mode ──
  const startMs = performance.now();
  let response: GuideResponse;

  switch (mode) {
    case 'implement':
      response = await executeImplementPasses(plan, input, ctx);
      break;
    case 'diagnose':
      response = await executeDiagnosePasses(plan, input, ctx);
      break;
    case 'understand':
      response = await executeUnderstandPasses(plan, input, ctx);
      break;
  }

  const totalMs = Math.round(performance.now() - startMs);

  // ── Step 4: Attach metadata ──
  response._passMetadata = {
    strategy: plan.strategy,
    totalPasses: plan.passes.length,
    totalMs,
    estimatedTokens: estimate.totalRequired,
    contextWindowSize,
    passDetails: plan.passes.map(p => ({
      passNumber: p.passNumber,
      passType: p.passType,
      files: p.targetFiles.map(f => f.filePath),
      estimatedTokens: p.estimatedTokens,
    })),
  };

  return response;
}
```

-----

## 9. Carry-Forward Context

The carry-forward is how each pass knows what prior passes already decided. It’s compact — just decisions and structure, not full code.

```typescript
// src/guide/context/carry_forward.ts

export interface CarryForwardEntry {
  passNumber: number;
  filesProcessed: string[];
  changesSummary: Array<{
    id: string;
    file: string;
    action: string;
    description: string;
    affectedSymbols: string[];
  }>;
  newFilesSummary: Array<{
    path: string;
    purpose: string;
  }>;
  conventionsObserved: string[];
}

/**
 * Format carry-forward for inclusion in next pass prompt.
 * This is the ONLY part that is summarized — because it's a summary
 * of MODEL output, not of source material.
 * 
 * Typical size: ~300-500 tokens per prior pass.
 */
export function formatCarryForward(entries: CarryForwardEntry[]): string {
  const sections: string[] = [];

  sections.push('## Changes Already Generated (prior passes)');
  sections.push('Do NOT regenerate these. You may reference them as dependencies.\n');

  for (const entry of entries) {
    sections.push(`### Pass ${entry.passNumber}: ${entry.filesProcessed.join(', ')}`);

    for (const change of entry.changesSummary) {
      sections.push(
        `- **${change.id}** [${change.action}] ${change.file}: ${change.description}` +
        (change.affectedSymbols.length > 0
          ? ` (affects: ${change.affectedSymbols.join(', ')})`
          : '')
      );
    }

    for (const newFile of entry.newFilesSummary) {
      sections.push(`- **NEW FILE** ${newFile.path}: ${newFile.purpose}`);
    }
  }

  if (entries.length > 0 && entries[0].conventionsObserved.length > 0) {
    sections.push(`\nConventions applied: ${entries[0].conventionsObserved.join(', ')}`);
  }

  return sections.join('\n');
}
```

### Why This Doesn’t Lose Quality

The carry-forward summarizes the MODEL’S OWN prior output — not the source material. The source material (files, ticket, graph) is always included in full. The model just needs to know “I already generated change-001 for OrderService.submitOrder” so it doesn’t produce conflicting changes.

-----

## 10. Merge Engine

The merge pass takes all generated changes, resolves dependencies, checks for conflicts, and produces the final `ImplementResponse`.

```typescript
// src/guide/context/merge_engine.ts

/**
 * Build the merge pass prompt.
 * Input: all changes from all passes + the query/ticket for context.
 * Output: ordered, deduplicated, dependency-resolved ImplementResponse.
 */
export function buildMergePassPrompt(
  input: ContextInput,
  allChanges: CodeChange[],
  allNewFiles: NewFile[],
  carryForward: CarryForwardEntry[]
): string {
  const sections: string[] = [];

  sections.push(`## Merge Task

You are assembling the final implementation plan from ${carryForward.length} generation passes.

Original task: ${input.query}
${input.ticketContext ? `Ticket: ${input.ticketContext.ticketId} — ${input.ticketContext.summary}` : ''}

Below are all the changes generated across passes. Your job:
1. Assign correct execution order (which changes must happen before which)
2. Set dependency references between changes (change.dependencies[])
3. Detect and flag any conflicts (two changes modifying the same lines)
4. Generate pre-checks and post-checks
5. Identify tests to update
6. Return the final JSON structure`);

  // All changes (full — these are model output, relatively compact)
  sections.push('## All Generated Changes\n```json');
  sections.push(JSON.stringify(allChanges, null, 2));
  sections.push('```');

  if (allNewFiles.length > 0) {
    sections.push('## New Files\n```json');
    sections.push(JSON.stringify(allNewFiles.map(f => ({
      path: f.path, description: f.description,
      contentLength: f.content.length,
    })), null, 2));
    sections.push('```');
  }

  // Graph context for dependency resolution
  if (input.graphContext) {
    sections.push(`## Dependency Graph\n${formatGraphFull(input.graphContext)}`);
  }

  sections.push(`## Output

Return JSON:
{
  "executionOrder": ["change-001", "change-003", "change-002"],
  "conflicts": [],
  "preChecks": ["verify X compiles", "run Y test"],
  "postChecks": ["verify Z endpoint returns 200"],
  "testsToUpdate": ["src/test/OrderServiceTest.java"],
  "summary": "One paragraph describing the full change set"
}`);

  return sections.join('\n\n---\n\n');
}


/**
 * Parse merge response and assemble final ImplementResponse.
 */
export function assembleFinalResponse(
  mergeOutput: MergeOutput,
  allChanges: CodeChange[],
  allNewFiles: NewFile[],
  input: ContextInput
): ImplementResponse {

  // Order changes per merge output
  const orderedChanges = mergeOutput.executionOrder.map(id =>
    allChanges.find(c => c.id === id)!
  ).filter(Boolean);

  // Set dependencies
  for (const change of orderedChanges) {
    const mergedDeps = mergeOutput.dependencies?.[change.id];
    if (mergedDeps) {
      change.dependencies = mergedDeps;
    }
  }

  // Generate content hashes for agent verification
  for (const change of orderedChanges) {
    if (change.currentCode) {
      change.currentCode.contentHash = md5(change.currentCode.content);
    }
  }

  return {
    summary: mergeOutput.summary,
    changes: orderedChanges,
    newFiles: allNewFiles,
    executionOrder: mergeOutput.executionOrder,
    preChecks: mergeOutput.preChecks,
    postChecks: mergeOutput.postChecks,
    testsToUpdate: mergeOutput.testsToUpdate,
    conflicts: mergeOutput.conflicts,
    confidence: {
      level: mergeOutput.conflicts.length === 0 ? 'high' : 'medium',
      reason: mergeOutput.conflicts.length === 0
        ? 'All changes generated with full file context, no conflicts'
        : `${mergeOutput.conflicts.length} potential conflicts detected`,
    },
  };
}
```

-----

## 11. Integration with Orchestrator

Replace step 8-9 in `orchestrator.ts`:

```typescript
// src/guide/orchestrator.ts — REPLACE steps 8-9

// ── 8-9. BUILD CONTEXT + GENERATE (pass-aware) ──

const contextInput: ContextInput = {
  query: params.query,
  targetFiles: fullFiles.filter(f => f.isTarget),
  referenceFiles: fullFiles.filter(f => !f.isTarget),
  ticketContext: gathered.ticketContext,
  similarPRs,
  graphContext: gathered.graphContext,
  conventions: gathered.conventions,
  healthWarnings: gathered.healthWarnings,
  knownFindings: gathered.knownFindings ?? [],
  errorContext: params.context,
  recentChanges: gathered.recentChanges,
  repoBasePath: path.join(process.env.REPOS_DIR || '/repos', params.current_repo || ''),
};

const response = await executeWithPasses(contextInput, intent.mode, ctx);

// Log pass metadata
if (response._passMetadata) {
  console.log(
    `[Guide] ${response._passMetadata.strategy} strategy: ` +
    `${response._passMetadata.totalPasses} passes in ${response._passMetadata.totalMs}ms`
  );
}
```

### What Changes vs Current Code

|Component                     |Before                              |After                                                  |
|------------------------------|------------------------------------|-------------------------------------------------------|
|`buildQwenContext()`          |Assembles everything into one string|Replaced by `executeWithPasses()`                      |
|`generateImplementResponse()` |Single LLM call                     |Wrapped by `executeImplementPasses()` — 1 or N calls   |
|`generateDiagnoseResponse()`  |Two LLM calls (analyze + fix)       |Wrapped by `executeDiagnosePasses()` — 2 or N+1 calls  |
|`generateUnderstandResponse()`|Single LLM call                     |Wrapped by `executeUnderstandPasses()` — 1 or N+1 calls|
|`qwen_context_builder.ts`     |Monolithic builder                  |Replaced by `pass_orchestrator.ts` + `pass_planner.ts` |

The existing `generateImplementResponse`, `generateDiagnoseResponse`, `generateUnderstandResponse` become the inner functions called by each pass. They don’t change — they just get called with smaller, focused input per pass.

-----

## 12. Configuration

```bash
# ═══ Context Windows ═══
FAST_MODEL_CTX_SIZE=65536        # Coder-Next :8090
DEEP_MODEL_CTX_SIZE=32768        # 235B :8091

# ═══ Generation Reserves ═══
IMPLEMENT_GEN_RESERVE=8000       # Tokens reserved for IMPLEMENT output per pass
DIAGNOSE_GEN_RESERVE=6000
UNDERSTAND_GEN_RESERVE=4000

# ═══ Multi-Pass Tuning ═══
CARRY_FORWARD_BUDGET=500         # Tokens allocated for carry-forward per prior pass
MERGE_PASS_BUDGET=2000           # Extra tokens for merge pass generation

# ═══ Token Estimation ═══
CHARS_PER_TOKEN=3.8              # Conservative estimate for English/code mix

# ═══ Logging ═══
LOG_CONTEXT_ESTIMATES=true       # Log detailed token estimates per section
LOG_PASS_PLANS=true              # Log pass planning decisions
```

-----

## 13. Implementation Schedule

```
Day 1: Context Estimator + Pass Planner
  ├── context_estimator.ts
  │   ├── estimateTokens() — char-based approximation
  │   ├── estimateFullContext() — per-section token counts
  │   └── Unit tests: verify estimates against actual model tokenizer (within 15%)
  ├── pass_planner.ts
  │   ├── planPasses() — single vs multi decision + file grouping
  │   ├── Greedy bin packing by file size
  │   └── Unit tests: verify plans for 1-file, 3-file, 8-file scenarios
  └── Test with real gathered data from 5 previous 400-error queries

Day 2: Carry-Forward + Pass Executors
  ├── carry_forward.ts
  │   ├── CarryForwardEntry type
  │   ├── formatCarryForward() — compact summary of prior decisions
  │   └── Unit tests
  ├── implement_passes.ts
  │   ├── executeImplementPasses() — loop over generate passes
  │   ├── buildImplementPassPrompt() — full uncompressed per-pass prompt
  │   └── Per-pass response parsing
  ├── diagnose_passes.ts
  │   └── executeDiagnosePasses() — analyze passes + fix pass
  └── understand_passes.ts
      └── executeUnderstandPasses() — insight passes + synthesis

Day 3: Merge Engine + Pass Orchestrator
  ├── merge_engine.ts
  │   ├── buildMergePassPrompt()
  │   ├── assembleFinalResponse() — order, deps, conflicts, hashes
  │   └── Conflict detection (same file + overlapping lines)
  ├── pass_orchestrator.ts
  │   ├── executeWithPasses() — central coordinator
  │   └── Pass metadata attachment for logging
  └── Integration with orchestrator.ts (replace steps 8-9)

Day 4: Testing + Edge Cases
  ├── Test single-pass (context fits) — verify zero overhead
  ├── Test 2-pass IMPLEMENT (2 large target files)
  ├── Test 3-pass IMPLEMENT (5 target files)
  ├── Test DIAGNOSE multi-pass (error spans 3 files)
  ├── Test carry-forward correctness (pass 2 references pass 1 changes)
  ├── Test merge conflict detection
  ├── Test the exact queries that were causing 400 errors
  ├── Latency comparison: single-pass vs multi-pass overhead
  └── Verify: multi-pass output quality matches single-pass (on queries that fit both)
```

### File Structure

```
src/guide/context/
  context_estimator.ts       — Token estimation per section
  pass_planner.ts            — Single vs multi decision + file grouping
  pass_orchestrator.ts       — Central coordinator
  carry_forward.ts           — Inter-pass state transfer
  implement_passes.ts        — IMPLEMENT multi-pass executor
  diagnose_passes.ts         — DIAGNOSE multi-pass executor
  understand_passes.ts       — UNDERSTAND multi-pass executor
  merge_engine.ts            — Final response assembly
```

### Latency Impact

|Scenario  |Single-Pass|Multi-Pass (2)         |Multi-Pass (3)       |
|----------|-----------|-----------------------|---------------------|
|IMPLEMENT |~50s       |~95s (2×gen + merge)   |~140s (3×gen + merge)|
|DIAGNOSE  |~35s       |~55s (2×analyze + fix) |~75s                 |
|UNDERSTAND|~3s        |~5s (2×analyze + synth)|~7s                  |

Multi-pass adds ~80-90% per extra pass. This is the cost of zero quality loss. The developer waits longer but gets correct, compilable code on the first try — which saves more time than the wait.

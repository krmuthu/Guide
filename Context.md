# OME Agent — Context Window Management Plan

> **Problem:** `ome_rag_guide` IMPLEMENT/DIAGNOSE prompts exceed Qwen3.5 context limit → 400 errors
> **Goal:** Fit all modes within 32K context (deep model) without losing code generation quality
> **Effort:** ~3 days

-----

## Table of Contents

1. [Root Cause Analysis](#1-root-cause-analysis)
1. [Context Priority Model](#2-context-priority-model)
1. [Token Budget Allocator](#3-token-budget-allocator)
1. [File Compression Strategy](#4-file-compression-strategy)
1. [Section-Level Compressors](#5-section-level-compressors)
1. [Implementation — TokenBudget Class](#6-implementation--tokenbudget-class)
1. [Implementation — Context Builder](#7-implementation--context-builder)
1. [Implementation — File Compressor](#8-implementation--file-compressor)
1. [Implementation — Section Compressors](#9-implementation--section-compressors)
1. [Integration with Orchestrator](#10-integration-with-orchestrator)
1. [Quality Safeguards](#11-quality-safeguards)
1. [Configuration](#12-configuration)
1. [Implementation Schedule](#13-implementation-schedule)

-----

## 1. Root Cause Analysis

### What’s Happening

`buildQwenContext()` assembles everything the gather phase collected with no budget awareness:

```
System prompt                   ~800 tokens
Target files (up to 8 full)     ~12,000-24,000 tokens  ← main culprit
Reference files (up to 4 full)  ~4,000-12,000 tokens   ← second culprit
Similar PRs (up to 3)           ~600-1,500 tokens
Ticket context (enriched)       ~300-1,000 tokens
Graph context (blast radius)    ~500-2,000 tokens
Conventions                     ~200-400 tokens
Health warnings                 ~100-300 tokens
Layer 1/2 known findings        ~200-800 tokens
──────────────────────────────────────────────
Worst case total:               ~19,000-42,000 tokens
+ Generation headroom needed:   ~8,000 tokens (IMPLEMENT output)
```

Deep model (:8091) has 32K context. An IMPLEMENT request on a large service with 8 target files + 4 reference files + enriched ticket + 3 similar PRs easily exceeds this.

### Why Blind Truncation Fails

Cutting files at a character limit loses:

- The function signature that sets up the entire method (cut from top)
- The return statement and closing logic (cut from middle)
- Related methods later in the file that follow the same pattern (cut from bottom)

The model generates code that doesn’t compile or misses conventions.

-----

## 2. Context Priority Model

Not all context is equal. What the model MUST have vs what helps vs what’s nice-to-have:

### IMPLEMENT Mode

|Priority|Section                                   |Why Essential                                                                                           |Max Budget %       |
|--------|------------------------------------------|--------------------------------------------------------------------------------------------------------|-------------------|
|P0      |Target file — matched functions/classes   |Model is editing THIS code. Must see exact lines, signatures, dependencies.                             |40%                |
|P0      |Target file — imports + class declaration |Without imports, generated code references non-existent modules.                                        |(included in above)|
|P1      |Ticket context (enriched)                 |Defines WHAT to build. Without it, model guesses requirements.                                          |10%                |
|P1      |Conventions (from fast model)             |Ensures generated code matches team patterns. Short already (~200 tokens).                              |5%                 |
|P2      |Reference files — signatures + key methods|Model needs to see patterns to follow. Full files unnecessary — signatures + method previews sufficient.|20%                |
|P2      |Similar PRs — compact                     |Shows how team solved analogous problems. Core problem + approach is enough.                            |8%                 |
|P3      |Graph context (blast radius, callers)     |Informs what else might break. Structured summary, not raw nodes.                                       |7%                 |
|P3      |Health warnings                           |Flags known problems in target files. Usually 1-3 lines.                                                |3%                 |
|P4      |Known findings (Layer 1/2)                |Anti-duplication hints. Ultra-compact format sufficient.                                                |2%                 |
|—       |Generation headroom                       |IMPLEMENT needs ~6,000-8,000 tokens for CodeChange[] output.                                            |25% reserved       |

### UNDERSTAND Mode (Fast model, 64K context — rarely hits limits)

Same priorities but more generous budgets. Usually fits without compression.

### DIAGNOSE Mode

|Priority|Section                                           |Budget %    |
|--------|--------------------------------------------------|------------|
|P0      |Error context (stacktrace, error message)         |15%         |
|P0      |Target file — error location ± 50 lines           |30%         |
|P1      |Reverse call chain (structured)                   |10%         |
|P1      |Recent changes to affected files                  |10%         |
|P2      |Config context (application.yml relevant sections)|10%         |
|—       |Generation headroom (analysis + fix)              |25% reserved|

-----

## 3. Token Budget Allocator

### Core Principle

**Allocate budget BEFORE assembling context. Fill sections by priority. Never exceed.**

```
Total context window:           32,768 tokens (deep model)
Reserved for system prompt:     -1,000
Reserved for generation:        -8,000 (IMPLEMENT) / -4,000 (UNDERSTAND) / -6,000 (DIAGNOSE)
═══════════════════════════════════════
Available for user context:     23,768 (IMPLEMENT) / 27,768 (UNDERSTAND) / 25,768 (DIAGNOSE)
```

Each section gets a budget allocation. If a section uses less than its allocation, the remainder flows to the next priority level (waterfall).

-----

## 4. File Compression Strategy

### Target Files — Smart Extraction (NOT truncation)

Target files need the highest fidelity. Instead of sending the entire 400-line file, send:

1. **File header** — imports, package declaration, class-level annotations (always)
1. **Matched regions** — the specific functions/classes that semantic search matched, with full content and exact line numbers
1. **Sibling signatures** — other methods in the same class as one-line signatures (so the model knows what else exists)
1. **Class fields** — all field declarations (critical for understanding state)

This preserves everything the model needs to generate correct code while cutting 60-70% of irrelevant methods.

```
BEFORE (full 380-line file, ~2,800 tokens):
  imports (20 lines)
  class OrderService {
    field declarations (5 lines)
    constructor (10 lines)
    submitOrder() — 45 lines         ← matched by search
    cancelOrder() — 35 lines
    getOrderStatus() — 20 lines
    validateOrder() — 60 lines       ← matched by search
    mapToDTO() — 25 lines
    ... 8 more utility methods (160 lines)
  }

AFTER (smart extraction, ~1,100 tokens):
  imports (20 lines)
  class OrderService {
    field declarations (5 lines)
    constructor (10 lines)
    submitOrder() — 45 lines FULL    ← matched
    cancelOrder(orderId: string): Promise<void>  ← signature only
    getOrderStatus(id: string): OrderDTO         ← signature only
    validateOrder() — 60 lines FULL  ← matched
    mapToDTO(order: Order): OrderDTO             ← signature only
    // ... 8 more methods (signatures omitted for brevity)
  }
```

### Reference Files — Signatures + Previews

Reference files exist so the model can follow conventions. It needs:

1. **Imports** — what libraries/patterns the team uses
1. **Class declaration + decorators** — `@Controller`, `@Service`, etc.
1. **Method signatures** — name, params, return type, decorators
1. **First 8-10 lines of each method body** — enough to see the pattern

This compresses a 300-line file to ~60-80 lines while preserving all convention-relevant information.

-----

## 5. Section-Level Compressors

### Ticket Context — Keep Essential, Drop Noise

```
KEEP (always):
  ticket_id, summary, business_goal, specific_task, acceptance_criteria

KEEP (if enriched):
  inferred_description (from sparse enrichment — this IS the ticket content)
  parent_context.summary + business_goal (1-2 sentences)

DROP (when budget tight):
  epic_context (redundant with parent)
  linked_documentation snippets (reference files cover this)
  full PR implementation summaries from linked PRs (similar PRs section covers this)
```

### Similar PRs — Compact Format

```
BEFORE (~500 tokens per PR × 3 = 1,500):
  PR-456: Implement circuit breaker for payment service
  Core Problem: Payment service was failing silently when the downstream
  settlement API returned 503 errors, causing orders to be marked as
  completed when they hadn't actually been processed...
  Implementation Summary: Added Resilience4j circuit breaker with a 50%
  failure rate threshold over a 10-request sliding window. When open,
  requests fail fast with a custom PaymentUnavailableException that the
  OrderController catches and returns a 503 to the client...
  Key Decisions: Used Resilience4j over Spring Retry because...

AFTER (~150 tokens per PR × 3 = 450):
  PR-456: Circuit breaker for payment service
  → Problem: Silent failures on downstream 503s
  → Approach: Resilience4j, 50% threshold, 10-request window, fail-fast with custom exception
  → Pattern: @CircuitBreaker annotation on service methods
```

### Graph Context — Structured Summary

```
BEFORE (raw blast radius output, ~800 tokens):
  [{ node: "OrderController.submitOrder", type: "function", ... },
   { node: "BatchProcessor.run", type: "function", ... },
   { node: "ScheduledJob.execute", ... },
   { node: "RetryHandler.retry", ... },
   { node: "OrderControllerTest", ... },
   ... 12 more entries with full metadata]

AFTER (~150 tokens):
  CALLERS (4): OrderController.submitOrder, BatchProcessor.run,
    ScheduledJob.execute, RetryHandler.retry
  CALLEES (3): validateOrder, mapToDTO, kafkaProducer.send
  BLAST RADIUS: 8 transitive dependents, 2 test files
  CENTRALITY: High (0.042) — core business function
```

### Known Findings — Ultra-Compact

```
BEFORE (~400 tokens):
  KNOWN FINDING: [security] In src/config/db.ts at line 42, a hardcoded
  database password was detected. Evidence: const password = 'admin123'
  appears in the database configuration section...

AFTER (~40 tokens):
  KNOWN: security:db.ts:42:hardcoded_secret, reliability:processor.ts:88:empty_catch
```

-----

## 6. Implementation — TokenBudget Class

```typescript
// src/guide/context/token_budget.ts

/**
 * Manages a token budget across sections with priority-based allocation
 * and waterfall overflow (unused budget flows to next priority).
 */
export class TokenBudget {
  private remaining: number;
  private allocations: Map<string, number> = new Map();
  private used: Map<string, number> = new Map();

  constructor(
    private readonly totalAvailable: number
  ) {
    this.remaining = totalAvailable;
  }

  /**
   * Allocate a percentage of total budget to a section.
   * Returns the allocated token count.
   */
  allocate(section: string, percentage: number): number {
    const tokens = Math.floor(this.totalAvailable * percentage);
    this.allocations.set(section, tokens);
    return tokens;
  }

  /**
   * Record actual usage for a section. Returns the content trimmed to budget.
   * Unused allocation flows back to remaining pool.
   */
  consume(section: string, content: string): string {
    const budget = this.allocations.get(section) ?? 0;
    const tokens = estimateTokens(content);

    if (tokens <= budget) {
      this.used.set(section, tokens);
      this.remaining -= tokens;
      return content;
    }

    // Over budget — truncate intelligently
    const trimmed = truncateToTokens(content, budget);
    this.used.set(section, budget);
    this.remaining -= budget;
    return trimmed;
  }

  /**
   * Get remaining unallocated + unused budget.
   * Call after all priority sections consumed — gives overflow to lower priorities.
   */
  getRemainingBudget(): number {
    let unused = 0;
    for (const [section, allocated] of this.allocations) {
      const actuallyUsed = this.used.get(section) ?? 0;
      unused += (allocated - actuallyUsed);
    }
    return unused;
  }

  /**
   * Consume from overflow pool (unused budget from higher-priority sections).
   */
  consumeOverflow(section: string, content: string): string {
    const overflow = this.getRemainingBudget();
    if (overflow <= 0) return '';

    const tokens = estimateTokens(content);
    if (tokens <= overflow) {
      this.used.set(section, tokens);
      return content;
    }

    const trimmed = truncateToTokens(content, overflow);
    this.used.set(section, overflow);
    return trimmed;
  }

  /** Debug: show allocation vs usage per section */
  getSummary(): Record<string, { allocated: number; used: number }> {
    const summary: Record<string, { allocated: number; used: number }> = {};
    for (const [section, allocated] of this.allocations) {
      summary[section] = { allocated, used: this.used.get(section) ?? 0 };
    }
    return summary;
  }
}

// ── Token estimation ──

/** Approximate token count: ~4 chars per token for English/code mix */
export function estimateTokens(text: string): number {
  if (!text) return 0;
  return Math.ceil(text.length / 3.8);
}

/** Truncate text to approximately N tokens, breaking at line boundaries */
export function truncateToTokens(text: string, maxTokens: number): string {
  const maxChars = Math.floor(maxTokens * 3.8);
  if (text.length <= maxChars) return text;

  // Break at last newline before the limit
  const truncated = text.slice(0, maxChars);
  const lastNewline = truncated.lastIndexOf('\n');
  if (lastNewline > maxChars * 0.8) {
    return truncated.slice(0, lastNewline) + '\n// ... truncated';
  }
  return truncated + '\n// ... truncated';
}
```

-----

## 7. Implementation — Context Builder

```typescript
// src/guide/context/context_builder.ts

import { TokenBudget, estimateTokens } from './token_budget.js';
import { compressTargetFile, compressReferenceFile } from './file_compressor.js';
import {
  compactTicket, compactPRs, compactGraphContext,
  compactFindings, compactConventions, compactHealthWarnings,
} from './section_compressors.js';

export interface ContextBuilderInput {
  mode: 'understand' | 'implement' | 'diagnose';
  query: string;
  targetFiles: GatheredFile[];      // Files being modified/analyzed
  referenceFiles: GatheredFile[];   // Files for convention reference
  matchedChunkIds: Map<string, string[]>;  // filePath → chunk IDs that search matched
  ticketContext: TicketContext | null;
  similarPRs: PRResult[];
  graphContext: GraphGatherResult;
  conventions: string | null;
  healthWarnings: HealthWarning[];
  knownFindings: string[];
  errorContext?: string;            // DIAGNOSE: stacktrace, error message
  recentChanges?: RecentChange[];   // DIAGNOSE: git log for affected files
  parser: Parser;                   // tree-sitter for AST extraction
}

/** Generation token reservation per mode */
const GENERATION_RESERVE: Record<string, number> = {
  understand: 4_000,
  implement: 8_000,
  diagnose: 6_000,
};

const SYSTEM_PROMPT_RESERVE = 1_000;

/**
 * Build the Qwen context prompt with budget-aware assembly.
 * Sections are filled by priority — higher priority sections get their
 * full allocation; lower priority sections get overflow from unused budgets.
 */
export function buildBudgetedContext(
  input: ContextBuilderInput,
  contextWindowSize: number = 32_768
): { prompt: string; budgetSummary: Record<string, any> } {

  const available = contextWindowSize - SYSTEM_PROMPT_RESERVE - GENERATION_RESERVE[input.mode];
  const budget = new TokenBudget(available);

  const sections: string[] = [];

  // ── QUERY (always first, minimal tokens) ──
  sections.push(`## Task\n${input.query}`);

  // ═══ MODE-SPECIFIC ASSEMBLY ═══

  if (input.mode === 'implement') {
    buildImplementContext(input, budget, sections);
  } else if (input.mode === 'diagnose') {
    buildDiagnoseContext(input, budget, sections);
  } else {
    buildUnderstandContext(input, budget, sections);
  }

  const prompt = sections.filter(Boolean).join('\n\n---\n\n');
  return { prompt, budgetSummary: budget.getSummary() };
}


function buildImplementContext(
  input: ContextBuilderInput,
  budget: TokenBudget,
  sections: string[]
): void {

  // ── P0: Target files — smart extraction (40%) ──
  budget.allocate('target_files', 0.40);
  const targetContent = input.targetFiles.map(f => {
    const matchedChunks = input.matchedChunkIds.get(f.filePath) ?? [];
    return compressTargetFile(f, matchedChunks, input.parser);
  }).join('\n\n');
  sections.push(budget.consume('target_files',
    `## Target Files (modify these)\n${targetContent}`
  ));

  // ── P1: Ticket context (10%) ──
  if (input.ticketContext) {
    budget.allocate('ticket', 0.10);
    sections.push(budget.consume('ticket',
      `## Ticket\n${compactTicket(input.ticketContext)}`
    ));
  }

  // ── P1: Conventions (5%) ──
  if (input.conventions) {
    budget.allocate('conventions', 0.05);
    sections.push(budget.consume('conventions',
      `## Conventions\n${compactConventions(input.conventions)}`
    ));
  }

  // ── P2: Reference files — signatures + previews (20%) ──
  budget.allocate('reference_files', 0.20);
  const refContent = input.referenceFiles.map(f =>
    compressReferenceFile(f, input.parser)
  ).join('\n\n');
  sections.push(budget.consume('reference_files',
    `## Reference Files (follow these patterns)\n${refContent}`
  ));

  // ── P2: Similar PRs (8%) ──
  if (input.similarPRs.length > 0) {
    budget.allocate('similar_prs', 0.08);
    sections.push(budget.consume('similar_prs',
      `## Similar PRs\n${compactPRs(input.similarPRs)}`
    ));
  }

  // ── P3: Graph context (7%) ──
  budget.allocate('graph', 0.07);
  sections.push(budget.consume('graph',
    `## Dependencies & Impact\n${compactGraphContext(input.graphContext)}`
  ));

  // ── P3: Health warnings (3%) ──
  if (input.healthWarnings.length > 0) {
    budget.allocate('health', 0.03);
    sections.push(budget.consume('health',
      `## Health Warnings\n${compactHealthWarnings(input.healthWarnings)}`
    ));
  }

  // ── P4: Known findings — overflow only (2%) ──
  if (input.knownFindings.length > 0) {
    budget.allocate('known_findings', 0.02);
    const compact = compactFindings(input.knownFindings);
    sections.push(budget.consume('known_findings',
      `## Known Issues (do not repeat)\n${compact}`
    ));
  }
}


function buildDiagnoseContext(
  input: ContextBuilderInput,
  budget: TokenBudget,
  sections: string[]
): void {

  // ── P0: Error context (15%) ──
  if (input.errorContext) {
    budget.allocate('error', 0.15);
    sections.push(budget.consume('error',
      `## Error\n\`\`\`\n${input.errorContext}\n\`\`\``
    ));
  }

  // ── P0: Target file — error location (30%) ──
  budget.allocate('target_files', 0.30);
  const targetContent = input.targetFiles.map(f => {
    const matchedChunks = input.matchedChunkIds.get(f.filePath) ?? [];
    return compressTargetFile(f, matchedChunks, input.parser);
  }).join('\n\n');
  sections.push(budget.consume('target_files',
    `## Error Location\n${targetContent}`
  ));

  // ── P1: Reverse call chain (10%) ──
  budget.allocate('graph', 0.10);
  sections.push(budget.consume('graph',
    `## Call Chain\n${compactGraphContext(input.graphContext)}`
  ));

  // ── P1: Recent changes (10%) ──
  if (input.recentChanges && input.recentChanges.length > 0) {
    budget.allocate('recent_changes', 0.10);
    const changes = input.recentChanges.map(c =>
      `${c.hash.slice(0, 8)} ${c.author} ${c.date}: ${c.message}`
    ).join('\n');
    sections.push(budget.consume('recent_changes',
      `## Recent Changes\n${changes}`
    ));
  }

  // ── P2: Config context (10%) ──
  if (input.referenceFiles.length > 0) {
    budget.allocate('config', 0.10);
    const configs = input.referenceFiles
      .filter(f => /\.(yml|yaml|properties|json)$/.test(f.filePath))
      .map(f => `### ${f.filePath}\n${f.content}`)
      .join('\n\n');
    sections.push(budget.consume('config', `## Config\n${configs}`));
  }
}


function buildUnderstandContext(
  input: ContextBuilderInput,
  budget: TokenBudget,
  sections: string[]
): void {

  // UNDERSTAND uses fast model with 64K context — rarely hits limits
  // Still apply budget discipline for very large codebases

  budget.allocate('target_files', 0.45);
  const targetContent = input.targetFiles.map(f => {
    const matchedChunks = input.matchedChunkIds.get(f.filePath) ?? [];
    return compressTargetFile(f, matchedChunks, input.parser);
  }).join('\n\n');
  sections.push(budget.consume('target_files',
    `## Code\n${targetContent}`
  ));

  budget.allocate('graph', 0.20);
  sections.push(budget.consume('graph',
    `## Architecture\n${compactGraphContext(input.graphContext)}`
  ));

  if (input.referenceFiles.length > 0) {
    budget.allocate('reference_files', 0.15);
    const refContent = input.referenceFiles.map(f =>
      compressReferenceFile(f, input.parser)
    ).join('\n\n');
    sections.push(budget.consume('reference_files',
      `## Related Code\n${refContent}`
    ));
  }

  if (input.ticketContext) {
    budget.allocate('ticket', 0.05);
    sections.push(budget.consume('ticket',
      `## Ticket\n${compactTicket(input.ticketContext)}`
    ));
  }

  if (input.similarPRs.length > 0) {
    budget.allocate('similar_prs', 0.05);
    sections.push(budget.consume('similar_prs',
      `## Related PRs\n${compactPRs(input.similarPRs)}`
    ));
  }
}
```

-----

## 8. Implementation — File Compressor

```typescript
// src/guide/context/file_compressor.ts

import Parser from 'tree-sitter';

interface CompressedFile {
  content: string;
  originalLines: number;
  compressedLines: number;
  strategy: 'full' | 'smart_extract' | 'signatures_only';
}

/**
 * Compress a TARGET file — preserve matched regions in full,
 * show everything else as signatures.
 *
 * Target files are what the model will MODIFY — it needs exact line numbers
 * and full content for the sections being changed.
 */
export function compressTargetFile(
  file: GatheredFile,
  matchedChunkIds: string[],
  parser: Parser
): string {
  const lines = file.content.split('\n');

  // Small file — send in full
  if (lines.length <= 150) {
    return formatFileWithLineNumbers(file.filePath, file.content);
  }

  const tree = parser.parse(file.content);
  const root = tree.rootNode;

  // ── Step 1: Identify matched line ranges from chunk metadata ──
  const matchedRanges: Array<[number, number]> = [];
  for (const chunkId of matchedChunkIds) {
    // Chunk metadata has start_line/end_line
    const range = file.chunkRanges?.get(chunkId);
    if (range) {
      matchedRanges.push([range.startLine, range.endLine]);
    }
  }

  // If no chunk ranges available, fall back to full file with head/tail truncation
  if (matchedRanges.length === 0) {
    return formatFileHeadTail(file.filePath, lines, 120, 40);
  }

  // ── Step 2: Extract file structure ──
  const imports = extractImportBlock(root, lines);
  const classDecl = extractClassDeclaration(root, lines);
  const fields = extractFieldDeclarations(root, lines);
  const methods = extractMethodInfo(root, lines);

  // ── Step 3: Build compressed output ──
  const parts: string[] = [];

  // Always include: imports + class declaration + fields
  parts.push(`### ${file.filePath}`);
  parts.push('```' + detectLanguage(file.filePath));

  if (imports) parts.push(imports);
  if (classDecl) parts.push(classDecl);
  if (fields) parts.push(fields);

  // For each method: full content if matched, signature-only if not
  for (const method of methods) {
    const isMatched = matchedRanges.some(([start, end]) =>
      method.startLine >= start && method.startLine <= end ||
      method.endLine >= start && method.endLine <= end ||
      start >= method.startLine && end <= method.endLine
    );

    if (isMatched) {
      // Full method with line numbers
      const methodLines = lines.slice(method.startLine, method.endLine + 1);
      const numbered = methodLines.map((line, i) =>
        `${String(method.startLine + i + 1).padStart(4)}| ${line}`
      ).join('\n');
      parts.push(numbered);
    } else {
      // Signature only
      parts.push(`  // L${method.startLine + 1}: ${method.signature}  { ... ${method.endLine - method.startLine + 1} lines }`);
    }
  }

  parts.push('```');

  return parts.join('\n');
}


/**
 * Compress a REFERENCE file — signatures + method previews.
 * The model uses this to understand patterns and conventions,
 * not to modify directly.
 */
export function compressReferenceFile(
  file: GatheredFile,
  parser: Parser
): string {
  const lines = file.content.split('\n');

  // Very small file — send in full
  if (lines.length <= 80) {
    return `### ${file.filePath} (reference)\n\`\`\`${detectLanguage(file.filePath)}\n${file.content}\n\`\`\``;
  }

  const tree = parser.parse(file.content);
  const root = tree.rootNode;

  const imports = extractImportBlock(root, lines);
  const classDecl = extractClassDeclaration(root, lines);
  const fields = extractFieldDeclarations(root, lines);
  const methods = extractMethodInfo(root, lines);

  const parts: string[] = [];
  parts.push(`### ${file.filePath} (reference — ${lines.length} lines)`);
  parts.push('```' + detectLanguage(file.filePath));

  if (imports) parts.push(imports);
  if (classDecl) parts.push(classDecl);
  if (fields) parts.push(fields);

  // Method signatures + first 8 lines of body (enough to see pattern)
  const PREVIEW_LINES = 8;
  for (const method of methods) {
    const methodLines = lines.slice(method.startLine, method.endLine + 1);
    if (methodLines.length <= PREVIEW_LINES + 2) {
      // Short method — include fully
      parts.push(methodLines.join('\n'));
    } else {
      // Preview: signature + first N lines + indicator
      const preview = methodLines.slice(0, PREVIEW_LINES + 1).join('\n');
      parts.push(`${preview}\n    // ... ${methodLines.length - PREVIEW_LINES - 1} more lines`);
    }
    parts.push('');
  }

  parts.push('```');
  return parts.join('\n');
}


// ── AST Extraction Helpers ──

function extractImportBlock(root: Parser.SyntaxNode, lines: string[]): string | null {
  const importNodes = root.children.filter(n =>
    n.type === 'import_statement' || n.type === 'import_declaration' ||
    n.type === 'lexical_declaration' && lines[n.startPosition.row]?.includes('require(') ||
    n.type === 'package_declaration'
  );
  if (importNodes.length === 0) return null;

  const startLine = importNodes[0].startPosition.row;
  const endLine = importNodes[importNodes.length - 1].endPosition.row;
  return lines.slice(startLine, endLine + 1).join('\n');
}

function extractClassDeclaration(root: Parser.SyntaxNode, lines: string[]): string | null {
  const classNode = root.children.find(n =>
    n.type === 'class_declaration' || n.type === 'export_statement'
  );
  if (!classNode) return null;

  // Get the class declaration line(s) including decorators, up to opening brace
  const row = classNode.startPosition.row;
  let endRow = row;
  for (let i = row; i < Math.min(row + 5, lines.length); i++) {
    if (lines[i]?.includes('{')) { endRow = i; break; }
    endRow = i;
  }
  return lines.slice(row, endRow + 1).join('\n');
}

function extractFieldDeclarations(root: Parser.SyntaxNode, lines: string[]): string | null {
  // Find field declarations (class properties, instance variables)
  const classBody = findFirstChild(root, 'class_body') ?? findFirstChild(root, 'class_declaration');
  if (!classBody) return null;

  const fields = classBody.children.filter(n =>
    n.type === 'field_definition' || n.type === 'public_field_definition' ||
    n.type === 'property_declaration' ||
    (n.type === 'field_declaration' && !lines[n.startPosition.row]?.includes('('))
  );

  if (fields.length === 0) return null;
  return fields.map(f => lines.slice(f.startPosition.row, f.endPosition.row + 1).join('\n')).join('\n');
}

interface MethodInfo {
  name: string;
  signature: string;
  startLine: number;
  endLine: number;
}

function extractMethodInfo(root: Parser.SyntaxNode, lines: string[]): MethodInfo[] {
  const methods: MethodInfo[] = [];

  function findMethods(node: Parser.SyntaxNode) {
    if (
      node.type === 'method_definition' || node.type === 'method_declaration' ||
      node.type === 'function_declaration' || node.type === 'function_definition' ||
      node.type === 'arrow_function' || node.type === 'public_field_definition'
    ) {
      const nameNode = node.childForFieldName('name');
      const name = nameNode?.text ?? 'anonymous';

      // Build signature from first line (decorators + method declaration)
      let sigStart = node.startPosition.row;
      // Include preceding decorator lines
      const prevSibling = node.previousNamedSibling;
      if (prevSibling?.type === 'decorator') {
        sigStart = prevSibling.startPosition.row;
      }

      const sigLine = lines[node.startPosition.row]?.trim() ?? '';
      const signature = sigLine.replace(/\{.*$/, '').trim();

      methods.push({
        name,
        signature,
        startLine: sigStart,
        endLine: node.endPosition.row,
      });
      return; // Don't recurse into method bodies
    }

    for (const child of node.children) {
      findMethods(child);
    }
  }

  findMethods(root);
  return methods;
}

function findFirstChild(node: Parser.SyntaxNode, type: string): Parser.SyntaxNode | null {
  for (const child of node.children) {
    if (child.type === type) return child;
    const found = findFirstChild(child, type);
    if (found) return found;
  }
  return null;
}

function formatFileWithLineNumbers(filePath: string, content: string): string {
  const lines = content.split('\n');
  const numbered = lines.map((line, i) =>
    `${String(i + 1).padStart(4)}| ${line}`
  ).join('\n');
  return `### ${filePath}\n\`\`\`${detectLanguage(filePath)}\n${numbered}\n\`\`\``;
}

function formatFileHeadTail(
  filePath: string, lines: string[], headLines: number, tailLines: number
): string {
  if (lines.length <= headLines + tailLines) {
    return formatFileWithLineNumbers(filePath, lines.join('\n'));
  }

  const head = lines.slice(0, headLines).map((l, i) =>
    `${String(i + 1).padStart(4)}| ${l}`
  ).join('\n');

  const tail = lines.slice(-tailLines).map((l, i) => {
    const lineNum = lines.length - tailLines + i + 1;
    return `${String(lineNum).padStart(4)}| ${l}`;
  }).join('\n');

  const omitted = lines.length - headLines - tailLines;

  return `### ${filePath}\n\`\`\`${detectLanguage(filePath)}\n${head}\n\n   // ... ${omitted} lines omitted ...\n\n${tail}\n\`\`\``;
}

function detectLanguage(filePath: string): string {
  const ext = filePath.split('.').pop()?.toLowerCase();
  const map: Record<string, string> = {
    ts: 'typescript', tsx: 'typescript', js: 'javascript', jsx: 'javascript',
    java: 'java', kt: 'kotlin', py: 'python', go: 'go',
    yml: 'yaml', yaml: 'yaml', json: 'json', xml: 'xml',
  };
  return map[ext ?? ''] ?? '';
}
```

-----

## 9. Implementation — Section Compressors

```typescript
// src/guide/context/section_compressors.ts

/**
 * Compact ticket context — keep essential fields only.
 */
export function compactTicket(ticket: TicketContext): string {
  const parts: string[] = [];

  parts.push(`${ticket.ticketId}: ${ticket.summary}`);

  // Prefer inferred description (from sparse enrichment) if original is weak
  if (ticket.inferredDescription) {
    parts.push(`Description: ${ticket.inferredDescription}`);
  } else if (ticket.businessGoal) {
    parts.push(`Goal: ${ticket.businessGoal}`);
  }

  if (ticket.specificTask) {
    parts.push(`Task: ${ticket.specificTask}`);
  }

  if (ticket.acceptanceCriteria) {
    parts.push(`AC: ${ticket.acceptanceCriteria}`);
  }

  // Parent context — one line only
  if (ticket.parentContext) {
    parts.push(`Parent (${ticket.parentContext.key}): ${ticket.parentContext.summary}`);
  }

  return parts.join('\n');
}


/**
 * Compact similar PRs — problem + approach in 2-3 lines each.
 */
export function compactPRs(prs: PRResult[]): string {
  return prs.map(pr => {
    const lines: string[] = [];
    lines.push(`PR-${pr.pr_number}: ${pr.title}`);
    if (pr.core_problem) {
      lines.push(`→ Problem: ${truncate(pr.core_problem, 120)}`);
    }
    if (pr.implementation_summary) {
      lines.push(`→ Approach: ${truncate(pr.implementation_summary, 180)}`);
    }
    if (pr.key_decisions) {
      lines.push(`→ Decision: ${truncate(pr.key_decisions, 120)}`);
    }
    return lines.join('\n');
  }).join('\n\n');
}


/**
 * Compact graph context — structured one-line summaries.
 */
export function compactGraphContext(graph: GraphGatherResult): string {
  const parts: string[] = [];

  // Callers
  if (graph.callers?.length) {
    const callerNames = graph.callers.slice(0, 6).map(c => c.name ?? c.symbol).join(', ');
    const more = graph.callers.length > 6 ? ` +${graph.callers.length - 6} more` : '';
    parts.push(`CALLED BY: ${callerNames}${more}`);
  }

  // Callees
  if (graph.callees?.length) {
    const calleeNames = graph.callees.slice(0, 6).map(c => c.name ?? c.symbol).join(', ');
    parts.push(`CALLS: ${calleeNames}`);
  }

  // Blast radius (if computed)
  if (graph.blastRadius) {
    const br = graph.blastRadius;
    parts.push(`BLAST RADIUS: ${br.directDependents ?? 0} direct, ${br.transitiveDependents ?? 0} transitive, ${br.affectedTests ?? 0} tests`);
  }

  // Centrality
  if (graph.centrality) {
    const label = graph.centrality.score > 0.02 ? 'HIGH' :
                  graph.centrality.score > 0.005 ? 'MODERATE' : 'LOW';
    parts.push(`CENTRALITY: ${label} (${graph.centrality.score.toFixed(3)})`);
  }

  // Related files (co-change)
  if (graph.relatedFiles?.length) {
    parts.push(`OFTEN CHANGES WITH: ${graph.relatedFiles.slice(0, 4).join(', ')}`);
  }

  return parts.join('\n');
}


/**
 * Ultra-compact known findings — one line per finding.
 */
export function compactFindings(findings: string[]): string {
  // Assume findings are already in a structured format from Layer 1/2
  // Compress to: dimension:file:line:type
  return findings.slice(0, 10).map(f => {
    // Try to extract key info from the finding string
    const match = f.match(/\[(\w+)\].*?(\S+\.\w+):?(\d+)?.*?(\w+)$/);
    if (match) {
      return `${match[1]}:${match[2]}:${match[3] ?? '?'}:${match[4]}`;
    }
    return truncate(f, 80);
  }).join('\n');
}


/**
 * Compact conventions — already short from fast model, just ensure format.
 */
export function compactConventions(conventions: string): string {
  // Conventions from fast model are usually ~200-400 tokens already
  // Just cap at 500 tokens
  return truncate(conventions, 1900); // ~500 tokens × 3.8 chars
}


/**
 * Compact health warnings — one line each.
 */
export function compactHealthWarnings(warnings: HealthWarning[]): string {
  return warnings.slice(0, 5).map(w =>
    `[${w.severity}] ${w.dimension}: ${truncate(w.message, 100)}`
  ).join('\n');
}


function truncate(text: string, maxChars: number): string {
  if (!text || text.length <= maxChars) return text;
  return text.slice(0, maxChars - 3) + '...';
}
```

-----

## 10. Integration with Orchestrator

Update step 8 in `orchestrator.ts`:

```typescript
// src/guide/orchestrator.ts — REPLACE step 8

// ── 8. BUILD QWEN CONTEXT (budget-aware) ──
const contextWindowSize = intent.mode === 'understand'
  ? parseInt(process.env.FAST_MODEL_CTX_SIZE || '65536', 10)
  : parseInt(process.env.DEEP_MODEL_CTX_SIZE || '32768', 10);

const { prompt: qwenContext, budgetSummary } = buildBudgetedContext({
  mode: intent.mode,
  query: params.query,
  targetFiles: fullFiles.filter(f => f.isTarget),
  referenceFiles: fullFiles.filter(f => !f.isTarget),
  matchedChunkIds: gathered.matchedChunkIds,   // NEW: pass through from gather
  ticketContext: gathered.ticketContext,
  similarPRs,
  graphContext: gathered.graphContext,
  conventions: gathered.conventions,
  healthWarnings: gathered.healthWarnings,
  knownFindings: gathered.knownFindings ?? [],
  errorContext: params.context,
  recentChanges: gathered.recentChanges,
  parser,
}, contextWindowSize);

// Log budget usage for monitoring
console.log(`[Guide] Context budget: ${JSON.stringify(budgetSummary)}`);
```

### matchedChunkIds Propagation

The gather executor needs to track which chunks matched which files, so the file compressor knows which methods to show in full:

```typescript
// In gather_executor.ts — when processing search results

const matchedChunkIds = new Map<string, string[]>();

for (const result of searchResults) {
  const existing = matchedChunkIds.get(result.filePath) ?? [];
  existing.push(result.chunkId);
  matchedChunkIds.set(result.filePath, existing);
}

gathered.matchedChunkIds = matchedChunkIds;
```

-----

## 11. Quality Safeguards

### Never Lose These

|Content                       |Rule                                         |Rationale                                                    |
|------------------------------|---------------------------------------------|-------------------------------------------------------------|
|Target file imports           |Always included in full                      |Generated code references these — wrong imports = broken code|
|Target file matched methods   |Full content with line numbers               |Model is EDITING this exact code                             |
|Target file field declarations|Always included                              |Model needs state context for method generation              |
|Ticket acceptance criteria    |Always included (even if truncated elsewhere)|Defines correctness                                          |
|Convention output             |Always included (already compact)            |Determines code style                                        |

### Compression Quality Checks

```typescript
// Run after compression to verify nothing critical was lost

function validateCompressedContext(
  input: ContextBuilderInput,
  compressed: string
): string[] {
  const warnings: string[] = [];

  // Check target files have line numbers (required for IMPLEMENT)
  if (input.mode === 'implement') {
    for (const f of input.targetFiles) {
      if (!compressed.includes(f.filePath)) {
        warnings.push(`CRITICAL: Target file ${f.filePath} missing from context`);
      }
    }
  }

  // Check ticket ID is present
  if (input.ticketContext && !compressed.includes(input.ticketContext.ticketId)) {
    warnings.push(`WARNING: Ticket ${input.ticketContext.ticketId} missing from context`);
  }

  // Check at least one convention line present
  if (input.conventions && !compressed.includes('Convention')) {
    warnings.push(`WARNING: Conventions section dropped`);
  }

  return warnings;
}
```

### Fallback: If Context STILL Too Large

If after all compression the prompt still exceeds the budget (e.g., 8 matched target files each with 100+ line matched methods), reduce target files:

```typescript
// In buildImplementContext — after initial compression
const estimatedTotal = estimateTokens(sections.join('\n'));
if (estimatedTotal > available * 0.95) {
  // Drop lowest-score target files until it fits
  // Keep minimum 2 target files
  console.warn(`[Guide] Context still over budget (${estimatedTotal} tokens). Trimming target files.`);
}
```

-----

## 12. Configuration

```bash
# ═══ Context Window Sizes ═══
FAST_MODEL_CTX_SIZE=65536        # Coder-Next :8090
DEEP_MODEL_CTX_SIZE=32768        # 235B :8091

# ═══ Budget Tuning ═══
IMPLEMENT_GENERATION_RESERVE=8000   # Tokens reserved for IMPLEMENT output
DIAGNOSE_GENERATION_RESERVE=6000
UNDERSTAND_GENERATION_RESERVE=4000

# ═══ File Compression ═══
TARGET_FILE_FULL_THRESHOLD=150     # Lines — files smaller than this sent in full
REF_FILE_FULL_THRESHOLD=80         # Lines — reference files smaller than this sent in full
REF_METHOD_PREVIEW_LINES=8         # Lines of method body to show in reference files
HEAD_TAIL_HEAD_LINES=120           # Head lines when no AST extraction possible
HEAD_TAIL_TAIL_LINES=40            # Tail lines when no AST extraction possible

# ═══ Section Limits ═══
MAX_SIMILAR_PRS_IN_CONTEXT=3
MAX_HEALTH_WARNINGS_IN_CONTEXT=5
MAX_KNOWN_FINDINGS_IN_CONTEXT=10
MAX_CALLERS_IN_CONTEXT=6
MAX_CALLEES_IN_CONTEXT=6
MAX_RELATED_FILES_IN_CONTEXT=4
```

-----

## 13. Implementation Schedule

```
Day 1: TokenBudget + Section Compressors
  ├── token_budget.ts (estimator, allocator, consumer, overflow)
  ├── section_compressors.ts (ticket, PRs, graph, findings, health, conventions)
  ├── Unit tests for each compressor
  └── Verify: same information, 50-70% fewer tokens

Day 2: File Compressor (AST-Based)
  ├── file_compressor.ts
  │   ├── compressTargetFile() — smart extraction using matched chunk ranges
  │   ├── compressReferenceFile() — signatures + method previews
  │   └── AST helpers (imports, class decl, fields, methods)
  ├── Test on 5 representative files from actual repos
  │   (large Spring service, React component, utility file, config, test)
  └── Verify: matched methods in full, unmatched as signatures

Day 3: Context Builder + Integration + Testing
  ├── context_builder.ts (budget-aware assembly per mode)
  ├── matchedChunkIds propagation from gather_executor
  ├── Integration with orchestrator.ts
  ├── Validation checks (no critical content dropped)
  ├── Test all 3 modes against real queries that previously caused 400 errors
  └── Log budget summaries — verify no mode exceeds context window
```

### File Structure

```
src/guide/context/
  token_budget.ts              — Budget allocation + consumption + overflow
  context_builder.ts           — Mode-specific assembly with budget
  file_compressor.ts           — AST-based target + reference file compression
  section_compressors.ts       — Ticket, PRs, graph, findings compactors
```

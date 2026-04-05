# OME Agent Phase 2 — Accuracy & Performance Engineering Plan

> **Document Version:** 1.0 — 2026-04-05
> **Scope:** Tool orchestration, prompt engineering, verification layers, and performance tuning for Jira-to-PR pipeline accuracy
> **Model:** Qwen3.5-122B-A10B (deep, :8091) + Qwen3-Coder-Next (fast, :8090)
> **Prerequisite:** Phase 1 complete, Phase 2 pipeline architecture defined
> **Goal:** Every generated CodeChange compiles, passes tests, and matches team conventions — or the pipeline aborts cleanly

-----

## Table of Contents

1. [Accuracy Problem Statement](#1-accuracy-problem-statement)
1. [Three-Pass Architecture](#2-three-pass-architecture)
1. [Pass 1 — Context Gathering (Tool Orchestration)](#3-pass-1--context-gathering)
1. [Pass 2 — Code Generation (Prompt Engineering)](#4-pass-2--code-generation)
1. [Pass 3 — Verification & Self-Correction](#5-pass-3--verification--self-correction)
1. [Tool Call Optimization Patterns](#6-tool-call-optimization-patterns)
1. [Prompt Templates — Full Specifications](#7-prompt-templates)
1. [Convention Detection & Enforcement](#8-convention-detection--enforcement)
1. [Multi-File Coherence](#9-multi-file-coherence)
1. [Hallucination Prevention — Deep](#10-hallucination-prevention)
1. [Performance Budget & Optimization](#11-performance-budget)
1. [SCIP Query-Time Verification Integration](#12-scip-verification)
1. [Accuracy Metrics & Evaluation](#13-accuracy-metrics)
1. [Prompt Tuning Feedback Loop](#14-prompt-tuning-feedback-loop)
1. [Implementation Schedule](#15-implementation-schedule)

-----

## 1. Accuracy Problem Statement

### 1.1 What “Accurate” Means for Jira-to-PR

An accurate pipeline output satisfies ALL of these:

|Criterion                      |Verification Method                              |
|-------------------------------|-------------------------------------------------|
|Code compiles                  |`tsc --noEmit` / `mvn compile` in worktree       |
|Lint passes                    |ESLint / Checkstyle on changed files             |
|Tests pass                     |Affected test suites in isolated worktree        |
|Line numbers are correct       |Content hash match against actual file           |
|Symbols exist                  |Knowledge graph + SCIP verification              |
|Imports are complete           |AST parse of generated code → resolve all imports|
|Conventions followed           |Convention snapshot comparison                   |
|No hallucinated files/functions|Filesystem + graph cross-check                   |
|Blast radius is accurate       |Reverse call chain traversal verified            |
|Change is minimal              |No unnecessary modifications outside scope       |

### 1.2 Where Errors Originate

|Error Source            |Root Cause                                              |Fix Strategy                                 |
|------------------------|--------------------------------------------------------|---------------------------------------------|
|Wrong line numbers      |Model guesses lines from truncated context              |Full file with line numbers in prompt        |
|Missing imports         |Model generates code referencing unseen modules         |Import resolution pass after generation      |
|Wrong API signatures    |Model hallucinates method parameters                    |SCIP symbol resolution + graph signature data|
|Convention violations   |Model uses its training patterns, not team patterns     |Convention snapshot injection into prompt    |
|Stale code references   |Plan generated on old code, file changed since          |Content hash gate before execution           |
|Incomplete changes      |Model modifies 1 of 3 files that need updating          |Blast radius-driven completeness check       |
|Wrong framework patterns|Model uses Spring patterns in React code (or vice versa)|Framework-specific prompt templates          |
|Scope creep             |Model refactors beyond the ticket requirement           |Explicit scope boundary in prompt            |

-----

## 2. Three-Pass Architecture

Every IMPLEMENT invocation runs three distinct passes. Each pass has a clear purpose and specific tool calls.

```
Pass 1: GATHER (Fast Model + RAG Tools)
  Purpose: Build a complete, accurate context window
  Model: Coder-Next (:8090) for intent/extraction only
  Tools: 8-12 RAG MCP calls in parallel
  Output: ContextBundle (files, graph, conventions, similar PRs)
  Time: ~3-5 seconds

Pass 2: GENERATE (Deep Model)
  Purpose: Produce CodeChange[] from context
  Model: Qwen3.5-122B (:8091)
  Tools: None — pure generation from assembled context
  Output: Raw CodeChange[] + NewFile[]
  Time: ~15-30 seconds

Pass 3: VERIFY (Fast Model + Tools + SCIP)
  Purpose: Catch and fix errors before execution
  Model: Coder-Next (:8090) for lightweight fixes
  Tools: Filesystem checks, SCIP verification, import resolution
  Output: Verified CodeChange[] with confidence scores
  Time: ~5-10 seconds

Total: ~25-45 seconds (vs ~45-90s with single-pass approach)
```

### 2.1 Why Three Passes Beat One Pass

Single-pass generation asks the model to simultaneously: understand the ticket, find relevant code, detect conventions, generate changes, verify correctness, and handle imports. This overloads context and produces errors.

Three passes let each stage focus on one concern:

- Gather ensures the model sees the RIGHT context (garbage in → garbage out prevention)
- Generate focuses purely on code quality with all context pre-assembled
- Verify catches what the model missed without asking it to self-evaluate during generation

-----

## 3. Pass 1 — Context Gathering (Tool Orchestration)

### 3.1 Gather Plan — What to Collect and Why

```typescript
// src/guide/gather/gather_plan.ts

export interface GatherPlan {
  // ── Core context (always gathered) ──
  semanticSearch: {
    query: string;
    scope: 'code';
    topK: 10;
    repoFilter?: string;
  };

  symbolLookups: {
    symbols: string[];        // Extracted from ticket + query
    includeCallChain: boolean;
    maxDepth: number;
  };

  projectInfo: {
    repoSlug: string;
    needsTechStack: true;
    needsDependencies: true;
  };

  // ── Targeted context (gathered based on extracted elements) ──
  fileReads: {
    files: string[];          // From semantic search results + current_file
    readFull: true;           // ALWAYS read full files — never truncated
    withLineNumbers: true;    // Numbered lines for accurate change targets
  };

  blastRadius: {
    symbols: string[];        // Primary symbols being modified
    maxDepth: 2;
  };

  // ── Convention context (always for IMPLEMENT) ──
  conventionSample: {
    filePatterns: string[];   // Same-directory files for style reference
    maxFiles: 3;
    maxLinesPerFile: 200;
  };

  // ── Historical context (when available) ──
  similarPRs: {
    query: string;
    maxResults: 3;
  };

  ticketContext: {
    ticketKey: string;
    enrichSparse: true;
  };

  // ── Dependency context (for import resolution) ──
  fileDependencies: {
    files: string[];          // Files being modified
    direction: 'both';       // imports AND imported-by
  };
}
```

### 3.2 Parallel Tool Execution Strategy

```typescript
// src/guide/gather/gather_executor.ts

export async function executeGatherPlan(
  plan: GatherPlan,
  ctx: AgentContext
): Promise<ContextBundle> {

  // ── Wave 1: Independent queries (all parallel) ──
  const wave1 = await Promise.allSettled([
    ctx.rag.searchCode(plan.semanticSearch.query, {
      topK: plan.semanticSearch.topK,
      repo: plan.semanticSearch.repoFilter,
    }),
    ctx.rag.getProjectInfo(plan.projectInfo.repoSlug),
    plan.ticketContext.ticketKey
      ? ctx.rag.getTicketContext(plan.ticketContext.ticketKey)
      : null,
    plan.similarPRs
      ? searchSimilarPRs(plan.similarPRs.query, ctx)
      : null,
  ]);

  const [searchResults, projectInfo, ticketContext, similarPRs] = extractSettled(wave1);

  // ── Wave 2: Depends on Wave 1 results (parallel within wave) ──
  // Extract file paths from search results for full file reads
  const targetFiles = deduplicateFiles([
    ...plan.fileReads.files,
    ...extractFilePaths(searchResults).slice(0, 8),
  ]);

  const wave2 = await Promise.allSettled([
    // Read full files from disk (not chunks — full files with line numbers)
    batchReadFilesWithLineNumbers(targetFiles, ctx),

    // Symbol lookups for all extracted symbols
    ...plan.symbolLookups.symbols.slice(0, 5).map(sym =>
      ctx.rag.getCodeContext(sym, plan.symbolLookups.maxDepth)
    ),

    // Blast radius for primary symbols
    ...plan.blastRadius.symbols.slice(0, 3).map(sym =>
      ctx.rag.analyzeBlastRadius(sym)
    ),

    // File dependencies for import context
    ...plan.fileDependencies.files.slice(0, 5).map(file =>
      ctx.rag.getFileDependencies(file)
    ),
  ]);

  // ── Wave 3: Convention sampling (depends on Wave 1 for repo/framework info) ──
  const framework = mapTechStackToFramework(projectInfo?.tech_stack);
  const conventionFiles = await gatherConventionSamples(
    plan.conventionSample, framework, targetFiles, ctx
  );

  // ── Assemble ContextBundle ──
  return assembleContext(
    searchResults, projectInfo, ticketContext, similarPRs,
    wave2, conventionFiles, framework, targetFiles
  );
}
```

### 3.3 Full File Reading with Line Numbers

This is the single most important accuracy decision. Never send truncated files to the deep model for IMPLEMENT mode.

```typescript
// src/guide/gather/file_reader.ts

export async function readFileWithLineNumbers(
  filePath: string,
  repoSlug: string
): Promise<NumberedFile> {
  const fullPath = path.join(process.env.REPOS_DIR!, repoSlug, filePath);
  const content = await fs.readFile(fullPath, 'utf-8');
  const lines = content.split('\n');

  // Add line numbers for precise change targeting
  const numbered = lines
    .map((line, i) => `${String(i + 1).padStart(4)}│ ${line}`)
    .join('\n');

  return {
    path: filePath,
    content: content,           // Raw content for hash verification
    numberedContent: numbered,  // Numbered for model prompt
    lineCount: lines.length,
    contentHash: md5(content),
  };
}

export async function batchReadFilesWithLineNumbers(
  files: string[],
  ctx: AgentContext
): Promise<NumberedFile[]> {
  const results: NumberedFile[] = [];

  for (const file of files) {
    try {
      // Determine repo from file path or use default
      const repoSlug = extractRepoFromPath(file) || ctx.currentRepo;
      const numbered = await readFileWithLineNumbers(file, repoSlug);

      // Size gate: skip files > 500 lines for context window management
      // But NEVER skip the primary target files
      if (numbered.lineCount > 500 && !ctx.targetFiles.includes(file)) {
        // Read head + tail for reference files
        const headTail = extractHeadTail(numbered, 300, 100);
        results.push({ ...numbered, numberedContent: headTail, truncated: true });
      } else {
        results.push(numbered);
      }
    } catch {
      // File not on disk — flag but don't fail
      results.push({
        path: file,
        content: '',
        numberedContent: `[FILE NOT FOUND: ${file}]`,
        lineCount: 0,
        contentHash: '',
        missing: true,
      });
    }
  }

  return results;
}
```

### 3.4 Convention Sampling

```typescript
// src/guide/gather/convention_sampler.ts

export async function gatherConventionSamples(
  plan: ConventionSamplePlan,
  framework: Framework,
  targetFiles: string[],
  ctx: AgentContext
): Promise<ConventionSnapshot> {
  const samples: ConventionFile[] = [];

  // Strategy 1: Same-directory files (strongest convention signal)
  for (const target of targetFiles.slice(0, 3)) {
    const dir = path.dirname(target);
    const siblings = await listDirectory(dir, ctx.currentRepo);
    const sameType = siblings
      .filter(f => f !== path.basename(target))
      .filter(f => matchesExtension(f, target))
      .slice(0, 2);

    for (const sibling of sameType) {
      const content = await readFileWithLineNumbers(
        path.join(dir, sibling), ctx.currentRepo
      );
      if (content.lineCount > 0 && content.lineCount <= 200) {
        samples.push({
          path: path.join(dir, sibling),
          content: content.numberedContent,
          relationship: 'same_directory',
        });
      }
    }
  }

  // Strategy 2: Similar-name files in other directories
  // e.g., if modifying BondService.java, find EquityService.java
  for (const target of targetFiles.slice(0, 2)) {
    const baseName = path.basename(target);
    const pattern = extractNamingPattern(baseName);
    if (pattern) {
      const similar = await ctx.rag.searchCode(
        `file similar to ${baseName} ${pattern.suffix}`,
        { topK: 2, repo: ctx.currentRepo }
      );
      for (const result of similar) {
        if (result.filePath && result.filePath !== target) {
          const content = await readFileWithLineNumbers(
            result.filePath, ctx.currentRepo
          );
          if (content.lineCount <= 200) {
            samples.push({
              path: result.filePath,
              content: content.numberedContent,
              relationship: 'naming_pattern_match',
            });
          }
        }
      }
    }
  }

  return {
    files: samples.slice(0, 5), // Cap at 5 convention reference files
    framework,
    detectedPatterns: [], // Filled by Coder-Next in convention detection step
  };
}
```

### 3.5 Context Bundle Assembly

```typescript
// src/guide/gather/context_assembler.ts

export interface ContextBundle {
  // ── Primary context (always present) ──
  targetFiles: NumberedFile[];         // Full files being modified, with line numbers
  referenceFiles: NumberedFile[];      // Convention samples + dependency files
  graphContext: Map<string, {
    callers: string[];
    callees: string[];
    centralityScore: number;
    decorators: string | null;
    signature: string | null;
    startLine: number;
    endLine: number;
  }>;

  // ── Ticket context ──
  ticketDescription: string;          // Enriched description
  acceptanceCriteria: string | null;
  parentContext: string | null;

  // ── Historical context ──
  similarPRs: {
    title: string;
    coreProblem: string;
    implementationSummary: string;
    keyDecisions: string;
  }[];

  // ── Framework context ──
  framework: Framework;
  techStack: any;
  conventions: ConventionSnapshot;

  // ── Dependency context ──
  imports: Map<string, string[]>;      // file → imported modules
  importedBy: Map<string, string[]>;   // file → files that import it

  // ── Blast radius ──
  blastRadius: {
    directlyAffected: string[];
    transitivelyAffected: string[];
    affectedTests: string[];
    crossRepoCallers: string[];
  };

  // ── Metadata ──
  totalTokenEstimate: number;
  gatherTimeMs: number;
  toolCallCount: number;
}

export function assembleContext(/* ... */): ContextBundle {
  // Token budget allocation:
  // Target files:     up to 12,000 tokens (full files with line numbers)
  // Reference files:  up to 4,000 tokens (convention samples)
  // Graph context:    up to 1,500 tokens (callers/callees/signatures)
  // Ticket + PRs:     up to 1,500 tokens
  // Framework prompt: up to 500 tokens
  // Blast radius:     up to 500 tokens
  // ─────────────────────────────────
  // Total context:    ~20,000 tokens (fits in 32K with room for generation)

  const bundle = buildBundle(/* ... */);

  // Fit to budget — trim reference files first, never target files
  return fitToBudget(bundle, 20_000);
}
```

-----

## 4. Pass 2 — Code Generation (Prompt Engineering)

### 4.1 Prompt Structure — Layered Architecture

The generation prompt follows a strict layered structure. Each layer serves a specific purpose and the model is told exactly what each section contains.

```
┌─────────────────────────────────────────────┐
│  SYSTEM PROMPT (role + constraints + rules)  │
├─────────────────────────────────────────────┤
│  FRAMEWORK CONTEXT (Spring Boot / React)     │
├─────────────────────────────────────────────┤
│  CONVENTIONS (from team's actual code)       │
├─────────────────────────────────────────────┤
│  TICKET REQUIREMENT (what to implement)      │
├─────────────────────────────────────────────┤
│  TARGET FILES (full, with line numbers)      │
├─────────────────────────────────────────────┤
│  REFERENCE FILES (similar patterns)          │
├─────────────────────────────────────────────┤
│  GRAPH CONTEXT (callers, callees, types)     │
├─────────────────────────────────────────────┤
│  SIMILAR PRs (how team solved this before)   │
├─────────────────────────────────────────────┤
│  BLAST RADIUS (what breaks if you're wrong)  │
├─────────────────────────────────────────────┤
│  OUTPUT FORMAT (strict JSON schema)          │
└─────────────────────────────────────────────┘
```

### 4.2 System Prompt — Role + Constraints

```typescript
// src/guide/prompts/system_prompt.ts

export function buildSystemPrompt(framework: Framework): string {
  return `You are a senior software engineer implementing changes in a ${frameworkLabel(framework)} codebase.

CRITICAL RULES — violating any of these makes your output unusable:

1. LINE NUMBERS MUST BE EXACT. You are given numbered source files. When specifying startLine/endLine, use the EXACT line numbers from the numbered file. Do not guess or approximate.

2. currentCode MUST BE VERBATIM. Copy the exact text from the numbered file between startLine and endLine. Do not paraphrase, reformat, or summarize. Byte-for-byte match.

3. IMPORTS MUST BE COMPLETE. If your proposedCode uses any symbol not already imported in the file, add the import statement as a separate change at the top of the file.

4. SCOPE MUST BE MINIMAL. Only change what is necessary for the ticket requirement. Do not refactor adjacent code, rename variables for style, or "improve" unrelated logic.

5. ONE CHANGE PER LOGICAL UNIT. Each change in the changes array modifies one continuous block of lines. Do not combine unrelated modifications into a single change.

6. PRESERVE EXISTING PATTERNS. Match the naming conventions, indentation, comment style, and architectural patterns visible in the CONVENTIONS section below.

7. NEW FILES follow existing directory structure. Place new files where similar files already exist.

8. RESPOND ONLY WITH THE JSON SCHEMA specified in OUTPUT FORMAT. No explanation, no markdown, no preamble.`;
}
```

### 4.3 Framework-Specific Prompt Layers

```typescript
// src/guide/prompts/framework_context.ts

export function buildFrameworkContext(framework: Framework): string {
  const contexts: Record<Framework, string> = {
    spring_boot: `<framework>
This is a Spring Boot microservice.
Architecture: Controller → Service → Repository pattern.
DI: Constructor injection (not field injection).
Transactions: @Transactional on service methods that write.
Validation: @Valid on controller request bodies.
Error handling: @ControllerAdvice with domain-specific exceptions.
Testing: @SpringBootTest for integration, @MockBean for unit.
Config: application.yml with Spring profiles.
</framework>`,

    react: `<framework>
This is a React micro-frontend using Module Federation.
Architecture: Component → Hook → Service/API pattern.
State: React hooks (useState, useReducer). No class components.
API calls: Custom hooks wrapping fetch/axios with error handling.
Routing: React Router v6+ with lazy loading.
Testing: Vitest or Jest with @testing-library/react.
Styles: CSS Modules or Tailwind (check convention files).
Error boundaries: Required at MFE root.
</framework>`,

    generic_typescript: `<framework>
TypeScript project. Follow existing patterns visible in convention files.
</framework>`,

    generic_java: `<framework>
Java project. Follow existing patterns visible in convention files.
</framework>`,

    unknown: `<framework>
Follow existing patterns visible in convention files.
</framework>`,
  };

  return contexts[framework] || contexts.unknown;
}
```

### 4.4 Convention Injection

```typescript
// src/guide/prompts/convention_prompt.ts

export function buildConventionSection(conventions: ConventionSnapshot): string {
  if (conventions.files.length === 0) {
    return '<conventions>\nNo convention reference files available. Follow standard patterns.\n</conventions>';
  }

  const sections = ['<conventions>',
    'These files show your team\'s actual coding patterns. Match them exactly.',
    ''
  ];

  for (const file of conventions.files) {
    sections.push(
      `--- ${file.path} (${file.relationship}) ---`,
      file.content,
      ''
    );
  }

  if (conventions.detectedPatterns.length > 0) {
    sections.push('Detected patterns:');
    for (const pattern of conventions.detectedPatterns) {
      sections.push(`- ${pattern}`);
    }
  }

  sections.push('</conventions>');
  return sections.join('\n');
}
```

### 4.5 Target File Prompt — Line-Numbered

```typescript
// src/guide/prompts/file_prompt.ts

export function buildTargetFileSection(files: NumberedFile[]): string {
  const sections = ['<target_files>',
    'These are the files you will modify. Line numbers are on the left.',
    'When specifying changes, use EXACT line numbers and EXACT content from these files.',
    ''
  ];

  for (const file of files) {
    sections.push(
      `=== ${file.path} (${file.lineCount} lines, hash: ${file.contentHash}) ===`,
      file.numberedContent,
      ''
    );
  }

  sections.push('</target_files>');
  return sections.join('\n');
}
```

### 4.6 Graph Context Prompt

```typescript
// src/guide/prompts/graph_prompt.ts

export function buildGraphContextSection(
  graphContext: ContextBundle['graphContext']
): string {
  if (graphContext.size === 0) return '';

  const sections = ['<graph_context>',
    'Symbol relationships from the knowledge graph. These are compiler-verified facts.',
    ''
  ];

  for (const [symbol, ctx] of graphContext) {
    const lines = [`${symbol}:`];

    if (ctx.signature) {
      lines.push(`  Signature: ${ctx.signature}`);
    }
    if (ctx.decorators) {
      lines.push(`  Decorators: ${ctx.decorators}`);
    }
    if (ctx.callers.length > 0) {
      lines.push(`  Called by: ${ctx.callers.join(', ')}`);
    }
    if (ctx.callees.length > 0) {
      lines.push(`  Calls: ${ctx.callees.join(', ')}`);
    }
    lines.push(`  Lines: ${ctx.startLine}-${ctx.endLine}`);
    lines.push(`  Centrality: ${ctx.centralityScore.toFixed(4)}`);

    sections.push(lines.join('\n'));
  }

  sections.push('</graph_context>');
  return sections.join('\n');
}
```

### 4.7 Blast Radius Awareness Prompt

```typescript
export function buildBlastRadiusSection(
  blastRadius: ContextBundle['blastRadius']
): string {
  const sections = ['<blast_radius>',
    'These files and services will be affected by your changes.',
    'Be aware of downstream impact. Do not break these callers.',
    ''
  ];

  if (blastRadius.directlyAffected.length > 0) {
    sections.push(`Direct callers: ${blastRadius.directlyAffected.join(', ')}`);
  }
  if (blastRadius.crossRepoCallers.length > 0) {
    sections.push(`Cross-repo callers (CRITICAL): ${blastRadius.crossRepoCallers.join(', ')}`);
  }
  if (blastRadius.affectedTests.length > 0) {
    sections.push(`Tests to update: ${blastRadius.affectedTests.join(', ')}`);
  }

  sections.push('</blast_radius>');
  return sections.join('\n');
}
```

### 4.8 Output Format Specification

```typescript
// src/guide/prompts/output_format.ts

export function buildOutputFormatSection(): string {
  return `<output_format>
Respond with a single JSON object. No markdown fences. No explanation.

{
  "summary": "One-paragraph description of all changes",
  "changes": [
    {
      "id": "change-001",
      "file": "exact/path/from/target_files.ts",
      "action": "modify",
      "description": "What this change does and why",
      "reason": "Which ticket requirement this satisfies",
      "currentCode": {
        "startLine": 42,
        "endLine": 48,
        "content": "EXACT verbatim text from the numbered file between these lines"
      },
      "proposedCode": "The replacement code with proper indentation",
      "affectedSymbols": ["functionName", "ClassName"]
    }
  ],
  "newFiles": [
    {
      "path": "src/new/file.ts",
      "content": "Complete file content",
      "reason": "Why this new file is needed"
    }
  ],
  "importChanges": [
    {
      "file": "exact/path.ts",
      "action": "add",
      "importStatement": "import { Thing } from './thing'",
      "insertAfterLine": 3
    }
  ],
  "testsToUpdate": ["path/to/test.spec.ts"],
  "executionOrder": ["change-001", "change-002"]
}

VALIDATION RULES:
- file must be an exact path from <target_files>
- startLine and endLine must be exact numbers from the numbered lines
- content in currentCode must be byte-for-byte match of those lines
- proposedCode must maintain the same indentation level
- importChanges must list every new import needed by proposedCode
- executionOrder must list change IDs in safe application order
</output_format>`;
}
```

-----

## 5. Pass 3 — Verification & Self-Correction

### 5.1 Verification Pipeline

After the deep model generates CodeChange[], run a multi-layer verification before execution.

```typescript
// src/guide/verify/verification_pipeline.ts

export async function verifyGeneratedChanges(
  changes: GeneratedOutput,
  context: ContextBundle,
  ctx: AgentContext
): Promise<VerifiedOutput> {
  const results: VerificationResult[] = [];
  let corrected = deepClone(changes);

  // ── Layer 1: File existence check ──
  for (const change of corrected.changes) {
    const exists = await fileExists(change.file, ctx.currentRepo);
    if (!exists && change.action !== 'insert') {
      results.push({
        changeId: change.id,
        check: 'file_exists',
        passed: false,
        error: `File ${change.file} does not exist on disk`,
        severity: 'fatal',
      });
    }
  }

  // ── Layer 2: Line content verification ──
  for (const change of corrected.changes) {
    if (!change.currentCode) continue;

    const file = context.targetFiles.find(f => f.path === change.file);
    if (!file) continue;

    const actualLines = file.content.split('\n')
      .slice(change.currentCode.startLine - 1, change.currentCode.endLine)
      .join('\n');

    const expectedContent = change.currentCode.content.trim();
    const actualContent = actualLines.trim();

    if (actualContent !== expectedContent) {
      // Try fuzzy match — content might be at different lines
      const fuzzyResult = fuzzyFindInFile(file.content, expectedContent, 0.85);

      if (fuzzyResult) {
        // Auto-correct line numbers
        change.currentCode.startLine = fuzzyResult.startLine;
        change.currentCode.endLine = fuzzyResult.endLine;
        change.currentCode.content = fuzzyResult.matchedContent;

        results.push({
          changeId: change.id,
          check: 'line_content',
          passed: true,
          warning: `Line numbers corrected: ${change.currentCode.startLine}-${change.currentCode.endLine} → ${fuzzyResult.startLine}-${fuzzyResult.endLine}`,
          severity: 'corrected',
        });
      } else {
        results.push({
          changeId: change.id,
          check: 'line_content',
          passed: false,
          error: `Content at lines ${change.currentCode.startLine}-${change.currentCode.endLine} does not match`,
          severity: 'fatal',
        });
      }
    }
  }

  // ── Layer 3: Symbol existence (graph + SCIP) ──
  for (const change of corrected.changes) {
    for (const symbol of change.affectedSymbols) {
      const found = await ctx.rag.findSymbol(symbol);
      if (!found || found.length === 0) {
        results.push({
          changeId: change.id,
          check: 'symbol_exists',
          passed: false,
          error: `Symbol '${symbol}' not found in knowledge graph`,
          severity: 'warning', // Not fatal — could be a new symbol
        });
      }
    }
  }

  // ── Layer 4: Import completeness ──
  const importIssues = await verifyImports(corrected, context, ctx);
  results.push(...importIssues);

  // ── Layer 5: Content hash generation ──
  for (const change of corrected.changes) {
    if (change.currentCode) {
      change.currentCode.contentHash = md5(change.currentCode.content.trim());
    }
  }

  // ── Layer 6: Scope check — no out-of-scope modifications ──
  const scopeIssues = checkScope(corrected, context);
  results.push(...scopeIssues);

  // ── Compute confidence ──
  const fatalCount = results.filter(r => r.severity === 'fatal').length;
  const warningCount = results.filter(r => r.severity === 'warning').length;
  const correctedCount = results.filter(r => r.severity === 'corrected').length;

  const confidence = Math.max(0, 1.0
    - (fatalCount * 0.2)
    - (warningCount * 0.05)
    - (correctedCount * 0.02)
  );

  return {
    changes: corrected,
    verificationResults: results,
    confidence: {
      score: confidence,
      level: confidence >= 0.85 ? 'high' : confidence >= 0.6 ? 'medium' : 'low',
      fatalIssues: fatalCount,
      warnings: warningCount,
      autoCorrected: correctedCount,
    },
  };
}
```

### 5.2 Import Resolution Verifier

```typescript
// src/guide/verify/import_verifier.ts

export async function verifyImports(
  output: GeneratedOutput,
  context: ContextBundle,
  ctx: AgentContext
): Promise<VerificationResult[]> {
  const results: VerificationResult[] = [];

  for (const change of output.changes) {
    if (!change.proposedCode) continue;

    // Extract symbols used in proposedCode
    const usedSymbols = extractUsedSymbols(change.proposedCode, context.framework);

    // Get existing imports in the file
    const file = context.targetFiles.find(f => f.path === change.file);
    if (!file) continue;
    const existingImports = parseImports(file.content, context.framework);

    // Check each used symbol
    for (const symbol of usedSymbols) {
      // Skip built-in types
      if (isBuiltinType(symbol, context.framework)) continue;

      // Skip symbols defined in same file
      if (isDefinedInFile(symbol, file.content)) continue;

      // Check if import exists
      const imported = existingImports.some(imp =>
        imp.symbols.includes(symbol) || imp.default === symbol
      );

      // Check if import is in the importChanges array
      const addedImport = output.importChanges?.some(ic =>
        ic.file === change.file && ic.importStatement.includes(symbol)
      );

      if (!imported && !addedImport) {
        // Try to find where this symbol comes from
        const symbolInfo = await ctx.rag.findSymbol(symbol);

        if (symbolInfo && symbolInfo.length > 0) {
          const sourcePath = symbolInfo[0].file_path;
          const relativePath = computeRelativePath(change.file, sourcePath);

          // Auto-add missing import
          const importStatement = buildImportStatement(
            symbol, relativePath, context.framework
          );

          if (!output.importChanges) output.importChanges = [];
          output.importChanges.push({
            file: change.file,
            action: 'add',
            importStatement,
            insertAfterLine: findLastImportLine(file.content),
          });

          results.push({
            changeId: change.id,
            check: 'import_completeness',
            passed: true,
            warning: `Auto-added missing import: ${importStatement}`,
            severity: 'corrected',
          });
        } else {
          results.push({
            changeId: change.id,
            check: 'import_completeness',
            passed: false,
            error: `Symbol '${symbol}' used but not imported and not found in knowledge graph`,
            severity: 'warning',
          });
        }
      }
    }
  }

  return results;
}
```

### 5.3 Scope Boundary Checker

```typescript
// src/guide/verify/scope_checker.ts

export function checkScope(
  output: GeneratedOutput,
  context: ContextBundle
): VerificationResult[] {
  const results: VerificationResult[] = [];

  // Files being modified must be in the blast radius or target files
  const allowedFiles = new Set([
    ...context.targetFiles.map(f => f.path),
    ...context.blastRadius.directlyAffected,
    ...context.blastRadius.affectedTests,
  ]);

  for (const change of output.changes) {
    if (!allowedFiles.has(change.file)) {
      results.push({
        changeId: change.id,
        check: 'scope_boundary',
        passed: false,
        error: `File ${change.file} is outside blast radius — possible scope creep`,
        severity: 'warning',
      });
    }
  }

  // Check for excessive changes (more than expected for ticket complexity)
  if (output.changes.length > 12) {
    results.push({
      changeId: 'global',
      check: 'scope_size',
      passed: false,
      error: `${output.changes.length} changes is unusually high — review for scope creep`,
      severity: 'warning',
    });
  }

  return results;
}
```

-----

## 6. Tool Call Optimization Patterns

### 6.1 Windsurf 20-Tool Budget Awareness

The pipeline runs server-side (not via Windsurf), so the 20-tool budget does not apply directly. However, minimizing RAG MCP calls improves latency. Target: 8-12 RAG calls per pipeline run.

### 6.2 Batch vs Sequential Tool Calls

```
PARALLEL (Wave 1 — 4 calls, ~2s):
  search_semantic_code ──┐
  get_project_info ──────┤──→ all resolve together
  get_ticket_context ────┤
  search_similar_prs ────┘

PARALLEL (Wave 2 — depends on Wave 1, ~3s):
  read_full_file ×3 ─────┐
  get_code_context ×2 ───┤──→ all resolve together
  analyze_blast_radius ──┤
  get_file_dependencies ─┘

SEQUENTIAL (Wave 3 — depends on Wave 2, ~1s):
  Convention sampling (needs framework from Wave 1)

Total: 10-12 calls in ~6 seconds (3 waves)
vs. sequential: 10-12 calls in ~15-20 seconds
```

### 6.3 Smart Tool Selection Based on Ticket Type

```typescript
// src/guide/gather/smart_gather.ts

export function buildSmartGatherPlan(
  intent: GuideMode,
  ticketType: string,
  complexity: string,
  extractedElements: ExtractedElements
): GatherPlan {
  const plan = baseGatherPlan();

  // Bug fix: focus on error location, recent changes
  if (ticketType === 'Bug') {
    plan.addTool('get_recent_changes', { repo: extractedElements.repo, limit: 5 });
    plan.addTool('get_file_history', { files: extractedElements.files });
    plan.blastRadius.maxDepth = 1; // Narrow blast for bug fixes
  }

  // Feature: need broader context, more convention samples
  if (ticketType === 'Story') {
    plan.conventionSample.maxFiles = 5; // More convention context
    plan.similarPRs.maxResults = 3;     // More historical reference
    plan.blastRadius.maxDepth = 2;      // Wider impact analysis
  }

  // Small complexity: fewer reference files, faster
  if (complexity === 'small') {
    plan.semanticSearch.topK = 5;       // Fewer search results
    plan.fileReads.maxFiles = 3;        // Fewer full file reads
    plan.conventionSample.maxFiles = 2;
  }

  // Large complexity: more context, more reference
  if (complexity === 'large') {
    plan.semanticSearch.topK = 15;
    plan.fileReads.maxFiles = 8;
    plan.conventionSample.maxFiles = 5;
    plan.addTool('search_architecture', { repo: extractedElements.repo });
  }

  // If specific symbols mentioned: add targeted lookups
  for (const symbol of extractedElements.symbols.slice(0, 5)) {
    plan.symbolLookups.symbols.push(symbol);
  }

  // If specific files mentioned: read them directly
  for (const file of extractedElements.files.slice(0, 5)) {
    plan.fileReads.files.push(file);
  }

  return plan;
}
```

-----

## 7. Prompt Templates — Full Specifications

### 7.1 Convention Detection Prompt (Fast Model)

```typescript
export function buildConventionDetectionPrompt(
  conventionFiles: ConventionFile[],
  framework: Framework
): string {
  return `Analyze these ${frameworkLabel(framework)} source files and extract the team's coding conventions.

${conventionFiles.map(f => `--- ${f.path} ---\n${f.content}`).join('\n\n')}

Extract ONLY conventions visible in the code above. Do not invent patterns.
Respond as a JSON array of strings. Each string is one convention.

Example output:
["Uses constructor injection, never field injection",
 "Service methods are annotated with @Transactional(readOnly=true) for reads",
 "Error messages use MessageFormat, not string concatenation",
 "Test classes use @ExtendWith(MockitoExtension.class)"]`;
}
```

### 7.2 Full IMPLEMENT Prompt Assembly

```typescript
// src/guide/prompts/implement_prompt.ts

export function assembleImplementPrompt(
  query: string,
  context: ContextBundle
): string {
  const sections = [
    buildSystemPrompt(context.framework),
    buildFrameworkContext(context.framework),
    buildConventionSection(context.conventions),

    // Ticket requirement
    `<requirement>
${query}

${context.ticketDescription ? `Description: ${context.ticketDescription}` : ''}
${context.acceptanceCriteria ? `Acceptance Criteria: ${context.acceptanceCriteria}` : ''}
</requirement>`,

    // Files with line numbers
    buildTargetFileSection(context.targetFiles),

    // Reference files
    context.referenceFiles.length > 0
      ? buildReferenceFileSection(context.referenceFiles)
      : '',

    // Graph context
    buildGraphContextSection(context.graphContext),

    // Similar PRs
    context.similarPRs.length > 0
      ? buildSimilarPRSection(context.similarPRs)
      : '',

    // Blast radius
    buildBlastRadiusSection(context.blastRadius),

    // Output format (always last — closest to generation)
    buildOutputFormatSection(),
  ];

  return sections.filter(Boolean).join('\n\n');
}
```

### 7.3 Similar PR Prompt Section

```typescript
export function buildSimilarPRSection(prs: ContextBundle['similarPRs']): string {
  const sections = ['<similar_prs>',
    'The team has solved similar problems before. Follow their approach if applicable.',
    ''
  ];

  for (const pr of prs) {
    sections.push(
      `PR: ${pr.title}`,
      `Problem: ${pr.coreProblem}`,
      `Solution: ${pr.implementationSummary}`,
      `Decisions: ${pr.keyDecisions}`,
      ''
    );
  }

  sections.push('</similar_prs>');
  return sections.join('\n');
}
```

-----

## 8. Convention Detection & Enforcement

### 8.1 Two-Stage Convention System

```
Stage 1: DETECT (Fast Model, ~1 second)
  Input: 3-5 sibling/similar files
  Output: JSON array of convention strings
  Cached: Per repository, refreshed daily

Stage 2: ENFORCE (Post-generation verification)
  Input: Generated code + detected conventions
  Output: Convention violations (if any)
  Action: Re-generate with violation feedback (1 retry)
```

### 8.2 Convention Cache

```typescript
// src/guide/analyze/convention_cache.ts

const CONVENTION_CACHE = new Map<string, {
  conventions: string[];
  detectedAt: number;
}>();

const CACHE_TTL_MS = 24 * 60 * 60 * 1000; // 24 hours

export async function getConventions(
  repoSlug: string,
  directory: string,
  framework: Framework,
  ctx: AgentContext
): Promise<string[]> {
  const cacheKey = `${repoSlug}:${directory}`;
  const cached = CONVENTION_CACHE.get(cacheKey);

  if (cached && Date.now() - cached.detectedAt < CACHE_TTL_MS) {
    return cached.conventions;
  }

  // Gather convention samples
  const samples = await gatherConventionSamples(
    { filePatterns: [directory + '/*'], maxFiles: 3, maxLinesPerFile: 200 },
    framework, [], ctx
  );

  // Detect via fast model
  const prompt = buildConventionDetectionPrompt(samples.files, framework);
  const response = await ctx.llm.chat(prompt, routeToModel('implement', 'convention_detect'));
  const conventions = parseJSONArray(response.content);

  CONVENTION_CACHE.set(cacheKey, {
    conventions,
    detectedAt: Date.now(),
  });

  return conventions;
}
```

### 8.3 Convention Violation Check (Post-Generation)

```typescript
export async function checkConventionViolations(
  output: GeneratedOutput,
  conventions: string[],
  ctx: AgentContext
): Promise<{ violations: string[]; needsRegeneration: boolean }> {
  if (conventions.length === 0) return { violations: [], needsRegeneration: false };

  const prompt = `Given these team conventions:
${conventions.map((c, i) => `${i + 1}. ${c}`).join('\n')}

Does this generated code follow all conventions? List ONLY actual violations.
If no violations, respond with an empty JSON array [].

Generated code:
${output.changes.map(c => `--- ${c.file} ---\n${c.proposedCode}`).join('\n\n')}

Respond with a JSON array of violation strings, or [] if none.`;

  const response = await ctx.llm.chat(prompt, routeToModel('implement', 'convention_detect'));
  const violations = parseJSONArray(response.content);

  return {
    violations,
    needsRegeneration: violations.length > 0,
  };
}
```

-----

## 9. Multi-File Coherence

When changes span multiple files, the model must maintain consistency (shared types, matching API contracts, import chains).

### 9.1 Coherence Strategy

```typescript
// src/guide/verify/coherence_checker.ts

export async function checkMultiFileCoherence(
  output: GeneratedOutput,
  context: ContextBundle
): Promise<CoherenceResult> {
  const issues: string[] = [];

  // 1. Type consistency: if a type/interface changes in file A,
  //    all usages in other changed files must match
  const typeChanges = extractTypeChanges(output);
  for (const [typeName, newDefinition] of typeChanges) {
    for (const change of output.changes) {
      if (change.proposedCode?.includes(typeName)) {
        const usageConsistent = validateTypeUsage(
          change.proposedCode, typeName, newDefinition
        );
        if (!usageConsistent) {
          issues.push(
            `Type '${typeName}' changed but usage in ${change.file} is inconsistent`
          );
        }
      }
    }
  }

  // 2. API contract: if a function signature changes,
  //    all callers must pass the new parameters
  const signatureChanges = extractSignatureChanges(output);
  for (const [funcName, newSignature] of signatureChanges) {
    const callers = context.graphContext.get(funcName)?.callers || [];
    for (const caller of callers) {
      const callerChange = output.changes.find(c =>
        c.affectedSymbols.includes(caller)
      );
      if (!callerChange && context.blastRadius.directlyAffected.includes(caller)) {
        issues.push(
          `Signature of '${funcName}' changed but caller '${caller}' is not updated`
        );
      }
    }
  }

  // 3. Import chain: if file A adds an import from file B,
  //    file B must actually export that symbol
  for (const importChange of output.importChanges || []) {
    const importedSymbol = extractImportedSymbol(importChange.importStatement);
    const sourceFile = resolveImportPath(
      importChange.file, importChange.importStatement
    );

    if (sourceFile) {
      const sourceContent = context.targetFiles.find(f => f.path === sourceFile);
      if (sourceContent) {
        const exports = extractExports(sourceContent.content);
        if (!exports.includes(importedSymbol)) {
          issues.push(
            `Import '${importedSymbol}' from '${sourceFile}' but symbol is not exported`
          );
        }
      }
    }
  }

  return {
    coherent: issues.length === 0,
    issues,
  };
}
```

-----

## 10. Hallucination Prevention — Deep

### 10.1 Four-Layer Verification

```
Layer 1: Filesystem check
  Every change.file must exist at REPOS_DIR/{repo}/{file}
  Every newFile.path directory must be a real directory pattern

Layer 2: Content hash match
  change.currentCode.content must match actual file content
  MD5 hash computed and stored for execution-time re-verification

Layer 3: Symbol graph check
  change.affectedSymbols verified against IGraphClient.findSymbol()
  Signatures verified against GraphNode.signature

Layer 4: SCIP verification (when available)
  Symbol resolution: does this symbol resolve to a real definition?
  Call chain: does this call chain actually exist?
  Type hierarchy: does this type implement the claimed interface?
  Content freshness: is the SCIP index current with HEAD?
```

### 10.2 SCIP Integration for Verification

```typescript
// src/guide/verify/scip_verifier.ts

export async function verifySCIPSymbols(
  changes: CodeChange[],
  repoSlug: string,
  ctx: AgentContext
): Promise<SCIPVerificationResult[]> {
  const results: SCIPVerificationResult[] = [];

  for (const change of changes) {
    for (const symbol of change.affectedSymbols) {
      // Query SCIP index for symbol resolution
      const scipResult = await ctx.rag.queryKnowledgeGraph({
        type: 'symbol_resolution',
        symbol,
        repo: repoSlug,
        file: change.file,
      });

      if (scipResult.resolved) {
        // Verify signature matches what the model expects
        if (scipResult.signature && change._expectedSignature) {
          const sigMatch = compareSignatures(
            scipResult.signature, change._expectedSignature
          );
          if (!sigMatch.compatible) {
            results.push({
              symbol,
              check: 'signature_mismatch',
              expected: change._expectedSignature,
              actual: scipResult.signature,
              severity: 'error',
            });
          }
        }

        results.push({
          symbol,
          check: 'symbol_resolved',
          resolved: true,
          definitionFile: scipResult.definitionFile,
          severity: 'ok',
        });
      } else {
        results.push({
          symbol,
          check: 'symbol_unresolved',
          resolved: false,
          severity: change.action === 'insert' ? 'info' : 'warning',
        });
      }
    }
  }

  return results;
}
```

-----

## 11. Performance Budget & Optimization

### 11.1 Latency Budget

```
Target: Plan generation in <45 seconds (end-to-end)

Pass 1 — Gather: 6 seconds
  Wave 1 (4 parallel RAG calls):      2.0s
  Wave 2 (6 parallel RAG + file I/O): 3.0s
  Wave 3 (convention sampling):        1.0s

Pass 2 — Generate: 25 seconds
  Prompt assembly:                     0.1s
  Qwen3.5-122B inference:
    Input: ~18K tokens @ ~1000 tok/s:  18.0s (prompt processing)
    Output: ~2K tokens @ 60 tok/s:      3.3s (generation)
  JSON parse:                          0.1s

Pass 3 — Verify: 8 seconds
  File existence checks:               0.1s
  Line content verification:           0.5s
  Import resolution:                   2.0s (includes findSymbol calls)
  Symbol verification:                 2.0s
  Scope check:                         0.1s
  Convention violation check:          3.0s (fast model call)
  Content hash generation:             0.1s

Buffer:                                6 seconds

Total:                                ~45 seconds
```

### 11.2 Performance Optimization Techniques

```typescript
// 1. Speculative prompt processing
// Start building the prompt while Wave 2 results are still arriving
// Send partial prompt to model for prompt cache pre-fill

// 2. Convention cache (24h TTL per repo/directory)
// Avoids re-detecting conventions for every pipeline run
// Refresh only when files in that directory change

// 3. Graph context pre-fetch
// If ticket mentions specific symbols, start graph queries in Wave 1
// Don't wait for semantic search results

// 4. File content cache (keyed by HEAD commit hash)
// Cache numbered file contents — invalidate on git pull
// Avoids re-reading same files across multiple pipeline runs

// 5. Parallel verification in Pass 3
// All verification checks are independent — run in parallel
const verificationResults = await Promise.allSettled([
  verifyFileExistence(changes, ctx),
  verifyLineContent(changes, context),
  verifyImports(changes, context, ctx),
  verifySymbols(changes, ctx),
  checkScope(changes, context),
  checkConventionViolations(changes, conventions, ctx),
]);

// 6. Model warm-up
// Keep Qwen3.5-122B loaded and warm (3 parallel slots)
// Avoid cold-start latency on first request
```

### 11.3 Token Budget Optimization

```typescript
// src/guide/prompts/token_budget.ts

export const TOKEN_BUDGET = {
  systemPrompt: 400,       // Fixed
  frameworkContext: 200,    // Fixed per framework
  conventions: 1500,       // Variable, cap at 1500
  requirement: 500,        // From ticket
  targetFiles: 12000,      // Main budget — full files with line numbers
  referenceFiles: 3000,    // Convention samples
  graphContext: 1000,       // Callers/callees/signatures
  similarPRs: 800,         // Historical reference
  blastRadius: 300,        // Impact summary
  outputFormat: 800,       // Fixed JSON schema spec
  // ────────────────────
  total: 20500,            // Leaves ~11K for generation in 32K context
};

export function fitToBudget(context: ContextBundle, maxTokens: number): ContextBundle {
  let estimate = estimateTokens(context);

  // Trim in priority order: reference files → similar PRs → graph → conventions
  while (estimate > maxTokens) {
    if (context.referenceFiles.length > 2) {
      context.referenceFiles.pop();
    } else if (context.similarPRs.length > 1) {
      context.similarPRs.pop();
    } else if (context.graphContext.size > 5) {
      // Remove lowest centrality entries
      const sorted = [...context.graphContext.entries()]
        .sort((a, b) => a[1].centralityScore - b[1].centralityScore);
      context.graphContext.delete(sorted[0][0]);
    } else {
      // Last resort: truncate largest target file (head+tail)
      const largest = context.targetFiles
        .filter(f => !f.truncated)
        .sort((a, b) => b.lineCount - a.lineCount)[0];
      if (largest) {
        largest.numberedContent = extractHeadTail(largest, 300, 100);
        largest.truncated = true;
      } else {
        break;
      }
    }

    estimate = estimateTokens(context);
  }

  return context;
}
```

-----

## 12. SCIP Query-Time Verification Integration

When SCIP query-time verification is operational (Stage 14.5 in the 16-stage pipeline), integrate its verifiers into Pass 3.

### 12.1 Four SCIP Verifiers

```typescript
// src/guide/verify/scip_gate.ts

export interface SCIPGateResult {
  decision: 'proceed' | 'proceed_with_caution' | 'request_confirmation' | 'abort';
  verifiers: {
    symbolResolution: { passed: boolean; unresolved: string[] };
    callChain: { passed: boolean; brokenLinks: string[] };
    typeHierarchy: { passed: boolean; mismatches: string[] };
    contentFreshness: { passed: boolean; staleFiles: string[] };
  };
  overallConfidence: number;
}

export async function runSCIPGate(
  changes: CodeChange[],
  repoSlug: string,
  ctx: AgentContext
): Promise<SCIPGateResult> {
  const [symbolRes, callChain, typeHier, freshness] = await Promise.allSettled([
    verifySCIPSymbolResolution(changes, repoSlug, ctx),
    verifySCIPCallChains(changes, repoSlug, ctx),
    verifySCIPTypeHierarchy(changes, repoSlug, ctx),
    verifySCIPContentFreshness(changes, repoSlug, ctx),
  ]);

  const results = {
    symbolResolution: extractResult(symbolRes),
    callChain: extractResult(callChain),
    typeHierarchy: extractResult(typeHier),
    contentFreshness: extractResult(freshness),
  };

  // Confidence gate decision
  const failedCount = Object.values(results).filter(r => !r.passed).length;

  let decision: SCIPGateResult['decision'];
  if (failedCount === 0) {
    decision = 'proceed';
  } else if (failedCount === 1 && results.contentFreshness.passed) {
    decision = 'proceed_with_caution';
  } else if (!results.contentFreshness.passed) {
    decision = 'abort'; // Stale SCIP index — cannot verify anything
  } else {
    decision = 'request_confirmation';
  }

  return {
    decision,
    verifiers: results,
    overallConfidence: 1.0 - (failedCount * 0.25),
  };
}
```

-----

## 13. Accuracy Metrics & Evaluation

### 13.1 Golden Test Suite

Maintain a suite of golden Jira tickets with known-correct implementations. Run after any prompt or pipeline change.

```typescript
// test/golden/implement_golden.test.ts

const GOLDEN_TESTS: GoldenTest[] = [
  {
    name: 'Add pagination to bond search',
    ticketKey: 'BMT-GOLDEN-001',
    ticketSummary: 'Add pagination support to /api/bonds/search endpoint',
    repo: 'ms-bonds-trading',
    expectedChanges: [
      { file: 'src/main/java/com/bmt/bonds/controller/BondSearchController.java',
        mustContain: ['@RequestParam', 'Pageable', 'Page<'],
        mustNotContain: ['List<'] },
      { file: 'src/main/java/com/bmt/bonds/service/BondSearchService.java',
        mustContain: ['Pageable', 'Page<'] },
      { file: 'src/main/java/com/bmt/bonds/repository/BondRepository.java',
        mustContain: ['Pageable'] },
    ],
    expectedNewFiles: [],
    maxChanges: 5,
    mustCompile: true,
  },
  {
    name: 'Add error boundary to equity MFE',
    ticketKey: 'BMT-GOLDEN-002',
    ticketSummary: 'Wrap equity dashboard in error boundary component',
    repo: 'mfe-equity-dashboard',
    expectedChanges: [
      { file: 'src/App.tsx',
        mustContain: ['ErrorBoundary'],
        mustNotContain: ['class ErrorBoundary'] },
    ],
    expectedNewFiles: [
      { path: 'src/components/ErrorBoundary.tsx',
        mustContain: ['componentDidCatch', 'getDerivedStateFromError'] },
    ],
    maxChanges: 3,
  },
];
```

### 13.2 Accuracy Metrics to Track

```typescript
export interface PipelineAccuracyMetrics {
  // Per-run metrics
  lineNumberAccuracy: number;      // % of changes with correct line numbers
  contentHashMatch: number;        // % of changes where currentCode matched
  importCompleteness: number;      // % of runs with no missing imports
  compilationSuccess: number;      // % of runs where code compiled
  testPassRate: number;            // % of runs where tests passed first try
  conventionCompliance: number;    // % of changes matching team conventions
  scopeAccuracy: number;           // % of changes within blast radius
  symbolExistence: number;         // % of referenced symbols found in graph

  // Aggregate metrics (daily/weekly)
  planApprovalRate: number;        // % of plans approved by humans
  firstAttemptSuccess: number;     // % of pipelines succeeding without auto-fix
  autoFixSuccess: number;          // % of auto-fix attempts that worked
  prMergeRate: number;             // % of created PRs that got merged
  avgConfidenceScore: number;      // Average confidence across all runs
  avgPlanGenerationTime: number;   // Seconds
  avgExecutionTime: number;        // Seconds
}
```

### 13.3 Continuous Accuracy Monitoring

```typescript
// src/pipeline/accuracy_monitor.ts

export async function recordAccuracyMetrics(
  runId: string,
  verificationResults: VerificationResult[],
  executionResult: ApplyResult,
  testResult: TestResult
): Promise<void> {
  const metrics: PipelineAccuracyMetrics = {
    lineNumberAccuracy: calcRate(verificationResults, 'line_content', 'passed'),
    contentHashMatch: calcRate(verificationResults, 'content_hash', 'passed'),
    importCompleteness: calcRate(verificationResults, 'import_completeness', 'passed'),
    compilationSuccess: executionResult.compilePassed ? 1.0 : 0.0,
    testPassRate: testResult.passed ? 1.0 : 0.0,
    conventionCompliance: calcRate(verificationResults, 'convention', 'passed'),
    scopeAccuracy: calcRate(verificationResults, 'scope_boundary', 'passed'),
    symbolExistence: calcRate(verificationResults, 'symbol_exists', 'passed'),
  };

  await agentDb.query(`
    INSERT INTO ome_agent.accuracy_metrics
    (pipeline_run_id, metrics, created_at)
    VALUES ($1, $2, NOW())
  `, [runId, JSON.stringify(metrics)]);

  // Alert on degradation
  const weeklyAvg = await getWeeklyAverageMetrics();
  if (metrics.compilationSuccess < weeklyAvg.compilationSuccess - 0.1) {
    await alertAccuracyDegradation('compilationSuccess', metrics, weeklyAvg);
  }
}
```

-----

## 14. Prompt Tuning Feedback Loop

### 14.1 Learning from Rejections

When a plan is rejected (Checkpoint 1) or a PR gets change requests (Checkpoint 2), feed the feedback back into prompt improvement.

```typescript
// src/pipeline/feedback_loop.ts

export async function recordRejectionFeedback(
  runId: string,
  rejectionNotes: string,
  reviewer: string
): Promise<void> {
  // Store feedback
  await agentDb.query(`
    INSERT INTO ome_agent.prompt_feedback
    (pipeline_run_id, feedback_type, feedback_text, reviewer, created_at)
    VALUES ($1, 'plan_rejected', $2, $3, NOW())
  `, [runId, rejectionNotes, reviewer]);

  // Classify rejection reason via fast model
  const prompt = `Classify this plan rejection reason into one category:
- line_number_error (wrong line numbers in changes)
- missing_import (generated code missing imports)
- wrong_pattern (used wrong framework pattern)
- scope_creep (changed too many files or unrelated code)
- convention_violation (didn't match team style)
- incomplete (missed a file that needed changing)
- incorrect_logic (wrong business logic)
- other

Rejection note: "${rejectionNotes}"

Respond with one category name only.`;

  const category = (await ctx.llm.chat(prompt,
    routeToModel('implement', 'classify_feedback')
  )).content.trim();

  await agentDb.query(`
    UPDATE ome_agent.prompt_feedback
    SET category = $1
    WHERE pipeline_run_id = $2
  `, [category, runId]);
}
```

### 14.2 Weekly Prompt Refinement Report

```typescript
// src/jobs/weekly_prompt_report.ts

cron.schedule('0 9 * * 1', async () => { // Monday 9 AM
  const weeklyFeedback = await agentDb.query(`
    SELECT category, COUNT(*) AS cnt, 
           ARRAY_AGG(feedback_text ORDER BY created_at DESC) AS samples
    FROM ome_agent.prompt_feedback
    WHERE created_at > NOW() - INTERVAL '7 days'
    GROUP BY category
    ORDER BY cnt DESC
  `);

  const report = {
    totalRejections: weeklyFeedback.rows.reduce((s, r) => s + parseInt(r.cnt), 0),
    topIssues: weeklyFeedback.rows.map(r => ({
      category: r.category,
      count: parseInt(r.cnt),
      examples: r.samples.slice(0, 3),
    })),
    recommendations: generatePromptRecommendations(weeklyFeedback.rows),
  };

  await sendSlackReport(report);
});
```

-----

## 15. Implementation Schedule

```
Week 1: Pass 1 — Context Gathering (Days 1-4)

  Day 1: Gather Plan + Smart Tool Selection
    ├── GatherPlan interface
    ├── buildSmartGatherPlan() — ticket-type-aware tool selection
    ├── Three-wave parallel executor
    └── Test: 3 ticket types → correct tool selection

  Day 2: Full File Reading + Line Numbers
    ├── readFileWithLineNumbers()
    ├── batchReadFilesWithLineNumbers()
    ├── Head+tail truncation for oversized reference files
    ├── Content hash computation at read time
    └── Test: read 10 files, verify line numbering

  Day 3: Convention Sampling + Cache
    ├── gatherConventionSamples()
    ├── Convention detection prompt (fast model)
    ├── Convention cache (24h TTL per repo/directory)
    └── Test: detect conventions in 3 repos, compare with manual review

  Day 4: Context Assembly + Token Budgeting
    ├── assembleContext() — bundle all gathered data
    ├── fitToBudget() — trim by priority
    ├── Token estimation function
    └── Test: assemble context for 5 tickets, verify fits in 20K tokens

Week 2: Pass 2 — Code Generation Prompts (Days 5-8)

  Day 5: System Prompt + Framework Context
    ├── buildSystemPrompt() — rules and constraints
    ├── buildFrameworkContext() — Spring Boot / React specifics
    ├── buildOutputFormatSection() — strict JSON schema
    └── Test: model produces valid JSON on 5 test prompts

  Day 6: Full Prompt Assembly
    ├── assembleImplementPrompt() — all sections combined
    ├── buildTargetFileSection() — numbered files
    ├── buildGraphContextSection() — callers/callees/signatures
    ├── buildSimilarPRSection() — historical reference
    ├── buildBlastRadiusSection() — impact awareness
    └── Test: 3 golden tickets → prompt → model → valid output

  Day 7: Convention Injection + Enforcement
    ├── buildConventionSection() — team patterns in prompt
    ├── checkConventionViolations() — post-generation check
    ├── Convention violation → single retry with feedback
    └── Test: detect violations in deliberately wrong code

  Day 8: Import Change Handling
    ├── importChanges in output schema
    ├── Import resolution in prompt instructions
    ├── Import ordering conventions per framework
    └── Test: generate code needing new imports → verify completeness

Week 3: Pass 3 — Verification (Days 9-12)

  Day 9: Core Verification Pipeline
    ├── verifyGeneratedChanges() — main pipeline
    ├── File existence checks
    ├── Line content verification with fuzzy matching
    ├── Auto-correction for shifted line numbers
    └── Test: inject wrong line numbers → verify auto-correction

  Day 10: Import + Symbol Verification
    ├── verifyImports() — extract used symbols, check imports
    ├── Auto-add missing imports via findSymbol()
    ├── Symbol existence via graph + SCIP
    ├── Signature compatibility check
    └── Test: generate code with missing imports → verify auto-fix

  Day 11: Coherence + Scope Checking
    ├── checkMultiFileCoherence() — type/API/import consistency
    ├── checkScope() — blast radius boundary check
    ├── Confidence score computation
    ├── SCIP gate integration (if available)
    └── Test: multi-file change → verify cross-file consistency

  Day 12: End-to-End Integration
    ├── Wire three passes together
    ├── Performance profiling (target <45s)
    ├── Token budget validation on real tickets
    └── Test: 5 golden tickets → full pipeline → verify accuracy

Week 4: Evaluation + Tuning (Days 13-15)

  Day 13: Golden Test Suite
    ├── 10+ golden test cases (MFE + MS)
    ├── Automated accuracy metric collection
    ├── Compilation verification in test worktrees
    └── Baseline accuracy measurement

  Day 14: Feedback Loop
    ├── Rejection feedback recording
    ├── Category classification (fast model)
    ├── Weekly prompt refinement report
    ├── Accuracy degradation alerts
    └── Test: simulate rejection → verify feedback flow

  Day 15: Performance Optimization
    ├── Parallel verification in Pass 3
    ├── Convention cache warm-up
    ├── File content cache (keyed by HEAD hash)
    ├── Speculative prompt assembly
    ├── Benchmark: 10 tickets, measure p50/p95 latency
    └── Document: accuracy results, known limitations, tuning guide
```

-----

## Summary — How Each Layer Ensures Accuracy

|Accuracy Risk                |Prevention Layer                                          |Mechanism                                            |
|-----------------------------|----------------------------------------------------------|-----------------------------------------------------|
|Wrong line numbers           |Pass 1: full files with line numbers                      |Model sees exact numbered source                     |
|Wrong line numbers (still)   |Pass 3: fuzzy matching                                    |Auto-corrects shifted lines, aborts if no match      |
|Missing imports              |Pass 3: import verifier                                   |Extracts used symbols, auto-adds via findSymbol      |
|Wrong API signatures         |Pass 1: graph context with signatures                     |Model sees compiler-verified signatures              |
|Wrong API signatures (still) |Pass 3: SCIP verification                                 |Compiler-level signature check                       |
|Convention violations        |Pass 1: convention samples + Pass 2: conventions in prompt|Model sees team’s actual code patterns               |
|Convention violations (still)|Pass 3: convention violation check                        |Fast model detects violations, triggers re-generation|
|Hallucinated symbols         |Pass 3: symbol existence check                            |Graph + SCIP verification                            |
|Hallucinated files           |Pass 3: filesystem check                                  |Every file verified on disk                          |
|Scope creep                  |Pass 2: scope constraint in prompt + Pass 3: scope check  |Blast radius boundary enforcement                    |
|Stale code changes           |Execution: content hash gate                              |MD5 mismatch → abort → re-plan                       |
|Multi-file inconsistency     |Pass 3: coherence checker                                 |Cross-file type/API/import validation                |
|Missing test updates         |Pass 1: blast radius → affected tests                     |Test discovery covers transitive impact              |
|Wrong framework patterns     |Pass 2: framework-specific system prompt                  |Spring Boot vs React rules enforced                  |

# OME Agent — Agentic Retrieval Loop: Complete Implementation Plan

> **Scope:** How every RAG MCP tool feeds the loop, file confirmation gates, cross-repo traversal, and tool-aware LLM routing
> **Builds On:** Agentic Retrieval Loop plan, Plan-Execute-Verify plan, Agent Plan
> **RAG Tools:** 43+ MCP tools across search, graph, SDLC, project, and diagnostic categories
> **Effort:** ~8 days

-----

## Table of Contents

1. [RAG MCP Tool Taxonomy](#1-rag-mcp-tool-taxonomy)
1. [Tool-to-Loop Mapping — When and Why Each Tool Is Called](#2-tool-to-loop-mapping)
1. [Response Data Handling — What to Extract from Each Tool](#3-response-data-handling)
1. [File Confirmation Gate — Iterate Before Committing](#4-file-confirmation-gate)
1. [Cross-Repo Retrieval](#5-cross-repo-retrieval)
1. [Tool-Aware LLM Routing](#6-tool-aware-llm-routing)
1. [Retrieval Phases — The Complete Flow](#7-retrieval-phases--the-complete-flow)
1. [Tool Dispatcher](#8-tool-dispatcher)
1. [Response Processors — Per-Tool Data Extraction](#9-response-processors)
1. [File Confirmation Engine](#10-file-confirmation-engine)
1. [Cross-Repo Resolver](#11-cross-repo-resolver)
1. [LLM Router for Analysis](#12-llm-router-for-analysis)
1. [Updated Loop Controller](#13-updated-loop-controller)
1. [Updated Context Accumulator](#14-updated-context-accumulator)
1. [Observability](#15-observability)
1. [Configuration](#16-configuration)
1. [Implementation Schedule](#17-implementation-schedule)

-----

## 1. RAG MCP Tool Taxonomy

All 43+ RAG MCP tools organized by purpose. The agentic loop doesn’t call all of them — it calls the right ones at the right time based on reasoning.

### Category A: Search Tools (Discovery)

|Tool                  |Purpose                                                   |Response Shape                                                         |Loop Role                                                                          |
|----------------------|----------------------------------------------------------|-----------------------------------------------------------------------|-----------------------------------------------------------------------------------|
|`search_semantic_code`|Dense + BM25 hybrid search for code                       |`{ results: [{ filePath, content, score, symbolName, graphContext }] }`|**Primary discovery** — always the seed. Finds candidate files and symbols.        |
|`search_exact_match`  |Exact identifier match via BM25                           |`{ results: [{ filePath, content, score }] }`                          |**Targeted followup** — when reasoning model knows exact symbol name.              |
|`search_architecture` |Search for architecture patterns (decorators, annotations)|`{ results: [{ filePath, pattern, decorators }] }`                     |**Pattern discovery** — finds `@Controller`, `@Service`, `@KafkaListener` patterns.|
|`search_confluence`   |Search Confluence documentation                           |`{ results: [{ pageTitle, breadcrumb, content, spaceKey }] }`          |**Documentation context** — when code alone isn’t enough (e.g., business rules).   |

### Category B: Symbol & Graph Tools (Understanding)

|Tool                         |Purpose                                  |Response Shape                                                                                |Loop Role                                                                                          |
|-----------------------------|-----------------------------------------|----------------------------------------------------------------------------------------------|---------------------------------------------------------------------------------------------------|
|`find_symbol`                |Locate symbol by name across repos       |`{ results: [{ name, type, filePath, repoSlug, startLine, endLine, signature, decorators }] }`|**Symbol resolution** — resolves a name to exact file + line. Cross-repo by default.               |
|`get_code_context`           |Callers, callees, centrality for a symbol|`{ callers, callees, centralityScore, centralityLabel, decorators, imports }`                 |**Structural understanding** — how a symbol connects to the codebase.                              |
|`analyze_blast_radius`       |Impact analysis via reverse call chain   |`{ directDependents, transitiveDependents, affectedTests, crossRepoConsumers }`               |**Impact assessment** — only for IMPLEMENT/DIAGNOSE when changes affect shared code.               |
|`query_knowledge_graph`      |Raw graph traversal with filters         |`{ nodes, edges }`                                                                            |**Deep exploration** — when specific graph queries are needed (e.g., “all classes implementing X”).|
|`get_file_dependencies`      |Imports and dependents for a file        |`{ imports: [{ module, symbols }], importedBy: [{ filePath }] }`                              |**Dependency mapping** — understand what a file needs and who needs it.                            |
|`suggest_related_files`      |co_change-based file suggestions         |`{ relatedFiles: [{ filePath, coChangeCount, relationship }] }`                               |**Temporal coupling** — files that always change together (architectural coupling).                |
|`get_cross_repo_dependencies`|Cross-repo callers/callees               |`{ consumers: [{ repoSlug, filePath, symbol }], providers: [...] }`                           |**Cross-repo awareness** — who in OTHER repos depends on this symbol.                              |

### Category C: File Tools (Content)

|Tool              |Purpose                          |Response Shape                                  |Loop Role                                                             |
|------------------|---------------------------------|------------------------------------------------|----------------------------------------------------------------------|
|`read_full_file`  |Read complete file from REPOS_DIR|`{ filePath, content, lines, language }`        |**Full content** — the actual code. Called after confirmation gate.   |
|`read_file_range` |Read specific line range         |`{ filePath, content, startLine, endLine }`     |**Targeted read** — when only a specific method/section is needed.    |
|`get_file_history`|Git log for a file               |`{ commits: [{ hash, author, date, message }] }`|**Change history** — who changed this and when. Critical for DIAGNOSE.|

### Category D: SDLC Tools (PRs, Jira, Tickets)

|Tool                 |Purpose                                     |Response Shape                                                                             |Loop Role                                                          |
|---------------------|--------------------------------------------|-------------------------------------------------------------------------------------------|-------------------------------------------------------------------|
|`get_ticket_context` |Enriched Jira ticket with condensed fields  |`{ ticketId, summary, businessGoal, specificTask, acceptanceCriteria, parentKey, epicKey }`|**Requirements** — what the task actually asks for.                |
|`search_pr_summaries`|Search PR condensed summaries (FTS + vector)|`{ results: [{ prNumber, title, coreProblem, implementationSummary, keyDecisions }] }`     |**Prior art** — how the team solved similar problems before.       |
|`get_pr_details`     |Full PR details (files changed, reviewers)  |`{ title, description, filesChanged, reviewComments, author }`                             |**Implementation reference** — exact changes in a past PR.         |
|`get_recent_changes` |Recent commits to a file/repo               |`{ commits: [{ hash, author, date, message, filesChanged }] }`                             |**Recent context** — what changed recently (critical for DIAGNOSE).|

### Category E: Project Tools (Metadata)

|Tool              |Purpose                               |Response Shape                                                 |Loop Role                                                                    |
|------------------|--------------------------------------|---------------------------------------------------------------|-----------------------------------------------------------------------------|
|`get_project_info`|Tech stack, project type, dependencies|`{ repoSlug, techStack, projectType, dependencies, framework }`|**Framework context** — needed for convention detection and pattern matching.|
|`get_repo_config` |application.yml, Dockerfile, compose  |`{ applicationYml, dockerfile, dockerCompose }`                |**Runtime config** — needed when changes involve configuration.              |
|`lookup_glossary` |Acronym/term expansion                |`{ expansions: [{ acronym, meaning }] }`                       |**Domain knowledge** — BMT-specific acronyms the model wouldn’t know.        |

### Category F: Diagnostic Tools

|Tool                 |Purpose                  |Response Shape                              |Loop Role                                                        |
|---------------------|-------------------------|--------------------------------------------|-----------------------------------------------------------------|
|`diagnose_search`    |Search health diagnostics|`{ coverage, camelCaseHits, sourceBalance }`|**System health** — not used in guide loop (internal diagnostic).|
|`check_hallucination`|Verify file/symbol exists|`{ exists, filePath, verified }`            |**Verification** — used in verify phase, not retrieval.          |

### Category G: Health Score Tools

|Tool                  |Purpose                       |Response Shape                             |Loop Role                                                                |
|----------------------|------------------------------|-------------------------------------------|-------------------------------------------------------------------------|
|`get_health_score`    |Architecture health for a repo|`{ overall, rating, dimensions, topRisks }`|**Quality context** — flags known problems in files being modified.      |
|`get_health_dimension`|Specific dimension details    |`{ score, findings }`                      |**Targeted warnings** — e.g., security warnings when modifying auth code.|

-----

## 2. Tool-to-Loop Mapping

### Which Tools Are Available at Which Loop Phase

```
SEED PHASE (always, no LLM reasoning needed):
  ├── search_semantic_code          ← PRIMARY: discover candidates
  ├── find_symbol (extracted)       ← TARGETED: resolve known names from query
  ├── get_ticket_context            ← IF ticket_id provided
  ├── get_project_info              ← IF current_repo known
  ├── lookup_glossary               ← IF domain terms detected
  └── read_full_file (current_file) ← IF developer has a file open

REASONING ITERATIONS (LLM chooses):
  ├── DISCOVERY actions:
  │   ├── search_semantic_code      ← Refined/rephrased search
  │   ├── search_exact_match        ← When exact identifier known
  │   ├── search_architecture       ← When looking for patterns (@Controller, etc.)
  │   └── search_confluence         ← When business rules/docs needed
  │
  ├── UNDERSTANDING actions:
  │   ├── find_symbol               ← Resolve type/class name to location
  │   ├── get_code_context          ← Callers + callees + centrality
  │   ├── get_file_dependencies     ← What does this file import/export
  │   ├── suggest_related_files     ← co_change temporal coupling
  │   ├── query_knowledge_graph     ← Custom graph traversal
  │   └── get_cross_repo_deps       ← Who in other repos uses this
  │
  ├── CONTENT actions:
  │   ├── read_full_file            ← After confirmation gate passes
  │   ├── read_file_range           ← When only specific section needed
  │   └── get_file_history          ← Recent changes to a file
  │
  ├── SDLC actions:
  │   ├── search_pr_summaries       ← Find similar past PRs
  │   ├── get_pr_details            ← Deep dive into a specific PR
  │   └── get_recent_changes        ← Recent commits
  │
  ├── CONFIG actions:
  │   ├── get_repo_config           ← When config context needed
  │   └── get_health_score          ← When quality context relevant
  │
  └── IMPACT actions:
      ├── analyze_blast_radius      ← Only when modifying shared code
      └── get_cross_repo_deps       ← Only when symbol is cross-repo

NEVER in retrieval loop (used elsewhere):
  ├── diagnose_search               ← Internal diagnostic
  ├── check_hallucination           ← Verify phase only
  ├── create_pull_request           ← Phase 2 execution only
  ├── generate_pr_description       ← Phase 2 execution only
  └── review_pull_request           ← Phase 2 execution only
```

### Mode-Specific Tool Relevance

|Tool                 |UNDERSTAND                  |IMPLEMENT                      |DIAGNOSE                   |
|---------------------|----------------------------|-------------------------------|---------------------------|
|search_semantic_code |★★★ Always                  |★★★ Always                     |★★★ Always                 |
|search_exact_match   |★★ If symbol known          |★★ If symbol known             |★★★ Error symbols          |
|search_architecture  |★★ Pattern exploration      |★★★ Convention reference       |★ Rarely                   |
|search_confluence    |★★ Architecture docs        |★★ Business rules              |★ Rarely                   |
|find_symbol          |★★★ Resolve names           |★★★ Resolve types              |★★★ Error symbols          |
|get_code_context     |★★★ Core understanding      |★★★ Dependency awareness       |★★★ Call chain to error    |
|analyze_blast_radius |★ Rarely                    |★★★ Impact of changes          |★ Rarely                   |
|get_cross_repo_deps  |★★ If cross-repo question   |★★★ If modifying shared code   |★★ If error crosses repos  |
|read_full_file       |★★★ The code being explained|★★★ The code being modified    |★★★ Error location         |
|get_file_history     |★ Rarely                    |★ Rarely                       |★★★ What changed recently  |
|get_ticket_context   |★ If asked about ticket     |★★★ Requirements               |★ If ticket-related bug    |
|search_pr_summaries  |★ Prior discussions         |★★★ How team solved this before|★★ Past fix patterns       |
|get_project_info     |★★ Framework context        |★★★ Convention baseline        |★★ Framework context       |
|get_repo_config      |★ If config question        |★★ If config changes needed    |★★★ If config-related error|
|get_health_score     |★ Rarely                    |★★ Known issues in target files|★★ Known issues            |
|lookup_glossary      |★★ Domain terms             |★★ Domain terms                |★ Rarely                   |
|suggest_related_files|★★ Architecture mapping     |★★★ Files that change together |★★ Coupled files           |
|get_file_dependencies|★★ Import chain             |★★★ What to import             |★★ Dependency chain        |
|query_knowledge_graph|★★ Custom exploration       |★★ Interface implementations   |★ Rarely                   |

-----

## 3. Response Data Handling

Each tool’s response needs specific processing — what to extract, what to store, and what signals to derive for the reasoning model.

```typescript
// src/guide/retrieval/response_processors.ts

export interface ProcessedResponse {
  /** Files discovered (candidates for reading) */
  discoveredFiles: DiscoveredFile[];
  /** Symbols resolved */
  resolvedSymbols: ResolvedSymbol[];
  /** Graph edges discovered */
  graphEdges: GraphEdge[];
  /** Cross-repo references found */
  crossRepoRefs: CrossRepoRef[];
  /** Signals for the reasoning model */
  signals: RetrievalSignal[];
  /** Raw data to store in accumulator */
  rawData: any;
  /** Estimated tokens this adds to context */
  tokenEstimate: number;
}

export type RetrievalSignal =
  | { type: 'high_centrality_symbol'; symbol: string; score: number }
  | { type: 'cross_repo_dependency'; repo: string; symbol: string }
  | { type: 'shared_type_referenced'; typeName: string; inFiles: string[] }
  | { type: 'config_dependency'; configKey: string; filePath: string }
  | { type: 'recent_change_detected'; filePath: string; daysAgo: number; author: string }
  | { type: 'test_coverage_gap'; filePath: string }
  | { type: 'health_warning'; dimension: string; severity: string; filePath: string }
  | { type: 'temporal_coupling'; fileA: string; fileB: string; coChangeCount: number }
  | { type: 'no_results'; query: string; suggestion: string }
  | { type: 'sparse_ticket'; ticketId: string }
  | { type: 'cross_repo_consumer'; consumerRepo: string; symbol: string }
  | { type: 'interface_implementors'; interfaceName: string; implementors: string[] }
  | { type: 'convention_pattern'; pattern: string; exampleFile: string };
```

### Per-Tool Response Processors

```typescript
// ── search_semantic_code ──
export function processSemanticSearch(results: any[]): ProcessedResponse {
  const discoveredFiles: DiscoveredFile[] = [];
  const resolvedSymbols: ResolvedSymbol[] = [];
  const signals: RetrievalSignal[] = [];
  const crossRepoRefs: CrossRepoRef[] = [];

  // Track which repos appear in results
  const repoSet = new Set<string>();

  for (const r of results) {
    repoSet.add(r.repoSlug);

    discoveredFiles.push({
      filePath: r.filePath,
      repoSlug: r.repoSlug,
      score: r.score,
      matchedSymbol: r.symbolName,
      snippet: r.content?.slice(0, 200),
      source: 'semantic_search',
      // Extract whether this looks like a target or reference
      likelyRole: r.score > 0.7 ? 'target' : 'candidate',
    });

    if (r.symbolName) {
      resolvedSymbols.push({
        name: r.symbolName,
        type: r.symbolType ?? inferSymbolType(r.content),
        filePath: r.filePath,
        repoSlug: r.repoSlug,
        line: r.startLine,
        signature: r.signature,
      });
    }

    // Signal: high centrality symbol found
    if (r.graphContext?.centralityScore > 0.02) {
      signals.push({
        type: 'high_centrality_symbol',
        symbol: r.symbolName,
        score: r.graphContext.centralityScore,
      });
    }

    // Signal: shared type referenced in content
    const typeRefs = extractTypeReferences(r.content ?? '');
    for (const typeName of typeRefs) {
      signals.push({
        type: 'shared_type_referenced',
        typeName,
        inFiles: [r.filePath],
      });
    }
  }

  // Signal: results span multiple repos
  if (repoSet.size > 1) {
    for (const repo of repoSet) {
      crossRepoRefs.push({
        repoSlug: repo,
        reason: 'Search results include this repo',
        filesFound: results.filter(r => r.repoSlug === repo).map(r => r.filePath),
      });
    }
  }

  // Signal: poor results
  if (results.length === 0 || (results.length > 0 && results[0].score < 0.3)) {
    signals.push({
      type: 'no_results',
      query: 'semantic search',
      suggestion: 'Try rephrased query or exact_search with specific identifier',
    });
  }

  return {
    discoveredFiles,
    resolvedSymbols,
    graphEdges: [],
    crossRepoRefs,
    signals,
    rawData: results,
    tokenEstimate: estimateTokens(JSON.stringify(results)),
  };
}


// ── get_code_context ──
export function processCodeContext(data: any, symbol: string): ProcessedResponse {
  const discoveredFiles: DiscoveredFile[] = [];
  const signals: RetrievalSignal[] = [];
  const graphEdges: GraphEdge[] = [];
  const crossRepoRefs: CrossRepoRef[] = [];

  // Callers — each is a potential file to explore
  for (const caller of data.callers ?? []) {
    discoveredFiles.push({
      filePath: caller.filePath,
      repoSlug: caller.repoSlug,
      score: 0.6,
      matchedSymbol: caller.name,
      snippet: `Calls ${symbol}`,
      source: 'graph_caller',
      likelyRole: 'candidate',
    });

    graphEdges.push({
      source: caller.name,
      target: symbol,
      type: 'calls',
      sourceFile: caller.filePath,
      targetFile: data.filePath,
      sourceRepo: caller.repoSlug,
    });

    // Cross-repo caller detection
    if (caller.repoSlug && caller.repoSlug !== data.repoSlug) {
      crossRepoRefs.push({
        repoSlug: caller.repoSlug,
        reason: `${caller.name} in ${caller.repoSlug} calls ${symbol}`,
        filesFound: [caller.filePath],
      });
      signals.push({
        type: 'cross_repo_consumer',
        consumerRepo: caller.repoSlug,
        symbol: caller.name,
      });
    }
  }

  // Callees — what this symbol depends on
  for (const callee of data.callees ?? []) {
    discoveredFiles.push({
      filePath: callee.filePath,
      repoSlug: callee.repoSlug,
      score: 0.5,
      matchedSymbol: callee.name,
      snippet: `Called by ${symbol}`,
      source: 'graph_callee',
      likelyRole: 'candidate',
    });

    graphEdges.push({
      source: symbol,
      target: callee.name,
      type: 'calls',
      sourceFile: data.filePath,
      targetFile: callee.filePath,
    });
  }

  // High centrality signal
  if (data.centralityScore > 0.02) {
    signals.push({
      type: 'high_centrality_symbol',
      symbol,
      score: data.centralityScore,
    });
  }

  return {
    discoveredFiles,
    resolvedSymbols: [],
    graphEdges,
    crossRepoRefs,
    signals,
    rawData: data,
    tokenEstimate: estimateTokens(JSON.stringify(data)),
  };
}


// ── find_symbol ──
export function processFindSymbol(results: any[], symbol: string): ProcessedResponse {
  const discoveredFiles: DiscoveredFile[] = [];
  const resolvedSymbols: ResolvedSymbol[] = [];
  const crossRepoRefs: CrossRepoRef[] = [];
  const signals: RetrievalSignal[] = [];

  const repoSet = new Set<string>();

  for (const r of results) {
    repoSet.add(r.repoSlug);

    discoveredFiles.push({
      filePath: r.filePath,
      repoSlug: r.repoSlug,
      score: 0.8,
      matchedSymbol: r.name,
      snippet: r.signature ?? `${r.type}: ${r.name}`,
      source: 'find_symbol',
      likelyRole: isTypeSymbol(r.type) ? 'type_definition' : 'candidate',
    });

    resolvedSymbols.push({
      name: r.name,
      type: r.type,
      filePath: r.filePath,
      repoSlug: r.repoSlug,
      line: r.startLine,
      signature: r.signature,
      decorators: r.decorators,
    });

    // Detect interface with multiple implementations
    if (r.type === 'interface') {
      signals.push({
        type: 'interface_implementors',
        interfaceName: r.name,
        implementors: [], // Will be filled by a graph query if reasoning model asks
      });
    }
  }

  // Cross-repo: symbol found in multiple repos
  if (repoSet.size > 1) {
    for (const repo of repoSet) {
      crossRepoRefs.push({
        repoSlug: repo,
        reason: `Symbol ${symbol} exists in ${repo}`,
        filesFound: results.filter(r => r.repoSlug === repo).map(r => r.filePath),
      });
    }
  }

  return {
    discoveredFiles,
    resolvedSymbols,
    graphEdges: [],
    crossRepoRefs,
    signals,
    rawData: results,
    tokenEstimate: estimateTokens(JSON.stringify(results)),
  };
}


// ── get_ticket_context ──
export function processTicketContext(data: any): ProcessedResponse {
  const signals: RetrievalSignal[] = [];

  // Detect sparse ticket
  const descLength = (
    (data.businessGoal ?? '').length +
    (data.specificTask ?? '').length +
    (data.acceptanceCriteria ?? '').length
  );

  if (descLength < 80) {
    signals.push({
      type: 'sparse_ticket',
      ticketId: data.ticketId,
    });
  }

  return {
    discoveredFiles: [],
    resolvedSymbols: [],
    graphEdges: [],
    crossRepoRefs: [],
    signals,
    rawData: data,
    tokenEstimate: estimateTokens(JSON.stringify(data)),
  };
}


// ── suggest_related_files ──
export function processRelatedFiles(data: any[], sourceFile: string): ProcessedResponse {
  const discoveredFiles: DiscoveredFile[] = [];
  const signals: RetrievalSignal[] = [];

  for (const r of data) {
    discoveredFiles.push({
      filePath: r.filePath,
      repoSlug: r.repoSlug,
      score: 0.4,
      matchedSymbol: null,
      snippet: `co_change with ${sourceFile} (${r.coChangeCount} times)`,
      source: 'co_change',
      likelyRole: 'candidate',
    });

    if (r.coChangeCount >= 5) {
      signals.push({
        type: 'temporal_coupling',
        fileA: sourceFile,
        fileB: r.filePath,
        coChangeCount: r.coChangeCount,
      });
    }
  }

  return {
    discoveredFiles,
    resolvedSymbols: [],
    graphEdges: [],
    crossRepoRefs: [],
    signals,
    rawData: data,
    tokenEstimate: estimateTokens(JSON.stringify(data)),
  };
}


// ── analyze_blast_radius ──
export function processBlastRadius(data: any, symbol: string): ProcessedResponse {
  const discoveredFiles: DiscoveredFile[] = [];
  const crossRepoRefs: CrossRepoRef[] = [];
  const signals: RetrievalSignal[] = [];

  // Cross-repo consumers are critical
  for (const consumer of data.crossRepoConsumers ?? []) {
    crossRepoRefs.push({
      repoSlug: consumer.repoSlug,
      reason: `${consumer.symbol} in ${consumer.repoSlug} depends on ${symbol}`,
      filesFound: [consumer.filePath],
    });
    signals.push({
      type: 'cross_repo_consumer',
      consumerRepo: consumer.repoSlug,
      symbol: consumer.symbol,
    });
  }

  // Affected tests should be read
  for (const test of data.affectedTests ?? []) {
    discoveredFiles.push({
      filePath: test.filePath,
      repoSlug: test.repoSlug,
      score: 0.3,
      matchedSymbol: null,
      snippet: `Test affected by changes to ${symbol}`,
      source: 'blast_radius_test',
      likelyRole: 'test',
    });
  }

  if (!data.affectedTests?.length) {
    signals.push({
      type: 'test_coverage_gap',
      filePath: data.filePath ?? '',
    });
  }

  return {
    discoveredFiles,
    resolvedSymbols: [],
    graphEdges: [],
    crossRepoRefs,
    signals,
    rawData: data,
    tokenEstimate: estimateTokens(JSON.stringify(data)),
  };
}


// ── get_health_score ──
export function processHealthScore(data: any, repo: string): ProcessedResponse {
  const signals: RetrievalSignal[] = [];

  for (const risk of data.topRisks ?? []) {
    signals.push({
      type: 'health_warning',
      dimension: risk.dimension,
      severity: risk.severity,
      filePath: risk.filePath,
    });
  }

  return {
    discoveredFiles: [],
    resolvedSymbols: [],
    graphEdges: [],
    crossRepoRefs: [],
    signals,
    rawData: data,
    tokenEstimate: estimateTokens(JSON.stringify(data)),
  };
}


// ── get_file_history ──
export function processFileHistory(data: any[], filePath: string): ProcessedResponse {
  const signals: RetrievalSignal[] = [];

  if (data.length > 0) {
    const latestCommit = data[0];
    const daysSinceChange = Math.floor(
      (Date.now() - new Date(latestCommit.date).getTime()) / (1000 * 60 * 60 * 24)
    );

    if (daysSinceChange <= 7) {
      signals.push({
        type: 'recent_change_detected',
        filePath,
        daysAgo: daysSinceChange,
        author: latestCommit.author,
      });
    }
  }

  return {
    discoveredFiles: [],
    resolvedSymbols: [],
    graphEdges: [],
    crossRepoRefs: [],
    signals,
    rawData: data,
    tokenEstimate: estimateTokens(JSON.stringify(data)),
  };
}


// ── search_pr_summaries ──
export function processPRSearch(results: any[]): ProcessedResponse {
  const signals: RetrievalSignal[] = [];

  for (const pr of results) {
    if (pr.implementationSummary) {
      // Extract conventions from PR implementation
      const patterns = extractConventionPatterns(pr.implementationSummary);
      for (const pattern of patterns) {
        signals.push({
          type: 'convention_pattern',
          pattern: pattern.description,
          exampleFile: pattern.filePath ?? `PR-${pr.prNumber}`,
        });
      }
    }
  }

  return {
    discoveredFiles: [],
    resolvedSymbols: [],
    graphEdges: [],
    crossRepoRefs: [],
    signals,
    rawData: results,
    tokenEstimate: estimateTokens(JSON.stringify(results)),
  };
}


// ── get_file_dependencies ──
export function processFileDependencies(data: any, filePath: string): ProcessedResponse {
  const discoveredFiles: DiscoveredFile[] = [];
  const crossRepoRefs: CrossRepoRef[] = [];
  const signals: RetrievalSignal[] = [];

  // Imports — files this file depends on
  for (const imp of data.imports ?? []) {
    if (imp.resolvedPath) {
      discoveredFiles.push({
        filePath: imp.resolvedPath,
        repoSlug: imp.repoSlug,
        score: 0.4,
        matchedSymbol: imp.symbols?.join(', '),
        snippet: `Imported by ${filePath}`,
        source: 'file_dependency',
        likelyRole: 'candidate',
      });

      // Cross-repo import
      if (imp.repoSlug && imp.repoSlug !== data.repoSlug) {
        crossRepoRefs.push({
          repoSlug: imp.repoSlug,
          reason: `${filePath} imports from ${imp.repoSlug}`,
          filesFound: [imp.resolvedPath],
        });
      }
    }
  }

  // Imported by — files that depend on this file
  for (const dep of data.importedBy ?? []) {
    discoveredFiles.push({
      filePath: dep.filePath,
      repoSlug: dep.repoSlug,
      score: 0.3,
      matchedSymbol: null,
      snippet: `Imports from ${filePath}`,
      source: 'reverse_dependency',
      likelyRole: 'candidate',
    });
  }

  // Detect shared type references
  for (const imp of data.imports ?? []) {
    for (const sym of imp.symbols ?? []) {
      if (isLikelyType(sym)) {
        signals.push({
          type: 'shared_type_referenced',
          typeName: sym,
          inFiles: [filePath],
        });
      }
    }
  }

  return {
    discoveredFiles,
    resolvedSymbols: [],
    graphEdges: [],
    crossRepoRefs,
    signals,
    rawData: data,
    tokenEstimate: estimateTokens(JSON.stringify(data)),
  };
}


// ── Helpers ──

function isTypeSymbol(type: string): boolean {
  return ['interface', 'type', 'enum', 'class'].includes(type);
}

function isLikelyType(name: string): boolean {
  return /^[A-Z]/.test(name) &&
    /(DTO|Entity|Type|Interface|Model|Request|Response|Config|Props|State)$/.test(name);
}

function inferSymbolType(content: string): string {
  if (/\b(interface|type)\b/.test(content)) return 'type';
  if (/\b(class)\b/.test(content)) return 'class';
  if (/\b(function|const\s+\w+\s*=\s*\(|async\s+\w+)/.test(content)) return 'function';
  return 'unknown';
}

function extractTypeReferences(content: string): string[] {
  const matches = content.match(
    /\b[A-Z][a-zA-Z0-9]*(?:DTO|Entity|Request|Response|Type|Interface|Model|Config|Props|State)\b/g
  );
  return [...new Set(matches ?? [])];
}

function extractConventionPatterns(text: string): Array<{ description: string; filePath?: string }> {
  const patterns: Array<{ description: string; filePath?: string }> = [];
  // Extract mentions of specific patterns or approaches
  if (/\b(Resilience4j|CircuitBreaker|@Retry)\b/i.test(text)) {
    patterns.push({ description: 'Uses Resilience4j for fault tolerance' });
  }
  if (/\b(Pageable|PageRequest|Page<)\b/i.test(text)) {
    patterns.push({ description: 'Uses Spring Pageable for pagination' });
  }
  // Add more pattern detectors as needed
  return patterns;
}
```

-----

## 4. File Confirmation Gate

### The Problem with Reading Everything

Reading every discovered file in full is wasteful. Semantic search returns 10 results — but only 2-3 are actually relevant target files. Each unnecessary file read costs 500-3,000 tokens in the execution prompt.

### Solution: Confirm Before Reading

After discovery (search, graph), the reasoning model evaluates each candidate file and decides:

- **Read in full** — this is a target or critical reference
- **Read range only** — only a specific method/section is needed
- **Read signature only** — just need to know the interface, not the implementation
- **Skip** — not relevant enough to read

```typescript
// src/guide/retrieval/file_confirmation.ts

export type FileReadDecision =
  | { decision: 'read_full'; reason: string }
  | { decision: 'read_range'; startLine: number; endLine: number; reason: string }
  | { decision: 'read_signature'; reason: string }
  | { decision: 'skip'; reason: string };

/**
 * The confirmation gate evaluates candidate files and decides
 * how much to read. This runs as part of the reasoning iteration.
 */
export function buildFileConfirmationPrompt(
  query: string,
  mode: GuideMode,
  candidates: DiscoveredFile[],
  alreadyRead: Set<string>,
  tokenBudgetRemaining: number
): string {
  // Filter out already-read files
  const unread = candidates.filter(c => !alreadyRead.has(c.filePath));

  if (unread.length === 0) return ''; // Nothing to confirm

  return `## File Read Decisions

You have discovered ${unread.length} candidate files. Decide how much of each to read.
Token budget remaining: ~${tokenBudgetRemaining} tokens

Query: ${query}
Mode: ${mode}

Candidates:
${unread.map((c, i) => `${i + 1}. ${c.filePath} (${c.repoSlug})
   Score: ${c.score.toFixed(2)} | Source: ${c.source}
   Match: ${c.matchedSymbol ?? 'none'} | Snippet: ${c.snippet ?? 'N/A'}
   Likely role: ${c.likelyRole}`).join('\n\n')}

For each file, decide:
- "read_full" — This file IS a target or critical reference. Need full content with line numbers.
- "read_range" — Only need specific lines (e.g., one method). Specify startLine and endLine.
- "read_signature" — Only need imports + class declaration + method signatures (for type compatibility).
- "skip" — Not relevant enough. Don't waste tokens.

Consider:
- For IMPLEMENT mode: target files (being modified) MUST be read_full. Reference files can be read_signature.
- For UNDERSTAND mode: the code being explained should be read_full. Supporting context can be read_signature.
- For DIAGNOSE mode: the error location MUST be read_full. Call chain files can be read_signature.
- Type definition files (DTOs, interfaces) are small — usually read_full is fine.
- Config files (application.yml) — read_full if config-related, skip otherwise.
- Test files — usually skip unless specifically asked about testing.

Respond with JSON:
{
  "decisions": [
    { "filePath": "...", "decision": "read_full", "reason": "..." },
    { "filePath": "...", "decision": "read_signature", "reason": "..." },
    { "filePath": "...", "decision": "skip", "reason": "..." }
  ]
}`;
}

/**
 * Execute file read decisions — parallel reads for confirmed files.
 */
export async function executeFileReads(
  decisions: FileReadDecisionEntry[],
  ctx: AgentContext,
  parser: Parser
): Promise<Map<string, FileReadResult>> {
  const results = new Map<string, FileReadResult>();
  const limit = pLimit(4);

  const readPromises = decisions
    .filter(d => d.decision !== 'skip')
    .map(d => limit(async () => {
      switch (d.decision) {
        case 'read_full': {
          const content = await ctx.fileReader.readFile(d.filePath);
          results.set(d.filePath, {
            filePath: d.filePath,
            content,
            lines: content.split('\n').length,
            readType: 'full',
            role: d.assignedRole ?? 'target',
          });
          break;
        }

        case 'read_range': {
          const content = await ctx.rag.readFileRange(d.filePath, d.startLine!, d.endLine!);
          results.set(d.filePath, {
            filePath: d.filePath,
            content,
            lines: d.endLine! - d.startLine! + 1,
            readType: 'range',
            role: 'reference',
            range: { start: d.startLine!, end: d.endLine! },
          });
          break;
        }

        case 'read_signature': {
          const fullContent = await ctx.fileReader.readFile(d.filePath);
          const sig = extractFileSignature(
            { filePath: d.filePath, content: fullContent, lines: 0, isTarget: false },
            parser
          );
          results.set(d.filePath, {
            filePath: d.filePath,
            content: formatSignatureForPlan(sig),
            lines: sig.totalLines,
            readType: 'signature',
            role: 'reference',
            originalLines: sig.totalLines,
          });
          break;
        }
      }
    }));

  await Promise.allSettled(readPromises);
  return results;
}
```

### How File Confirmation Integrates with the Loop

```
Iteration 1: SEED
  → semantic_search → 10 candidates discovered
  → find_symbol → 3 more candidates

Iteration 2: REASON + CONFIRM
  → Reasoning model sees 13 candidates with snippets and scores
  → Decides: read_full for top 3, read_signature for 2 types, skip 8
  → Execute reads in parallel

Iteration 3: REASON
  → Now has 3 full files + 2 signatures
  → "I see OrderService calls BondValidator — need that too"
  → get_code_context(BondValidator) → discovers BondValidator.java
  → Confirm: read_full(BondValidator.java)
  → "Sufficient — have all targets and types"
```

The confirmation step naturally merges into the reasoning iteration. The reasoning prompt includes candidate files, and the model’s action output includes read decisions. This is NOT a separate phase — it’s part of what the reasoning model decides each iteration.

-----

## 5. Cross-Repo Retrieval

### The Problem

Current retrieval is scoped to `current_repo`. But BMT has 47 repos with cross-repo dependencies:

- Shared libraries used by multiple services
- MFE shells importing remote modules from other MFE repos
- Services calling other services (REST or Kafka)
- Shared type definitions in common repos

### Cross-Repo Discovery Signals

The retrieval loop detects cross-repo needs via signals from tool responses:

```
Signal Source                          → Cross-Repo Action
─────────────────────────────────────────────────────────────
find_symbol returns results in         → Read the symbol definition
  multiple repos                         in each repo

get_code_context shows callers         → Discover the consumer in
  from other repos                       the other repo

analyze_blast_radius returns           → Must understand the consumer's
  crossRepoConsumers                     usage for safe changes

get_file_dependencies shows            → Read the shared dependency
  imports from shared lib repo

search_semantic_code returns           → Results already span repos
  results across repos                   — accumulator tracks per-repo

search_architecture finds              → Pattern exists in multiple
  same pattern in multiple repos         repos — reference for conventions
```

### Cross-Repo Resolver

```typescript
// src/guide/retrieval/cross_repo_resolver.ts

export interface CrossRepoRef {
  repoSlug: string;
  reason: string;
  filesFound: string[];
}

export interface CrossRepoContext {
  /** Primary repo (from user's current_repo or most results) */
  primaryRepo: string;
  /** Other repos involved */
  secondaryRepos: CrossRepoDetail[];
  /** Shared types/interfaces used across repos */
  sharedTypes: SharedTypeRef[];
  /** Cross-repo call edges */
  crossRepoCalls: CrossRepoCall[];
}

export interface CrossRepoDetail {
  repoSlug: string;
  relationship: 'consumer' | 'provider' | 'shared_lib' | 'sibling';
  filesInvolved: string[];
  reason: string;
}

export interface SharedTypeRef {
  typeName: string;
  definedIn: { repoSlug: string; filePath: string };
  usedIn: Array<{ repoSlug: string; filePath: string }>;
}

export interface CrossRepoCall {
  callerRepo: string;
  callerSymbol: string;
  callerFile: string;
  calleeRepo: string;
  calleeSymbol: string;
  calleeFile: string;
}

/**
 * Resolve cross-repo dependencies from accumulated signals.
 * Called after each reasoning iteration when new cross-repo refs are detected.
 */
export async function resolveCrossRepoContext(
  crossRepoRefs: CrossRepoRef[],
  primaryRepo: string,
  accumulator: ContextAccumulator,
  ctx: AgentContext
): Promise<RetrievalAction[]> {
  const actions: RetrievalAction[] = [];
  const processedRepos = new Set<string>();

  for (const ref of crossRepoRefs) {
    if (ref.repoSlug === primaryRepo) continue;
    if (processedRepos.has(ref.repoSlug)) continue;
    processedRepos.add(ref.repoSlug);

    // Get project info for the secondary repo (if not already known)
    if (!accumulator.hasProjectInfoFor(ref.repoSlug)) {
      actions.push({
        action: 'get_project_info',
        repo: ref.repoSlug,
      });
    }

    // Read the specific files found in the cross-repo reference
    for (const filePath of ref.filesFound) {
      if (!accumulator.hasFile(filePath)) {
        // For cross-repo files, default to signature read (less token cost)
        // The reasoning model can upgrade to read_full if needed
        actions.push({
          action: 'read_full_file',
          filePath,
        });
      }
    }

    // Check for shared types between primary and secondary repo
    const sharedTypes = findSharedTypesBetweenRepos(
      primaryRepo, ref.repoSlug, accumulator
    );

    for (const typeName of sharedTypes) {
      if (!accumulator.hasSymbol(typeName)) {
        actions.push({
          action: 'find_symbol',
          symbol: typeName,
        });
      }
    }
  }

  return actions;
}

/**
 * Find types that are referenced in both repos.
 */
function findSharedTypesBetweenRepos(
  repoA: string,
  repoB: string,
  accumulator: ContextAccumulator
): string[] {
  const typesInA = new Set<string>();
  const typesInB = new Set<string>();

  for (const signal of accumulator.signals) {
    if (signal.type === 'shared_type_referenced') {
      for (const filePath of signal.inFiles) {
        const repo = accumulator.getRepoForFile(filePath);
        if (repo === repoA) typesInA.add(signal.typeName);
        if (repo === repoB) typesInB.add(signal.typeName);
      }
    }
  }

  // Types referenced in both repos
  return [...typesInA].filter(t => typesInB.has(t));
}
```

### Cross-Repo in the Reasoning Prompt

```typescript
// Added to reasoning prompt when cross-repo refs exist:

`### Cross-Repo Dependencies Detected
${accumulator.crossRepoContext.secondaryRepos.map(r =>
  `- ${r.repoSlug} (${r.relationship}): ${r.reason}
   Files: ${r.filesInvolved.join(', ')}`
).join('\n')}

${accumulator.crossRepoContext.sharedTypes.length > 0
  ? `### Shared Types Across Repos
${accumulator.crossRepoContext.sharedTypes.map(t =>
  `- ${t.typeName}: defined in ${t.definedIn.repoSlug}, used in ${t.usedIn.map(u => u.repoSlug).join(', ')}`
).join('\n')}`
  : ''}

Consider:
- If modifying a symbol consumed by other repos, you should fetch blast_radius
- Shared type changes affect ALL repos that import them
- Cross-repo files are usually read as signatures unless directly modified`
```

-----

## 6. Tool-Aware LLM Routing

### The Insight

Not all tool responses need LLM analysis. Some are structured data to accumulate. Some need the code-aware fast model. Some need deep reasoning.

```
Tool Response                    → Analysis Needed         → LLM
──────────────────────────────────────────────────────────────────────
search_semantic_code results     → Extract signals          → NONE (programmatic)
find_symbol results              → Extract signals          → NONE (programmatic)
get_code_context callers/callees → Extract signals          → NONE (programmatic)
get_project_info                 → Store metadata           → NONE (programmatic)
lookup_glossary                  → Store expansions         → NONE (programmatic)
get_repo_config                  → Store config             → NONE (programmatic)
get_file_history                 → Extract signals          → NONE (programmatic)

read_full_file content           → Understand code structure → FAST (code-aware)
                                   identify target methods
                                   detect patterns/conventions

get_ticket_context               → Enrich sparse ticket      → FAST (reasoning)
                                   identify technical scope

search_pr_summaries              → Extract relevant patterns → FAST (reasoning)
                                   match to current task

analyze_blast_radius             → Assess impact severity     → FAST (reasoning)
                                   decide if cross-repo
                                   investigation needed

get_health_score                 → Identify relevant warnings → NONE (programmatic)
                                   for target files

File confirmation decisions      → Decide read/skip per file  → FAST (reasoning)

Sufficiency determination        → Should we stop or continue → FAST (reasoning)

Convention detection              → Extract patterns from     → FAST (code-aware)
                                   reference files
```

### LLM Router Implementation

```typescript
// src/guide/retrieval/analysis_router.ts

export type AnalysisLevel = 'none' | 'fast' | 'deep';

export interface AnalysisRouting {
  level: AnalysisLevel;
  reason: string;
  /** For 'fast' — specific analysis task */
  task?: string;
  /** For 'fast' — max tokens for analysis output */
  maxTokens?: number;
  /** Temperature for analysis */
  temperature?: number;
}

/**
 * Determine what level of LLM analysis a tool response needs.
 * Most responses need NO LLM — just programmatic signal extraction.
 * Some need the fast model for code understanding or reasoning.
 * The deep model is NEVER used during retrieval — only during generation.
 */
export function routeAnalysis(
  action: string,
  responseData: any,
  mode: GuideMode,
  context: AnalysisContext
): AnalysisRouting {

  switch (action) {
    // ── NO LLM NEEDED — programmatic signal extraction ──
    case 'search_semantic_code':
    case 'search_exact_match':
    case 'find_symbol':
    case 'get_code_context':
    case 'get_file_dependencies':
    case 'suggest_related_files':
    case 'get_project_info':
    case 'get_repo_config':
    case 'lookup_glossary':
    case 'get_file_history':
    case 'get_recent_changes':
    case 'get_health_score':
    case 'get_health_dimension':
    case 'query_knowledge_graph':
      return { level: 'none', reason: 'Structured data — programmatic extraction sufficient' };

    // ── FAST MODEL — code understanding ──
    case 'read_full_file':
      // Only analyze if this is a target file in IMPLEMENT mode
      if (mode === 'implement' && context.fileRole === 'target') {
        return {
          level: 'fast',
          reason: 'Need to identify modifiable methods, detect patterns, understand structure',
          task: 'analyze_target_file',
          maxTokens: 500,
          temperature: 0.1,
        };
      }
      // Reference files — no analysis, just store
      return { level: 'none', reason: 'Reference file — store for context' };

    // ── FAST MODEL — ticket reasoning ──
    case 'get_ticket_context':
      if (context.isSparseTicket) {
        return {
          level: 'fast',
          reason: 'Sparse ticket — need inference from parent/epic context',
          task: 'sparse_ticket_enrich',
          maxTokens: 300,
          temperature: 0.1,
        };
      }
      return { level: 'none', reason: 'Ticket has sufficient detail' };

    // ── FAST MODEL — PR pattern matching ──
    case 'search_pr_summaries':
      if (mode === 'implement' && responseData.length > 0) {
        return {
          level: 'fast',
          reason: 'Match PR patterns to current task — extract relevant conventions',
          task: 'extract_pr_conventions',
          maxTokens: 300,
          temperature: 0.2,
        };
      }
      return { level: 'none', reason: 'No PRs found or not IMPLEMENT mode' };

    // ── FAST MODEL — blast radius assessment ──
    case 'analyze_blast_radius':
      if (responseData.crossRepoConsumers?.length > 0) {
        return {
          level: 'fast',
          reason: 'Cross-repo consumers found — assess impact and decide if further investigation needed',
          task: 'assess_cross_repo_impact',
          maxTokens: 200,
          temperature: 0.1,
        };
      }
      return { level: 'none', reason: 'No cross-repo impact — programmatic extraction sufficient' };

    // ── FAST MODEL — architecture search ──
    case 'search_architecture':
      if (mode === 'implement') {
        return {
          level: 'fast',
          reason: 'Match architecture patterns to implementation task',
          task: 'match_architecture_pattern',
          maxTokens: 200,
          temperature: 0.2,
        };
      }
      return { level: 'none', reason: 'Store patterns for context' };

    // ── FAST MODEL — Confluence content ──
    case 'search_confluence':
      return {
        level: 'fast',
        reason: 'Extract relevant requirements/specs from documentation',
        task: 'extract_doc_requirements',
        maxTokens: 400,
        temperature: 0.1,
      };

    default:
      return { level: 'none', reason: 'Unknown action — no analysis' };
  }
}


/**
 * Execute tool-specific LLM analysis on a response.
 * Returns extracted insights that augment the accumulator signals.
 */
export async function analyzeToolResponse(
  routing: AnalysisRouting,
  action: string,
  responseData: any,
  query: string,
  ctx: AgentContext
): Promise<ToolAnalysisResult> {
  if (routing.level === 'none') {
    return { insights: [], augmentedSignals: [] };
  }

  const prompt = buildToolAnalysisPrompt(routing.task!, action, responseData, query);

  const llmRouting = routeToModel('understand', routing.task! as ModelTask);
  llmRouting.maxTokens = routing.maxTokens ?? 300;
  llmRouting.temperature = routing.temperature ?? 0.1;

  const response = await ctx.llm.chat(prompt, llmRouting);
  return parseToolAnalysisResponse(response.content, routing.task!);
}

function buildToolAnalysisPrompt(
  task: string,
  action: string,
  data: any,
  query: string
): string {
  switch (task) {
    case 'analyze_target_file':
      return `You are analyzing a file that will be modified for this task: "${query}"

File content:
\`\`\`
${typeof data === 'string' ? data.slice(0, 8000) : JSON.stringify(data).slice(0, 8000)}
\`\`\`

Identify:
1. Which methods/functions will need modification for this task
2. What patterns/conventions does this file follow
3. What types/interfaces are critical for compatibility

Respond with JSON:
{
  "modifiableMethods": ["methodName"],
  "patterns": ["pattern description"],
  "criticalTypes": ["TypeName"],
  "conventionNotes": "brief note on coding style"
}`;

    case 'sparse_ticket_enrich':
      return `This Jira ticket has a sparse description. Infer what it requires.

Ticket: ${JSON.stringify(data)}

Write 2-3 sentences describing the technical requirements. JSON:
{ "inferredDescription": "..." }`;

    case 'extract_pr_conventions':
      return `Extract implementation patterns from these similar PRs relevant to: "${query}"

PRs: ${JSON.stringify(data).slice(0, 3000)}

JSON:
{ "relevantPatterns": ["pattern 1", "pattern 2"], "keyDecisions": ["decision 1"] }`;

    case 'assess_cross_repo_impact':
      return `These repos consume the symbol being modified. Assess if they need investigation.

${JSON.stringify(data).slice(0, 2000)}

JSON:
{ "needsInvestigation": ["repo-slug-1"], "safeToIgnore": ["repo-slug-2"], "reasoning": "..." }`;

    default:
      return `Analyze this data for: "${query}"\n${JSON.stringify(data).slice(0, 3000)}`;
  }
}
```

### LLM Cost Model

```
Per retrieval loop — LLM calls breakdown:

ALWAYS (part of the loop):
  Reasoning calls:  1-5 × fast model (~200 tokens out, ~1-2s each)

SOMETIMES (tool-specific analysis):
  Target file analysis:    0-3 × fast model (~500 tokens out, ~2s each)
  Sparse ticket enrichment: 0-1 × fast model (~300 tokens out, ~1s)
  PR convention extraction: 0-1 × fast model (~300 tokens out, ~1s)
  Cross-repo assessment:    0-1 × fast model (~200 tokens out, ~1s)

NEVER during retrieval:
  Deep model — reserved for generation phase only

Worst case retrieval LLM cost:
  5 reasoning + 3 file analysis + 1 ticket + 1 PR + 1 cross-repo
  = 11 fast model calls × ~1.5s = ~16s
  (Most queries: 2-4 calls × ~1.5s = ~3-6s)
```

-----

## 7. Retrieval Phases — The Complete Flow

```
PHASE 0: SEED (no LLM, parallel, ~500ms)
  │
  ├── search_semantic_code(query)                    → processSemanticSearch()
  ├── find_symbol(extracted symbols from query)      → processFindSymbol()
  ├── get_ticket_context(ticket_id) if present       → processTicketContext()
  ├── get_project_info(current_repo) if known        → (store metadata)
  ├── lookup_glossary(domain terms) if found          → (store expansions)
  └── read_full_file(current_file) if open in IDE    → (store + analyze if IMPLEMENT)
  │
  ▼ Process all responses → extract signals → populate accumulator

PHASE 1: AGENTIC LOOP (fast model reasoning, max 5 iterations)
  │
  │  ┌─────────────────────────────────────────────────────────────┐
  │  │ ITERATION:                                                   │
  │  │                                                              │
  │  │ 1. Build reasoning prompt:                                   │
  │  │    - Accumulated context summary                             │
  │  │    - Discovered candidate files (with snippets)              │
  │  │    - Resolved symbols                                        │
  │  │    - Signals (cross-repo, shared types, health warnings)     │
  │  │    - Token budget remaining                                  │
  │  │    - Cross-repo context (if any)                             │
  │  │    - Available actions                                       │
  │  │    - File confirmation decisions needed                      │
  │  │                                                              │
  │  │ 2. Fast model responds with:                                 │
  │  │    - Reasoning: "Found controller, need the service..."      │
  │  │    - File decisions: { "OrderService.java": "read_full" }    │
  │  │    - Actions: [get_code_context("submitOrder"), ...]         │
  │  │    - OR: sufficient = true                                   │
  │  │                                                              │
  │  │ 3. Execute in parallel:                                      │
  │  │    a. File reads (per decisions)                             │
  │  │    b. Tool calls (per actions)                               │
  │  │                                                              │
  │  │ 4. Process responses:                                        │
  │  │    a. Route each response to analysis level (none/fast)      │
  │  │    b. Run programmatic signal extraction                     │
  │  │    c. Run fast model analysis where needed                   │
  │  │    d. Check for new cross-repo references                    │
  │  │    e. Generate cross-repo follow-up actions if needed        │
  │  │                                                              │
  │  │ 5. Accumulate results                                        │
  │  │                                                              │
  │  │ 6. Check hard limits → continue or exit                      │
  │  └─────────────────────────────────────────────────────────────┘
  │
  ▼ When sufficient or limits reached

PHASE 2: POST-LOOP ENRICHMENT (no LLM or fast model only, ~2-3s)
  │
  ├── Convention detection (fast model on reference files)          ← IF implement mode
  ├── Cross-repo context assembly (programmatic)                   ← IF cross-repo detected
  ├── File role classification (target/reference/type/test)        ← Programmatic
  ├── Health warnings for target files (get_health_score)          ← IF not already fetched
  └── Sparse ticket enrichment (fast model)                        ← IF sparse ticket detected
  │
  ▼ Output: ContextInput for Plan-Execute-Verify pipeline
```

-----

## 8. Tool Dispatcher

Central dispatcher that routes tool calls to RAG MCP and processes responses.

```typescript
// src/guide/retrieval/tool_dispatcher.ts

export class ToolDispatcher {
  constructor(
    private rag: RAGClient,
    private fileReader: FileReader,
    private ragDb: Pool,     // Read-only otak access
    private parser: Parser,
  ) {}

  /**
   * Execute a retrieval action and return processed response.
   * Handles: RAG MCP call → response processing → signal extraction → optional LLM analysis
   */
  async dispatch(
    action: RetrievalAction,
    ctx: AgentContext,
    mode: GuideMode,
    accumulator: ContextAccumulator
  ): Promise<DispatchResult> {
    const startMs = performance.now();

    // 1. Execute the raw tool call
    const rawResult = await this.executeRawAction(action);

    if (!rawResult.success) {
      return {
        action,
        success: false,
        processed: null,
        analysis: null,
        durationMs: Math.round(performance.now() - startMs),
        error: rawResult.error,
      };
    }

    // 2. Process response — extract signals, files, symbols, cross-repo refs
    const processed = this.processResponse(action, rawResult.data);

    // 3. Route to LLM analysis if needed
    const analysisRouting = routeAnalysis(
      action.action,
      rawResult.data,
      mode,
      {
        fileRole: accumulator.getFileRole(
          (action as any).filePath ?? ''
        ),
        isSparseTicket: processed.signals.some(s => s.type === 'sparse_ticket'),
      }
    );

    let analysis: ToolAnalysisResult | null = null;
    if (analysisRouting.level !== 'none') {
      analysis = await analyzeToolResponse(
        analysisRouting, action.action, rawResult.data, ctx.query, ctx
      );
    }

    return {
      action,
      success: true,
      processed,
      analysis,
      durationMs: Math.round(performance.now() - startMs),
    };
  }

  private async executeRawAction(action: RetrievalAction): Promise<RawActionResult> {
    try {
      switch (action.action) {
        case 'semantic_search':
          return { success: true, data: await this.rag.searchCode(action.query, { source: action.source, topK: 10 }) };
        case 'exact_search':
          return { success: true, data: await this.rag.searchExact(action.query) };
        case 'find_symbol':
          return { success: true, data: await this.rag.findSymbol(action.symbol) };
        case 'read_full_file':
          return { success: true, data: await this.fileReader.readFile(action.filePath) };
        case 'get_code_context':
          return { success: true, data: await this.rag.getCodeContext(action.symbol) };
        case 'get_callers':
          return { success: true, data: (await this.rag.getCodeContext(action.symbol))?.callers ?? [] };
        case 'get_callees':
          return { success: true, data: (await this.rag.getCodeContext(action.symbol))?.callees ?? [] };
        case 'get_blast_radius':
          return { success: true, data: await this.rag.analyzeBlastRadius(action.symbol) };
        case 'get_related_files':
          return { success: true, data: await this.rag.suggestRelatedFiles(action.filePath) };
        case 'get_file_dependencies':
          return { success: true, data: await this.rag.getFileDependencies(action.filePath) };
        case 'get_recent_changes':
          return { success: true, data: await this.rag.getRecentChanges(action.filePath) };
        case 'get_file_history':
          return { success: true, data: await this.rag.getFileHistory(action.filePath) };
        case 'get_project_info':
          return { success: true, data: await this.rag.getProjectInfo(action.repo) };
        case 'get_repo_config':
          return { success: true, data: await this.rag.getRepoConfig(action.repo) };
        case 'search_similar_prs':
          return { success: true, data: await this.searchPRs(action.query) };
        case 'search_architecture':
          return { success: true, data: await this.rag.searchArchitecture(action.query) };
        case 'search_confluence':
          return { success: true, data: await this.rag.searchConfluence(action.query) };
        case 'get_health_score':
          return { success: true, data: await this.rag.getHealthScore(action.repo) };
        case 'get_cross_repo_deps':
          return { success: true, data: await this.rag.getCrossRepoDependencies(action.symbol) };
        case 'query_knowledge_graph':
          return { success: true, data: await this.rag.queryKnowledgeGraph(action.query) };
        case 'lookup_glossary':
          return { success: true, data: await this.rag.lookupGlossary(action.terms) };
        case 'sufficient':
          return { success: true, data: null };
        default:
          return { success: false, error: `Unknown action: ${(action as any).action}` };
      }
    } catch (error) {
      return { success: false, error: (error as Error).message };
    }
  }

  private processResponse(action: RetrievalAction, data: any): ProcessedResponse {
    switch (action.action) {
      case 'semantic_search': return processSemanticSearch(data);
      case 'find_symbol': return processFindSymbol(data, action.symbol);
      case 'get_code_context': return processCodeContext(data, action.symbol);
      case 'get_ticket_context': return processTicketContext(data);
      case 'suggest_related_files': return processRelatedFiles(data, action.filePath);
      case 'analyze_blast_radius': return processBlastRadius(data, action.symbol);
      case 'get_health_score': return processHealthScore(data, action.repo);
      case 'get_file_history': return processFileHistory(data, action.filePath);
      case 'search_similar_prs': return processPRSearch(data);
      case 'get_file_dependencies': return processFileDependencies(data, action.filePath);
      default: return {
        discoveredFiles: [], resolvedSymbols: [], graphEdges: [],
        crossRepoRefs: [], signals: [], rawData: data,
        tokenEstimate: estimateTokens(JSON.stringify(data)),
      };
    }
  }

  private async searchPRs(query: string): Promise<any[]> {
    // Hybrid search against vec_pr_summaries + fts_pr_summaries
    // (Uses ragDb for direct access to otak tables)
    const [vecResults, ftsResults] = await Promise.all([
      this.ragDb.query(`
        SELECT vps.pr_id, pr.repo_slug, pr.pr_number, pr.title,
               pr.core_problem, pr.implementation_summary, pr.key_decisions,
               1 - (vps.embedding <=> $1::halfvec) AS score
        FROM otak.vec_pr_summaries vps
        JOIN otak.pull_requests pr ON vps.pr_id = pr.id
        ORDER BY vps.embedding <=> $1::halfvec LIMIT 5
      `, [query]),
      this.ragDb.query(`
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
}
```

-----

## 9. Updated Reasoning Prompt

The reasoning prompt now includes file confirmation, cross-repo context, and signal-based guidance.

```typescript
// src/guide/retrieval/reasoning_prompt.ts — UPDATED

export function buildReasoningPrompt(
  query: string,
  mode: GuideMode,
  accumulator: ContextAccumulator,
  iteration: number,
  maxIterations: number
): string {
  const summary = accumulator.toReasoningSummary();

  const sections: string[] = [];

  sections.push(`You are a retrieval planner for a code assistant. Decide what to fetch next.

## Query
${query}

## Mode: ${mode.toUpperCase()}`);

  // ── What we have so far ──
  sections.push(`## Accumulated Context (Iteration ${iteration}/${maxIterations})`);

  // Files
  if (summary.files.length > 0) {
    sections.push(`### Files
${summary.files.map(f =>
  `- ${f.filePath} (${f.repoSlug}) [${f.readType ?? 'not read'}] role:${f.role} — ${f.relevance}`
).join('\n')}`);
  }

  // Unread candidates (need file confirmation)
  const unreadCandidates = summary.files.filter(f => !f.hasFullContent && f.role !== 'skip');
  if (unreadCandidates.length > 0) {
    sections.push(`### Unread Candidate Files (need your read decision)
${unreadCandidates.map((c, i) =>
  `${i + 1}. ${c.filePath} (${c.repoSlug}) score:${c.score?.toFixed(2)} source:${c.source}
     snippet: ${c.snippet ?? 'N/A'}`
).join('\n')}`);
  }

  // Symbols
  if (summary.symbols.length > 0) {
    sections.push(`### Symbols Found
${summary.symbols.map(s =>
  `- ${s.name} (${s.type}) in ${s.filePath}:${s.line}${s.context ? ' — ' + s.context : ''}`
).join('\n')}`);
  }

  // Graph
  if (summary.graphSummary) {
    sections.push(`### Graph Context\n${summary.graphSummary}`);
  }

  // Cross-repo
  if (summary.crossRepoRefs.length > 0) {
    sections.push(`### Cross-Repo References
${summary.crossRepoRefs.map(r =>
  `- ${r.repoSlug}: ${r.reason} (files: ${r.filesFound.join(', ')})`
).join('\n')}

⚠️ Cross-repo dependencies detected. Consider:
- Use find_symbol to resolve shared types across repos
- Use get_cross_repo_deps for symbols consumed by other repos
- Cross-repo files default to signature read unless directly modified`);
  }

  // Signals
  const activeSignals = summary.signals.filter(s =>
    !['no_results'].includes(s.type) || iteration === 1 // Only show no_results on first iteration
  );
  if (activeSignals.length > 0) {
    sections.push(`### Signals
${activeSignals.map(s => `- [${s.type}] ${formatSignal(s)}`).join('\n')}`);
  }

  // Ticket
  if (summary.ticketSummary) {
    sections.push(`### Ticket\n${summary.ticketSummary}`);
  }

  // PR
  if (summary.prSummary) {
    sections.push(`### Similar PRs\n${summary.prSummary}`);
  }

  // Token budget
  sections.push(`### Token Budget
Accumulated: ~${summary.totalTokens} tokens
Remaining for code generation: ~${summary.remainingBudget} tokens
${summary.remainingBudget < 8000 ? '⚠️ LOW BUDGET — prioritize essential reads only' : ''}`);

  // Available actions
  sections.push(`## Available Actions
${getAvailableActionsForMode(mode, accumulator).join('\n')}`);

  // Sufficiency criteria
  sections.push(`## Sufficiency Criteria
${getSufficiencyCriteria(mode)}`);

  // Output format
  sections.push(`## Respond with JSON

{
  "reasoning": "What's missing and why I chose these actions",

  "fileDecisions": {
    "path/to/file.ts": { "decision": "read_full|read_signature|skip", "reason": "..." }
  },

  "actions": [
    { "action": "action_name", ...params }
  ],

  "sufficient": false,

  "crossRepoActionsNeeded": false
}

Rules:
- Maximum 4 actions per iteration
- ALWAYS include fileDecisions for unread candidates (don't leave them unconfirmed)
- If cross-repo refs exist and you haven't investigated them, set crossRepoActionsNeeded: true
- Don't re-fetch what you already have
- For IMPLEMENT: target files MUST be read_full. Type files read_full (they're small). Reference files read_signature.
- If sufficient, actions and fileDecisions must be empty`);

  return sections.join('\n\n');
}

function formatSignal(signal: RetrievalSignal): string {
  switch (signal.type) {
    case 'high_centrality_symbol': return `${signal.symbol} has high centrality (${signal.score.toFixed(3)}) — core function`;
    case 'cross_repo_dependency': return `Cross-repo dep: ${signal.symbol} in ${signal.repo}`;
    case 'shared_type_referenced': return `Type ${signal.typeName} referenced in ${signal.inFiles.join(', ')} — may need resolution`;
    case 'config_dependency': return `Config key ${signal.configKey} in ${signal.filePath}`;
    case 'recent_change_detected': return `${signal.filePath} changed ${signal.daysAgo} days ago by ${signal.author}`;
    case 'test_coverage_gap': return `No tests found for ${signal.filePath}`;
    case 'health_warning': return `[${signal.severity}] ${signal.dimension} issue in ${signal.filePath}`;
    case 'temporal_coupling': return `${signal.fileA} ↔ ${signal.fileB} change together (${signal.coChangeCount}x)`;
    case 'no_results': return `Search returned poor results — ${signal.suggestion}`;
    case 'sparse_ticket': return `Ticket ${signal.ticketId} has sparse description — needs enrichment`;
    case 'cross_repo_consumer': return `${signal.symbol} in ${signal.consumerRepo} consumes the symbol being modified`;
    case 'interface_implementors': return `${signal.interfaceName} is an interface — may have multiple implementations`;
    case 'convention_pattern': return `Convention: ${signal.pattern} (see ${signal.exampleFile})`;
  }
}
```

-----

## 10. Configuration

```bash
# ═══ Retrieval Loop ═══
RETRIEVAL_MAX_ITERATIONS=5
RETRIEVAL_MAX_TOOL_CALLS=25           # Increased for cross-repo
RETRIEVAL_MAX_ACTIONS_PER_ITERATION=4
RETRIEVAL_MAX_PARALLEL_ACTIONS=4
RETRIEVAL_MIN_REMAINING_BUDGET=5000

# ═══ File Confirmation ═══
FILE_CONFIRM_ENABLED=true             # Gate file reads behind confirmation
FILE_MAX_FULL_READS=6                 # Max files read in full per loop
FILE_MAX_SIGNATURE_READS=8            # Max files read as signatures
FILE_AUTO_READ_THRESHOLD=0.85         # Score above which files auto-read without confirmation

# ═══ Cross-Repo ═══
CROSS_REPO_ENABLED=true               # Follow cross-repo references
CROSS_REPO_MAX_SECONDARY_REPOS=3      # Max additional repos to investigate
CROSS_REPO_DEFAULT_READ=signature     # Default read level for cross-repo files

# ═══ Tool Analysis LLM ═══
TOOL_ANALYSIS_MODEL=fast              # fast (:8090) for all tool analysis
TOOL_ANALYSIS_MAX_TOKENS=500
TOOL_ANALYSIS_TEMPERATURE=0.1

# ═══ Reasoning Model ═══
RETRIEVAL_REASONING_MODEL=fast
RETRIEVAL_REASONING_MAX_TOKENS=400
RETRIEVAL_REASONING_TEMPERATURE=0.1
```

-----

## 11. Implementation Schedule

```
Day 1: Response Processors + Signal System
  ├── response_processors.ts
  │   ├── processSemanticSearch()       — file candidates, symbols, type refs, cross-repo
  │   ├── processCodeContext()          — graph edges, callers/callees, cross-repo callers
  │   ├── processFindSymbol()           — symbol resolution, multi-repo detection
  │   ├── processTicketContext()        — sparse ticket signal
  │   ├── processRelatedFiles()         — temporal coupling signals
  │   ├── processBlastRadius()          — cross-repo consumers, test gaps
  │   ├── processHealthScore()          — health warning signals
  │   ├── processFileHistory()          — recent change signals
  │   ├── processPRSearch()             — convention pattern signals
  │   └── processFileDependencies()     — import chain, cross-repo imports
  ├── Types: ProcessedResponse, RetrievalSignal, DiscoveredFile, ResolvedSymbol
  └── Test: process mock responses from each tool, verify signals extracted

Day 2: Tool Dispatcher + Analysis Router
  ├── tool_dispatcher.ts
  │   ├── ToolDispatcher class
  │   ├── dispatch() — execute → process → route analysis → analyze
  │   ├── executeRawAction() — maps action to RAG MCP call
  │   └── processResponse() — dispatches to per-tool processor
  ├── analysis_router.ts
  │   ├── routeAnalysis() — decides none/fast per tool + context
  │   ├── analyzeToolResponse() — builds analysis prompt, calls fast model
  │   └── buildToolAnalysisPrompt() per task type
  └── Test: dispatch 5 actions, verify routing + analysis decisions

Day 3: File Confirmation Gate
  ├── file_confirmation.ts
  │   ├── FileReadDecision types
  │   ├── Confirmation integrated into reasoning prompt
  │   ├── executeFileReads() — parallel reads per decisions
  │   └── read_full / read_range / read_signature / skip logic
  ├── Updated reasoning prompt with candidate files + decision format
  └── Test: present 10 candidates, verify model makes good read decisions

Day 4: Cross-Repo Resolver
  ├── cross_repo_resolver.ts
  │   ├── CrossRepoRef, CrossRepoContext types
  │   ├── resolveCrossRepoContext() — generate follow-up actions
  │   ├── findSharedTypesBetweenRepos()
  │   └── Cross-repo signals in reasoning prompt
  ├── Updated accumulator to track per-repo context
  ├── Updated reasoning prompt with cross-repo section
  └── Test: simulate cross-repo find_symbol result, verify follow-up actions generated

Day 5: Updated Context Accumulator
  ├── context_accumulator.ts — UPDATED
  │   ├── Absorb processed responses (not raw tool output)
  │   ├── Per-repo file tracking
  │   ├── Signal accumulation and dedup
  │   ├── Cross-repo context assembly
  │   ├── File role classification with confirmation state
  │   ├── toReasoningSummary() — includes signals, cross-repo, unread candidates
  │   └── toExecutionInput() — full context for Plan-Execute-Verify
  └── Test: full accumulation flow from seed through 3 iterations

Day 6: Updated Loop Controller + Reasoning Prompt
  ├── loop_controller.ts — UPDATED
  │   ├── Seed phase with all seed tools
  │   ├── Reasoning iteration with file confirmation + cross-repo
  │   ├── Tool dispatch with analysis routing
  │   ├── Cross-repo action injection after each iteration
  │   ├── Post-loop enrichment (conventions, health, ticket)
  │   └── File role classification
  ├── reasoning_prompt.ts — UPDATED
  │   ├── Candidate files with read decisions
  │   ├── Cross-repo context section
  │   ├── Signals section
  │   └── Combined action + fileDecision output format
  └── Integration with orchestrator.ts

Day 7: Observability + Metrics
  ├── retrieval_metrics table (updated schema)
  │   ├── Per-iteration details (reasoning, actions, file decisions, signals)
  │   ├── Cross-repo stats
  │   ├── Tool analysis LLM call count
  │   └── File confirmation stats (read/skip/signature breakdown)
  ├── Structured logging for every loop iteration
  └── Dashboard queries for retrieval efficiency analysis

Day 8: Testing + Tuning
  ├── Test suite — 15 golden queries across modes:
  │   UNDERSTAND × 5:
  │   ├── "What does submitOrder do?" — should exit in 1-2 iterations
  │   ├── "Explain auth flow" — should follow call chain across files
  │   ├── "How does bond search work?" — should find controller + service
  │   ├── "What's the architecture of ms-orders?" — should fetch project info + graph
  │   └── "How does BondDTO get to the frontend?" — should cross repos (MS → MFE)
  │
  │   IMPLEMENT × 5:
  │   ├── "Add pagination to bond search" — should find target + type + reference
  │   ├── "Implement BMT-1234" — should fetch ticket + enrich if sparse
  │   ├── "Add circuit breaker to PaymentService" — should find similar PR
  │   ├── "Refactor AuthService to Strategy" — should blast radius + cross-repo
  │   └── "Add logging to OrderController" — should exit FAST (1-2 iterations)
  │
  │   DIAGNOSE × 5:
  │   ├── "NPE in SettlementService line 42" — should read file + trace callers
  │   ├── "Kafka consumer timeout" — should fetch config + consumer code
  │   ├── "Bond search returns 500" — should recent changes + call chain
  │   ├── "Auth token not refreshing" — should cross-repo (shared auth lib)
  │   └── "Memory leak in batch processor" — should file history + code context
  │
  ├── Compare metrics: static gather vs agentic loop
  │   ├── Token count in execution prompt
  │   ├── Relevance of gathered context (manual review)
  │   ├── Time (retrieval phase)
  │   ├── Number of tool calls
  │   └── Accuracy of final generated code
  │
  └── Tune:
      ├── Reasoning prompt — does model make good decisions?
      ├── File confirmation — too aggressive skipping? Too conservative?
      ├── Cross-repo — does it investigate the right repos?
      ├── Analysis routing — are the right tools getting LLM analysis?
      └── Sufficiency criteria — stopping too early or too late?
```

### File Structure

```
src/guide/retrieval/
  loop_controller.ts              — Main loop: seed → reason → dispatch → accumulate
  tool_dispatcher.ts              — Central dispatcher: action → RAG call → process → analyze
  response_processors.ts          — Per-tool response processing + signal extraction
  analysis_router.ts              — Tool-aware LLM routing (none/fast per tool)
  context_accumulator.ts          — Tracks everything: files, symbols, signals, cross-repo
  file_confirmation.ts            — Gate file reads behind model confirmation
  cross_repo_resolver.ts          — Detect + resolve cross-repo dependencies
  reasoning_prompt.ts             — Build reasoning prompt with all context
  reasoning_parser.ts             — Parse model's decisions (actions + file reads)
  sufficiency.ts                  — Hard limits + soft signals
  actions.ts                      — Action types (unchanged)
  metrics.ts                      — Observability + logging

REMOVED:
  src/guide/gather_planner.ts     — Static checklist (replaced)
  src/guide/gather_executor.ts    — Blind parallel execution (replaced)
```

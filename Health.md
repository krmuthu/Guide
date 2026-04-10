# Financial Code Integrity Checks — Final Consolidated Plan

> **Scope:** 5 checks + 1 new MCP tool across all OCBC banking lines — Bonds, Equity, Loans, Deposits, Funds
> **Integration:** Layer 1 AST analyzers + Layer 2 Graph + Layer 3 LLM in health score, standalone CLI, AND new MCP tool for Windsurf
> **Stack:** tree-sitter AST · 55 MCP tools · IGraphClient · SCIP · Hybrid Search · PR Vectors · Katz Centrality · Qwen2.5-7B (CPU, :8082) · Co-change · `trace_api_flow` · `get_scip_callers`/`callees`
> **Effort:** ~7 days
> **Last Updated:** April 2026

-----

## Table of Contents

### Part A — Architecture & Tooling

1. [Overview — The 5 Checks](#1-overview)
1. [Paired Operations Reference — All Business Lines](#2-paired-operations-reference)
1. [MCP Tool Integration Map (55 Tools → Financial Checks)](#3-mcp-tool-integration-map)
1. [New MCP Tool: `check_financial_integrity`](#4-new-mcp-tool-check_financial_integrity)
1. [Improvements Over Initial Design](#5-improvements-over-initial-design)

### Part B — Batch Processing

1. [Large File Batch Processing](#6-large-file-batch-processing)

### Part C — 5 Checks (AST Implementation)

1. [Check 1 — Duplicate / Single-Legged / Looping Debit-Credit](#7-check-1)
1. [Check 2 — Transaction Rollback Integrity](#8-check-2)
1. [Check 3 — Unclosed Conditions (Missing Catch-All)](#9-check-3)
1. [Check 4 — Unhandled Errors / Sudden Termination](#10-check-4)
1. [Check 5 — Maturity / Expiry Without Handler](#11-check-5)

### Part D — RAG Enrichment & Graph Verification

1. [Phase 1: RAG-Enriched Gathering (Parallel)](#12-phase-1-gathering)
1. [Phase 4: Graph Verification + SCIP + Severity Boosting](#13-phase-4-verification)
1. [Cross-Service Transaction Tracing via API Dependency Tools](#14-cross-service-tracing)

### Part E — Layer 3 LLM

1. [LLM Prompt Architecture](#15-llm-prompt-architecture)
1. [System Prompt](#16-system-prompt)
1. [Per-Check Prompts (Full Implementation)](#17-per-check-prompts)
1. [LLM Orchestration + Response Parser](#18-llm-orchestration)

### Part F — Integration & Execution

1. [6-Phase Orchestrator](#19-orchestrator)
1. [Shared Types](#20-shared-types)
1. [Standalone CLI Mode](#21-standalone-cli)
1. [Health Score Integration (9th Dimension)](#22-health-score-integration)
1. [Implementation Schedule (7 Days)](#23-implementation-schedule)
1. [File Structure](#24-file-structure)

-----

# Part A — Architecture & Tooling

## 1. Overview

These checks target **financial transaction safety** across all OCBC banking lines — Bonds, Equity, Loans, Deposits, and Funds. They run in a **6-phase pipeline**: RAG Gathering → AST Analysis → Dedup → Graph/SCIP Verification → LLM Verification → Score.

|#|Check                                           |What It Catches                                                                                                                                              |Severity          |
|-|------------------------------------------------|-------------------------------------------------------------------------------------------------------------------------------------------------------------|------------------|
|1|Duplicate / Single-Legged / Looping Debit-Credit|Unbalanced ledger entries across all paired operations: debit/credit, buy/sell, disburse/repay, place/withdraw, subscribe/redeem                             |**major**         |
|2|Transaction Rollback                            |Missing `@Transactional`, manual commits without rollback, save-then-throw, readOnly TX with writes, cross-service partial commits                           |**major**         |
|3|Unclosed Conditions                             |if/else-if without final else, switch without default, enum switches missing cases — severity escalated when condition involves financial-sensitive variables|**moderate–major**|
|4|Unhandled Errors / Sudden Termination           |Empty catch, System.exit in service code, swallowed exceptions, missing finally, @Async void without handler, broad catch in financial methods               |**moderate–major**|
|5|Maturity / Expiry Without Handler               |Date-driven events missing equals-check, no business-day adjustment, rollover missing a leg, @Scheduled maturity processor without retry/alert               |**major**         |

-----

## 2. Paired Operations Reference — All Business Lines

|Business Line |Outflow (Debit-side)                                                                                    |Inflow (Credit-side)                                                                                                               |Notes                                          |
|--------------|--------------------------------------------------------------------------------------------------------|-----------------------------------------------------------------------------------------------------------------------------------|-----------------------------------------------|
|**Bonds**     |`debit`, `sell`, `redeemBond`, `payoutCoupon`, `payInterest`                                            |`credit`, `buy`, `purchaseBond`, `receiveCoupon`, `collectInterest`                                                                |Coupon paired with accrual                     |
|**Equity**    |`sell`, `closePosition`, `deallocate`, `exerciseOption`, `shortSell`, `disposeHolding`, `reducePosition`|`buy`, `purchase`, `openPosition`, `allocate`, `assignOption`, `coverShort`, `addHolding`, `increasePosition`                      |Position open/close must balance               |
|**Loans**     |`disburse`, `disburseLoan`, `drawdown`, `releaseCollateral`, `writeOff`, `waiveFee`, `forgivePrincipal` |`repay`, `repayLoan`, `paydown`, `pledgeCollateral`, `collectFee`, `accrue`, `capitalize`, `collectRepayment`, `collectInstallment`|Disbursement without repayment schedule suspect|
|**Deposits**  |`place`, `placeDeposit`, `breakDeposit`, `earlyWithdraw`, `terminateDeposit`, `closeDeposit`            |`withdraw`, `accrueInterest`, `creditInterest`, `renewDeposit`                                                                     |Rollover = close old + open new                |
|**Funds**     |`redeem`, `cancelUnits`, `collectFee`, `distributeDividend`, `surrenderUnits`                           |`subscribe`, `allocateUnits`, `rebate`, `reinvestDividend`, `issueUnits`                                                           |NAV-driven, units+cash must balance            |
|**Cross-line**|`transferOut`, `settlePayable`, `remit`, `payOut`                                                       |`transferIn`, `settleReceivable`, `receive`, `payIn`                                                                               |Always two-legged                              |

### Financial-Sensitive Variable Names

Used by Check 3 and Check 4 to escalate severity when these appear in conditions:

|Category          |Terms                                                                                                                                       |
|------------------|--------------------------------------------------------------------------------------------------------------------------------------------|
|**Transaction**   |`type`, `transactionType`, `txnType`, `entryType`, `side`, `direction`, `action`, `operationType`, `movementType`                           |
|**Product**       |`product`, `productType`, `instrumentType`, `assetClass`, `assetType`, `securityType`                                                       |
|**Time**          |`tenor`, `maturity`, `maturityDate`, `expiryDate`, `valueDate`, `settlementDate`, `frequency`, `couponDate`, `rolloverDate`, `repaymentDate`|
|**Rate/Amount**   |`rate`, `rateType`, `interestType`, `fixedOrFloat`, `currency`, `ccy`, `amount`, `notional`, `principal`                                    |
|**Classification**|`status`, `category`, `tier`, `channel`, `segment`, `tranche`, `facility`, `loanType`, `depositType`, `fundType`, `accountType`             |

-----

## 3. MCP Tool Integration Map (55 Tools → Financial Checks)

Of the 55 MCP tools, **32 are directly useful** for financial integrity checks. Here’s exactly how each is used.

### 3.1 Tools Used in Phase 1 — Gathering

|MCP Tool                |What We Gather                                                                                                                                    |Used By                                                        |
|------------------------|--------------------------------------------------------------------------------------------------------------------------------------------------|---------------------------------------------------------------|
|`get_project_info`      |`tech_stack`, `dependencies`, `project_type` — determines if repo has Spring TX, Resilience4j, business-day lib                                   |All checks (false positive reduction)                          |
|`get_repo_config`       |`application.yml` parsed — TX isolation level, scheduler cron, retry config, datasource settings                                                  |Check 2 (TX config), Check 5 (scheduler config)                |
|`search_files`          |Find all files matching `*Service.java`, `*Processor.java`, `*Handler.java` for financial file discovery                                          |All checks (file identification)                               |
|`get_file_skeleton`     |**Lightweight pre-screen** — get method signatures + annotations without reading full file. Filter non-financial files BEFORE loading them for AST|All checks (performance optimization)                          |
|`check_dependency_usage`|Verify if `spring-tx`, `resilience4j-retry`, `jollyday` are used across repos                                                                     |Check 2, Check 4, Check 5                                      |
|`batch_read_files`      |Read multiple counterpart files in one call (e.g., `DebitService.java` + `CreditService.java`)                                                    |Check 1 (counterpart verification)                             |
|`get_file_dependencies` |`direction=imports` — what does this financial service import? `direction=imported_by` — who depends on it?                                       |Check 1 (counterpart imported?), Check 2 (TX manager imported?)|
|`suggest_related_files` |Co-change files — if `DebitService` always changes with `CreditService`, they’re likely a paired operation split                                  |Check 1 (architectural split detection)                        |
|`get_file_history`      |Git commit count for financial files — frequently changed + untested = higher severity                                                            |All checks (severity boosting)                                 |
|`get_file_pr_context`   |PR history — has this file been part of a “fix” PR recently? Was it a financial bug fix?                                                          |All checks (risk amplification)                                |
|`get_recent_changes`    |Recent changes in last 30 days — recently modified financial code gets higher priority                                                            |All checks (recency weighting)                                 |
|`get_ticket_context`    |Linked Jira tickets — has there been a bug report about this service? About debit/credit imbalance?                                               |All checks (corroborating evidence)                            |
|`lookup_glossary`       |Resolve acronyms like STP, DLT, T+2, FD, TD — ensures LLM prompts use correct terminology                                                         |LLM prompts                                                    |
|`search_architecture`   |Confluence docs — find architectural rules: “All financial operations MUST use @Transactional”, “Rollover MUST be atomic”                         |Check 2 (TX rules), Check 5 (rollover rules)                   |

### 3.2 Tools Used in Phase 2 — AST Analysis Support

|MCP Tool            |How It Supports AST                                                                                                       |Used By                                          |
|--------------------|--------------------------------------------------------------------------------------------------------------------------|-------------------------------------------------|
|`read_full_file`    |Fallback when `REPOS_DIR` is unavailable — read file via MCP instead of disk                                              |All checks (file loading fallback)               |
|`search_exact_match`|Find all occurrences of `debit(`, `credit(`, `@Transactional`, `rollover` across codebase — confirms pattern exists       |Check 1 (validate regex patterns match real code)|
|`search_by_code`    |Given a debit code snippet from one repo, find similar patterns in other repos — cross-repo consistency check             |Check 1 (cross-repo paired operation discovery)  |
|`batch_search`      |Run multiple semantic queries in one call: “debit service”, “credit service”, “transaction boundary” — reduces round-trips|All checks (multi-query gathering)               |

### 3.3 Tools Used in Phase 4 — Verification

|MCP Tool                  |Verification Purpose                                                                                        |Used By                                                               |
|--------------------------|------------------------------------------------------------------------------------------------------------|----------------------------------------------------------------------|
|`find_symbol`             |Resolve a method name to its full definition with callers/callees/file/class                                |Check 1 (counterpart verification), Check 2 (TX verification)         |
|`get_code_context`        |Get full context around a flagged line — callers, callees, imports at that location                         |All checks (finding context enrichment)                               |
|`get_scip_callers`        |**Compiler-verified** callers of a function — more accurate than graph for call chain verification          |Check 1 (who calls debit?), Check 5 (who calls maturity handler?)     |
|`get_scip_callees`        |**Compiler-verified** callees — what does this method actually call? Does debit() call credit() internally? |Check 1 (internal counterpart?), Check 2 (does method call save()?)   |
|`get_scip_type_definition`|Resolve type to definition — confirms `TransactionType` is an enum, confirms `CalendarService` exists       |Check 3 (enum definition lookup), Check 5 (BusinessDayConvention type)|
|`get_scip_implementations`|Find interface implementations — if `PaymentService` implements `TransactionalService`, TX may be inherited |Check 2 (inherited @Transactional?)                                   |
|`get_scip_references`     |All references to a symbol — every place `debit()` is called, every place `maturityDate` is read            |Check 1 (debit usage map), Check 5 (maturity date usage map)          |
|`query_knowledge_graph`   |Custom graph queries — find all nodes with `@Transactional`, find all nodes calling any DEBIT_PATTERN method|Check 1 (global paired operation map), Check 2 (TX coverage map)      |
|`analyze_blast_radius`    |Impact analysis — if this financial method has a bug, how many services/consumers are affected?             |All checks (severity boosting by blast radius)                        |
|`analyze_graph_health`    |Dead code + architecture compliance — is this financial method actually dead code?                          |All checks (filter dead code from findings)                           |
|`check_ai_hallucinations` |Verify that LLM-generated additional findings reference real symbols/files                                  |LLM verification (Phase 5)                                            |
|`compare_implementations` |Compare how two repos handle debit/credit — inconsistency may indicate a bug in one                         |Check 1 (cross-repo consistency)                                      |

### 3.4 Tools Used in Phase 4 — Cross-Service Tracing

|MCP Tool              |Cross-Service Purpose                                                                              |Used By                                                              |
|----------------------|---------------------------------------------------------------------------------------------------|---------------------------------------------------------------------|
|`trace_api_flow`      |Trace a POST `/api/debit` → all downstream services it calls → does any of them call `/api/credit`?|Check 1 (cross-service paired operations)                            |
|`get_api_dependencies`|Which repos call this repo’s APIs? Which repos does this repo call? Map the transaction flow       |Check 1 (cross-repo counterpart), Check 2 (cross-service TX boundary)|
|`get_api_surface`     |All endpoints of a service — find `/debit`, `/credit`, `/transfer` endpoints and verify pairing    |Check 1 (API-level paired operation verification)                    |

### 3.5 Tools Used in Phase 5 — LLM Context Building

|MCP Tool               |LLM Context Purpose                                                                                                 |Used By                                          |
|-----------------------|--------------------------------------------------------------------------------------------------------------------|-------------------------------------------------|
|`assemble_file_context`|Build rich file context from indexed chunks — more efficient than reading full file for LLM                         |All LLM prompts (targeted context window filling)|
|`get_chunks_for_symbol`|Get exact code for a specific symbol — focused LLM context without loading entire file                              |LLM prompts (method-level context)               |
|`search_semantic_code` |Find similar financial methods for few-shot examples in LLM prompt — “here’s how other services handle debit/credit”|LLM prompts (few-shot pattern examples)          |

### 3.6 Tools NOT Used (and Why)

|MCP Tool                                                            |Why Not Used                                                                                 |
|--------------------------------------------------------------------|---------------------------------------------------------------------------------------------|
|`parse_repository_graph`                                            |Graph already built by bootstrap Phase 2g — never rebuild during analysis                    |
|`jira_add_comment` / `jira_transition_issue` / `jira_create_subtask`|Write operations — health score is read-only                                                 |
|`create_pull_request`                                               |Write operation — health score doesn’t create PRs                                            |
|`scaffold_project` / `get_project_templates`                        |Project creation — not relevant                                                              |
|`pipeline_*`                                                        |Pipeline management — not relevant                                                           |
|`rate_search_result`                                                |Search quality feedback — not relevant                                                       |
|`get_current_user`                                                  |User identity — not relevant                                                                 |
|`get_server_status`                                                 |Server health — not relevant at analysis time                                                |
|`get_file_diff`                                                     |Commit-level diffs — `get_file_history` + `get_recent_changes` cover this                    |
|`check_component_security`                                          |CVE scanning — separate concern, but could flag vulnerable TX/retry libs (future enhancement)|

-----

## 4. New MCP Tool: `check_financial_integrity`

### 4.1 Why a New Tool?

The 5 checks run nightly in the health score. But developers need **on-demand** checks from Windsurf:

- “Before I merge this PR, check if my debit/credit is balanced”
- “Is this switch statement covering all transaction types?”
- “Does this maturity handler account for non-business days?”

A new MCP tool exposes the financial integrity checks as a **developer-facing query** — not just a health score batch.

### 4.2 Tool Definition

```typescript
// src/mcp/tools/check_financial_integrity.ts

export const CHECK_FINANCIAL_INTEGRITY = {
  name: 'check_financial_integrity',
  description: `Run financial integrity checks on a file or method.

Checks for: unbalanced debit/credit (across all business lines — bonds, equity, loans, deposits, funds),
missing @Transactional on multi-write methods, unclosed conditions on financial enums/types,
swallowed exceptions in financial methods, and maturity/expiry date handling issues.

Returns findings with severity (minor/moderate/major), evidence, and recommendations.`,

  inputSchema: {
    type: 'object',
    properties: {
      repo_slug: {
        type: 'string',
        description: 'Repository slug (e.g., "BMT/ms-bonds-trading")',
      },
      file_path: {
        type: 'string',
        description: 'File to check (e.g., "src/main/java/com/ocbc/service/DebitService.java")',
      },
      method_name: {
        type: 'string',
        description: 'Optional — check a specific method only. If omitted, checks entire file.',
      },
      checks: {
        type: 'array',
        items: { type: 'string', enum: ['debit_credit', 'transaction', 'conditions', 'errors', 'maturity', 'all'] },
        description: 'Which checks to run. Default: all',
      },
      include_llm: {
        type: 'boolean',
        description: 'Run LLM verification on ambiguous findings. Slower but more accurate. Default: false',
      },
    },
    required: ['repo_slug', 'file_path'],
  },
};
```

### 4.3 Implementation

```typescript
export async function handleCheckFinancialIntegrity(
  params: CheckFinancialIntegrityParams,
  deps: ServerDeps
): Promise<FinancialIntegrityResult> {
  const { repo_slug, file_path, method_name, checks = ['all'], include_llm = false } = params;

  // ── Load file ──
  const repoBasePath = path.join(process.env.REPOS_DIR || '/repos', repo_slug.replace('/', '/'));
  const parser = deps.parserPool;
  const file = await loadAndParseFile(path.join(repoBasePath, file_path), parser);

  if (!file) {
    return { error: `File not found: ${file_path}`, findings: [], score: null };
  }

  // ── Create analysis windows ──
  let windows = createAnalysisWindows(file);

  // Filter to specific method if requested
  if (method_name) {
    windows = windows.filter(w =>
      w.methods.some(m => extractFunctionName(m, file.language) === method_name)
    );
  }

  // ── Quick gather (lighter than full nightly gather) ──
  const ctx = await gatherFinancialContextLight(
    deps.pool, deps.graphClient, repo_slug, file_path
  );

  // ── Run selected checks ──
  const runAll = checks.includes('all');
  let findings: FinancialFinding[] = [];

  for (const window of windows) {
    if (runAll || checks.includes('debit_credit'))
      findings.push(...analyzeDebitCreditWindow(window, ctx));
    if (runAll || checks.includes('transaction'))
      findings.push(...analyzeTransactionRollbackWindow(window, ctx));
    if (runAll || checks.includes('conditions'))
      findings.push(...analyzeUnclosedConditionsWindow(window));
    if (runAll || checks.includes('errors'))
      findings.push(...analyzeErrorHandlingWindow(window, ctx));
    if (runAll || checks.includes('maturity'))
      findings.push(...analyzeMaturityExpiryWindow(window, ctx));
  }

  findings = deduplicateFindings(findings);

  // Graph verification for single-legged
  findings = await verifyWithCallChainSCIP(findings, deps, repo_slug);

  // Optional LLM verification
  if (include_llm && findings.some(f => f.confidence === 'possible')) {
    findings = await runLLMVerification(
      findings, windows, ctx, deps.qwen, deps.searchPipeline, deps.pool, repo_slug
    );
  }

  findings = findings.filter(f => f.confidence !== 'unlikely');

  return {
    findings,
    score: computeScore(findings),
    summary: {
      total: findings.length,
      major: findings.filter(f => f.severity === 'major').length,
      moderate: findings.filter(f => f.severity === 'moderate').length,
      minor: findings.filter(f => f.severity === 'minor').length,
      checks_run: checks.includes('all') ? 5 : checks.length,
    },
  };
}
```

-----

## 5. Improvements Over Initial Design

### 5.1 SCIP-First Symbol Verification (replaces raw DB queries)

**Before:** Raw SQL against `scip_symbol_info` table to verify if `debit()` resolves to a financial type.

**After:** Use the existing `get_scip_callers`, `get_scip_callees`, `get_scip_type_definition`, and `get_scip_references` MCP tools. These handle SCIP data access, fallback, and formatting — no raw SQL needed.

```typescript
// ── IMPROVED: Use SCIP tools instead of raw DB queries ──

async function verifyWithCallChainSCIP(
  findings: FinancialFinding[],
  deps: ServerDeps,
  repo: string
): Promise<FinancialFinding[]> {
  const verified: FinancialFinding[] = [];

  for (const finding of findings) {
    if (!finding.needsGraphVerification) {
      verified.push(finding);
      continue;
    }

    const missingType = finding.check === 'single_legged_debit' ? 'credit' : 'debit';
    const counterpartPatterns = missingType === 'credit' ? CREDIT_PATTERNS : DEBIT_PATTERNS;

    // ── Step 1: SCIP callers (compiler-verified, more accurate than graph) ──
    let counterpartFound = false;

    try {
      const scipCallers = await deps.mcpClient.call('get_scip_callers', {
        symbol: finding.function,
        repo_slug: repo,
        limit: 20,
      });

      for (const caller of scipCallers?.callers ?? []) {
        // Read the caller's source
        const callerCode = await deps.mcpClient.call('get_chunks_for_symbol', {
          symbol: caller.name,
          repo_slug: repo,
        });

        const callerText = callerCode?.chunks?.map((c: any) => c.content).join('\n') ?? '';
        if (counterpartPatterns.some(p => p.test(callerText))) {
          counterpartFound = true;
          finding.evidence += ` (SCIP-verified: caller '${caller.name}' contains matching ${missingType} operation)`;
          break;
        }
      }
    } catch {
      // SCIP unavailable — fall back to graph
      const graphCallers = await deps.graphClient.getReverseCallChain(finding.function, repo, 3);
      for (const caller of graphCallers) {
        if (!caller.file_path) continue;
        try {
          const content = await fs.readFile(
            path.join(process.env.REPOS_DIR || '/repos', repo.replace('/', '/'), caller.file_path),
            'utf-8'
          );
          if (counterpartPatterns.some(p => p.test(content))) {
            counterpartFound = true;
            break;
          }
        } catch { continue; }
      }
    }

    // ── Step 2: Cross-service check via API flow ──
    if (!counterpartFound) {
      try {
        // Does this service expose an API that another service's credit endpoint calls?
        const apiDeps = await deps.mcpClient.call('get_api_dependencies', {
          repo, direction: 'both',
        });

        const consumers = apiDeps?.consumers ?? [];
        const providers = apiDeps?.providers ?? [];

        // Check if any consumer or provider has the counterpart
        for (const dep of [...consumers, ...providers]) {
          const surface = await deps.mcpClient.call('get_api_surface', { repo: dep.repo });
          const endpoints = surface?.endpoints ?? [];

          const hasCounterpart = endpoints.some((ep: any) =>
            counterpartPatterns.some(p => p.test(ep.path + ' ' + ep.handler))
          );

          if (hasCounterpart) {
            counterpartFound = true;
            finding.evidence += ` (Cross-service: ${dep.repo} has matching ${missingType} endpoint)`;
            finding.confidence = 'unlikely'; // Architectural split across services
            finding.severity = 'minor';
            break;
          }
        }
      } catch {
        // API tools unavailable — skip cross-service check
      }
    }

    if (counterpartFound && finding.confidence !== 'unlikely') {
      finding.confidence = 'unlikely';
      finding.severity = 'minor';
    } else if (!counterpartFound) {
      finding.confidence = 'likely';
      finding.evidence += ` (Verified: no counterpart in SCIP call chain or cross-service APIs)`;
    }

    verified.push(finding);
  }

  return verified;
}
```

### 5.2 `get_file_skeleton` Pre-Screening (performance)

**Before:** Load every file in the repo, parse with tree-sitter, then check `isFinancialFile()`.

**After:** Use `get_file_skeleton` to get method signatures + annotations cheaply. Only load files that pass the financial pre-screen.

```typescript
async function preScreenFinancialFiles(
  deps: ServerDeps,
  repo: string,
  allFilePaths: string[]
): Promise<string[]> {
  const candidates: string[] = [];

  // Step 1: Path-based filter (cheap, no MCP call)
  const pathFiltered = allFilePaths.filter(fp => isFinancialFilePath(fp));

  // Step 2: Skeleton-based filter (one MCP call per file, but lightweight)
  const SKELETON_BATCH = 20;
  for (let i = 0; i < pathFiltered.length; i += SKELETON_BATCH) {
    const batch = pathFiltered.slice(i, i + SKELETON_BATCH);

    const skeletons = await Promise.allSettled(
      batch.map(fp => deps.mcpClient.call('get_file_skeleton', { repo_slug: repo, file_path: fp }))
    );

    for (let j = 0; j < batch.length; j++) {
      if (skeletons[j].status !== 'fulfilled') {
        candidates.push(batch[j]); // Can't get skeleton — include it
        continue;
      }

      const skeleton = (skeletons[j] as PromiseFulfilledResult<any>).value;
      const methods = skeleton?.methods ?? [];
      const annotations = skeleton?.annotations ?? [];

      // Check if any method name or annotation suggests financial operation
      const skeletonText = [
        ...methods.map((m: any) => m.name),
        ...annotations,
      ].join(' ');

      if (isFinanciallySensitive(skeletonText) || isFinancialContent(skeletonText)) {
        candidates.push(batch[j]);
      }
    }
  }

  return candidates;
}
```

### 5.3 `trace_api_flow` for Cross-Service TX Boundary (Check 2)

**Before:** Check 2 only detected missing `@Transactional` within a single service.

**After:** Use `trace_api_flow` to detect **cross-service partial commits**: Service A commits, calls Service B’s API, Service B fails → Service A’s commit is orphaned.

```typescript
async function detectCrossServicePartialCommits(
  deps: ServerDeps,
  repo: string,
  findings: FinancialFinding[]
): Promise<FinancialFinding[]> {
  // Find methods that call external APIs within a @Transactional boundary
  const newFindings: FinancialFinding[] = [];

  for (const f of findings) {
    if (f.check !== 'multi_write_no_transaction') continue;

    // Check if the method also makes HTTP calls
    // (already parsed by AST — look for RestTemplate, WebClient, Feign patterns)
    // If it does, trace the API flow
    try {
      const apiSurface = await deps.mcpClient.call('get_api_surface', { repo });
      const endpoints = apiSurface?.endpoints ?? [];

      // Find if the flagged method is an endpoint handler
      const matchedEndpoint = endpoints.find((ep: any) =>
        ep.handler?.includes(f.function) || ep.method_name === f.function
      );

      if (matchedEndpoint) {
        const flow = await deps.mcpClient.call('trace_api_flow', {
          endpoint_path: matchedEndpoint.path,
          http_method: matchedEndpoint.http_method,
        });

        const downstreamServices = flow?.downstream ?? [];

        if (downstreamServices.length > 0) {
          newFindings.push({
            check: 'cross_service_partial_commit',
            severity: 'major',
            filePath: f.filePath,
            line: f.line,
            function: f.function,
            evidence: `Method '${f.function}' writes to local DB AND calls ${downstreamServices.length} ` +
                      `downstream service(s): ${downstreamServices.map((d: any) => d.repo).join(', ')}. ` +
                      `If downstream call fails after local commit, data is inconsistent across services. ` +
                      `This requires saga/outbox pattern, not just @Transactional.`,
            recommendation: 'Implement outbox pattern: write to outbox table in same TX, then process ' +
                            'outbox asynchronously. Or use saga with compensating transactions.',
            confidence: 'likely',
          });
        }
      }
    } catch {
      // API flow tracing unavailable — skip
    }
  }

  return [...findings, ...newFindings];
}
```

### 5.4 `compare_implementations` for Cross-Repo Consistency (Check 1)

**Before:** Each repo checked independently.

**After:** Compare how two repos handle the same financial pattern. If `ms-bonds-trading` uses paired debit/credit but `ms-equity-trading` has single-legged debit, that’s a consistency issue.

```typescript
async function checkCrossRepoConsistency(
  deps: ServerDeps,
  financialRepos: string[],
  query: string
): Promise<FinancialFinding[]> {
  const findings: FinancialFinding[] = [];

  // Compare repos pairwise for key financial patterns
  const patterns = [
    'debit credit balance transaction',
    'transaction rollback error handling',
    'maturity expiry business day',
  ];

  for (const pattern of patterns) {
    for (let i = 0; i < financialRepos.length; i++) {
      for (let j = i + 1; j < financialRepos.length; j++) {
        try {
          const comparison = await deps.mcpClient.call('compare_implementations', {
            query: pattern,
            repo_a: financialRepos[i],
            repo_b: financialRepos[j],
            top_k: 3,
          });

          // If one repo has a pattern the other doesn't, flag it
          if (comparison?.differences?.length > 0) {
            for (const diff of comparison.differences) {
              if (diff.only_in_a || diff.only_in_b) {
                findings.push({
                  check: 'cross_repo_inconsistency',
                  severity: 'moderate',
                  filePath: diff.file_path ?? '',
                  line: 0,
                  function: diff.symbol ?? 'unknown',
                  evidence: `'${pattern}' pattern found in ${diff.only_in_a ? financialRepos[i] : financialRepos[j]} ` +
                            `but not in ${diff.only_in_a ? financialRepos[j] : financialRepos[i]}. ` +
                            `Inconsistent financial patterns across repos may indicate missing implementation.`,
                  recommendation: 'Review whether this pattern should be present in both repos.',
                  confidence: 'possible',
                });
              }
            }
          }
        } catch { continue; }
      }
    }
  }

  return findings;
}
```

### 5.5 `assemble_file_context` for LLM Context Building

**Before:** Read full file from disk, truncate at 350 lines (head 245 + tail 70).

**After:** Use `assemble_file_context` which returns chunked file content with relevance scoring — the LLM gets the most relevant chunks, not arbitrary head/tail.

```typescript
async function buildLLMFileContext(
  deps: ServerDeps,
  repo: string,
  filePath: string,
  finding: FinancialFinding,
  budgetTokens: number = 14000
): Promise<string> {
  try {
    const assembled = await deps.mcpClient.call('assemble_file_context', {
      repo_slug: repo,
      file_path: filePath,
      role: 'auditor', // Context assembled for code review
    });

    if (assembled?.content) {
      // Truncate to budget
      const content = assembled.content;
      if (estimateTokens(content) <= budgetTokens) return content;

      // Too big — use chunks for the specific symbol instead
      const chunks = await deps.mcpClient.call('get_chunks_for_symbol', {
        symbol: finding.function,
        repo_slug: repo,
      });

      return chunks?.chunks?.map((c: any) => c.content).join('\n\n') ?? content.slice(0, budgetTokens * 4);
    }
  } catch {
    // Fallback to disk read
    const content = await fs.readFile(
      path.join(process.env.REPOS_DIR || '/repos', repo.replace('/', '/'), filePath),
      'utf-8'
    );
    const lines = content.split('\n');
    if (lines.length <= 350) return content;
    return lines.slice(0, 245).join('\n') + '\n// ... omitted ...\n' + lines.slice(-70).join('\n');
  }

  return '';
}
```

### 5.6 Additional Improvements Summary

|Improvement                                      |Description                                                                                                        |Impact                            |
|-------------------------------------------------|-------------------------------------------------------------------------------------------------------------------|----------------------------------|
|**`check_ai_hallucinations` on LLM output**      |Verify LLM-discovered findings reference real symbols/files                                                        |Eliminates false LLM findings     |
|**`search_architecture` for TX rules**           |Query Confluence for org-wide rules like “all financial writes MUST be @Transactional” — use as severity escalation|Better severity calibration       |
|**`lookup_glossary` in LLM prompts**             |Resolve banking acronyms before sending to Qwen — “FD” → “Fixed Deposit”, “STP” → “Straight-Through Processing”    |Better LLM comprehension          |
|**`get_scip_implementations` for interface TX**  |If `PaymentService implements TransactionalService`, TX may be inherited from interface                            |Reduces false positives in Check 2|
|**Cross-service partial commit (new check type)**|Uses `trace_api_flow` to detect local commit + HTTP call pattern                                                   |Catches distributed TX bugs       |
|**Cross-repo consistency (new check type)**      |Uses `compare_implementations` to find repos with inconsistent financial patterns                                  |Catches org-wide gaps             |
|**Dead code filter via `analyze_graph_health`**  |Don’t report findings on dead financial methods — they’re noise                                                    |Reduces false positives           |

-----

# Part B — Batch Processing

## 6. Large File Batch Processing

### 6.1 The Problem

Financial service classes (e.g. `SettlementService.java` at 2,500 lines) exceed AST traversal efficiency limits and LLM context windows.

### 6.2 Strategy: Method-Level Windowing

Split at **method boundaries** via tree-sitter. Each window holds ≤10 methods or ≤500 lines. Class-level context (annotations, fields, imports) is shared across all windows.

```typescript
// src/lib/health_score/analyzers/financial_integrity/batch_processor.ts

export interface AnalysisWindow {
  file: LoadedFile;
  methods: MethodNode[];
  windowIndex: number;
  totalWindows: number;
  classContext: ClassContext;
}

export interface ClassContext {
  className: string;
  classAnnotations: string[];
  hasClassTransactional: boolean;
  fieldDeclarations: string[];
  imports: string[];
  implementedInterfaces: string[];
  extendedClass: string | null;
  filePath: string;
  totalLines: number;
  totalMethods: number;
}

const METHODS_PER_WINDOW = 10;
const MAX_LINES_PER_WINDOW = 500;

export function createAnalysisWindows(file: LoadedFile): AnalysisWindow[] {
  const tree = file.tree;
  const classNode = findClassDeclaration(tree);

  const classContext: ClassContext = {
    className: classNode?.childForFieldName('name')?.text ??
               path.basename(file.path, path.extname(file.path)),
    classAnnotations: extractAnnotations(classNode),
    hasClassTransactional: extractAnnotations(classNode).some(a => a === 'Transactional'),
    fieldDeclarations: extractFieldDeclarations(classNode, file.language),
    imports: extractImports(tree, file.language),
    implementedInterfaces: extractImplementedInterfaces(classNode, file.language),
    extendedClass: extractExtendedClass(classNode, file.language),
    filePath: file.path,
    totalLines: file.lineCount,
    totalMethods: 0,
  };

  const methods = findMethodDeclarations(tree);
  classContext.totalMethods = methods.length;

  // Small file — no windowing
  if (methods.length <= METHODS_PER_WINDOW && file.lineCount <= MAX_LINES_PER_WINDOW) {
    return [{ file, methods, windowIndex: 0, totalWindows: 1, classContext }];
  }

  // Window by method count, respecting line cap
  const windows: AnalysisWindow[] = [];
  let batch: typeof methods = [];
  let batchLines = 0;

  for (const method of methods) {
    const mLines = method.endPosition.row - method.startPosition.row + 1;

    // Giant method → solo window
    if (mLines > MAX_LINES_PER_WINDOW) {
      if (batch.length > 0) {
        windows.push({ file, methods: batch, windowIndex: windows.length, totalWindows: -1, classContext });
        batch = []; batchLines = 0;
      }
      windows.push({ file, methods: [method], windowIndex: windows.length, totalWindows: -1, classContext });
      continue;
    }

    if (batch.length >= METHODS_PER_WINDOW || batchLines + mLines > MAX_LINES_PER_WINDOW) {
      windows.push({ file, methods: batch, windowIndex: windows.length, totalWindows: -1, classContext });
      batch = []; batchLines = 0;
    }

    batch.push(method);
    batchLines += mLines;
  }

  if (batch.length > 0) {
    windows.push({ file, methods: batch, windowIndex: windows.length, totalWindows: -1, classContext });
  }

  for (const w of windows) w.totalWindows = windows.length;
  return windows;
}
```

### 6.3 Giant Method Sub-Splitting for LLM

```typescript
export function splitGiantMethodForLLM(
  method: SyntaxNode,
  budgetLines: number = 300
): { segments: string[]; overlap: number } {
  const lines = method.text.split('\n');
  if (lines.length <= budgetLines) return { segments: [method.text], overlap: 0 };

  const OVERLAP = 30;
  const segments: string[] = [];
  const signature = lines[0];

  for (let i = 0; i < lines.length; i += budgetLines - OVERLAP) {
    const start = Math.max(0, i);
    const end = Math.min(lines.length, i + budgetLines);
    const prefix = start > 0 ? `// ... continued from line ${start}\n${signature}\n` : '';
    segments.push(prefix + lines.slice(start, end).join('\n'));
    if (end >= lines.length) break;
  }

  return { segments, overlap: OVERLAP };
}
```

### 6.4 Cross-Window Dedup

```typescript
export function deduplicateFindings(findings: FinancialFinding[]): FinancialFinding[] {
  const seen = new Map<string, FinancialFinding>();
  const MAX_PER_CHECK_PER_FILE = 5;

  for (const f of findings) {
    const key = `${f.check}::${f.filePath}::${f.function}`;
    const existing = seen.get(key);
    if (!existing || compareSeverity(f, existing) > 0) {
      seen.set(key, f);
    }
  }

  // Cap per check-type per file
  const grouped = new Map<string, FinancialFinding[]>();
  for (const f of seen.values()) {
    const gk = `${f.check}::${f.filePath}`;
    if (!grouped.has(gk)) grouped.set(gk, []);
    grouped.get(gk)!.push(f);
  }

  const result: FinancialFinding[] = [];
  for (const [, group] of grouped) {
    if (group.length <= MAX_PER_CHECK_PER_FILE) {
      result.push(...group);
    } else {
      const sorted = group.sort((a, b) => compareSeverity(b, a));
      result.push(...sorted.slice(0, MAX_PER_CHECK_PER_FILE));
      result.push({
        ...group[0],
        line: 0, function: '(multiple)',
        evidence: `${group.length - MAX_PER_CHECK_PER_FILE} additional '${group[0].check}' occurrences suppressed.`,
        recommendation: 'This pattern repeats across the file. Consider structural refactor.',
      });
    }
  }

  return result;
}
```

-----

# Part C — 5 Checks (AST Implementation)

## 7. Check 1 — Duplicate / Single-Legged / Looping Debit-Credit

### 7.1 Detection

Per window: find all DEBIT_PATTERN and CREDIT_PATTERN calls. Flag single-legged, loop-without-guard, and duplicate.

### 7.2 Pattern Arrays

```typescript
// ── OUTFLOW / DEBIT-SIDE (50+ patterns) ──
const DEBIT_PATTERNS = [
  // Generic
  /\bdebit\s*\(/i, /\bwithdraw\s*\(/i, /\bsubtractBalance\s*\(/i, /\bcharge\s*\(/i,
  /\bdecrease(?:Balance|Amount|Funds)\s*\(/i,
  /\.setTransactionType\s*\(\s*["'](?:DEBIT|DR|WITHDRAWAL)["']\s*\)/i,
  /TransactionType\s*\.\s*(?:DEBIT|DR|WITHDRAWAL)/i,
  /\.setEntryType\s*\(\s*["']D["']\s*\)/i,
  /\.setMovementType\s*\(\s*["'](?:OUT|OUTFLOW|DEBIT)["']\s*\)/i,
  // Equity
  /\bsell\s*\(/i, /\bshortSell\s*\(/i, /\bclosePosition\s*\(/i, /\bdeallocate\s*\(/i,
  /\bexerciseOption\s*\(/i, /\bdisposeHolding\s*\(/i, /\breducePosition\s*\(/i,
  /OrderSide\s*\.\s*SELL/i, /TradeSide\s*\.\s*SELL/i,
  // Loans
  /\bdisburse\s*\(/i, /\bdisburseLoan\s*\(/i, /\bdrawdown\s*\(/i,
  /\breleaseCollateral\s*\(/i, /\bwriteOff\s*\(/i, /\bwaiveFee\s*\(/i, /\bforgivePrincipal\s*\(/i,
  // Deposits
  /\bplace(?:Deposit)?\s*\(/i, /\bbreakDeposit\s*\(/i, /\bearlyWithdraw\s*\(/i,
  /\bterminateDeposit\s*\(/i, /\bcloseDeposit\s*\(/i,
  // Funds
  /\bredeem\s*\(/i, /\bcancelUnits?\s*\(/i, /\bcollectFee\s*\(/i,
  /\bdistributeDividend\s*\(/i, /\bsurrenderUnits?\s*\(/i,
  // Bonds
  /\bpayoutCoupon\s*\(/i, /\bpayInterest\s*\(/i, /\bredeemBond\s*\(/i,
  // Cross-line
  /\btransferOut\s*\(/i, /\bsettlePayable\s*\(/i, /\bremit\s*\(/i, /\bpayOut\s*\(/i,
];

// ── INFLOW / CREDIT-SIDE (50+ patterns) ──
const CREDIT_PATTERNS = [
  // Generic
  /\bcredit\s*\(/i, /\bdeposit\s*\(/i, /\baddBalance\s*\(/i,
  /\bincrease(?:Balance|Amount|Funds)\s*\(/i,
  /\.setTransactionType\s*\(\s*["'](?:CREDIT|CR|DEPOSIT)["']\s*\)/i,
  /TransactionType\s*\.\s*(?:CREDIT|CR|DEPOSIT)/i,
  /\.setEntryType\s*\(\s*["']C["']\s*\)/i,
  /\.setMovementType\s*\(\s*["'](?:IN|INFLOW|CREDIT)["']\s*\)/i,
  // Equity
  /\bbuy\s*\(/i, /\bpurchase\s*\(/i, /\bopenPosition\s*\(/i,
  /\ballocate(?:Shares|Units|Position)?\s*\(/i, /\bassignOption\s*\(/i,
  /\bcoverShort\s*\(/i, /\baddHolding\s*\(/i, /\bincreasePosition\s*\(/i,
  /OrderSide\s*\.\s*BUY/i, /TradeSide\s*\.\s*BUY/i,
  // Loans
  /\brepay\s*\(/i, /\brepayLoan\s*\(/i, /\bpaydown\s*\(/i,
  /\bpledgeCollateral\s*\(/i, /\bcollectRepayment\s*\(/i,
  /\baccrue(?:Interest)?\s*\(/i, /\bcapitalize(?:Interest)?\s*\(/i, /\bcollectInstallment\s*\(/i,
  // Deposits
  /\bplaceDeposit\s*\(/i, /\baccrueInterest\s*\(/i, /\bcreditInterest\s*\(/i, /\brenewDeposit\s*\(/i,
  // Funds
  /\bsubscribe\s*\(/i, /\ballocateUnits?\s*\(/i, /\brebate\s*\(/i,
  /\breinvestDividend\s*\(/i, /\bissueUnits?\s*\(/i,
  // Bonds
  /\breceiveCoupon\s*\(/i, /\bcollectInterest\s*\(/i, /\bpurchaseBond\s*\(/i,
  // Cross-line
  /\btransferIn\s*\(/i, /\bsettleReceivable\s*\(/i, /\breceive\s*\(/i, /\bpayIn\s*\(/i,
];

const IDEMPOTENCY_PATTERNS = [
  /idempoten/i, /dedup(?:lication)?/i, /alreadyProcessed/i, /\.contains\s*\(/,
  /processedIds/i, /transactionId/i, /ifAbsent/i, /computeIfAbsent/i,
  /INSERT\s+.*\s+ON\s+CONFLICT/i, /INSERT\s+IGNORE/i, /MERGE\s+INTO/i,
  /referenceNumber/i, /requestId/i,
];
```

### 7.3 Window-Level Analyzer

```typescript
export function analyzeDebitCreditWindow(
  window: AnalysisWindow,
  ctx: FinancialGatheredContext
): FinancialFinding[] {
  const findings: FinancialFinding[] = [];
  const { file, methods, classContext } = window;

  for (const method of methods) {
    const funcName = extractFunctionName(method, file.language);
    const funcBody = method.text;
    const funcLines = funcBody.split('\n');

    const debits = findPatternCalls(funcBody, DEBIT_PATTERNS, 'debit', method, file);
    const credits = findPatternCalls(funcBody, CREDIT_PATTERNS, 'credit', method, file);

    // Single-legged
    if (debits.length > 0 && credits.length === 0) {
      findings.push({
        check: 'single_legged_debit', severity: 'major', filePath: file.path,
        line: debits[0].line, function: funcName,
        evidence: `'${funcName}' has ${debits.length} outflow op(s) [${debits.map(d => d.methodName).join(', ')}] but no inflow. Risk: unbalanced ledger.`,
        recommendation: 'Verify intentional (fee/accrual/dividend). Add counterpart or ensure caller provides it.',
        needsGraphVerification: true, confidence: 'possible',
      });
    }
    if (credits.length > 0 && debits.length === 0) {
      findings.push({
        check: 'single_legged_credit', severity: 'major', filePath: file.path,
        line: credits[0].line, function: funcName,
        evidence: `'${funcName}' has ${credits.length} inflow op(s) [${credits.map(c => c.methodName).join(', ')}] but no outflow.`,
        recommendation: 'Verify intentional (interest accrual, subscription). Add counterpart or ensure caller provides it.',
        needsGraphVerification: true, confidence: 'possible',
      });
    }

    // Loop without guard
    for (const op of [...debits, ...credits]) {
      if (op.inLoop && !op.hasIdempotencyGuard) {
        findings.push({
          check: 'loop_debit_credit', severity: 'major', filePath: file.path,
          line: op.line, function: funcName,
          evidence: `${op.type}() '${op.methodName}' inside ${op.loopType ?? 'loop'} without idempotency guard. Risk: duplicate processing.`,
          recommendation: 'Add processed-ID set, DB unique constraint, or INSERT ON CONFLICT.',
          confidence: 'likely',
        });
      }
    }

    // Duplicate calls
    const debitNames = debits.map(d => d.methodName);
    for (const dupName of new Set(debitNames.filter((n, i) => debitNames.indexOf(n) !== i))) {
      const dupCalls = debits.filter(d => d.methodName === dupName);
      if (!areInDifferentBranches(dupCalls, method, file.language)) {
        findings.push({
          check: 'duplicate_debit', severity: 'major', filePath: file.path,
          line: dupCalls[1].line, function: funcName,
          evidence: `'${dupName}' called ${dupCalls.length}x at lines ${dupCalls.map(d => d.line).join(', ')} — not in exclusive branches. Risk: double-charge.`,
          recommendation: 'If multi-leg, add comment. Otherwise remove duplicate or add guard.',
          confidence: 'likely',
        });
      }
    }
  }

  return findings;
}
```

## 8–11. Checks 2–5

> **Implementation is identical to the previous plan documents.** The key change is every analyzer now operates on `AnalysisWindow` instead of `LoadedFile`, and receives `FinancialGatheredContext` for false-positive reduction. Refer to the base plan for full code of:
> 
> - `analyzeTransactionRollbackWindow()` — Check 2 (§4.2 in base plan)
> - `analyzeTypescriptTransactionsWindow()` — Check 2 TS variant (§4.3 in base plan)
> - `analyzeUnclosedConditionsWindow()` — Check 3 (§5.2 in base plan)
> - `analyzeErrorHandlingWindow()` — Check 4 (§6.2 in base plan)
> - `analyzeMaturityExpiryWindow()` — Check 5 (§7.2 in base plan)

**One addition per check from tool integration:**

|Check  |Tool-Powered Enhancement                                                                                                                                                                          |
|-------|--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
|Check 2|`get_scip_implementations` — check if `@Transactional` is inherited from interface. `trace_api_flow` — detect cross-service partial commits (new check type `cross_service_partial_commit`)       |
|Check 3|`get_scip_type_definition` — resolve enum type and get ALL values for completeness check (replaces regex-based enum search across files)                                                          |
|Check 4|`search_architecture` — query Confluence for “error handling standards” to determine if log-only is org-approved                                                                                  |
|Check 5|`check_dependency_usage('jollyday')` — if business-day lib exists in repo, downgrade severity. `get_scip_references` — find ALL references to `maturityDate` to see if adjustment happens upstream|

-----

# Part D — RAG Enrichment & Graph Verification

## 12. Phase 1: RAG-Enriched Gathering (Parallel)

```typescript
export async function gatherFinancialContext(
  deps: ServerDeps,
  repo: string,
  financialFiles: LoadedFile[]
): Promise<FinancialGatheredContext> {
  const filePaths = financialFiles.map(f => f.path);

  const [
    projectInfo, repoConfig, centralityResult, testCoverage,
    fileImports, relatedPRs, bugTickets, fileHistory,
    archRules, glossary, graphHealth, apiDeps
  ] = await Promise.allSettled([

    // 1. get_project_info → framework, dependencies
    deps.mcpClient.call('get_project_info', { repo_slug: repo }),

    // 2. get_repo_config → application.yml parsed
    deps.mcpClient.call('get_repo_config', { repo_slug: repo }),

    // 3. IGraphClient centrality
    deps.graphClient.computeCentrality(repo),

    // 4. test_coverage table
    deps.pool.query('SELECT source_file, test_file FROM test_coverage WHERE repo_slug = $1', [repo]),

    // 5. file_imports table
    deps.pool.query(
      'SELECT file_path, imported_module FROM file_imports WHERE repo_slug = $1 AND file_path = ANY($2)',
      [repo, filePaths]
    ),

    // 6. batch_search for similar financial patterns + PRs
    deps.mcpClient.call('batch_search', {
      queries: [
        { query: 'debit credit transaction rollback fix', repo_slug: repo, top_k: 5 },
        { query: 'maturity expiry business day rollover fix', repo_slug: repo, top_k: 5 },
        { query: '@Transactional save delete repository', repo_slug: repo, top_k: 5 },
      ],
    }),

    // 7. get_ticket_context for financial bug tickets
    deps.pool.query(`
      SELECT issue_key, summary, status FROM otak.jira_tickets
      WHERE repo_slug = $1 AND (
        summary ILIKE '%debit%' OR summary ILIKE '%credit%' OR summary ILIKE '%transaction%'
        OR summary ILIKE '%maturity%' OR summary ILIKE '%rollover%' OR summary ILIKE '%balance%'
      ) ORDER BY created_at DESC LIMIT 10`, [repo]),

    // 8. get_file_history for change frequency
    Promise.all(filePaths.slice(0, 30).map(fp =>
      deps.mcpClient.call('get_file_history', { repo_slug: repo, file_path: fp, limit: 50 })
        .then((h: any) => ({ path: fp, count: h?.commits?.length ?? 0 }))
        .catch(() => ({ path: fp, count: 0 }))
    )),

    // 9. search_architecture for financial rules
    deps.mcpClient.call('search_architecture', {
      query: 'transaction financial debit credit rollback maturity',
      aspect: 'standards',
    }).catch(() => null),

    // 10. lookup_glossary for financial terms
    deps.mcpClient.call('lookup_glossary', {
      terms: ['STP', 'DLT', 'FD', 'TD', 'NAV', 'T+2', 'ACT/360', 'FOLLOWING'],
    }).catch(() => null),

    // 11. analyze_graph_health for dead code
    deps.mcpClient.call('analyze_graph_health', { repo_slug: repo }).catch(() => null),

    // 12. get_api_dependencies for cross-service mapping
    deps.mcpClient.call('get_api_dependencies', { repo, direction: 'both' }).catch(() => null),
  ]);

  // ── Assemble into FinancialGatheredContext ──
  return assembleContext({
    projectInfo, repoConfig, centralityResult, testCoverage,
    fileImports, relatedPRs, bugTickets, fileHistory,
    archRules, glossary, graphHealth, apiDeps,
    graphClient: deps.graphClient, repo, filePaths,
  });
}
```

## 13. Phase 4: Graph Verification + SCIP + Severity Boosting

### 13.1 Severity Boosting

```typescript
export function boostSeverityFromContext(
  findings: FinancialFinding[],
  ctx: FinancialGatheredContext
): FinancialFinding[] {
  return findings.map(f => {
    const boosted = { ...f };
    const reasons: string[] = [];

    // High centrality → boost
    const centrality = ctx.highCentralityMethods.get(f.function);
    if (centrality && centrality > 0.02) {
      if (boosted.severity === 'moderate') boosted.severity = 'major';
      reasons.push(`High-traffic (centrality ${centrality.toFixed(3)})`);
    }

    // No test coverage → boost
    if (ctx.untestedFinancialFiles.includes(f.filePath)) {
      if (boosted.severity === 'minor') boosted.severity = 'moderate';
      reasons.push('No test coverage');
    }

    // Frequently changed + no test → boost
    const changes = ctx.fileChangeFrequency.get(f.filePath) ?? 0;
    if (changes > 10 && !ctx.testedFiles.has(f.filePath)) {
      if (boosted.severity === 'moderate') boosted.severity = 'major';
      reasons.push(`${changes} changes in 6 months, untested`);
    }

    // Cross-repo consumers → boost (blast radius)
    const consumers = ctx.crossRepoConsumers.get(f.function);
    if (consumers && consumers.length > 0) {
      if (boosted.severity === 'moderate') boosted.severity = 'major';
      reasons.push(`${consumers.length} cross-repo consumer(s)`);
    }

    // Past bug tickets → boost confidence
    const bugs = ctx.relatedBugTickets.filter(t =>
      t.summary?.toLowerCase().includes(f.function.toLowerCase())
    );
    if (bugs.length > 0) {
      boosted.confidence = 'likely';
      reasons.push(`Bug ticket: ${bugs[0].issue_key}`);
    }

    // Architecture rule violation → boost
    if (ctx.archRules?.some((r: string) => r.toLowerCase().includes(f.check.replace(/_/g, ' ')))) {
      reasons.push('Violates documented architecture standard');
    }

    // Dead code → suppress
    if (ctx.deadMethods?.has(f.function)) {
      boosted.confidence = 'unlikely';
      boosted.evidence += ' [Dead code — no live callers]';
    }

    if (reasons.length > 0) boosted.evidence += ` [Boosted: ${reasons.join('; ')}]`;
    return boosted;
  });
}
```

## 14. Cross-Service Transaction Tracing

```typescript
export async function traceFinancialAPIFlows(
  deps: ServerDeps,
  repo: string,
  findings: FinancialFinding[]
): Promise<FinancialFinding[]> {
  const additional: FinancialFinding[] = [];

  try {
    const surface = await deps.mcpClient.call('get_api_surface', { repo });
    const financialEndpoints = (surface?.endpoints ?? []).filter((ep: any) =>
      isFinanciallySensitive(ep.path + ' ' + (ep.handler ?? ''))
    );

    for (const ep of financialEndpoints) {
      const flow = await deps.mcpClient.call('trace_api_flow', {
        endpoint_path: ep.path,
        http_method: ep.http_method ?? 'POST',
      });

      const downstream = flow?.downstream ?? [];

      if (downstream.length > 0) {
        // Check if the handler method writes to DB (from existing findings)
        const handlerFindings = findings.filter(f =>
          f.function === ep.handler && f.check.startsWith('multi_write')
        );

        if (handlerFindings.length > 0) {
          additional.push({
            check: 'cross_service_partial_commit',
            severity: 'major',
            filePath: handlerFindings[0].filePath,
            line: handlerFindings[0].line,
            function: ep.handler,
            evidence: `${ep.http_method} ${ep.path} writes locally AND calls ` +
                      `${downstream.map((d: any) => `${d.http_method} ${d.repo}${d.path}`).join(', ')}. ` +
                      `Local commit + downstream failure = cross-service inconsistency.`,
            recommendation: 'Use outbox pattern: write intent to outbox table in same TX, ' +
                            'process outbox asynchronously. Or implement saga with compensation.',
            confidence: 'likely',
          });
        }
      }
    }
  } catch {
    // API tracing unavailable
  }

  return [...findings, ...additional];
}
```

-----

# Part E — Layer 3 LLM

## 15. LLM Prompt Architecture

Every LLM call follows this 6-block structure:

```
SYSTEM: Financial auditor role + business lines + output JSON format
USER:
  ├── <class_context>      — class metadata, annotations, fields (from ClassContext)
  ├── <graph_context>      — callers, callees, centrality, test status (from GatheredContext)
  ├── <project_context>    — TX config, retry libs, business-day libs (from get_repo_config)
  ├── <known_findings>     — AST findings for this method (anti-duplication)
  ├── <similar_past_fixes> — related PRs (from batch_search, few-shot examples)
  ├── <code>               — method body (from assemble_file_context or window)
  └── QUESTION             — check-specific verification question
```

## 16. System Prompt

```typescript
const SYSTEM_PROMPT = `You are a senior financial software auditor specializing in OCBC banking systems (Bonds, Equity, Loans, Deposits, Funds).

You review code for transaction safety: unbalanced ledgers, partial commits, silent failures, missed lifecycle events.

RULES:
1. Only flag genuine risks — intentional patterns (fee collection, saga orchestration, interest accrual) are NOT bugs
2. Consider the full call chain — single-legged methods may have counterpart in a caller or downstream service
3. Consider the framework — Spring @Transactional provides automatic rollback; inherited @Transactional counts
4. Consider the business line — loan disbursement without immediate repayment is normal
5. If uncertain, say "possible" not "likely"
6. NEVER invent file paths or symbol names — only reference what you see in the code

TERMINOLOGY:
${glossaryBlock}

OUTPUT — ONLY JSON array, no markdown:
[{
  "is_genuine_risk": boolean,
  "check_type": "check name from AST finding",
  "severity": "minor"|"moderate"|"major"|"dismiss",
  "reason": "1-2 sentence explanation",
  "missed_by_ast": boolean,
  "additional_finding": null | {
    "check": "string", "line": number,
    "evidence": "string", "recommendation": "string"
  }
}]`;
```

## 17. Per-Check Prompts

### Check 1 — Debit/Credit Verification

```typescript
function buildCheck1Prompt(
  window: AnalysisWindow, finding: FinancialFinding,
  ctx: FinancialGatheredContext, relatedPRs: PRContext[]
): string {
  const callers = ctx.financialMethodCallers.get(finding.function) ?? [];
  const callees = ctx.financialMethodCallees.get(finding.function) ?? [];
  const centrality = ctx.highCentralityMethods.get(finding.function);
  const coChange = ctx.coChangePartners.get(window.classContext.filePath) ?? [];

  return `
<class_context>
File: ${window.classContext.filePath} | Class: ${window.classContext.className}
Annotations: ${window.classContext.classAnnotations.join(', ') || 'none'}
Fields: ${window.classContext.fieldDeclarations.slice(0, 10).join('; ')}
Implements: ${window.classContext.implementedInterfaces.join(', ') || 'none'}
</class_context>

<graph_context>
Callers of '${finding.function}' (SCIP-verified): ${callers.slice(0, 8).join(', ') || 'none'}
Callees: ${callees.slice(0, 8).join(', ') || 'none'}
${centrality ? `Centrality: ${centrality.toFixed(3)} (${centrality > 0.02 ? 'HIGH-TRAFFIC' : 'normal'})` : ''}
Co-change files: ${coChange.slice(0, 5).join(', ') || 'none'}
Test coverage: ${ctx.testedFiles.has(finding.filePath) ? 'YES' : 'NO'}
Changes (6mo): ${ctx.fileChangeFrequency.get(finding.filePath) ?? 0}
</graph_context>

<known_findings>
${finding.check}: ${finding.evidence}
</known_findings>

${relatedPRs.length > 0 ? `<similar_past_fixes>
${relatedPRs.slice(0, 2).map(pr =>
  `PR-${pr.prNumber}: "${pr.title}" — ${pr.coreProblem ?? 'N/A'}`
).join('\n')}
</similar_past_fixes>` : ''}

<code>
${window.methods.map(m => m.text).join('\n\n')}
</code>

QUESTION: AST detected ${finding.check === 'single_legged_debit' ? 'outflow without inflow' : 'inflow without outflow'} in '${finding.function}'.

Analyze:
A) Genuine single-legged bug (money/units move without counterpart)?
B) Intentional (fee collection, interest accrual, saga where counterpart is in another service)?
C) Architectural split (counterpart in a caller listed above)?

Also check for issues AST missed:
- Amount mismatch between debit/credit (different variables)
- Currency mismatch between paired operations
- Missing null/zero check on amount before financial operation
- Race condition on balance check-then-debit (read-then-write without lock)`;
}
```

### Check 2 — Transaction Rollback Verification

```typescript
function buildCheck2Prompt(
  window: AnalysisWindow, finding: FinancialFinding, ctx: FinancialGatheredContext
): string {
  return `
<class_context>
File: ${window.classContext.filePath} | Class: ${window.classContext.className}
Annotations: ${window.classContext.classAnnotations.join(', ') || 'none'}
Class-level @Transactional: ${window.classContext.hasClassTransactional ? 'YES' : 'NO'}
Repository fields: ${window.classContext.fieldDeclarations
  .filter(f => /Repository|DAO|jdbcTemplate|entityManager/.test(f)).join('; ') || 'none'}
</class_context>

<project_context>
Spring TX dependency: ${ctx.hasSpringTx ? 'YES' : 'NO'}
TX isolation: ${ctx.txIsolationLevel ?? 'default (READ_COMMITTED)'}
Retry library: ${ctx.hasRetryLib ? 'YES' : 'NO'}
Cross-service calls: ${ctx.crossServiceDeps.length > 0 ? ctx.crossServiceDeps.join(', ') : 'none detected'}
</project_context>

<known_findings>
${finding.check}: ${finding.evidence}
</known_findings>

<code>
${window.methods.map(m => m.text).join('\n\n')}
</code>

QUESTION: AST detected '${finding.check}' in '${finding.function}'.

Analyze:
1. Are the writes actually interdependent (need atomicity)?
2. Is @Transactional inherited from parent class or AOP proxy?
3. Does Spring Data default save() run in its own TX — is multi-write still risky?
4. For save-then-throw: is the throw actually reachable after the save?

Additional issues to check:
- @Transactional on private method (Spring proxy won't intercept)
- Self-invocation bypassing proxy (this.otherMethod() inside @Transactional)
- Long-running operation inside TX boundary (holding DB locks)
- @Transactional(propagation=REQUIRES_NEW) causing unexpected independent commits`;
}
```

### Check 3 — Unclosed Conditions Verification

```typescript
function buildCheck3Prompt(
  window: AnalysisWindow, finding: FinancialFinding, ctx: FinancialGatheredContext
): string {
  // Try to resolve enum definition via SCIP
  let enumContext = '';
  if (finding.check === 'enum_switch_missing_cases') {
    const enumName = finding.evidence.match(/enum '(\w+)'/)?.[1];
    if (enumName && ctx.resolvedEnums?.has(enumName)) {
      const def = ctx.resolvedEnums.get(enumName)!;
      enumContext = `\n<enum_definition>\nFile: ${def.file}\nenum ${enumName} { ${def.values.join(', ')} }\n</enum_definition>`;
    }
  }

  return `
<class_context>
File: ${window.classContext.filePath} | Class: ${window.classContext.className}
Business line: ${detectBusinessLine(window.classContext.filePath)}
</class_context>
${enumContext}

<known_findings>
${finding.check}: ${finding.evidence}
</known_findings>

<code>
${window.methods.map(m => m.text).join('\n\n')}
</code>

QUESTION: AST detected unclosed condition in '${finding.function}'.

Analyze:
1. Are the conditions EXHAUSTIVE for the business domain?
2. Could a new value be added that silently falls through?
3. Is there upstream validation guaranteeing only handled values reach here?
4. Would a missing branch cause financial impact (wrong amount, wrong account, skipped processing)?

For enum: list ALL values and confirm handled vs missing.
For if/else-if: determine if conditions are logically exhaustive.`;
}
```

### Check 5 — Maturity/Expiry Verification

```typescript
function buildCheck5Prompt(
  window: AnalysisWindow, finding: FinancialFinding, ctx: FinancialGatheredContext
): string {
  return `
<class_context>
File: ${window.classContext.filePath} | Class: ${window.classContext.className}
Annotations: ${window.classContext.classAnnotations.join(', ') || 'none'}
Business line: ${detectBusinessLine(window.classContext.filePath)}
</class_context>

<project_context>
Business-day library: ${ctx.hasBusinessDayLib ? 'YES' : 'NO'}
Scheduler config: ${JSON.stringify(ctx.schedulerConfig).slice(0, 200) || 'none'}
Retry library: ${ctx.hasRetryLib ? 'YES' : 'NO'}
</project_context>

<known_findings>
${finding.check}: ${finding.evidence}
</known_findings>

<code>
${window.methods.map(m => m.text).join('\n\n')}
</code>

QUESTION: AST detected maturity/expiry issue in '${finding.function}'.

${finding.check === 'maturity_no_equals_check' ?
`Does the comparison handle the exact-match case? Would !isAfter() fix it?` :
finding.check === 'maturity_no_business_day' ?
`Is business-day adjustment done upstream? Check imports for CalendarService.` :
finding.check.startsWith('rollover_') ?
`Is rollover split across two methods called by an orchestrator? Is it atomic?` :
`Is this @Scheduled method idempotent? What happens to failed items?`}

Additional checks:
- Timezone mismatch (UTC vs SGT) in date comparison
- maturityDate DATE vs TIMESTAMP inconsistency
- Day-count convention (ACT/360, ACT/365, 30/360) not accounted for
- Rollover creating new instrument with wrong tenor/rate from old one`;
}
```

## 18. LLM Orchestration

```typescript
export async function runLLMVerification(
  findings: FinancialFinding[],
  windows: AnalysisWindow[],
  ctx: FinancialGatheredContext,
  deps: ServerDeps,
  repo: string
): Promise<FinancialFinding[]> {
  const needsVerification = findings.filter(f => f.confidence === 'possible');
  const decided = findings.filter(f => f.confidence !== 'possible');

  if (needsVerification.length === 0) return findings;

  // Resolve glossary terms for system prompt
  const glossary = ctx.glossaryTerms ?? {};
  const glossaryBlock = Object.entries(glossary)
    .map(([k, v]) => `${k}: ${v}`)
    .join('\n') || 'Standard banking terms';

  const systemPrompt = SYSTEM_PROMPT.replace('${glossaryBlock}', glossaryBlock);

  // Group by file+function for batched calls
  const groups = new Map<string, FinancialFinding[]>();
  for (const f of needsVerification) {
    const key = `${f.filePath}::${f.function}`;
    if (!groups.has(key)) groups.set(key, []);
    groups.get(key)!.push(f);
  }

  const verified: FinancialFinding[] = [...decided];
  const limit = pLimit(2); // 2 concurrent Qwen calls

  const tasks = [...groups.entries()].map(([key, groupFindings]) =>
    limit(async () => {
      const window = windows.find(w =>
        w.file.path === groupFindings[0].filePath &&
        w.methods.some(m => extractFunctionName(m, w.file.language) === groupFindings[0].function)
      );
      if (!window) { verified.push(...groupFindings); return; }

      // Get related PRs via batch_search
      const relatedPRs = await findRelatedPRs(groupFindings[0], deps, repo);

      // Build prompt
      const prompt = selectPromptBuilder(groupFindings[0].check)(
        window, groupFindings[0], ctx, relatedPRs
      );

      try {
        const response = await deps.qwen.generateAnalysis(prompt, {
          systemPrompt, maxTokens: 1500, temperature: 0.1,
        });

        const llmResults = parseLLMResponse(response);

        // Validate LLM output with hallucination checker
        for (const result of llmResults) {
          if (result.additional_finding) {
            const hallCheck = await deps.mcpClient.call('check_ai_hallucinations', {
              code: JSON.stringify(result.additional_finding),
              repo_slug: repo,
            });
            if (hallCheck?.issues?.length > 0) {
              result.additional_finding = null; // LLM hallucinated
            }
          }
        }

        for (const f of groupFindings) {
          const match = llmResults.find(r => r.check_type === f.check);
          if (match) {
            if (match.severity === 'dismiss') f.confidence = 'unlikely';
            else {
              f.severity = match.severity as any;
              f.confidence = match.is_genuine_risk ? 'likely' : 'possible';
              f.evidence += ` [LLM: ${match.reason}]`;
            }
            if (match.additional_finding && match.missed_by_ast) {
              verified.push({
                check: match.additional_finding.check as FinancialCheckType,
                severity: 'moderate', filePath: f.filePath,
                line: match.additional_finding.line, function: f.function,
                evidence: `[LLM-discovered] ${match.additional_finding.evidence}`,
                recommendation: match.additional_finding.recommendation,
                confidence: 'possible',
              });
            }
          }
          verified.push(f);
        }
      } catch (err) {
        console.error(`[FinancialLLM] Failed for ${key}:`, err);
        verified.push(...groupFindings);
      }
    })
  );

  await Promise.allSettled(tasks);
  return verified;
}
```

-----

# Part F — Integration & Execution

## 19. 6-Phase Orchestrator

```typescript
export async function runFinancialIntegrityChecks(
  files: LoadedFile[],
  deps: ServerDeps,
  repo: string,
  config: ProjectConfig
): Promise<FinancialIntegrityReport> {
  const t0 = performance.now();

  // ═══ Phase 0: Pre-screen + Window ═══
  const financialFiles = files.filter(f => !f.isTestFile && isFinancialFile(f));
  const windows = financialFiles.flatMap(createAnalysisWindows);
  const t1 = performance.now();

  // ═══ Phase 1: RAG Gathering (parallel, all 12 sources) ═══
  const ctx = await gatherFinancialContext(deps, repo, financialFiles);
  const t2 = performance.now();

  // ═══ Phase 2: AST Analysis (per window, 4x parallel) ═══
  let findings: FinancialFinding[] = [];
  const astLimit = pLimit(4);
  const astResults = await Promise.all(
    windows.map(w => astLimit(async () => [
      ...analyzeDebitCreditWindow(w, ctx),
      ...analyzeTransactionRollbackWindow(w, ctx),
      ...analyzeUnclosedConditionsWindow(w),
      ...analyzeErrorHandlingWindow(w, ctx),
      ...analyzeMaturityExpiryWindow(w, ctx),
    ]))
  );
  findings = astResults.flat();
  const t3 = performance.now();

  // ═══ Phase 3: Dedup ═══
  findings = deduplicateFindings(findings);

  // ═══ Phase 4: Graph/SCIP/API Verification + Boosting ═══
  findings = await verifyWithCallChainSCIP(
    findings.filter(f => f.needsGraphVerification), deps, repo
  ).then(verified => [...findings.filter(f => !f.needsGraphVerification), ...verified]);

  findings = boostSeverityFromContext(findings, ctx);
  findings = reduceFalsePositives(findings, ctx);
  findings = await traceFinancialAPIFlows(deps, repo, findings);
  const t4 = performance.now();

  // ═══ Phase 5: LLM Verification ═══
  findings = await runLLMVerification(findings, windows, ctx, deps, repo);
  findings = findings.filter(f => f.confidence !== 'unlikely');
  const t5 = performance.now();

  // ═══ Phase 6: Score ═══
  const DEDUCTIONS = { minor: 0.5, moderate: 1.0, major: 2.0 };
  let score = 10;
  for (const f of findings) score -= DEDUCTIONS[f.severity];
  score = Math.max(0, Math.min(10, score));

  return {
    findings,
    score: Math.round(score * 10) / 10,
    rating: score >= 8.5 ? 'Healthy' : score >= 6.0 ? 'Watch' : 'Critical',
    stats: {
      filesScanned: files.length,
      financialFiles: financialFiles.length,
      analysisWindows: windows.length,
      checksRun: 5,
      timings: {
        windowMs: Math.round(t1 - t0),
        gatherMs: Math.round(t2 - t1),
        astMs: Math.round(t3 - t2),
        verifyMs: Math.round(t4 - t3),
        llmMs: Math.round(t5 - t4),
        totalMs: Math.round(t5 - t0),
      },
      llmCallsMade: findings.filter(f => f.evidence.includes('[LLM')).length,
      scipVerified: findings.filter(f => f.evidence.includes('SCIP')).length,
      graphBoosted: findings.filter(f => f.evidence.includes('[Boosted')).length,
      crossServiceFindings: findings.filter(f => f.check === 'cross_service_partial_commit').length,
    },
    context: {
      relatedBugTickets: ctx.relatedBugTickets.map(t => t.issue_key),
      untestedFinancialFiles: ctx.untestedFinancialFiles.length,
      highCentralityMethods: [...ctx.highCentralityMethods.entries()]
        .filter(([, c]) => c > 0.02).map(([name, c]) => ({ name, centrality: c })),
      archRulesChecked: ctx.archRules?.length ?? 0,
    },
  };
}
```

## 20. Shared Types

```typescript
export type FinancialCheckType =
  // Check 1
  | 'single_legged_debit' | 'single_legged_credit'
  | 'loop_debit_credit' | 'duplicate_debit'
  // Check 2
  | 'multi_write_no_transaction' | 'multi_write_single_repo_no_transaction'
  | 'commit_without_rollback' | 'save_then_throw'
  | 'readonly_transaction_with_writes'
  | 'multi_mutation_no_error_handling' | 'optimistic_update_no_rollback'
  | 'cross_service_partial_commit'          // NEW — from trace_api_flow
  // Check 3
  | 'if_chain_no_else' | 'switch_no_default' | 'enum_switch_missing_cases'
  // Check 4
  | 'empty_catch' | 'catch_log_no_rethrow' | 'catch_returns_null'
  | 'system_exit_in_service' | 'resource_no_finally'
  | 'async_void_no_handler' | 'broad_catch_in_financial'
  // Check 5
  | 'maturity_no_equals_check' | 'maturity_no_business_day'
  | 'rollover_missing_open_leg' | 'rollover_missing_close_leg'
  | 'scheduled_maturity_no_retry' | 'scheduled_maturity_no_alert'
  // Cross-repo
  | 'cross_repo_inconsistency';             // NEW — from compare_implementations
```

**29 check types** (27 base + 2 new from tool integration).

## 21. Standalone CLI

```bash
# Single repo
npx tsx src/scripts/run_financial_checks.ts BMT/ms-bonds-trading

# All backend repos
npx tsx src/scripts/run_financial_checks.ts --all --filter=backend

# Specific file + LLM verification
npx tsx src/scripts/run_financial_checks.ts BMT/ms-loans --file=src/.../DisbursementService.java --llm

# Cross-repo consistency check
npx tsx src/scripts/run_financial_checks.ts --cross-repo --repos=BMT/ms-bonds,BMT/ms-equity,BMT/ms-loans
```

## 22. Health Score Integration (9th Dimension)

```typescript
const DIMENSION_WEIGHTS = {
  spring_boot: { financial_integrity: 1.3 },
  react: { financial_integrity: 0.6 },
};
```

Only scored for repos classified as `backend` or `fullstack` by ProjectAnalyzer.

## 23. Implementation Schedule (7 Days)

```
Day 1: Check 3 + Check 4 + Shared Infrastructure
  ├── types.ts — 29 FinancialCheckType values
  ├── helpers.ts — isFinancialFile (all business lines), isFinanciallySensitive
  ├── unclosed_conditions.ts — if chains, switch/default, enum (SCIP type resolution)
  ├── error_handling.ts — empty catch, System.exit, @Async, broad catch
  └── Unit tests with mock AST trees

Day 2: Check 2 + Batch Processing
  ├── transaction_rollback.ts — @Transactional, manual commit, save-then-throw, readOnly
  ├── transaction_rollback_ts.ts — multi-mutation, optimistic update
  ├── batch_processor.ts — createAnalysisWindows, splitGiantMethodForLLM
  ├── dedup.ts — cross-window deduplication
  └── Test batch processing on 2000+ line service files

Day 3: Check 1 + SCIP/Graph Verification
  ├── debit_credit.ts — all 50+ patterns per side, loop detection, idempotency
  ├── SCIP verification — get_scip_callers/callees for counterpart resolution
  ├── severity_booster.ts — centrality, test coverage, change frequency, blast radius
  └── Test single-legged detection + graph downgrade

Day 4: Check 5 + RAG Gathering
  ├── maturity_expiry.ts — date comparison, business day, rollover, @Scheduled retry
  ├── gather.ts — 12-source parallel gathering (all MCP tools + DB)
  ├── Cross-service TX tracing via trace_api_flow + get_api_dependencies
  └── Wire pre-screening with get_file_skeleton

Day 5: LLM Layer
  ├── prompts/ — system.ts + per-check prompt builders (5 files)
  ├── llm_orchestrator.ts — batched LLM verification, concurrency=2
  ├── llm_parser.ts — JSON response parser + check_ai_hallucinations validation
  └── Test LLM on 10+ ambiguous findings

Day 6: MCP Tool + CLI + Integration
  ├── check_financial_integrity MCP tool — on-demand checks from Windsurf
  ├── CLI runner — single repo, all repos, cross-repo, specific file modes
  ├── 6-phase orchestrator (index.ts)
  ├── Wire into health score as 9th dimension
  └── Cross-repo consistency via compare_implementations

Day 7: Testing + Calibration
  ├── Run against 5+ repos across all business lines (bonds, equity, loans, deposits, funds)
  ├── Validate batch processing on large service classes
  ├── Measure false positive rate per check per business line (target: <15%)
  ├── Benchmark total runtime per repo (target: <3 minutes without LLM, <8 with)
  ├── Calibrate severity boosting thresholds
  └── Dashboard rendering for financial integrity section
```

## 24. File Structure

```
src/lib/health_score/analyzers/financial_integrity/
  # ── Core ──
  index.ts                    — 6-phase orchestrator
  types.ts                    — 29 check types + interfaces
  helpers.ts                  — isFinancialFile, isFinanciallySensitive, detectBusinessLine

  # ── Batch Processing ──
  batch_processor.ts          — createAnalysisWindows, splitGiantMethodForLLM
  dedup.ts                    — cross-window deduplication

  # ── AST Checks ──
  debit_credit.ts             — Check 1: paired operations (all business lines)
  transaction_rollback.ts     — Check 2: Java TX integrity
  transaction_rollback_ts.ts  — Check 2: TypeScript mutations
  unclosed_conditions.ts      — Check 3: missing else/default/enum cases
  error_handling.ts           — Check 4: empty catch, exit, resources, @Async
  maturity_expiry.ts          — Check 5: maturity, business day, rollover, retry

  # ── RAG + Graph + SCIP ──
  gather.ts                   — 12-source parallel gathering
  severity_booster.ts         — centrality/test/change/blast-radius boosting
  scip_verify.ts              — SCIP callers/callees + cross-service API verification
  cross_service.ts            — trace_api_flow + get_api_dependencies TX tracing
  cross_repo.ts               — compare_implementations consistency checks

  # ── LLM Layer ──
  llm_orchestrator.ts         — batched LLM verification
  prompts/
    system.ts                 — shared system prompt with glossary
    check1_debit_credit.ts    — paired operation verification
    check2_transaction.ts     — TX boundary verification
    check3_conditions.ts      — exhaustiveness verification
    check4_errors.ts          — error handling verification
    check5_maturity.ts        — maturity/expiry verification
  llm_parser.ts               — JSON parser + hallucination guard

  # ── MCP Tool ──
  mcp_tool.ts                 — check_financial_integrity tool handler

src/scripts/
  run_financial_checks.ts     — standalone CLI (single, all, cross-repo modes)
```

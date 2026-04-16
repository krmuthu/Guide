# Project Sentinel — Master Implementation Plan

## HP NonStop COBOL → Java Conversion Assurance System

**Version:** 4.0 — Master Plan
**Date:** April 15, 2026
**Status:** Approved for Development
**Runtime:** RHEL 128-core · User-space only · No sudo · No external API dependencies
**Source:** HP NonStop (Tandem) COBOL — Guardian/OSS, NMCOBOL, COBOL85
**Target:** Java (converted via Windsurf IDE)
**Owner:** Project Apollo Team

-----

## Table of Contents

1. [The Problem We Are Solving](#1-the-problem-we-are-solving)
1. [Overall Goal & Success Definition](#2-overall-goal--success-definition)
1. [What Makes This Hard — HP NonStop Context](#3-what-makes-this-hard--hp-nonstop-context)
1. [Solution Strategy](#4-solution-strategy)
1. [Architecture Overview](#5-architecture-overview)
1. [Technology Stack — Final Decisions](#6-technology-stack--final-decisions)
1. [Database Schema — Single Source of Truth](#7-database-schema--single-source-of-truth)
1. [Phase 1 — COBOL Inventory (ProLeap)](#8-phase-1--cobol-inventory-proleap)
1. [Phase 2 — Java Inventory (SCIP + Tree-sitter)](#9-phase-2--java-inventory-scip--tree-sitter)
1. [Phase 3 — Deterministic Mapping](#10-phase-3--deterministic-mapping)
1. [Phase 3.5 — Stub Detection](#11-phase-35--stub-detection)
1. [Phase 4 — Dependency Validation (Recursive CTEs)](#12-phase-4--dependency-validation-recursive-ctes)
1. [Phase 5 — Behavioral Validation (Golden File Parity)](#13-phase-5--behavioral-validation-golden-file-parity)
1. [Phase 6 — Semantic Embedding Layer (Optional Flag)](#14-phase-6--semantic-embedding-layer-optional-flag)
1. [Phase 7 — Gap Analysis Dashboard](#15-phase-7--gap-analysis-dashboard)
1. [Phase 8 — Reporting & Review Workflow](#16-phase-8--reporting--review-workflow)
1. [Phase 9 — Continuous Governance](#17-phase-9--continuous-governance)
1. [HP NonStop — Construct Handling Reference](#18-hp-nonstop--construct-handling-reference)
1. [Feature Flags](#19-feature-flags)
1. [Repository Structure](#20-repository-structure)
1. [Configuration & Environment](#21-configuration--environment)
1. [Testing Strategy](#22-testing-strategy)
1. [Timeline & Milestones](#23-timeline--milestones)
1. [Risk Register](#24-risk-register)
1. [Decommission Gate — The Final Checkpoint](#25-decommission-gate--the-final-checkpoint)
1. [Appendices](#26-appendices)

-----

## 1. The Problem We Are Solving

We converted HP NonStop (Tandem) COBOL applications to Java using Windsurf IDE. The conversion was automated. Three things can go wrong — and all three are invisible without a deliberate verification system:

**Problem 1: Missing conversions.** A COBOL paragraph exists, Windsurf created a Java method with the right name and signature, but the method body contains only comments and a `return BigDecimal.ZERO` (or `return false`, `return null`, `return ""`). The method is structurally present but functionally empty. Standard code review often misses this because the naming looks correct.

**Problem 2: Incomplete conversions.** The Java method has some logic but not all of it. A COBOL `EVALUATE` with three branches became a Java `switch` with one branch. The ACT/360 path works; ACT/365 and 30/360 are silently ignored. The COBOL had `COMPUTE WS-FEE-AMOUNT = WS-PAY-AMOUNT * WS-FEE-PCT`; the Java method never computes the fee.

**Problem 3: NonStop-specific constructs not converted.** HP NonStop COBOL uses `ENTER TAL` and `ENTER C` to call TAL or C routines at the hardware level. It uses Enscribe Guardian files (`$DATA2.ACCTDB.ACCTFILE`) instead of standard I/O. It uses TMF transactions (`TMF_BEGIN_`, `TMF_COMMIT_`) for fault-tolerant transaction management. These have no direct ANSI COBOL equivalent and no standard Java conversion path. Windsurf may have left them as comments or stubs.

**The risk if these go undetected:**

- Programs pass structural review (names match)
- Programs pass basic smoke tests (no runtime errors)
- Programs are deployed to production
- Business logic silently produces wrong results — wrong interest rates, wrong payment limits, missing fees, no transaction rollback on failure

This is not a theoretical risk. It is the most common failure mode in automated COBOL-to-Java migrations.

**Project Sentinel exists to make these problems impossible to miss.**

-----

## 2. Overall Goal & Success Definition

### 2.1 The Goal

**Prove, with auditable evidence, that every COBOL construct has been faithfully converted to Java — before any COBOL program is decommissioned.**

Faithful conversion means:

1. Every structural element exists in Java (method, class, field)
1. Every structural element is actually implemented (not stubbed)
1. Dependency chains are preserved (if COBOL paragraph A calls B, Java method A’ calls B’)
1. Given identical inputs, COBOL and Java produce identical outputs

### 2.2 Success Criteria — Per Program

A COBOL program may only be decommissioned when ALL of the following pass:

|Criterion                |Target                                         |Measured By                  |
|-------------------------|-----------------------------------------------|-----------------------------|
|Structural coverage      |100% paragraphs mapped                         |traceability_map             |
|Stub-free                |0 confirmed stubs                              |java_constructs.is_stub      |
|Dependency integrity     |0 broken call chains                           |dependency_validation_results|
|Behavioral parity        |100% golden file tests pass                    |behavioral_results           |
|ENTER TAL/C verified     |All calls have Java equivalents + parity tested|graph_edges REPLACED_BY      |
|Guardian I/O verified    |All file patterns preserved                    |dependency_validation_results|
|TMF transactions verified|All transaction boundaries mapped              |nonstop_metadata             |
|Zero open drift alerts   |No unacknowledged changes post-mapping         |drift_alerts                 |
|Three sign-offs          |Tech lead + business owner + Legal             |decommission_log             |

### 2.3 Success Criteria — System Level

|Metric                    |Target                             |
|--------------------------|-----------------------------------|
|Composite coverage score  |≥ 95% overall                      |
|Structural layer          |≥ 95% individually                 |
|Dependency layer          |≥ 95% individually                 |
|Behavioral layer          |≥ 95% individually                 |
|Mean time to detect a stub|< 1 hour after Java commit         |
|Audit trail completeness  |Every decision has who + when + why|

### 2.4 What Sentinel Is Not

- Not a code quality tool (no linting, no style checking)
- Not a testing framework (no test generation beyond edge cases)
- Not a migration tool (conversion already done by Windsurf)
- Not a permanent production system (decommissioned when last COBOL program is retired)

-----

## 3. What Makes This Hard — HP NonStop Context

Standard COBOL-to-Java traceability tools assume IBM mainframe COBOL. HP NonStop COBOL is a different dialect with these unique constructs:

### 3.1 ENTER TAL / ENTER C

```cobol
ENTER TAL "RATE_TABLE_LOOKUP"
    USING WS-RATE-TABLE-PTR
          WS-CURRENCY-CODE
          WS-EFFECTIVE-DATE
          WS-RATE-RESULT
          WS-RETURN-STATUS.
ENTER COBOL.
```

This calls a TAL (Transaction Application Language) routine at the native instruction level. No COBOL parser can follow this call. No ANSI standard covers it. The Java conversion must replace it with a pure Java implementation or JNI wrapper — and prove they are equivalent.

### 3.2 Guardian File System

```cobol
SELECT ACCT-FILE ASSIGN TO "$DATA2.ACCTDB.ACCTFILE".
...
OPEN OUTPUT ACCT-FILE.
LOCKFILE ACCT-FILE TIME LIMIT 30.
READ ACCT-FILE KEY WS-ACCT-KEY INVALID KEY PERFORM 900-ERROR.
WRITE ACCT-REC.
```

Guardian uses physical Enscribe file paths (`$VOL.SUBVOL.FILE`), record-level locking (LOCKFILE), keyed access, and NonStop-specific file status codes. Java conversion must use JDBC, JPA, or standard I/O and preserve the locking semantics.

### 3.3 TMF Transactions

```cobol
ENTER TAL "TMF_BEGIN_".
ENTER COBOL.
...
ENTER TAL "TMF_COMMIT_".
ENTER COBOL.
```

NonStop Transaction Management Facility — the fault-tolerant transaction system. Java must use JTA, Spring `@Transactional`, or equivalent. Transaction isolation and rollback behaviour must match exactly for financial correctness.

### 3.4 Pathway Server Classes

Inter-process communication via Pathway — the NonStop TP monitor. Java conversion typically uses REST APIs or message queues, but the delivery guarantees differ.

### 3.5 COMP-3 Packed Decimal

```cobol
05 WS-DAILY-RATE    PIC S9(5)V9(8)  COMP-3.
05 WS-PRINCIPAL     PIC S9(11)V9(2) COMP-3.
```

COMP-3 is packed binary decimal with exact precision. Java `double` is IEEE 754 floating point — fundamentally different arithmetic. Java conversion must use `BigDecimal` throughout, with the correct scale and rounding mode. This is the most common source of silent numeric errors.

### 3.6 Why Standard Tools Fail

- **Tree-sitter COBOL grammars:** Based on ANSI COBOL85. No Tandem format support. Cannot parse `ENTER TAL`. Parse failures on Guardian file syntax.
- **GnuCOBOL:** Cannot compile NonStop COBOL. Cannot run behavioral tests.
- **IBM-specific tools:** Built for z/OS COBOL. Different dialects, different runtime.
- **Generic code embedding models:** None trained on COBOL. Most trained on Python, Java, JavaScript.

-----

## 4. Solution Strategy

### 4.1 Four-Layer Defense-in-Depth

No single technique is sufficient. We use four independent layers, each catching what the previous layers miss:

```
┌────────────────────────────────────────────────────────────────┐
│                   COMPOSITE SCORE                               │
│   0.25×Structural + 0.30×Dependency + 0.45×Behavioral          │
│         (when embeddings enabled: 0.20+0.20+0.25+0.35)         │
│              Decommission gate: each layer ≥ 95%                │
└──────────┬──────────────┬──────────────┬───────────────────────┘
           │              │              │
        Layer 1        Layer 2        Layer 3        Layer 4
        Structural     Dependency     Behavioral     Semantic
        + Stub         (Recursive     (Golden        (Optional
        Detection      CTEs)          File Parity)   Embeddings)
           │              │              │              │
        Catches:       Catches:       Catches:       Catches:
        Missing        Broken         Logic          Refactored
        methods        chains         errors         logic
        Empty          Missing        COMP-3         Renamed
        stubs          fan-out        precision      constructs
        Convention     Data flow      I/O format     Fuzzy
        mismatches     gaps           diffs          matches
```

### 4.2 Windsurf-Specific Insight

Because conversion was done by Windsurf systematically (not manual rewrite), deterministic matching covers 85-90% of cases. Embeddings are optional — only activate if unmapped count after Phase 3 exceeds 50 constructs.

### 4.3 Stub Detection as Mandatory Phase

Windsurf generates stubs — methods with real signatures but placeholder bodies (`return BigDecimal.ZERO`, `return false`, comments only). These pass structural checks (name matches) but behavioral tests will fail. Detecting stubs statically at parse time catches them before wasting test cycles.

### 4.4 Postgres-Only Architecture

No graph database (Apache AGE, Neo4j, Kùzu). All dependency validation via recursive CTEs on a `graph_edges` table in the same Postgres instance. This means:

- Zero new infrastructure
- JOIN with traceability, gap, and behavioral tables natively
- One backup strategy, one source of truth
- Same accuracy as purpose-built graph databases for our bounded (2-5 hop) query patterns

### 4.5 Golden File Parity for Behavioral Testing

Cannot run GnuCOBOL on NonStop COBOL. Instead: capture input/output pairs from the live NonStop system before decommission, then validate Java produces identical outputs.

-----

## 5. Architecture Overview

```
┌─────────────────────────────────────────────────────────────────────────┐
│                         PROJECT SENTINEL                                 │
│                                                                          │
│  INPUT                                                                    │
│  ┌──────────────────────┐     ┌───────────────────────────────┐          │
│  │ HP NonStop COBOL      │     │ Converted Java (Windsurf)     │          │
│  │ ($DATA1.SRCLIB.*)     │     │ src/main/java/...             │          │
│  └──────────┬────────────┘     └──────────────┬────────────────┘          │
│             │                                  │                          │
│  PARSING    │                                  │                          │
│  ┌──────────▼────────────┐     ┌──────────────▼────────────────┐          │
│  │ ProLeap (ANTLR4)       │     │ scip-java (compiler-grade)    │          │
│  │ · Native Tandem format │     │ · Symbols + references        │          │
│  │ · AST + ASG            │     │ · Call graph (from SCIP)      │          │
│  │ · COPY preprocessing   │     │ · Type hierarchy              │          │
│  │ · ENTER TAL/C extract  │     ├───────────────────────────────┤          │
│  │ · EXEC SQL/MP extract  │     │ Tree-sitter Java (supplement) │          │
│  │ · Guardian file detect │     │ · AST fingerprints            │          │
│  └──────────┬─────────────┘     │ · Complexity scoring          │          │
│             │                   └──────────────┬────────────────┘          │
│             │                                  │                          │
│  ┌──────────▼──────────────────────────────────▼────────────────┐          │
│  │                   POSTGRES — sentinel schema                   │          │
│  │                                                                │          │
│  │  cobol_constructs   java_constructs   traceability_map         │          │
│  │  scip_references    graph_edges        coverage_gaps           │          │
│  │  golden_files       test_cases         behavioral_results      │          │
│  │  review_log         drift_alerts       decommission_log        │          │
│  │                                                                │          │
│  │  [OPTIONAL — only when SENTINEL_ENABLE_EMBEDDINGS=true]       │          │
│  │  cobol_embeddings (1024d)    java_embeddings (1024d)           │          │
│  └────────────────────────────────────────────────────────────────┘          │
│                                                                          │
│  VALIDATION PIPELINE                                                      │
│  ┌──────────────────────────────────────────────────────────────┐         │
│  │                                                               │         │
│  │  Phase 3: Deterministic Mapping                               │         │
│  │  · Name convention matching (COBOL-KEBAB → javaMethod)        │         │
│  │  · Structural fingerprint matching (AST node sequences)       │         │
│  │  · Windsurf metadata extraction (conversion comments)         │         │
│  │                                                               │         │
│  │  Phase 3.5: Stub Detection                                    │         │
│  │  · 7 signal groups (empty returns, comments-only, unused      │         │
│  │    params, identity passthrough, COBOL structural mismatch)   │         │
│  │  · Flags methods that are mapped but not implemented          │         │
│  │                                                               │         │
│  │  Phase 4: Dependency Validation (Recursive CTEs)              │         │
│  │  · PERFORMS A→B must exist as JAVA_CALLS A'→B'                │         │
│  │  · Fan-out/fan-in delta ≤ 2                                   │         │
│  │  · ENTER TAL must have REPLACED_BY edge                       │         │
│  │  · Guardian file access patterns preserved                    │         │
│  │  · TMF transaction boundaries mapped                          │         │
│  │                                                               │         │
│  │  Phase 5: Behavioral Validation (Golden File Parity)          │         │
│  │  · Capture I/O from live NonStop system                       │         │
│  │  · Java produces identical output (byte-level + COMP-3 ε)     │         │
│  │                                                               │         │
│  │  Phase 6: Semantic Embeddings [OPTIONAL FLAG]                 │         │
│  │  · snowflake-arctic on NL summaries (both sides 1024d)        │         │
│  │  · Cross-language cosine similarity                           │         │
│  │  · Auto-map > 0.85, review 0.70–0.85, gap < 0.70             │         │
│  └──────────────────────────────────────────────────────────────┘         │
│                                                                          │
│  OUTPUTS                                                                  │
│  ┌──────────────────────────────────────────────────────────────┐         │
│  │                                                               │         │
│  │  Dashboard (React + Express)  ←──  Postgres LISTEN/NOTIFY    │         │
│  │  · Gap Analysis (5 views)         Server-Sent Events         │         │
│  │  · Stub Detection Panel                                       │         │
│  │  · Review Workflow                                            │         │
│  │  · Decommission Gate                                          │         │
│  │                                                               │         │
│  │  Reports (Pandoc → PDF)                                       │         │
│  │  · Executive Summary · Audit Trail · Gap Report               │         │
│  │  · Behavioral Report · Stub Report · Decommission Certificate │         │
│  └──────────────────────────────────────────────────────────────┘         │
└─────────────────────────────────────────────────────────────────────────┘
```

-----

## 6. Technology Stack — Final Decisions

### 6.1 Decision Table

|Component         |Choice                                      |Why This                                                                                                                |Why Not Others                                                                 |
|------------------|--------------------------------------------|------------------------------------------------------------------------------------------------------------------------|-------------------------------------------------------------------------------|
|COBOL parser      |**ProLeap (ANTLR4)**                        |Native Tandem format; AST+ASG with data/control flow; COPY preprocessing; EXEC SQL/ENTER TAL as text blocks; NIST tested|Tree-sitter: no Tandem support, no ASG. IBM tools: z/OS only                   |
|Java indexer      |**scip-java**                               |Compiler-grade; resolved symbols; cross-file refs; Gradle/Maven native; TypeScript bindings                             |Tree-sitter alone: no type resolution, no cross-file                           |
|Java AST          |**Tree-sitter Java** (supplement)           |AST fingerprints for structural matching; complexity scoring                                                            |scip-java doesn’t expose raw AST structure                                     |
|Dependency graph  |**Postgres recursive CTEs**                 |Already installed; native JOINs with all tables; same accuracy as graph DBs for 2-5 hop queries; zero new infra         |Apache AGE: compilation fragility; Neo4j: separate process; Kùzu: less mature  |
|COBOL embeddings  |**snowflake-arctic-embed-l-v2.0** (optional)|COBOL reads like English prose; strong text model; same 1024-dim space as Java summaries                                |nomic-embed-code: not trained on COBOL                                         |
|Java embeddings   |**snowflake-arctic-embed-l-v2.0** (optional)|Same vector space as COBOL for direct comparison                                                                        |nomic-embed-code trained on Java but different dims = no cross-language compare|
|Embedding strategy|**NL summaries** (not raw code)             |ProLeap ASG → NL summary for COBOL; LLM pseudocode for Java; text model understands English better than COBOL syntax    |Raw code embedding: model has never seen COBOL                                 |
|Behavioral testing|**Golden file parity**                      |Only option: GnuCOBOL cannot run NonStop COBOL                                                                          |GnuCOBOL: incompatible dialect and runtime                                     |
|Stub detection    |**Static pattern analysis**                 |7 signal groups; no model needed; catches comment-only stubs + empty returns                                            |LLM-only: slower, inconsistent, expensive                                      |
|COBOL orchestrator|**Node.js 24 + TypeScript**                 |Consistent with Apollo; ProLeap called via Java CLI wrapper                                                             |Python: inconsistent with existing stack                                       |
|Test framework    |**Vitest 3 (`^3.2.4`)**                     |Stable on Node.js 24; consistent with Apollo                                                                            |Vitest 4: instability on Node.js 24 backend                                    |
|Dashboard         |**React 18 + Express**                      |No build step needed; CDN or vendor bundle                                                                              |Next.js: overkill; Vue: team unfamiliar                                        |
|PDF reports       |**Pandoc**                                  |Already on RHEL; zero deps; Markdown → PDF                                                                              |Puppeteer: requires Chromium                                                   |

### 6.2 What We Explicitly Rejected

|Tool           |Reason Rejected                                                                                                 |
|---------------|----------------------------------------------------------------------------------------------------------------|
|Apache AGE     |Requires building against Postgres source; fragile across upgrades; Cypher expressiveness not needed            |
|Neo4j          |Separate JVM process to manage; second source of truth; same accuracy for our bounded queries                   |
|Kùzu           |Less mature; unproven at scale; adds third storage layer                                                        |
|GnuCOBOL       |Cannot compile or run HP NonStop COBOL dialect                                                                  |
|IBM COBOL tools|z/OS specific; wrong dialect                                                                                    |
|Codestral Embed|External API — violates no-external-dependency constraint                                                       |
|Voyage Code 3  |External API — violates no-external-dependency constraint                                                       |
|Windsurf alone |Cannot hold full codebase in context; no persistent verification state; not auditable; cannot execute both sides|

-----

## 7. Database Schema — Single Source of Truth

All data lives in the `sentinel` schema within the existing Apollo Postgres instance.

### 7.1 Schema Groups

```
sentinel schema
│
├── INVENTORY
│   ├── cobol_constructs       All COBOL constructs parsed by ProLeap
│   ├── java_constructs        All Java constructs from scip-java + Tree-sitter
│   │   └── is_stub            Stub detection result (Phase 3.5)
│   └── scip_references        Cross-file symbol references from SCIP
│
├── TRACEABILITY
│   ├── traceability_map       COBOL construct ↔ Java construct mappings
│   └── coverage_gaps          Unmapped items with severity + NonStop flags
│
├── DEPENDENCY GRAPH
│   └── graph_edges            All edges: COBOL, Java, cross-language
│       (replaces Apache AGE)  Queried via recursive CTEs
│
├── BEHAVIORAL
│   ├── golden_files           I/O pairs captured from NonStop system
│   ├── test_cases             Test definitions per program
│   └── behavioral_results     Java parity results vs golden files
│
├── GOVERNANCE
│   ├── review_log             Audit trail: every human action
│   ├── drift_alerts           Similarity degradation alerts (optional)
│   └── decommission_log       Per-program sign-off records
│
├── OPTIONAL (flag-gated)
│   ├── cobol_embeddings       vector(1024) — snowflake-arctic on ASG summaries
│   └── java_embeddings        vector(1024) — snowflake-arctic on LLM descriptions
│
└── VIEWS
    ├── coverage_summary       Layer-by-layer coverage % (materialized)
    ├── program_coverage       Per-program metrics (materialized)
    ├── nonstop_coverage       TAL/C, Guardian, TMF, Pathway status
    ├── stub_summary           All confirmed stubs with signals
    ├── program_stub_counts    Stub counts per program
    └── untested_stubs         Stubs with no behavioral test coverage
```

### 7.2 Key Design Decisions

**graph_edges replaces Apache AGE:**

```sql
CREATE TABLE graph_edges (
    id          SERIAL PRIMARY KEY,
    from_type   TEXT NOT NULL,   -- 'cobol_paragraph', 'java_method', etc.
    from_id     INT  NOT NULL,
    to_type     TEXT NOT NULL,
    to_id       INT  NOT NULL,
    edge_type   TEXT NOT NULL,   -- 'PERFORMS', 'JAVA_CALLS', 'CONVERTS_TO',
                                 -- 'ENTER_TAL', 'REPLACED_BY', etc.
    metadata    JSONB DEFAULT '{}',
    created_at  TIMESTAMPTZ DEFAULT NOW(),
    UNIQUE (from_type, from_id, to_type, to_id, edge_type)
);
```

Queried with recursive CTEs for all dependency validation. Joins natively with `traceability_map` and `coverage_gaps`.

**cobol_constructs has NonStop metadata:**

```sql
nonstop_metadata JSONB DEFAULT '{}'
-- Contains:
-- { talRoutine: "RATE_TABLE_LOOKUP",
--   talSourceFile: "$DATA1.TALLIB.RATELKP",
--   talParams: ["rate_table_ptr INT:ref", ...],
--   guardianFileName: "$DATA2.ACCTDB.ACCTFILE",
--   tmfOperation: "COMMIT",
--   pathwayServerClass: "PAYMENT-SVC" }
```

**java_constructs has stub detection fields:**

```sql
is_stub           BOOLEAN DEFAULT FALSE,
stub_confidence   NUMERIC DEFAULT 0.0,
stub_severity     TEXT,   -- 'critical', 'major', 'minor', 'none'
stub_signals      JSONB DEFAULT '[]',
stub_summary      TEXT,
stub_detected_at  TIMESTAMPTZ
```

**Composite score formula:**

```sql
-- Without embeddings (default)
0.25 × structural + 0.30 × dependency + 0.45 × behavioral

-- With embeddings enabled
0.20 × structural + 0.20 × semantic + 0.25 × dependency + 0.35 × behavioral
```

-----

## 8. Phase 1 — COBOL Inventory (ProLeap)

### 8.1 Goal

Parse every HP NonStop COBOL source file and build a complete structural + semantic inventory in `cobol_constructs`.

### 8.2 Why ProLeap

ProLeap is an ANTLR4-based COBOL parser (Java) that produces both AST and an Abstract Semantic Graph (ASG) — resolved variable access and control flow. It has **native Tandem source format** support (`CobolSourceFormatEnum.TANDEM`), making it the only open-source parser that handles HP NonStop COBOL without dialect failures.

ENTER TAL/C and EXEC SQL blocks are extracted as text by ProLeap and post-processed by regex to capture routine names, parameters, and source files.

### 8.3 What Gets Extracted

|Construct                       |ProLeap          |Regex Post-Process  |
|--------------------------------|-----------------|--------------------|
|PROGRAM-ID, SECTION, PARAGRAPH  |✓                |—                   |
|COPY statements (with expansion)|✓                |—                   |
|PERFORM targets                 |✓ (ASG)          |—                   |
|CALL targets                    |✓ (ASG)          |—                   |
|Data items (level, PIC, USAGE)  |✓ (ASG)          |—                   |
|File descriptors                |✓                |Guardian path regex |
|EXEC SQL/MP blocks              |Extracted as text|SQL/MP syntax parser|
|ENTER TAL blocks                |Extracted as text|TAL param extractor |
|ENTER C blocks                  |Extracted as text|C param extractor   |
|LOCKFILE statements             |✓                |—                   |
|CHECKPOINT constructs           |✓                |—                   |

### 8.4 ASG Summary Generation

ProLeap’s ASG gives us resolved data flow. For each paragraph, generate a plain English summary:

> “Performs 4 arithmetic computations. Contains 2 multi-branch decisions (ACT/360, ACT/365, 30/360). Reads: WS-ANNUAL-RATE, WS-PRINCIPAL. Modifies: WS-DAILY-RATE, WS-DAILY-AMOUNT. Calls: 310-CALC-30-360, 320-APPLY-COMPOUND. Calls TAL routine: RATE_TABLE_LOOKUP.”

This summary is stored in `cobol_constructs.asg_summary` and used by Phase 6 embeddings.

### 8.5 Installation

```bash
# User-space, no sudo required
git clone https://github.com/uwol/proleap-cobol-parser.git \
    ~/.local/sentinel/proleap
cd ~/.local/sentinel/proleap && mvn clean install -DskipTests

# Node.js calls ProLeap via Java CLI wrapper
java -jar proleap-cobol-parser.jar \
    --format TANDEM \
    --input /path/to/program.cbl \
    --output-format json \
    > constructs.json
```

### 8.6 Exit Criteria

- [ ] All COBOL source files processed (Tandem format)
- [ ] Parse failure rate < 5%
- [ ] All ENTER TAL/C blocks extracted and cataloged
- [ ] ASG summaries generated for paragraphs > 5 LOC
- [ ] All Guardian file references captured
- [ ] Baseline snapshot created in `conversion_baseline`

-----

## 9. Phase 2 — Java Inventory (SCIP + Tree-sitter)

### 9.1 Goal

Build a compiler-grade inventory of the converted Java codebase in `java_constructs` and `scip_references`.

### 9.2 Two-Layer Approach

**scip-java** (primary): Runs at the root of the Gradle/Maven project. Produces `index.scip` (protobuf) containing every symbol definition, reference, and type relationship. Compiler-accurate — resolves overloads, generics, and cross-module calls. Supports Java 8–21.

**Tree-sitter Java** (supplementary): Raw AST structure for fingerprint-based structural matching in Phase 3. Provides complexity scoring.

```bash
# Install scip-java (user-space binary)
curl -fLo ~/.local/sentinel/scip/scip-java \
    https://github.com/sourcegraph/scip-java/releases/latest/download/scip-java-linux-x86_64
chmod +x ~/.local/sentinel/scip/scip-java

# Run
cd /path/to/converted-java
scip-java index  # produces index.scip
```

### 9.3 Pseudocode Summary (flag-gated)

Only when `SENTINEL_ENABLE_EMBEDDINGS=true`: generate an LLM-based natural language description of each Java method > 5 LOC. Stored in `java_constructs.pseudocode_summary`. Used for cross-language embedding comparison in Phase 6.

### 9.4 Exit Criteria

- [ ] `scip-java index` completes successfully
- [ ] All symbols and references loaded into `java_constructs` and `scip_references`
- [ ] AST fingerprints from Tree-sitter generated for all methods
- [ ] Baseline snapshot updated with Java-side metrics

-----

## 10. Phase 3 — Deterministic Mapping

### 10.1 Goal

Establish links between COBOL constructs and Java constructs using three deterministic strategies. Expected to cover 85-90% of mappings with high/medium confidence.

### 10.2 Strategy 1: Convention-Based Name Matching

Windsurf follows naming conventions:

|COBOL                |Java                                           |Example             |
|---------------------|-----------------------------------------------|--------------------|
|PROGRAM-ID → class   |`CALC-INTEREST` → `CalcInterest`               |                    |
|PARAGRAPH → method   |`300-COMPUTE-DAILY-RATE` → `computeDailyRate()`|strip numeric prefix|
|COPYBOOK → DTO       |`CUST-REC.cpy` → `CustRec.java`                |                    |
|WS data item → field |`WS-DAILY-RATE` → `dailyRate`                  |                    |
|CALL target → service|`CALL 'VALIDATE'` → `ValidateService`          |                    |

Scoring: exact = 1.0, prefix match = 0.9, Levenshtein ≤ 2 = 0.8.

### 10.3 Strategy 2: Structural Fingerprint Matching

Normalize COBOL AST node sequences and Java AST node sequences to a common vocabulary. Compare via Longest Common Subsequence (LCS):

|COBOL     |Java                 |Token        |
|----------|---------------------|-------------|
|EVALUATE  |switch/if-else chain |BRANCH       |
|COMPUTE   |arithmetic expression|ARITHMETIC   |
|PERFORM   |method invocation    |CALL         |
|READ/WRITE|JDBC/InputStream     |IO           |
|CALL      |class instantiation  |EXTERNAL_CALL|

### 10.4 Strategy 3: Windsurf Metadata

Search Java files for conversion comments:

```java
// Converted from: CALC-INTEREST.300-COMPUTE-DAILY-RATE
// Source: CALCINT.cbl line 247
```

Also search git log for conversion commit messages.

### 10.5 Exit Criteria

- [ ] All three strategies executed
- [ ] `traceability_map` populated (high/medium/low confidence)
- [ ] Unmapped constructs logged in `coverage_gaps` with severity
- [ ] **Decision point: count unmapped items**
  - < 20 items → skip Phase 6, send to SME review
  - 20–50 items → management decision on Phase 6
  - 50+ items → enable `SENTINEL_ENABLE_EMBEDDINGS=true`

-----

## 11. Phase 3.5 — Stub Detection

### 11.1 Goal

Find Java methods that are structurally mapped (name matches, method exists) but contain no real implementation. This is the most dangerous conversion failure because it appears correct to all other layers.

### 11.2 The Problem

Windsurf sometimes generates:

```java
public BigDecimal calculateDailyRate(String rateType, BigDecimal annualRate) {
    // Calculates the daily interest rate based on day-count convention
    return BigDecimal.ZERO;
}
```

- Name matches COBOL paragraph ✓
- Signature looks correct ✓
- Method exists in call graph ✓
- Plain English comment makes it look deliberate ✓
- **But it returns zero regardless of inputs ✗**

### 11.3 Seven Detection Signal Groups

|Group                       |Signals    |Key Indicators                                                                                                                                                                |
|----------------------------|-----------|------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
|1. Empty return patterns    |18 patterns|`return BigDecimal.ZERO`, `return ""`, `return false`, `return null`, `return Optional.empty()`, `return Collections.emptyList()`, `throw UnsupportedOperationException`, etc.|
|2. Body structural emptiness|3 signals  |Empty body `{}`, comments-only body, comment-heavy with ≤ 2 code lines                                                                                                        |
|3. Unused parameters        |1 signal   |Parameters declared in signature but never referenced in body                                                                                                                 |
|4. Identity passthrough     |1 signal   |`return annualRate` or `BigDecimal r = annualRate; return r`                                                                                                                  |
|5. Semantic stubs           |2 signals  |Computes something but returns empty anyway; assigns empty then returns it                                                                                                    |
|6. COBOL structural mismatch|5 signals  |LOC ratio < 0.15, complexity ratio < 0.30, no arithmetic when COBOL had COMPUTE, no branching when COBOL had EVALUATE, no method calls when COBOL had PERFORM×3               |
|7. Explicit markers         |3 signals  |TODO/FIXME/XXX comments, COBOL references in comments, “placeholder”/“stub”/“not implemented”                                                                                 |

Confidence ≥ 0.50 → flagged as stub. Confidence ≥ 0.70 → severity = critical.

### 11.4 Signal Group 6 is Key

Groups 1-5 and 7 cover explicit stubs. Group 6 is what detects **silent stubs** — methods with plain comments and empty returns that look deliberate:

```
COBOL: 87 LOC, complexity 12, COMPUTE×4, EVALUATE×2, PERFORM×3
Java:   4 LOC, complexity  1, no arithmetic, no branching, no method calls
```

These mismatches together give ~0.90 confidence even with no TODO comment.

### 11.5 Integration

```typescript
// Runs during Phase 2 Java parsing
// When cobol_match is available (from Phase 3):
const result = detectStub(javaSource, cobolContext);
await db.query(`
    UPDATE java_constructs
    SET is_stub = $1, stub_confidence = $2,
        stub_severity = $3, stub_signals = $4
    WHERE id = $5
`, [result.isStub, result.confidence, result.severity,
    JSON.stringify(result.signals), constructId]);
```

Stub gaps are always severity = ‘critical’ and added to `coverage_gaps` with `gap_type = 'stub_conversion'`.

### 11.6 Exit Criteria

- [ ] All mapped Java methods scanned for stubs
- [ ] Zero confirmed stubs with confidence ≥ 0.70 unreviewed
- [ ] Stub count reported per program in dashboard
- [ ] `untested_stubs` view shows stubs without behavioral coverage

-----

## 12. Phase 4 — Dependency Validation (Recursive CTEs)

### 12.1 Goal

Verify that COBOL call chains, data flows, and NonStop-specific dependencies are preserved in the Java conversion.

### 12.2 Edge Table (replaces Apache AGE)

```sql
CREATE TABLE graph_edges (
    from_type  TEXT NOT NULL,  -- 'cobol_paragraph', 'java_method', 'tal_routine', etc.
    from_id    INT  NOT NULL,
    to_type    TEXT NOT NULL,
    to_id      INT  NOT NULL,
    edge_type  TEXT NOT NULL,  -- 'PERFORMS', 'JAVA_CALLS', 'CONVERTS_TO',
                               -- 'ENTER_TAL', 'REPLACED_BY', 'READS', etc.
    metadata   JSONB DEFAULT '{}'
);
```

Built from:

- ProLeap ASG → COBOL edges (PERFORMS, CALLS, COPIES, READS, WRITES, ENTER_TAL, ENTER_C)
- SCIP references → Java edges (JAVA_CALLS, JAVA_IMPORTS, JAVA_EXTENDS, JAVA_IMPLEMENTS)
- traceability_map → cross-language edges (CONVERTS_TO, REPLACED_BY, MIGRATED_TO)

### 12.3 Eight Validation Checks

|Check                     |Query Pattern                                                                     |Severity|
|--------------------------|----------------------------------------------------------------------------------|--------|
|Broken call chains        |For every COBOL A PERFORMS B where both are mapped, verify Java A’ CALLS B’       |Critical|
|Fan-out divergence        |If COBOL paragraph calls N sub-paragraphs, Java method should call ~N methods (±2)|Major   |
|Fan-in divergence         |If N COBOL paragraphs call X, N Java methods should call X’ (±2)                  |Major   |
|Unmapped ENTER TAL/C      |Every ENTER TAL/C must have a REPLACED_BY edge                                    |Critical|
|Guardian file access      |If COBOL paragraph READs a file, mapped Java method must have file/DB I/O         |Critical|
|TMF transaction boundaries|TMF_BEGIN_/TMF_COMMIT_ TAL calls must have Java transaction equivalents           |Critical|
|Copybook fan-out          |Every Java DTO derived from a copybook must be used by all expected classes       |Major   |
|SQL/MP equivalence        |EXEC SQL blocks must map to JDBC/JPA calls on same tables                         |Major   |

### 12.4 Key Recursive CTE — Broken Chain Detection

```sql
WITH broken_chains AS (
    SELECT
        c_edge.from_id AS cobol_caller_id,
        c_edge.to_id   AS cobol_callee_id,
        cc_from.construct_name AS caller_name,
        cc_to.construct_name   AS callee_name,
        cc_from.program_id
    FROM graph_edges c_edge
    JOIN cobol_constructs cc_from ON cc_from.id = c_edge.from_id
    JOIN cobol_constructs cc_to   ON cc_to.id   = c_edge.to_id
    WHERE c_edge.edge_type = 'PERFORMS'
    -- Both paragraphs are mapped to Java
    AND EXISTS (SELECT 1 FROM graph_edges WHERE from_id = c_edge.from_id AND edge_type = 'CONVERTS_TO')
    AND EXISTS (SELECT 1 FROM graph_edges WHERE from_id = c_edge.to_id   AND edge_type = 'CONVERTS_TO')
    -- But no corresponding Java call exists between the mapped methods
    AND NOT EXISTS (
        SELECT 1
        FROM graph_edges ct1
        JOIN graph_edges ct2 ON ct2.from_id = c_edge.to_id AND ct2.edge_type = 'CONVERTS_TO'
        JOIN graph_edges je  ON je.from_id = ct1.to_id AND je.to_id = ct2.to_id AND je.edge_type = 'JAVA_CALLS'
        WHERE ct1.from_id = c_edge.from_id AND ct1.edge_type = 'CONVERTS_TO'
    )
)
SELECT * FROM broken_chains;
```

### 12.5 Exit Criteria

- [ ] All edges populated in `graph_edges`
- [ ] All 8 validation checks executed
- [ ] All broken chains logged in `dependency_validation_results`
- [ ] Critical failures added to `coverage_gaps`

-----

## 13. Phase 5 — Behavioral Validation (Golden File Parity)

### 13.1 Goal

Prove functional equivalence by running identical inputs through Java and comparing outputs against captured NonStop (COBOL) outputs.

### 13.2 Why Golden Files, Not Parallel Execution

GnuCOBOL cannot compile or run HP NonStop COBOL. The NonStop-specific runtime (Guardian OS, TMF, Pathway) does not exist on RHEL. Therefore:

1. **Before decommission:** Run COBOL programs on the live NonStop system with known inputs. Capture outputs (stdout, output files, DB state changes, return codes).
1. **On RHEL:** Run Java with the same inputs. Compare against captured outputs.

### 13.3 Golden File Capture

```bash
# Run on NonStop system before decommission
# Captures: input files, stdout, output files, return codes, DB state

PROGRAM=$1
INPUT=$2
OUTPUT_DIR="/golden-files/$PROGRAM/$(date +%Y%m%d)"

mkdir -p "$OUTPUT_DIR"
sqlci <<< "SELECT * FROM test_acct WHERE id IN (...);" > "$OUTPUT_DIR/db_before.txt"
run $PROGRAM /IN $INPUT /OUT "$OUTPUT_DIR/stdout.txt"
echo $? > "$OUTPUT_DIR/return_code.txt"
cp $DATA2.ACCTDB.OUTPUT "$OUTPUT_DIR/output_file.dat"
sqlci <<< "SELECT * FROM test_acct WHERE id IN (...);" > "$OUTPUT_DIR/db_after.txt"
md5sum "$OUTPUT_DIR"/* > "$OUTPUT_DIR/checksums.md5"
```

### 13.4 COMP-3 Precision Testing

COMP-3 packed decimal → Java BigDecimal is the highest-risk conversion. Use epsilon comparison:

```typescript
const COMP3_EPSILON = 0.005;       // general numeric
const CURRENCY_EPSILON = 0.01;     // monetary fields

function comparePrecision(cobolValue: string, javaValue: string, pic: string): boolean {
    const scale = extractScale(pic);  // from V9(n) in PIC clause
    const delta = Math.abs(parseFloat(cobolValue) - parseFloat(javaValue));
    return delta <= (scale <= 2 ? CURRENCY_EPSILON : COMP3_EPSILON);
}
```

### 13.5 LLM Edge Case Generation

Use Ollama to generate NonStop-specific edge cases from COBOL source:

- Guardian file status codes (00, 10, 23, 35, 39, 41, 42, 46, 47, 48)
- LOCKFILE timeout scenarios (TIME LIMIT exceeded)
- TMF ABORT recovery paths
- COMP-3 boundary values (odd-length fields, sign nibble edge cases)
- PIC 9(n)V9(m) overflow/underflow
- REDEFINES memory overlap scenarios

### 13.6 Exit Criteria

- [ ] Golden files captured from NonStop for all critical programs
- [ ] Java parity tests executed against golden files
- [ ] COMP-3 precision tests passing within epsilon
- [ ] LLM-generated edge cases executed
- [ ] All failures documented with diff detail in `behavioral_results`

-----

## 14. Phase 6 — Semantic Embedding Layer (Optional Flag)

### 14.1 Default State: DISABLED

```bash
SENTINEL_ENABLE_EMBEDDINGS=false  # default
```

### 14.2 When to Enable

After Phase 3, count unmapped COBOL constructs:

- **< 20:** Skip Phase 6. Send directly to SME review.
- **20-50:** Management decision based on time/budget.
- **50+:** Enable Phase 6. Expected to recover most of these.

### 14.3 The Cross-Language Embedding Problem

No embedding model is trained on COBOL. Direct code embedding of COBOL fails because the model treats COBOL syntax as noise.

**Solution:** Embed natural language summaries in the same vector space:

```
COBOL paragraph
    → ProLeap ASG summary (English text)
    → snowflake-arctic-embed-l-v2.0
    → vector(1024)
                    ↕ cosine similarity
Java method
    → LLM pseudocode description (English text)
    → snowflake-arctic-embed-l-v2.0
    → vector(1024)
```

Both sides use the same model on the same kind of text → directly comparable.

**Example COBOL summary:** “Performs 4 arithmetic computations. Contains 2 multi-branch decisions (ACT/360, ACT/365, 30/360). Reads WS-ANNUAL-RATE, WS-PRINCIPAL. Modifies WS-DAILY-RATE, WS-DAILY-AMOUNT. Calls TAL routine RATE_TABLE_LOOKUP.”

**Example Java description:** “Calculates daily interest rate by dividing annual rate by 360. Only handles ACT/360 convention. Returns BigDecimal with 8 decimal places using HALF_UP rounding.”

Low similarity (0.62) correctly signals incomplete conversion.

### 14.4 Thresholds

|Similarity |Action      |Confidence|
|-----------|------------|----------|
|> 0.85     |Auto-map    |medium    |
|0.70 – 0.85|Review queue|low       |
|< 0.70     |Coverage gap|unmatched |

### 14.5 Exit Criteria

- [ ] All non-trivial constructs embedded (both sides, same 1024-dim space)
- [ ] Cross-language nearest-neighbor search complete
- [ ] Additional mappings added to `traceability_map`
- [ ] Remaining gaps in `coverage_gaps` with similarity scores

-----

## 15. Phase 7 — Gap Analysis Dashboard

### 15.1 Goal

Single-pane-of-glass for all stakeholders to monitor progress, review gaps, and manage sign-offs.

### 15.2 Architecture

```
React SPA (dashboard/src/)
    │  HTTP / Server-Sent Events
Express API (src/dashboard/api-server.ts) port 3001
    │
Postgres (materialized views + live queries)
    │
Postgres LISTEN/NOTIFY → SSE → browser auto-refresh
```

### 15.3 Five Views

|View        |Primary User      |Purpose                                                    |
|------------|------------------|-----------------------------------------------------------|
|**Overview**|Management        |Composite score, layer bars, coverage trend, NonStop status|
|**Gaps**    |Engineer/Tech Lead|Filterable gap list with full detail panel per gap         |
|**Stubs**   |Engineer/SME      |Detected stubs with signal breakdown and source comparison |
|**Programs**|Tech Lead         |Per-program health cards with drill-down                   |
|**Sign-Off**|Legal/Management  |Decommission gate status + sign-off flow                   |

### 15.4 Gap Detail Panel (8 Sections)

Each gap row expands with collapsible sections:

1. **Source Code** — COBOL vs Java side-by-side with syntax highlighting
1. **Data Items** — PIC, USAGE, COMP-3/Guardian warnings
1. **Similarity Analysis** — Cosine score bar with threshold markers (flag-gated)
1. **Dependency Issues** — Broken chains, fan-out mismatches from Phase 4
1. **Behavioral Tests** — Pass/fail/not-run counts with failing test diff samples
1. **ENTER TAL/C Detail** — TAL routine name, source file, parameters, required actions (NonStop only)
1. **Verb Analysis** — COBOL verb frequency with risk coloring
1. **Audit Log** — Timeline of all review actions

Action buttons: Confirm Match · Reject Match · Escalate to SME · Flag Dead Code

### 15.5 Review Workflow

**Tier 1 (automatic):** High confidence mappings auto-accepted. No human needed. ~70-80% of mappings.

**Tier 2 (Tech Lead):** Medium/low confidence items in review queue. Confirm/reject/reassign flow with mandatory notes. All actions logged to `review_log`.

**Tier 3 (SME):** Critical gaps, ENTER TAL/C items, Guardian I/O patterns. Domain expert verifies correctness. Decision recorded with reasoning.

### 15.6 Real-Time Updates

Postgres NOTIFY → Express SSE → browser hook:

```sql
-- Trigger on key tables
CREATE TRIGGER notify_coverage_gaps AFTER INSERT OR UPDATE ON sentinel.coverage_gaps
    FOR EACH ROW EXECUTE FUNCTION sentinel.notify_change();
```

Browser refreshes only affected panels without full page reload.

### 15.7 API Endpoints (Key)

|Endpoint                                   |Purpose                      |
|-------------------------------------------|-----------------------------|
|`GET /api/coverage/summary`                |Layer coverage %             |
|`GET /api/gaps?program=X&severity=critical`|Filtered gaps                |
|`GET /api/gaps/:id`                        |Full detail with all sections|
|`GET /api/stubs`                           |All detected stubs           |
|`PATCH /api/review/mapping/:id/confirm`    |Review action                |
|`GET /api/decommission/:programId`         |Gate checklist               |
|`POST /api/decommission/:programId/signoff`|Record sign-off              |
|`GET /api/events`                          |SSE stream                   |

-----

## 16. Phase 8 — Reporting & Review Workflow

### 16.1 Report Types

|Report                  |Audience         |Format      |When               |
|------------------------|-----------------|------------|-------------------|
|Executive Summary       |Management, Legal|PDF         |Weekly (cron)      |
|Audit Trail             |Legal (Sarah)    |CSV         |On demand          |
|Gap Report              |Tech Lead        |PDF         |On demand          |
|Stub Report             |Engineering      |PDF         |After Phase 3.5    |
|Behavioral Report       |QA               |PDF         |After each test run|
|Decommission Certificate|All stakeholders |PDF (signed)|Per program        |

### 16.2 Generation Pipeline

```
Postgres → coverage-engine.ts → Markdown template → Pandoc → PDF
```

Pandoc is already on RHEL. Zero additional dependencies.

### 16.3 Decommission Certificate

The formal sign-off artifact per program:

```
DECOMMISSION CERTIFICATE

Program:       CALC-INTEREST
Guardian File: $DATA1.SRCLIB.CALCINT
Date:          2026-05-01

Verification Scores:
  Structural Coverage:  100%  ≥ 95% ✓
  Dependency Integrity: 100%  ≥ 95% ✓
  Behavioral Parity:    100%  ≥ 95% ✓
  Stubs Confirmed:        0   = 0   ✓
  Open Gaps:              0   = 0   ✓

NonStop Verification:
  ENTER TAL/C:    3/3 verified ✓
  Guardian I/O:   2/2 verified ✓
  TMF:            1/1 verified ✓

Sign-offs:
  Tech Lead:       MK  2026-04-28 ✓
  Business Owner:  RJ  2026-04-29 ✓
  Legal:           SM  2026-05-01 ✓

APPROVED FOR DECOMMISSION
```

-----

## 17. Phase 9 — Continuous Governance

### 17.1 Git Hook (post-commit on Java repo)

```bash
# On every Java commit:
# 1. Re-parse changed files → update java_constructs
# 2. Re-run stub detection on changed methods
# 3. Re-compute affected graph edges
# 4. Re-validate broken chains for affected methods
# Alert if a confirmed mapping is disrupted
```

### 17.2 Drift Detection (optional, flag-gated)

```bash
SENTINEL_ENABLE_DRIFT_DETECTION=true  # requires ENABLE_EMBEDDINGS=true
```

Weekly cron: re-embed all Java constructs modified in the past 7 days. Recompute similarity against mapped COBOL counterparts.

|Similarity Drop|Alert Level|Action                          |
|---------------|-----------|--------------------------------|
|> 0.10         |Warning    |Notify tech lead                |
|> 0.20         |Critical   |Block deployment, require review|

### 17.3 Decommission Gate (blocking)

A program cannot be decommissioned until **all** pass. See Section 25.

-----

## 18. HP NonStop — Construct Handling Reference

### 18.1 Construct Type Matrix

|NonStop Construct                  |construct_type   |Risk    |Java Equivalent                |
|-----------------------------------|-----------------|--------|-------------------------------|
|`ENTER TAL "routine"`              |`enter_tal`      |Critical|Pure Java or JNI               |
|`ENTER C "routine"`                |`enter_c`        |Critical|Pure Java or JNI               |
|Guardian file (`$DATA.SUBVOL.FILE`)|`file_descriptor`|Critical|JDBC or java.io                |
|`LOCKFILE`                         |paragraph (verb) |Critical|DB locking or java.nio.FileLock|
|`TMF_BEGIN_` / `TMF_COMMIT_`       |`checkpoint`     |Critical|JTA / @Transactional           |
|Pathway server class               |`pathway_server` |Major   |REST / MQ                      |
|`$RECEIVE` IPC                     |`receive_block`  |Major   |MQ / gRPC                      |
|EXEC SQL/MP                        |`sql_block`      |Major   |JDBC/JPA                       |
|SCOBOL (Screen COBOL)              |Out of scope     |—       |Web UI                         |

### 18.2 ENTER TAL/C Verification Process

For each ENTER TAL/C block, complete these steps before marking as verified:

1. Identify TAL/C routine source file from NonStop (e.g., `$DATA1.TALLIB.RATELKP`)
1. Document all parameters: names, types, pass-by-ref vs pass-by-value
1. Capture I/O golden files from NonStop system
1. Find or implement Java equivalent
1. Run behavioral parity test against golden files
1. Create `REPLACED_BY` edge in `graph_edges`
1. Set `nonstop_metadata.talVerified = true`
1. Tech lead sign-off in `review_log`

### 18.3 COMP-3 Precision Rules

|COBOL PIC          |Java Type         |Epsilon|Risk        |
|-------------------|------------------|-------|------------|
|S9(n)V9(2) COMP-3  |BigDecimal scale 2|0.01   |Currency    |
|S9(n)V9(4-8) COMP-3|BigDecimal scale n|0.005  |Rates       |
|9(n) COMP          |long              |0      |Count       |
|S9(n) COMP         |int/long          |0      |Signed count|
|9(n) DISPLAY       |String/long       |0      |Display     |

Conversion rule: **always `BigDecimal`, never `double` or `float`** for COMP-3 fields.

-----

## 19. Feature Flags

```bash
# .env

# ─── REQUIRED ───────────────────────────────────────────────────
SENTINEL_DB_HOST=localhost
SENTINEL_DB_PORT=5432
SENTINEL_DB_NAME=apollo
SENTINEL_DB_SCHEMA=sentinel
OLLAMA_BASE_URL=http://localhost:11434
OLLAMA_LLM_MODEL=codellama:34b

# ─── PATHS ──────────────────────────────────────────────────────
COBOL_SOURCE_DIR=/path/to/nonstop/cobol
COBOL_COPYBOOK_DIRS=/path/to/copybooks
JAVA_SOURCE_DIR=/path/to/converted/java
PROLEAP_JAR=~/.local/sentinel/proleap/target/proleap-cobol-parser.jar
SCIP_JAVA_BIN=~/.local/sentinel/scip/scip-java
GOLDEN_FILES_DIR=/path/to/golden-files

# ─── THRESHOLDS ─────────────────────────────────────────────────
SIMILARITY_AUTO_MAP=0.85
SIMILARITY_REVIEW=0.70
STUB_CONFIDENCE_THRESHOLD=0.50
STUB_CRITICAL_THRESHOLD=0.70
COMP3_EPSILON=0.005
DECOMMISSION_MIN_COVERAGE=95

# ─── FEATURE FLAGS (all default off except LLM edge cases) ──────
SENTINEL_ENABLE_EMBEDDINGS=false        # Phase 6 — enable if 50+ unmapped
SENTINEL_ENABLE_DRIFT_DETECTION=false   # Phase 9 — requires embeddings
SENTINEL_ENABLE_LLM_EDGE_CASES=true     # Phase 5 — edge case generation
SENTINEL_ENABLE_AUTO_RESOLVE=false      # Auto-resolve minor gaps

# ─── DASHBOARD ──────────────────────────────────────────────────
SENTINEL_DASHBOARD_PORT=3001
PANDOC_BIN=/usr/bin/pandoc
SENTINEL_REPORTS_DIR=/var/sentinel/reports
```

### 19.1 Flag Dependencies

```
ENABLE_DRIFT_DETECTION → requires ENABLE_EMBEDDINGS
ENABLE_EMBEDDINGS      → requires pgvector extension
ENABLE_EMBEDDINGS      → requires OLLAMA_EMBED_MODEL_TEXT
```

Validated at startup in `src/config/feature-flags.ts`.

-----

## 20. Repository Structure

```
sentinel/
├── README.md
├── package.json                     # Node.js 24, TypeScript
├── tsconfig.json
├── vitest.config.ts                 # Vitest 3 (^3.2.4)
├── .env.example
├── .windsurfrules
│
├── sql/
│   ├── 001_schema.sql
│   ├── 002_inventory.sql            # cobol_constructs, java_constructs
│   ├── 003_traceability.sql         # traceability_map, coverage_gaps
│   ├── 004_graph_edges.sql          # graph_edges + dependency_validation_results
│   ├── 005_behavioral.sql           # golden_files, test_cases, behavioral_results
│   ├── 006_governance.sql           # review_log, drift_alerts, decommission_log
│   ├── 007_views.sql                # materialized views
│   ├── 008_indexes.sql
│   ├── 009_stub_detection.sql       # is_stub columns, stub views
│   └── optional/
│       └── 010_embeddings.sql       # Only when ENABLE_EMBEDDINGS=true
│
├── src/
│   ├── config/
│   │   ├── database.ts
│   │   ├── ollama.ts
│   │   ├── feature-flags.ts
│   │   └── environment.ts
│   │
│   ├── parsers/
│   │   ├── cobol/
│   │   │   ├── proleap-runner.ts
│   │   │   ├── proleap-json-parser.ts
│   │   │   ├── enter-tal-extractor.ts   # ENTER TAL/C regex
│   │   │   ├── guardian-file-extractor.ts
│   │   │   ├── sqlmp-extractor.ts
│   │   │   ├── verb-classifier.ts
│   │   │   ├── asg-summarizer.ts        # ASG → NL summary
│   │   │   └── construct-emitter.ts
│   │   │
│   │   └── java/
│   │       ├── scip-runner.ts
│   │       ├── scip-parser.ts
│   │       ├── scip-call-graph.ts
│   │       ├── tree-sitter-parser.ts
│   │       ├── stub-detector.ts         # Phase 3.5 core module
│   │       └── construct-emitter.ts
│   │
│   ├── mapping/
│   │   ├── name-matcher.ts
│   │   ├── structure-matcher.ts
│   │   ├── windsurf-metadata.ts
│   │   └── mapping-engine.ts
│   │
│   ├── dependency/                      # Replaces graph/ (no AGE)
│   │   ├── edge-builder.ts
│   │   ├── chain-validator.ts           # Recursive CTE queries
│   │   ├── fanout-validator.ts
│   │   ├── data-flow-validator.ts
│   │   └── nonstop-validator.ts         # TAL/C, Guardian, TMF
│   │
│   ├── behavioral/
│   │   ├── golden-file-manager.ts
│   │   ├── java-runner.ts
│   │   ├── diff-engine.ts
│   │   ├── precision-comparator.ts
│   │   └── edge-case-generator.ts
│   │
│   ├── embedding/                       # Flag-gated
│   │   ├── enabled.ts
│   │   ├── cobol-summarizer.ts
│   │   ├── java-summarizer.ts
│   │   ├── embed-client.ts
│   │   ├── cross-match.ts
│   │   └── gap-detector.ts
│   │
│   ├── dashboard/
│   │   ├── api-server.ts
│   │   ├── coverage-engine.ts
│   │   └── routes/
│   │       ├── coverage.ts
│   │       ├── gaps.ts
│   │       ├── stubs.ts
│   │       ├── review.ts
│   │       ├── reports.ts
│   │       ├── decommission.ts
│   │       └── sse.ts
│   │
│   ├── reporting/
│   │   ├── executive-summary.ts
│   │   ├── audit-trail.ts
│   │   ├── gap-report.ts
│   │   ├── stub-report.ts
│   │   ├── behavioral-report.ts
│   │   ├── decommission-certificate.ts
│   │   └── markdown-to-pdf.ts
│   │
│   ├── governance/
│   │   ├── git-hook-handler.ts
│   │   ├── drift-detector.ts
│   │   └── decommission-gate.ts
│   │
│   └── shared/
│       ├── types.ts
│       ├── logger.ts
│       ├── timing.ts
│       └── hash.ts
│
├── proleap-wrapper/                     # Java CLI for Node.js invocation
│   ├── pom.xml
│   └── src/main/java/sentinel/
│       ├── ProLeapCli.java
│       └── AsgJsonExporter.java
│
├── scripts/
│   ├── setup-db.sh
│   ├── install-proleap.sh
│   ├── install-scip-java.sh
│   ├── install-tree-sitter.sh
│   ├── capture-golden-files.sh          # Run on NonStop system
│   ├── run-inventory.sh                 # Phases 1-2
│   ├── run-mapping.sh                   # Phase 3
│   ├── run-stub-detection.sh            # Phase 3.5
│   ├── run-dependency.sh                # Phase 4
│   ├── run-behavioral.sh                # Phase 5
│   ├── run-embedding.sh                 # Phase 6 (flag-gated)
│   ├── run-reports.sh
│   └── run-full-pipeline.sh
│
├── dashboard/
│   ├── index.html
│   ├── vendor/                          # Bundled React/Recharts for offline
│   └── src/
│       ├── App.jsx
│       ├── components/ (Gap, Stub, Program, Review, Decommission, Shared)
│       ├── hooks/ (useApi, useGaps, useStubs, useCoverage, useSSE)
│       └── utils/ (api, severity, format, filters)
│
├── hooks/
│   ├── post-commit
│   └── install-hooks.sh
│
└── tests/
    ├── parsers/
    │   ├── proleap-parser.test.ts
    │   ├── enter-tal-extractor.test.ts
    │   ├── stub-detector.test.ts        # 27 tests, 12 groups
    │   └── scip-parser.test.ts
    ├── mapping/
    ├── dependency/
    ├── behavioral/
    ├── embedding/                       # Skipped when flag off
    └── fixtures/
        ├── nonstop-cobol/
        ├── sample-java/
        ├── golden-files/
        └── tal-routines/
```

-----

## 21. Configuration & Environment

See Section 19 for full `.env` file.

**Installation order (first-time setup):**

```bash
# 1. Database
./scripts/setup-db.sh

# 2. ProLeap (COBOL parser)
./scripts/install-proleap.sh

# 3. SCIP (Java indexer)
./scripts/install-scip-java.sh

# 4. Tree-sitter (AST fingerprints)
./scripts/install-tree-sitter.sh

# 5. Verify
node dist/config/verify-installation.js
```

**First pipeline run:**

```bash
./scripts/run-full-pipeline.sh
# Opens http://localhost:3001 when complete
```

-----

## 22. Testing Strategy

|Layer        |Tool         |Coverage |What It Tests                                                             |
|-------------|-------------|---------|--------------------------------------------------------------------------|
|Unit         |Vitest 3     |90%      |Parsers, stub detector (27 tests), name matcher, diff engine, CTE builders|
|Integration  |Vitest 3     |80%      |DB interactions, ProLeap invocation, SCIP parsing, edge builder           |
|E2E          |Shell scripts|Key flows|Full pipeline on fixture data                                             |
|Dashboard API|Supertest    |85%      |All API endpoints, filter params, pagination                              |

**Stub detector test coverage (27 tests, 12 groups):**

1. Explicit stubs (TODO/FIXME, throws)
1. Silent stubs — plain comment + empty return (the hard case)
1. Empty bodies
1. Comment-only bodies
1. Unused parameters
1. Identity passthrough
1. Semantic stubs
1. COBOL structural comparison signals
1. Batch detection
1. False positives — real implementations must NOT be flagged
1. Utility function coverage
1. Edge cases and boundary conditions

**Critical: Group 10 (false positives)** — real implementations with real switch statements, multi-branch if/else, `return false` as legitimate early exits, simple getters, constructors. These must all return `isStub = false`.

-----

## 23. Timeline & Milestones

|Week             |Phase       |Deliverable                                           |
|-----------------|------------|------------------------------------------------------|
|1                |Setup       |Repo, DB schema, ProLeap + SCIP installed and verified|
|1–2              |Phase 1     |COBOL inventory complete, NonStop constructs cataloged|
|2                |Phase 2     |Java inventory complete, SCIP index loaded            |
|2–3              |Phase 3     |Deterministic mapping, unmapped count known           |
|2–3              |Phase 3.5   |Stub detection complete, stub report generated        |
|**After Phase 3**|**Decision**|**Enable embeddings if unmapped > 50**                |
|3–5              |Phase 4     |Dependency validation via recursive CTEs              |
|3–6              |Phase 5     |Golden file capture (NonStop) + Java parity tests     |
|5–6              |Phase 6     |Embedding layer (only if flag enabled)                |
|6–7              |Phase 7     |Dashboard all 5 views + SSE live updates              |
|7–8              |Phase 8     |Reports + review workflow + decommission sign-off     |
|8+               |Phase 9     |Continuous governance, git hooks, drift detection     |

**Critical dependency:** Phase 5 golden file capture requires access to the live NonStop system. Coordinate with ops team in Week 2 — this is the longest lead-time item.

**Parallelism:**

- Phases 1 and 2 run in parallel (independent codebases)
- Phases 3, 3.5, 4 run sequentially (each depends on previous)
- Phase 5 golden file capture starts in Week 3 (independent of mapping phases)
- Dashboard development starts in Week 6 (depends on API data being available)

-----

## 24. Risk Register

|#  |Risk                                              |Probability|Impact  |Mitigation                                                                                                |
|---|--------------------------------------------------|-----------|--------|----------------------------------------------------------------------------------------------------------|
|R1 |ProLeap fails on specific NonStop COBOL extensions|High       |High    |Regex fallback for ENTER TAL/C and Guardian paths. ProLeap extracts EXEC blocks as text — always parseable|
|R2 |scip-java incompatible with project’s Java version|Medium     |High    |Test in Week 1. Fallback: Tree-sitter-only Java parsing (loses cross-file refs)                           |
|R3 |COMP-3 precision loss in Java                     |High       |Critical|Dedicated COMP-3 precision tests. BigDecimal enforcement check. Epsilon-based comparison                  |
|R4 |NonStop system unavailable for golden file capture|Medium     |Critical|Coordinate with ops team Week 2. Use pre-production NonStop if production unavailable                     |
|R5 |TAL/C routine source code unavailable             |High       |High    |Document signatures from ENTER blocks. Capture golden I/O even without source                             |
|R6 |Windsurf generated > 100 stubs                    |Medium     |High    |Phase 3.5 catches all. Stubs escalate to engineering for implementation                                   |
|R7 |Deterministic mapping < 80% (many renames)        |Medium     |Medium  |Enable embedding flag early. Adjust thresholds based on corpus                                            |
|R8 |Dynamic CALL targets (variable program names)     |High       |High    |Flag as manual-review-required. Cannot trace statically                                                   |
|R9 |Recursive CTE performance on large codebase       |Low        |Medium  |Proper indexes on graph_edges. Depth limit (5 hops max) in all recursive queries                          |
|R10|Engineer fatigue on large review queue            |Medium     |Medium  |Tier 1 auto-resolves ~75%. Queue shows highest LOC×complexity first                                       |

-----

## 25. Decommission Gate — The Final Checkpoint

A COBOL program may ONLY be decommissioned when every item below is satisfied:

### Structural Layer (must be ≥ 95%)

- [ ] 100% of paragraphs have a mapped Java method (confidence ≥ medium)
- [ ] 100% of copybooks mapped to Java classes
- [ ] All data items accounted for (COMP-3 items have verified BigDecimal equivalents)
- [ ] No unmapped CALL targets

### Stub Layer (must be 0)

- [ ] Zero confirmed stubs (is_stub = TRUE, unresolved)
- [ ] All dismissed stubs have a review_log entry explaining why they are not stubs

### Dependency Layer (must be ≥ 95%)

- [ ] Zero broken call chains
- [ ] Fan-out/fan-in delta ≤ 2 for all mapped paragraphs
- [ ] All file I/O patterns preserved
- [ ] All inter-program CALL paths verified

### NonStop-Specific (must be 100%)

- [ ] All ENTER TAL calls: Java equivalent implemented + behavioral parity verified
- [ ] All ENTER C calls: Java equivalent implemented + behavioral parity verified
- [ ] All Guardian file access patterns: Java I/O semantics verified
- [ ] TMF transaction boundaries: Java transaction equivalents verified
- [ ] Pathway IPC patterns: Java equivalents verified (if applicable)
- [ ] $RECEIVE handlers: Java equivalents verified (if applicable)

### Behavioral Layer (must be ≥ 95%)

- [ ] All golden file parity tests passing
- [ ] All COMP-3 precision tests passing within epsilon
- [ ] LLM-generated edge cases executed and passing
- [ ] All failures documented and resolved

### Governance

- [ ] Zero open drift alerts
- [ ] Review log shows every gap was reviewed (no auto-accepted stubs)
- [ ] Tech Lead sign-off recorded with timestamp
- [ ] Business Owner sign-off recorded with timestamp
- [ ] Legal (Sarah) sign-off recorded with timestamp

### Documentation

- [ ] Traceability matrix exported and archived (CSV)
- [ ] Golden files archived permanently
- [ ] Behavioral test results archived
- [ ] Decommission Certificate generated with all scores and sign-offs

-----

## 26. Appendices

### Appendix A — COBOL Verb Risk Classification

|Risk    |Verbs                                   |Conversion Challenge                    |
|--------|----------------------------------------|----------------------------------------|
|Critical|ENTER TAL, ENTER C, LOCKFILE, CHECKPOINT|NonStop-specific, no standard equivalent|
|High    |COMPUTE, EVALUATE, EXEC SQL             |Precision, branch coverage, SQL/MP→JDBC |
|Medium  |PERFORM, CALL, SORT, MERGE, SEARCH      |Call chains, external dependencies      |
|Low     |MOVE, INITIALIZE, SET, DISPLAY, ACCEPT  |Straightforward data movement           |

### Appendix B — Stub Signal Quick Reference

|Signal                      |Weight|Fires When                                     |
|----------------------------|------|-----------------------------------------------|
|`empty_body`                |0.75  |Method body is `{}` or whitespace only         |
|`comments_only_body`        |0.70  |Body has comments but zero code lines          |
|`throws_unsupported`        |0.75  |`throw new UnsupportedOperationException`      |
|`throws_not_implemented`    |0.70  |`throw new RuntimeException("Not implemented")`|
|`return_bigdecimal_zero`    |0.55  |`return BigDecimal.ZERO` in minimal body       |
|`return_null`               |0.50  |`return null` in minimal body                  |
|`return_empty_string`       |0.55  |`return ""` in minimal body                    |
|`return_false`              |0.45  |`return false` in minimal body                 |
|`return_optional_empty`     |0.55  |`return Optional.empty()` in minimal body      |
|`return_empty_list`         |0.55  |`return Collections.emptyList()`               |
|`identity_passthrough`      |0.55  |Single param, body returns it unchanged        |
|`unused_parameters`         |0.50  |All params unused (100% ratio)                 |
|`extreme_loc_mismatch`      |0.55  |Java LOC < 10% of COBOL LOC                    |
|`missing_arithmetic`        |0.55  |COBOL had COMPUTE×2+, Java has no arithmetic   |
|`missing_branching`         |0.50  |COBOL had EVALUATE×1+, Java has no if/switch   |
|`missing_business_logic`    |0.65  |COBOL had 5+ logic verbs, Java has nothing     |
|`comment_heavy_minimal_code`|0.45  |Comments ≥ code lines in small method          |
|`computes_but_returns_empty`|0.65  |Has arithmetic expression but returns zero     |

Threshold: ≥ 0.50 = stub. ≥ 0.70 = critical severity.

### Appendix C — Recursive CTE Patterns Reference

```sql
-- Pattern 1: Direct edge existence
SELECT * FROM graph_edges WHERE from_id = $1 AND edge_type = 'PERFORMS';

-- Pattern 2: N-hop reachability
WITH RECURSIVE r AS (
    SELECT to_id, ARRAY[from_id, to_id] AS path, 1 AS depth
    FROM graph_edges WHERE from_id = $1 AND edge_type = 'PERFORMS'
    UNION ALL
    SELECT e.to_id, r.path || e.to_id, r.depth + 1
    FROM r JOIN graph_edges e ON e.from_id = r.to_id AND e.edge_type = 'PERFORMS'
    WHERE r.depth < 5 AND NOT (e.to_id = ANY(r.path))
)
SELECT DISTINCT to_id, depth FROM r;

-- Pattern 3: Chain isomorphism (A→B in COBOL ∴ A'→B' in Java)
WHERE NOT EXISTS (
    SELECT 1 FROM graph_edges ct1
    JOIN graph_edges ct2 ON ct2.from_id = $callee_id AND ct2.edge_type = 'CONVERTS_TO'
    JOIN graph_edges je  ON je.from_id = ct1.to_id AND je.to_id = ct2.to_id AND je.edge_type = 'JAVA_CALLS'
    WHERE ct1.from_id = $caller_id AND ct1.edge_type = 'CONVERTS_TO'
)

-- Pattern 4: Fan-out comparison
COUNT(CASE WHEN edge_type = 'PERFORMS' THEN 1 END) AS cobol_fanout,
COUNT(CASE WHEN edge_type = 'JAVA_CALLS' THEN 1 END) AS java_fanout
```

### Appendix D — ProLeap Node Types Used by Sentinel

|Node Type             |Information Extracted                   |
|----------------------|----------------------------------------|
|`ProgramUnit`         |PROGRAM-ID, divisions                   |
|`Paragraph`           |Name, lines, contained statements       |
|`DataDescriptionEntry`|Level, PIC, USAGE, REDEFINES, OCCURS    |
|`FileDescriptionEntry`|File name, access mode, organization    |
|`PerformStatement`    |Target paragraph (ASG-resolved)         |
|`CallStatement`       |Target program name, USING parameters   |
|`ComputeStatement`    |Target variable, expression             |
|`EvaluateStatement`   |Subject, WHEN branches                  |
|`ReadStatement`       |File name, INVALID KEY handler          |
|`WriteStatement`      |File name, FROM clause                  |
|`ExecSqlStatement`    |SQL text (as string for post-processing)|

### Appendix E — scip-java Symbol Format

```
Symbols follow pattern: "package/ClassName#methodName(paramTypes)."
Example: "com/bank/CalcInterest#calculateDailyRate(String,BigDecimal)."

Relationship types used:
  is_implementation  → class implements interface
  is_type_definition → field type reference
  is_reference       → method call

Occurrence roles (bitmask):
  1  = Definition
  2  = Reference
  4  = Implementation
```

-----

*Project Sentinel — Master Implementation Plan v4.0*
*HP NonStop COBOL → Java · ProLeap + SCIP + Stub Detection + Recursive CTEs + Optional Embeddings*
*RHEL 128-core · Single Postgres Database · Zero External API Dependencies*
*April 2026*

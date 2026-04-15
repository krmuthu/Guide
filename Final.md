# Project Sentinel — Final Implementation Plan

## HP NonStop COBOL → Java Conversion Traceability & Coverage System

**Version:** 2.0 — Final
**Date:** April 15, 2026
**Status:** Ready for Development
**Runtime:** RHEL 128-core, user-space only, no sudo, no external API dependencies
**Source Dialect:** HP NonStop (Tandem) COBOL — Guardian/OSS, NMCOBOL, COBOL85 compilers
**Target:** Java (converted via Windsurf)

-----

## Table of Contents

1. [Executive Summary](#1-executive-summary)
1. [Architecture Overview](#2-architecture-overview)
1. [Technology Decisions — Final](#3-technology-decisions--final)
1. [Repository Structure](#4-repository-structure)
1. [Database Schema — Complete DDL](#5-database-schema--complete-ddl)
1. [Phase 1 — COBOL Inventory Engine (ProLeap)](#6-phase-1--cobol-inventory-engine-proleap)
1. [Phase 2 — Java Inventory Engine (SCIP + Tree-sitter)](#7-phase-2--java-inventory-engine-scip--tree-sitter)
1. [Phase 3 — Deterministic Mapping Engine](#8-phase-3--deterministic-mapping-engine)
1. [Phase 4 — Semantic Embedding Layer (Dual-Model)](#9-phase-4--semantic-embedding-layer-dual-model)
1. [Phase 5 — Graph Dependency Validation (Apache AGE)](#10-phase-5--graph-dependency-validation-apache-age)
1. [Phase 6 — Behavioral Validation (Golden File Parity)](#11-phase-6--behavioral-validation-golden-file-parity)
1. [Phase 7 — Gap Analysis Dashboard](#12-phase-7--gap-analysis-dashboard)
1. [Phase 8 — Reporting & Review Workflow](#13-phase-8--reporting--review-workflow)
1. [Phase 9 — Continuous Governance](#14-phase-9--continuous-governance)
1. [HP NonStop-Specific Handling](#15-hp-nonstop-specific-handling)
1. [Configuration & Environment](#16-configuration--environment)
1. [Testing Strategy](#17-testing-strategy)
1. [Timeline & Milestones](#18-timeline--milestones)
1. [Risk Register](#19-risk-register)
1. [Appendix A — COBOL Verb Classification & Risk](#20-appendix-a--cobol-verb-classification--risk)
1. [Appendix B — SCIP Java Index Schema](#21-appendix-b--scip-java-index-schema)
1. [Appendix C — ProLeap ASG Node Types](#22-appendix-c--proleap-asg-node-types)
1. [Appendix D — Decommission Gate Checklist](#23-appendix-d--decommission-gate-checklist)
1. [Appendix E — Report Templates](#24-appendix-e--report-templates)

-----

## 1. Executive Summary

### 1.1 Problem

We converted HP NonStop (Tandem) COBOL applications to Java using Windsurf. We need auditable, fool-proof evidence that every COBOL construct has been faithfully converted — covering structural mapping, semantic similarity, call-graph integrity, and behavioral I/O parity.

HP NonStop COBOL introduces dialect-specific challenges not present in standard IBM mainframe COBOL: ENTER TAL/C interop calls, Guardian file system I/O (Enscribe), TMF transaction management, Pathway/TS server classes, $RECEIVE IPC, and SQL/MP embedded SQL.

### 1.2 Solution

Project Sentinel is a standalone system that provides four independent layers of verification, each catching what the previous layers miss:

|Layer     |Weight|Method                             |Catches                                                  |
|----------|------|-----------------------------------|---------------------------------------------------------|
|Structural|20%   |ProLeap ASG + name/pattern matching|Direct 1:1 mappings, naming convention matches           |
|Semantic  |20%   |Dual-model embeddings + pgvector   |Refactored logic, renamed constructs, fuzzy matches      |
|Graph     |25%   |Apache AGE call-chain isomorphism  |Broken call chains, missing dependencies, data flow gaps |
|Behavioral|35%   |Golden file parity testing         |Logic errors, edge case failures, numeric precision drift|

### 1.3 Key Technology Decisions

|Decision          |Choice                           |Rationale                                                                                        |
|------------------|---------------------------------|-------------------------------------------------------------------------------------------------|
|COBOL parser      |**ProLeap** (ANTLR4)             |Native Tandem format support, AST + ASG (semantic graph), COPY preprocessing, EXEC SQL extraction|
|Java indexer      |**scip-java**                    |Compiler-grade symbol resolution, cross-file references, type hierarchy                          |
|Java AST          |**Tree-sitter** (supplementary)  |AST fingerprints for structural pattern matching                                                 |
|COBOL embeddings  |**snowflake-arctic-embed-l-v2.0**|COBOL reads like English prose; text embedder handles it better than code embedder               |
|Java embeddings   |**nomic-embed-code**             |Trained on Java; strong at code semantics                                                        |
|Behavioral testing|**Golden file parity**           |Cannot run GnuCOBOL for NonStop dialect; capture I/O from live NonStop system instead            |

### 1.4 Success Criteria

|Metric              |Target                                         |
|--------------------|-----------------------------------------------|
|Structural coverage |100% of COBOL paragraphs mapped                |
|Semantic similarity |Average ≥ 0.85 across all mappings             |
|Graph integrity     |0 broken call chains                           |
|Behavioral parity   |100% of test cases produce identical output    |
|ENTER TAL/C coverage|Every TAL/C call has a verified Java equivalent|
|Audit readiness     |Every mapping has verifier + timestamp         |
|Decommission gate   |All 4 layers ≥ 95% individually                |

### 1.5 Relationship to Project Apollo

Sentinel is standalone. It reuses Apollo’s infrastructure (Postgres/pgvector, Apache AGE, Ollama, Node.js orchestrator) but maintains its own `sentinel` schema, `sentinel_graph` namespace, and codebase.

-----

## 2. Architecture Overview

```
┌─────────────────────────────────────────────────────────────────────────┐
│                        PROJECT SENTINEL                                │
│                 HP NonStop COBOL → Java Traceability                   │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                        │
│  ┌──────────────────────┐          ┌───────────────────────┐           │
│  │  HP NonStop COBOL     │          │  Converted Java       │           │
│  │  (Tandem dialect)     │          │  (Windsurf output)    │           │
│  └──────────┬────────────┘          └──────────┬────────────┘           │
│             │                                   │                       │
│     ┌───────▼────────┐               ┌──────────▼──────────┐           │
│     │  ProLeap        │               │  scip-java           │           │
│     │  (ANTLR4)       │               │  (compiler-grade)    │           │
│     │                 │               │                      │           │
│     │  ► AST          │               │  ► Symbols           │           │
│     │  ► ASG (semantic│               │  ► References        │           │
│     │    graph: data  │               │  ► Call graph         │           │
│     │    flow, control│               │  ► Type hierarchy     │           │
│     │    flow)        │               │                      │           │
│     │  ► COPY preproc │               ├──────────────────────┤           │
│     │  ► EXEC SQL     │               │  Tree-sitter Java    │           │
│     │    extraction   │               │  (supplementary)     │           │
│     │  ► Tandem fmt   │               │                      │           │
│     │    native       │               │  ► AST fingerprints  │           │
│     └───────┬─────────┘               │  ► Complexity scores │           │
│             │                          └──────────┬──────────┘           │
│             │                                     │                     │
│     ┌───────▼─────────┐               ┌───────────▼──────────┐          │
│     │  ENTER TAL/C     │               │                      │          │
│     │  Regex Extractor │               │                      │          │
│     │  (post-ProLeap)  │               │                      │          │
│     └───────┬──────────┘               │                      │          │
│             │                          │                      │          │
│  ┌──────────▼──────────────────────────▼──────────────────┐   │          │
│  │                    POSTGRES                             │   │          │
│  │  sentinel schema                                        │   │          │
│  │  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐  │   │          │
│  │  │cobol_         │  │java_          │  │traceability_ │  │   │          │
│  │  │constructs     │  │constructs     │  │map           │  │   │          │
│  │  └──────┬────────┘  └──────┬────────┘  └──────────────┘  │   │          │
│  │         │                  │                              │   │          │
│  │  ┌──────▼────────┐  ┌─────▼─────────┐                    │   │          │
│  │  │cobol_          │  │java_           │   pgvector         │   │          │
│  │  │embeddings      │  │embeddings      │   cosine search    │   │          │
│  │  │(arctic 1024d)  │  │(nomic 3584d)   │                    │   │          │
│  │  └───────────────┘  └────────────────┘                    │   │          │
│  └───────────────────────────────────────────────────────────┘   │          │
│                                                                  │          │
│  ┌────────────────────────────────────────────────────────────┐  │          │
│  │                    APACHE AGE                               │  │          │
│  │  sentinel_graph                                             │  │          │
│  │                                                              │  │          │
│  │  CobolProgram ──PERFORMS──► CobolParagraph                    │  │          │
│  │       │                         │                             │  │          │
│  │       │CALLS                    │CONVERTS_TO                  │  │          │
│  │       ▼                         ▼                             │  │          │
│  │  CobolProgram              JavaMethod                         │  │          │
│  │                                 │                             │  │          │
│  │  CobolParagraph ──READS──► CobolFile     JavaMethod ──CALLS──►│  │          │
│  │                                                               │  │          │
│  │  TalRoutine ◄──ENTER_TAL── CobolParagraph                    │  │          │
│  └────────────────────────────────────────────────────────────┘  │          │
│                                                                  │          │
│  ┌────────────────────────────────────────────────────────────┐  │          │
│  │              BEHAVIORAL VALIDATION                          │  │          │
│  │                                                              │  │          │
│  │  Golden Files ───► Java Runner ───► Diff Engine              │  │          │
│  │  (captured from     (Vitest 3)      (byte-level +            │  │          │
│  │   live NonStop)                      COMP-3 precision)       │  │          │
│  └────────────────────────────────────────────────────────────┘  │          │
│                                                                  │          │
│  ┌────────────────────────────────────────────────────────────┐  │          │
│  │              DASHBOARD + REPORTING                          │  │          │
│  │                                                              │  │          │
│  │  Gap Analysis ◄── Coverage Engine ──► Stakeholder Reports   │  │          │
│  │  (React SPA)       (materialized      (MD → PDF via Pandoc)  │  │          │
│  │                     views)                                    │  │          │
│  └────────────────────────────────────────────────────────────┘  │          │
└─────────────────────────────────────────────────────────────────────────┘
```

### 2.1 Defense-in-Depth Coverage Model

```
 ┌─────────────────────────────────────────────────────────────┐
 │                  COMPOSITE SCORE                             │
 │  0.20×Structural + 0.20×Semantic + 0.25×Graph + 0.35×Behav  │
 │              Decommission Gate: Each ≥ 95%                   │
 └──────┬──────────┬───────────┬──────────┬────────────────────┘
        │          │           │          │
  Layer 1     Layer 2     Layer 3     Layer 4
  Structural  Semantic    Graph       Behavioral
  (20%)       (20%)       (25%)       (35%)
  ProLeap     Dual-model  AGE call    Golden file
  ASG + name  embeddings  chain       parity +
  matching    + pgvector  isomorphism COMP-3 test
        │          │           │          │
        ▼          ▼           ▼          ▼
  Catches:    Catches:    Catches:    Catches:
  Direct 1:1  Refactored  Broken      Logic errors
  mappings    logic       chains      Edge cases
  Convention  Renamed     Missing     Numeric
  matches     constructs  deps        precision
              Fuzzy       Data flow   I/O format
              matches     gaps        diffs
```

-----

## 3. Technology Decisions — Final

### 3.1 COBOL Parsing — ProLeap (ANTLR4)

**Why ProLeap over Tree-sitter for HP NonStop:**

ProLeap is an ANTLR4-based COBOL parser that produces both an AST and an Abstract Semantic Graph (ASG). The ASG provides data flow and control flow information — resolved variable references, not just syntax structure.

Critically, ProLeap has native Tandem source format support:

```java
CobolSourceFormatEnum format =
    CobolSourceFormatEnum.TANDEM;
```

|Capability             |Tree-sitter  |ProLeap                       |
|-----------------------|-------------|------------------------------|
|Tandem source format   |No           |Yes (native)                  |
|Abstract Semantic Graph|No (AST only)|Yes (data flow + control flow)|
|COPY preprocessing     |Manual       |Built-in                      |
|EXEC SQL extraction    |No           |Yes (extracted as text blocks)|
|ENTER TAL/C            |No           |Extracted as text blocks      |
|Variable resolution    |No           |Yes                           |
|NIST test suite        |Yes          |Yes                           |
|Language               |C/WASM       |Java                          |

**What ProLeap extracts that Tree-sitter cannot:**

- Resolved variable access: which paragraphs read/modify which data items
- Control flow: which paragraphs PERFORM which other paragraphs
- Copy expansion: inline COPY resolution before parsing
- EXEC SQL blocks: extracted as text for separate analysis
- ENTER TAL/C blocks: extracted as text for regex post-processing

**Installation (user-space, Java-based):**

```bash
#!/bin/bash
# scripts/install-proleap.sh
set -euo pipefail

INSTALL_DIR="$HOME/.local/sentinel/proleap"
mkdir -p "$INSTALL_DIR"

git clone --depth 1 https://github.com/uwol/proleap-cobol-parser.git \
    "$INSTALL_DIR/proleap-cobol-parser"

cd "$INSTALL_DIR/proleap-cobol-parser"
mvn clean install -DskipTests

echo "ProLeap installed. JAR at target/proleap-cobol-parser-*.jar"
```

**Integration with Node.js orchestrator:**

ProLeap runs as a Java process invoked from Node.js. The orchestrator calls ProLeap via a thin Java CLI wrapper that outputs JSON:

```bash
java -jar proleap-cobol-parser.jar \
    --format TANDEM \
    --input /path/to/program.cbl \
    --output-format json \
    > constructs.json
```

The Node.js orchestrator reads `constructs.json` and inserts into `cobol_constructs`.

### 3.2 Java Parsing — scip-java + Tree-sitter

**scip-java** provides compiler-grade symbol resolution:

- Run at the root of the Gradle/Maven build: `scip-java index`
- Produces `index.scip` (protobuf binary) containing every symbol definition, reference, and relationship
- TypeScript bindings available via protobuf schema
- Supports Java 8, 11, 17, 21

**Tree-sitter Java** supplements with raw AST structure for fingerprint-based pattern matching in Phase 3.

|Data                         |Source     |Purpose                             |
|-----------------------------|-----------|------------------------------------|
|Symbol definitions with types|SCIP       |Populate java_constructs.scip_symbol|
|Cross-file references        |SCIP       |Populate scip_references            |
|Method call graph            |SCIP       |Build Java subgraph in AGE          |
|Type hierarchy               |SCIP       |EXTENDS/IMPLEMENTS edges            |
|Raw source extraction        |Tree-sitter|java_constructs.raw_source          |
|AST fingerprints             |Tree-sitter|Structural pattern matching         |
|Complexity scoring           |Tree-sitter|java_constructs.complexity          |

### 3.3 Embedding — Dual-Model Strategy

No embedding model is trained on COBOL. The best approach:

|Side |Model                         |Dimensions|Rationale                                                                                                                  |
|-----|------------------------------|----------|---------------------------------------------------------------------------------------------------------------------------|
|COBOL|snowflake-arctic-embed-l-v2.0 |1024      |COBOL reads like English prose. Text embedder understands semantic intent better than a code model that’s never seen COBOL.|
|Java |nomic-embed-code (GGUF Q4_K_M)|3584      |Trained on Java via CoRNStack dataset. Strong at code semantics.                                                           |

**Cross-language matching strategy:**

For COBOL→Java similarity, we cannot directly compare 1024-dim and 3584-dim vectors. Instead:

1. Use ProLeap ASG to generate a **structured natural language summary** of each COBOL paragraph: “This paragraph calculates daily interest rate using day-count conventions (ACT/360, ACT/365, 30/360), handles invalid rate types, applies minimum threshold, and performs compounding.”
1. Embed that summary using `snowflake-arctic` (1024 dims)
1. Generate a **pseudocode description** of each Java method using Ollama LLM
1. Embed that description using `snowflake-arctic` (1024 dims)
1. Compare COBOL summary embedding vs Java description embedding in the same 1024-dim space

This bridges the language gap by comparing **intent descriptions** in the same vector space, rather than raw COBOL syntax against raw Java syntax.

### 3.4 Behavioral Testing — Golden File Parity

**Why not GnuCOBOL:** HP NonStop COBOL uses Tandem-specific runtime libraries, Guardian file system, ENTER TAL/C interop, and TMF transactions. GnuCOBOL cannot replicate this environment.

**Golden file approach:**

1. Before decommission, capture production-like I/O from the live NonStop system: input files, output files, return codes, database state changes
1. Feed identical inputs to the Java application
1. Compare Java output against golden (NonStop-captured) output byte-for-byte
1. COMP-3 fields get epsilon-based comparison

### 3.5 Graph Database — Apache AGE

Separate graph namespace: `sentinel_graph`. Additional node types for NonStop-specific constructs: `TalRoutine`, `CRoutine`, `GuardianFile`, `PathwayServer`.

### 3.6 Test Framework — Vitest 3 (`^3.2.4`)

Backend-only, Node.js 24. Consistent with Apollo.

-----

## 4. Repository Structure

```
sentinel/
├── README.md
├── package.json
├── tsconfig.json
├── vitest.config.ts
├── .env.example
├── .windsurfrules
│
├── sql/
│   ├── 001_schema.sql
│   ├── 002_inventory.sql
│   ├── 003_traceability.sql
│   ├── 004_embeddings.sql
│   ├── 005_graph.sql
│   ├── 006_behavioral.sql
│   ├── 007_governance.sql
│   ├── 008_views.sql
│   └── 009_indexes.sql
│
├── src/
│   ├── config/
│   │   ├── database.ts
│   │   ├── ollama.ts
│   │   └── environment.ts
│   │
│   ├── parsers/
│   │   ├── cobol/
│   │   │   ├── proleap-runner.ts        # Invokes ProLeap JAR
│   │   │   ├── proleap-json-parser.ts   # Parses ProLeap JSON output
│   │   │   ├── enter-tal-extractor.ts   # Regex for ENTER TAL/C blocks
│   │   │   ├── guardian-file-extractor.ts # $DATA.SUBVOL.FILE patterns
│   │   │   ├── sqlmp-extractor.ts       # SQL/MP specific syntax
│   │   │   ├── verb-classifier.ts       # COBOL verb classification
│   │   │   ├── asg-summarizer.ts        # Generates NL summaries from ASG
│   │   │   └── construct-emitter.ts     # Writes to cobol_constructs
│   │   │
│   │   └── java/
│   │       ├── scip-runner.ts           # Invokes scip-java CLI
│   │       ├── scip-parser.ts           # Parses index.scip protobuf
│   │       ├── scip-call-graph.ts       # Extracts call graph from SCIP
│   │       ├── tree-sitter-parser.ts    # AST fingerprints
│   │       └── construct-emitter.ts     # Writes to java_constructs
│   │
│   ├── mapping/
│   │   ├── name-matcher.ts
│   │   ├── structure-matcher.ts
│   │   ├── windsurf-metadata.ts
│   │   └── mapping-engine.ts
│   │
│   ├── embedding/
│   │   ├── cobol-summarizer.ts          # ASG → NL summary → arctic embed
│   │   ├── java-summarizer.ts           # LLM pseudocode → arctic embed
│   │   ├── embed-client.ts             # Ollama embedding calls
│   │   ├── cross-match.ts              # Cross-language NN search
│   │   └── gap-detector.ts
│   │
│   ├── graph/
│   │   ├── cobol-graph-builder.ts
│   │   ├── java-graph-builder.ts
│   │   ├── scip-graph-enricher.ts
│   │   ├── nonstop-graph-builder.ts    # TAL/C, Guardian, TMF nodes
│   │   ├── cross-linker.ts
│   │   └── chain-validator.ts
│   │
│   ├── behavioral/
│   │   ├── golden-file-manager.ts      # Manages NonStop-captured I/O
│   │   ├── java-runner.ts
│   │   ├── diff-engine.ts
│   │   ├── precision-comparator.ts     # COMP-3 vs BigDecimal
│   │   ├── edge-case-generator.ts      # LLM-powered
│   │   └── nonstop-capture-guide.ts    # Generates capture scripts
│   │
│   ├── dashboard/
│   │   ├── coverage-engine.ts
│   │   ├── api-server.ts
│   │   └── gap-detail-api.ts
│   │
│   ├── reporting/
│   │   ├── executive-summary.ts
│   │   ├── audit-trail.ts
│   │   ├── gap-report.ts
│   │   ├── behavioral-report.ts
│   │   ├── decommission-certificate.ts
│   │   └── markdown-to-pdf.ts          # Pandoc wrapper
│   │
│   ├── governance/
│   │   ├── git-hook-handler.ts
│   │   ├── drift-detector.ts
│   │   └── decommission-gate.ts
│   │
│   └── shared/
│       ├── logger.ts
│       ├── timing.ts
│       ├── hash.ts
│       └── types.ts
│
├── proleap-wrapper/
│   ├── pom.xml
│   └── src/main/java/sentinel/
│       ├── ProLeapCli.java             # CLI wrapper for Node.js
│       └── AsgJsonExporter.java        # ASG → JSON serializer
│
├── scripts/
│   ├── setup-db.sh
│   ├── install-proleap.sh
│   ├── install-scip-java.sh
│   ├── install-tree-sitter.sh
│   ├── capture-golden-files.sh         # Run on NonStop system
│   ├── run-inventory.sh
│   ├── run-mapping.sh
│   ├── run-embedding.sh
│   ├── run-graph-validation.sh
│   ├── run-behavioral.sh
│   ├── run-reports.sh
│   └── run-full-pipeline.sh
│
├── hooks/
│   ├── post-commit
│   └── install-hooks.sh
│
├── dashboard/
│   └── sentinel-gap-detail.jsx         # React gap analysis UI
│
├── tests/
│   ├── parsers/
│   │   ├── proleap-parser.test.ts
│   │   ├── enter-tal-extractor.test.ts
│   │   ├── scip-parser.test.ts
│   │   └── asg-summarizer.test.ts
│   ├── mapping/
│   │   ├── name-matcher.test.ts
│   │   └── structure-matcher.test.ts
│   ├── embedding/
│   │   └── cross-match.test.ts
│   ├── graph/
│   │   └── chain-validator.test.ts
│   ├── behavioral/
│   │   ├── diff-engine.test.ts
│   │   └── precision-comparator.test.ts
│   └── fixtures/
│       ├── nonstop-cobol/              # HP NonStop COBOL samples
│       ├── sample-java/
│       ├── sample-scip/
│       ├── golden-files/
│       └── tal-routines/               # TAL routine signatures
│
└── docs/
    ├── architecture.md
    ├── nonstop-dialect-guide.md
    ├── proleap-integration.md
    ├── golden-file-capture-runbook.md
    ├── review-workflow.md
    ├── runbook.md
    └── decommission-checklist.md
```

-----

## 5. Database Schema — Complete DDL

```sql
-- sql/001_schema.sql
CREATE SCHEMA IF NOT EXISTS sentinel;
SET search_path TO sentinel, public;
SELECT create_graph('sentinel_graph');
```

```sql
-- sql/002_inventory.sql
SET search_path TO sentinel, public;

CREATE TABLE cobol_constructs (
    id                    SERIAL PRIMARY KEY,
    file_path             TEXT NOT NULL,
    guardian_file_name    TEXT,                -- $DATA1.SRCLIB.CALCINT
    program_id            TEXT,
    construct_type        TEXT NOT NULL
        CHECK (construct_type IN (
            'program', 'section', 'paragraph', 'copybook',
            'data_item', 'file_descriptor', 'sql_block',
            'call_target', 'perform_target', 'condition_name',
            'enter_tal', 'enter_c', 'pathway_server',
            'receive_block', 'checkpoint'
        )),
    construct_name        TEXT NOT NULL,
    start_line            INT NOT NULL,
    end_line              INT NOT NULL,
    loc                   INT NOT NULL,
    parent_construct_id   INT REFERENCES cobol_constructs(id),
    raw_source            TEXT,
    asg_summary           TEXT,               -- ProLeap ASG NL summary
    content_hash          TEXT NOT NULL,
    complexity            INT DEFAULT 1,
    metadata              JSONB DEFAULT '{}',
    -- NonStop-specific metadata
    nonstop_metadata      JSONB DEFAULT '{}',
        -- { talRoutine: "RATE_TABLE_LOOKUP",
        --   talSourceFile: "$DATA1.TALLIB.RATELKP",
        --   talParams: [...],
        --   guardianFileName: "$DATA2.ACCTDB.ACCTFILE",
        --   tmfOperation: "COMMIT",
        --   pathwayServerClass: "PAYMENT-SVC" }
    created_at            TIMESTAMPTZ DEFAULT NOW(),
    updated_at            TIMESTAMPTZ DEFAULT NOW()
);

CREATE INDEX idx_cc_type ON cobol_constructs(construct_type);
CREATE INDEX idx_cc_program ON cobol_constructs(program_id);
CREATE INDEX idx_cc_hash ON cobol_constructs(content_hash);
CREATE INDEX idx_cc_nonstop ON cobol_constructs
    USING gin (nonstop_metadata jsonb_path_ops);
CREATE UNIQUE INDEX idx_cc_dedup
    ON cobol_constructs(file_path, construct_type, construct_name, start_line);

CREATE TABLE java_constructs (
    id                    SERIAL PRIMARY KEY,
    file_path             TEXT NOT NULL,
    package_name          TEXT,
    class_name            TEXT,
    construct_type        TEXT NOT NULL
        CHECK (construct_type IN (
            'class', 'interface', 'enum', 'method', 'constructor',
            'field', 'import', 'annotation', 'exception_handler',
            'jdbc_call', 'inner_class', 'lambda', 'jni_call'
        )),
    construct_name        TEXT NOT NULL,
    signature             TEXT,
    start_line            INT NOT NULL,
    end_line              INT NOT NULL,
    loc                   INT NOT NULL,
    parent_construct_id   INT REFERENCES java_constructs(id),
    raw_source            TEXT,
    pseudocode_summary    TEXT,               -- LLM-generated description
    content_hash          TEXT NOT NULL,
    complexity            INT DEFAULT 1,
    scip_symbol           TEXT,
    metadata              JSONB DEFAULT '{}',
    created_at            TIMESTAMPTZ DEFAULT NOW(),
    updated_at            TIMESTAMPTZ DEFAULT NOW()
);

CREATE INDEX idx_jc_type ON java_constructs(construct_type);
CREATE INDEX idx_jc_class ON java_constructs(class_name);
CREATE INDEX idx_jc_scip ON java_constructs(scip_symbol);
CREATE UNIQUE INDEX idx_jc_dedup
    ON java_constructs(file_path, construct_type, construct_name, start_line);

CREATE TABLE scip_references (
    id              SERIAL PRIMARY KEY,
    from_symbol     TEXT NOT NULL,
    to_symbol       TEXT NOT NULL,
    from_file       TEXT NOT NULL,
    to_file         TEXT,
    line_number     INT NOT NULL,
    reference_kind  TEXT NOT NULL
        CHECK (reference_kind IN (
            'definition', 'reference', 'implementation', 'type_definition'
        )),
    created_at      TIMESTAMPTZ DEFAULT NOW()
);

CREATE INDEX idx_sr_from ON scip_references(from_symbol);
CREATE INDEX idx_sr_to ON scip_references(to_symbol);

CREATE TABLE conversion_baseline (
    id              SERIAL PRIMARY KEY,
    snapshot_date   DATE DEFAULT CURRENT_DATE,
    side            TEXT NOT NULL CHECK (side IN ('cobol', 'java')),
    metric_name     TEXT NOT NULL,
    metric_value    NUMERIC,
    breakdown       JSONB DEFAULT '{}',
    UNIQUE (snapshot_date, side, metric_name)
);
```

```sql
-- sql/003_traceability.sql
SET search_path TO sentinel, public;

CREATE TABLE traceability_map (
    id                  SERIAL PRIMARY KEY,
    cobol_construct_id  INT NOT NULL REFERENCES cobol_constructs(id),
    java_construct_id   INT NOT NULL REFERENCES java_constructs(id),
    mapping_type        TEXT NOT NULL
        CHECK (mapping_type IN (
            'name_match', 'structure_match', 'semantic_match',
            'manual', 'windsurf_metadata', 'scip_cross_ref',
            'asg_data_flow', 'nonstop_interop'
        )),
    confidence          TEXT NOT NULL
        CHECK (confidence IN ('high', 'medium', 'low', 'unmatched')),
    similarity_score    NUMERIC,
    mapping_detail      JSONB DEFAULT '{}',
    notes               TEXT,
    verified_by         TEXT,
    verified_at         TIMESTAMPTZ,
    created_at          TIMESTAMPTZ DEFAULT NOW(),
    updated_at          TIMESTAMPTZ DEFAULT NOW(),
    UNIQUE (cobol_construct_id, java_construct_id, mapping_type)
);

CREATE INDEX idx_tm_cobol ON traceability_map(cobol_construct_id);
CREATE INDEX idx_tm_java ON traceability_map(java_construct_id);
CREATE INDEX idx_tm_confidence ON traceability_map(confidence);

CREATE TABLE coverage_gaps (
    id                  SERIAL PRIMARY KEY,
    side                TEXT NOT NULL
        CHECK (side IN ('cobol_unmapped', 'java_orphan')),
    construct_id        INT NOT NULL,
    construct_type      TEXT NOT NULL,
    construct_name      TEXT NOT NULL,
    program_or_class    TEXT NOT NULL,
    loc                 INT NOT NULL,
    complexity          INT DEFAULT 1,
    nearest_match_id    INT,
    nearest_match_name  TEXT,
    nearest_similarity  NUMERIC,
    gap_type            TEXT
        CHECK (gap_type IN (
            'missing_conversion', 'partial_conversion',
            'logic_drift', 'dead_code', 'added_code'
        )),
    severity            TEXT NOT NULL
        CHECK (severity IN ('critical', 'major', 'minor')),
    -- NonStop-specific
    is_enter_tal        BOOLEAN DEFAULT FALSE,
    is_enter_c          BOOLEAN DEFAULT FALSE,
    is_guardian_io       BOOLEAN DEFAULT FALSE,
    is_tmf              BOOLEAN DEFAULT FALSE,
    is_pathway          BOOLEAN DEFAULT FALSE,
    is_sqlmp            BOOLEAN DEFAULT FALSE,
    -- Resolution
    resolution          TEXT,
    resolved_by         TEXT,
    resolved_at         TIMESTAMPTZ,
    created_at          TIMESTAMPTZ DEFAULT NOW()
);

CREATE INDEX idx_cg_severity ON coverage_gaps(severity);
CREATE INDEX idx_cg_nonstop ON coverage_gaps(is_enter_tal, is_enter_c, is_guardian_io, is_tmf);
```

```sql
-- sql/004_embeddings.sql
SET search_path TO sentinel, public;

-- COBOL uses snowflake-arctic (1024 dims) on NL summaries
CREATE TABLE cobol_embeddings (
    id              SERIAL PRIMARY KEY,
    construct_id    INT NOT NULL REFERENCES cobol_constructs(id) ON DELETE CASCADE,
    embedding       vector(1024) NOT NULL,
    summary_text    TEXT NOT NULL,            -- NL summary from ASG
    chunk_tokens    INT,
    model           TEXT DEFAULT 'snowflake-arctic-embed-l-v2.0',
    created_at      TIMESTAMPTZ DEFAULT NOW(),
    UNIQUE (construct_id)
);

CREATE INDEX idx_ce_vec ON cobol_embeddings
    USING ivfflat (embedding vector_cosine_ops)
    WITH (lists = 100);

-- Java uses snowflake-arctic (1024 dims) on pseudocode descriptions
-- (same vector space as COBOL for cross-language comparison)
CREATE TABLE java_embeddings (
    id              SERIAL PRIMARY KEY,
    construct_id    INT NOT NULL REFERENCES java_constructs(id) ON DELETE CASCADE,
    embedding       vector(1024) NOT NULL,
    summary_text    TEXT NOT NULL,            -- LLM pseudocode description
    chunk_tokens    INT,
    model           TEXT DEFAULT 'snowflake-arctic-embed-l-v2.0',
    created_at      TIMESTAMPTZ DEFAULT NOW(),
    UNIQUE (construct_id)
);

CREATE INDEX idx_je_vec ON java_embeddings
    USING ivfflat (embedding vector_cosine_ops)
    WITH (lists = 100);
```

```sql
-- sql/005_graph.sql
SET search_path TO sentinel, public;

CREATE TABLE graph_validation_results (
    id                  SERIAL PRIMARY KEY,
    validation_type     TEXT NOT NULL
        CHECK (validation_type IN (
            'call_chain', 'fan_out', 'fan_in', 'data_flow',
            'copybook_dependency', 'inter_program_call',
            'sql_equivalence', 'file_io_pattern',
            'enter_tal_mapping', 'enter_c_mapping',
            'tmf_transaction', 'pathway_ipc',
            'receive_handler', 'guardian_file_access'
        )),
    cobol_source        TEXT NOT NULL,
    java_target         TEXT,
    status              TEXT NOT NULL
        CHECK (status IN ('pass', 'fail', 'warning', 'skipped')),
    detail              JSONB DEFAULT '{}',
    run_id              TEXT NOT NULL,
    run_at              TIMESTAMPTZ DEFAULT NOW()
);

CREATE INDEX idx_gvr_status ON graph_validation_results(status);
CREATE INDEX idx_gvr_run ON graph_validation_results(run_id);
```

```sql
-- sql/006_behavioral.sql
SET search_path TO sentinel, public;

CREATE TABLE golden_files (
    id              SERIAL PRIMARY KEY,
    cobol_program   TEXT NOT NULL,
    test_name       TEXT NOT NULL,
    input_file_path TEXT NOT NULL,
    input_hash      TEXT NOT NULL,
    output_file_path TEXT NOT NULL,
    output_hash     TEXT NOT NULL,
    return_code     INT,
    db_state_before JSONB,
    db_state_after  JSONB,
    captured_from   TEXT DEFAULT 'nonstop_production',
    captured_at     TIMESTAMPTZ NOT NULL,
    notes           TEXT,
    created_at      TIMESTAMPTZ DEFAULT NOW()
);

CREATE TABLE test_cases (
    id              SERIAL PRIMARY KEY,
    cobol_program   TEXT NOT NULL,
    java_class      TEXT NOT NULL,
    test_name       TEXT NOT NULL,
    golden_file_id  INT REFERENCES golden_files(id),
    input_data      JSONB NOT NULL,
    expected_output JSONB,
    category        TEXT NOT NULL
        CHECK (category IN (
            'regression', 'edge_case', 'boundary',
            'negative', 'smoke', 'llm_generated',
            'comp3_precision', 'guardian_io', 'tmf_transaction'
        )),
    source          TEXT NOT NULL
        CHECK (source IN (
            'production', 'qa', 'generated', 'manual',
            'llm', 'nonstop_capture'
        )),
    priority        TEXT DEFAULT 'medium'
        CHECK (priority IN ('critical', 'high', 'medium', 'low')),
    active          BOOLEAN DEFAULT TRUE,
    created_at      TIMESTAMPTZ DEFAULT NOW()
);

CREATE TABLE behavioral_results (
    id                  SERIAL PRIMARY KEY,
    test_case_id        INT NOT NULL REFERENCES test_cases(id),
    golden_file_id      INT REFERENCES golden_files(id),
    cobol_program       TEXT NOT NULL,
    java_class          TEXT NOT NULL,
    input_hash          TEXT NOT NULL,
    golden_output_hash  TEXT,               -- from NonStop capture
    java_output_hash    TEXT,
    golden_return_code  INT,
    java_return_code    INT,
    match               BOOLEAN NOT NULL,
    diff_summary        TEXT,
    diff_detail         JSONB DEFAULT '{}',
    precision_issues    JSONB DEFAULT '{}', -- COMP-3 specific diffs
    java_duration_ms    INT,
    run_id              TEXT NOT NULL,
    run_at              TIMESTAMPTZ DEFAULT NOW()
);

CREATE INDEX idx_br_match ON behavioral_results(match);
CREATE INDEX idx_br_run ON behavioral_results(run_id);
```

```sql
-- sql/007_governance.sql
SET search_path TO sentinel, public;

CREATE TABLE review_log (
    id              SERIAL PRIMARY KEY,
    entity_type     TEXT NOT NULL
        CHECK (entity_type IN ('gap', 'mapping', 'test_result')),
    entity_id       INT NOT NULL,
    action          TEXT NOT NULL
        CHECK (action IN (
            'flagged', 'reviewed', 'confirmed', 'rejected',
            'escalated', 'resolved', 'reopened'
        )),
    performed_by    TEXT NOT NULL,
    notes           TEXT,
    created_at      TIMESTAMPTZ DEFAULT NOW()
);

CREATE INDEX idx_rl_entity ON review_log(entity_type, entity_id);
CREATE INDEX idx_rl_action ON review_log(action);

CREATE TABLE drift_alerts (
    id                      SERIAL PRIMARY KEY,
    cobol_construct_id      INT NOT NULL REFERENCES cobol_constructs(id),
    java_construct_id       INT NOT NULL REFERENCES java_constructs(id),
    traceability_map_id     INT REFERENCES traceability_map(id),
    previous_similarity     NUMERIC NOT NULL,
    current_similarity      NUMERIC NOT NULL,
    drift_delta             NUMERIC GENERATED ALWAYS AS
        (previous_similarity - current_similarity) STORED,
    java_previous_hash      TEXT,
    java_current_hash       TEXT,
    alert_level             TEXT NOT NULL
        CHECK (alert_level IN ('warning', 'critical')),
    acknowledged_by         TEXT,
    acknowledged_at         TIMESTAMPTZ,
    resolution_notes        TEXT,
    created_at              TIMESTAMPTZ DEFAULT NOW()
);

CREATE TABLE decommission_log (
    id                          SERIAL PRIMARY KEY,
    cobol_program               TEXT NOT NULL UNIQUE,
    guardian_file                TEXT,
    total_constructs            INT NOT NULL,
    mapped_constructs           INT NOT NULL,
    structural_coverage_pct     NUMERIC NOT NULL,
    semantic_avg_similarity     NUMERIC NOT NULL,
    broken_call_chains          INT NOT NULL,
    enter_tal_count             INT NOT NULL DEFAULT 0,
    enter_tal_verified          INT NOT NULL DEFAULT 0,
    guardian_io_count           INT NOT NULL DEFAULT 0,
    guardian_io_verified        INT NOT NULL DEFAULT 0,
    behavioral_total_tests      INT NOT NULL,
    behavioral_pass_count       INT NOT NULL,
    behavioral_pass_rate_pct    NUMERIC NOT NULL,
    comp3_precision_tests       INT NOT NULL DEFAULT 0,
    comp3_precision_pass        INT NOT NULL DEFAULT 0,
    open_drift_alerts           INT NOT NULL,
    critical_gaps               INT NOT NULL,
    tech_lead_signoff           TEXT,
    tech_lead_signoff_at        TIMESTAMPTZ,
    business_owner_signoff      TEXT,
    business_owner_signoff_at   TIMESTAMPTZ,
    legal_signoff               TEXT,
    legal_signoff_at            TIMESTAMPTZ,
    status                      TEXT NOT NULL DEFAULT 'pending'
        CHECK (status IN ('pending', 'ready', 'blocked', 'decommissioned')),
    blocked_reason              TEXT,
    decommissioned_at           TIMESTAMPTZ,
    created_at                  TIMESTAMPTZ DEFAULT NOW(),
    updated_at                  TIMESTAMPTZ DEFAULT NOW()
);
```

```sql
-- sql/008_views.sql
SET search_path TO sentinel, public;

CREATE MATERIALIZED VIEW coverage_summary AS
SELECT 'structural' AS layer,
    COUNT(*) FILTER (WHERE confidence IN ('high','medium'))::NUMERIC
    / NULLIF((SELECT COUNT(*) FROM cobol_constructs
              WHERE construct_type IN ('paragraph','section')), 0) * 100
    AS coverage_pct,
    COUNT(*) FILTER (WHERE confidence = 'unmatched') AS gap_count,
    NOW() AS computed_at
FROM traceability_map
UNION ALL
SELECT 'semantic',
    AVG(similarity_score) FILTER (WHERE similarity_score IS NOT NULL) * 100,
    COUNT(*) FILTER (WHERE similarity_score < 0.70 OR similarity_score IS NULL),
    NOW()
FROM traceability_map
UNION ALL
SELECT 'graph',
    COUNT(*) FILTER (WHERE status = 'pass')::NUMERIC
    / NULLIF(COUNT(*), 0) * 100,
    COUNT(*) FILTER (WHERE status = 'fail'), NOW()
FROM graph_validation_results
WHERE run_id = (SELECT run_id FROM graph_validation_results ORDER BY run_at DESC LIMIT 1)
UNION ALL
SELECT 'behavioral',
    COUNT(*) FILTER (WHERE match = true)::NUMERIC
    / NULLIF(COUNT(*), 0) * 100,
    COUNT(*) FILTER (WHERE match = false), NOW()
FROM behavioral_results
WHERE run_id = (SELECT run_id FROM behavioral_results ORDER BY run_at DESC LIMIT 1);

CREATE MATERIALIZED VIEW program_coverage AS
SELECT
    cc.program_id,
    cc.guardian_file_name,
    COUNT(DISTINCT cc.id) AS total_constructs,
    COUNT(DISTINCT cc.id) FILTER (WHERE cc.construct_type = 'paragraph') AS total_paragraphs,
    COUNT(DISTINCT cc.id) FILTER (WHERE cc.construct_type IN ('enter_tal','enter_c')) AS enter_tal_c_count,
    COUNT(DISTINCT tm.id) FILTER (WHERE tm.confidence = 'high') AS high_confidence,
    COUNT(DISTINCT tm.id) FILTER (WHERE tm.confidence = 'medium') AS medium_confidence,
    COUNT(DISTINCT cg.id) FILTER (WHERE cg.severity = 'critical') AS critical_gaps,
    COALESCE(
        COUNT(DISTINCT tm.id) FILTER (WHERE tm.confidence IN ('high','medium'))::NUMERIC
        / NULLIF(COUNT(DISTINCT cc.id)
                 FILTER (WHERE cc.construct_type IN ('paragraph','section','enter_tal','enter_c')), 0) * 100,
        0
    ) AS structural_coverage_pct,
    NOW() AS computed_at
FROM cobol_constructs cc
LEFT JOIN traceability_map tm ON tm.cobol_construct_id = cc.id
LEFT JOIN coverage_gaps cg ON cg.construct_id = cc.id AND cg.side = 'cobol_unmapped'
WHERE cc.construct_type IN ('paragraph','section','enter_tal','enter_c')
GROUP BY cc.program_id, cc.guardian_file_name;

CREATE MATERIALIZED VIEW nonstop_coverage AS
SELECT
    construct_type,
    COUNT(*) AS total,
    COUNT(*) FILTER (WHERE id IN (SELECT cobol_construct_id FROM traceability_map
                                   WHERE confidence IN ('high','medium'))) AS mapped,
    COUNT(*) FILTER (WHERE id NOT IN (SELECT cobol_construct_id FROM traceability_map
                                       WHERE confidence IN ('high','medium'))) AS unmapped
FROM cobol_constructs
WHERE construct_type IN ('enter_tal', 'enter_c', 'pathway_server',
                          'receive_block', 'checkpoint')
GROUP BY construct_type;
```

-----

## 6. Phase 1 — COBOL Inventory Engine (ProLeap)

### 6.1 Objective

Parse every HP NonStop COBOL source file using ProLeap and populate `cobol_constructs` with a complete structural and semantic inventory.

### 6.2 Pipeline

```
COBOL source files ($DATA1.SRCLIB.*)
        │
        ▼
  ┌─────────────────┐
  │  ProLeap Parser  │  format = TANDEM
  │  (ANTLR4/Java)   │
  │                   │
  │  1. COPY preproc  │
  │  2. AST generate  │
  │  3. ASG analyze   │
  │  4. JSON export   │
  └────────┬──────────┘
           │
           ▼
  ┌─────────────────┐
  │  ENTER TAL/C     │  Regex post-processing on
  │  Extractor       │  ProLeap-extracted text blocks
  └────────┬──────────┘
           │
           ▼
  ┌─────────────────┐
  │  ASG Summarizer  │  Generates NL summary from
  │  (Node.js)       │  ASG data flow + control flow
  └────────┬──────────┘
           │
           ▼
  ┌─────────────────┐
  │  Construct       │  Batch insert to Postgres
  │  Emitter         │  cobol_constructs table
  └──────────────────┘
```

### 6.3 ProLeap ASG Summary Generation

The ASG gives us resolved data flow. For each paragraph, generate a natural language summary:

```typescript
// src/parsers/cobol/asg-summarizer.ts
export function generateSummary(asgNode: AsgParagraph): string {
    const parts: string[] = [];

    // What it does
    if (asgNode.verbCounts.COMPUTE > 0) {
        parts.push(`Performs ${asgNode.verbCounts.COMPUTE} arithmetic computation(s)`);
    }
    if (asgNode.verbCounts.EVALUATE > 0) {
        parts.push(`Contains ${asgNode.verbCounts.EVALUATE} multi-branch decision(s)`);
    }

    // What it reads/modifies
    const reads = asgNode.dataAccess.filter(d => d.accessType === 'read');
    const modifies = asgNode.dataAccess.filter(d => d.accessType === 'modify');
    if (reads.length > 0) {
        parts.push(`Reads: ${reads.map(r => r.name).join(', ')}`);
    }
    if (modifies.length > 0) {
        parts.push(`Modifies: ${modifies.map(m => m.name).join(', ')}`);
    }

    // What it calls
    const performs = asgNode.performTargets;
    if (performs.length > 0) {
        parts.push(`Calls: ${performs.join(', ')}`);
    }

    // File I/O
    const fileOps = asgNode.fileOperations;
    if (fileOps.length > 0) {
        parts.push(`File I/O: ${fileOps.map(f => `${f.operation} ${f.fileName}`).join(', ')}`);
    }

    // NonStop specifics
    if (asgNode.enterTal) {
        parts.push(`Calls TAL routine: ${asgNode.enterTal.routineName}`);
    }

    return parts.join('. ') + '.';
}
```

**Example output:** “Performs 4 arithmetic computation(s). Contains 2 multi-branch decision(s) (ACT/360, ACT/365, 30/360). Reads: WS-ANNUAL-RATE, WS-PRINCIPAL, WS-RATE-TYPE, WS-DAY-COUNT. Modifies: WS-DAILY-RATE, WS-DAILY-AMOUNT. Calls: 310-CALC-30-360, 320-APPLY-COMPOUND, 900-ERROR-HANDLER.”

This summary is what gets embedded by `snowflake-arctic` in Phase 4.

### 6.4 ENTER TAL/C Extraction

ProLeap extracts ENTER blocks as text. Post-process with regex to capture:

```typescript
// src/parsers/cobol/enter-tal-extractor.ts
const ENTER_TAL_REGEX = /ENTER\s+TAL\s+"([^"]+)"\s*(USING\s+(.+?))\.\s*ENTER\s+COBOL/gs;
const ENTER_C_REGEX   = /ENTER\s+C\s+"([^"]+)"\s*(USING\s+(.+?))\.\s*ENTER\s+COBOL/gs;

interface EnterTalBlock {
    routineName: string;
    language: 'TAL' | 'C';
    parameters: string[];
    startLine: number;
    endLine: number;
    sourceFile?: string;   // TAL source file if known
}
```

### 6.5 Exit Criteria

- [ ] All COBOL source files processed via ProLeap (Tandem format)
- [ ] Parse failure rate < 5%
- [ ] ASG summaries generated for all paragraphs with >5 LOC
- [ ] All ENTER TAL/C blocks extracted and cataloged
- [ ] All Guardian file references captured
- [ ] Baseline snapshot created

-----

## 7. Phase 2 — Java Inventory Engine (SCIP + Tree-sitter)

### 7.1 Objective

Run `scip-java` on the converted Java codebase, parse the index, supplement with Tree-sitter AST fingerprints, and populate `java_constructs` and `scip_references`.

### 7.2 Pipeline

```
Converted Java project
        │
        ├──► scip-java index ──► index.scip ──► ScipParser.ts
        │                                           │
        │                                    ┌──────▼──────┐
        │                                    │ java_        │
        │                                    │ constructs   │
        │                                    │ (scip_symbol)│
        │                                    │              │
        │                                    │ scip_        │
        │                                    │ references   │
        │                                    └──────────────┘
        │
        └──► Tree-sitter Java ──► AST fingerprints
                                  complexity scores
                                  raw source extraction
```

### 7.3 LLM Pseudocode Description

For cross-language embedding, generate a natural language description of each Java method using Ollama:

```typescript
// src/parsers/java/java-summarizer.ts
const JAVA_SUMMARY_PROMPT = `
Describe what this Java method does in plain English.
Focus on: what it computes, what data it reads/modifies,
what other methods it calls, and any I/O operations.
Keep it under 100 words. Do not include Java syntax.

Method:
{source}
`;
```

**Example output:** “Calculates the daily interest rate by dividing the annual rate by 360. Only handles the ACT/360 day-count convention. Returns a BigDecimal with 8 decimal places using HALF_UP rounding. Does not handle ACT/365 or 30/360 conventions.”

This description gets embedded by `snowflake-arctic` (same model as COBOL summaries) so they’re comparable in the same vector space.

### 7.4 Exit Criteria

- [ ] `scip-java index` runs successfully
- [ ] All symbols and references extracted
- [ ] Pseudocode descriptions generated for all methods >5 LOC
- [ ] AST fingerprints generated
- [ ] Baseline snapshot updated

-----

## 8. Phase 3 — Deterministic Mapping Engine

### 8.1 Three Strategies

**Strategy 1: Convention-based name matching**

Normalization: strip numeric prefixes → replace hyphens with camelCase → case-insensitive compare → Levenshtein distance.

|COBOL                |Java                                 |Level           |
|---------------------|-------------------------------------|----------------|
|PROGRAM-ID → class   |`CALC-INTEREST` → `CalcInterest`     |program→class   |
|PARAGRAPH → method   |`COMPUTE-RATE` → `computeRate()`     |paragraph→method|
|COPYBOOK → DTO       |`CUST-REC.cpy` → `CustRec.java`      |copybook→class  |
|WS item → field      |`WS-TOTAL-AMT` → `totalAmt`          |data_item→field |
|CALL target → service|`CALL 'VALIDATE'` → `ValidateService`|call→class      |

**Strategy 2: Structural pattern matching (AST fingerprints)**

Compare normalized node-type sequences. COBOL EVALUATE(3 branches) + COMPUTE(2) → Java switch(3 cases) + arithmetic(2). Score via LCS.

**Strategy 3: Windsurf metadata extraction**

Search Java files for `// Converted from:`, `// Source:`, `// COBOL:` comments. Search git log for conversion commits.

### 8.2 Exit Criteria

- [ ] All three strategies executed
- [ ] traceability_map populated
- [ ] Unmapped constructs logged in coverage_gaps

-----

## 9. Phase 4 — Semantic Embedding Layer (Dual-Model)

### 9.1 Embedding Pipeline

```
COBOL paragraph
    │
    ▼
ProLeap ASG summary (NL text)
    │
    ▼
snowflake-arctic-embed-l-v2.0 ──► vector(1024)
    │                                    │
    │                              cosine similarity
    │                              (same vector space)
    │                                    │
Java method                              │
    │                                    │
    ▼                                    │
LLM pseudocode description (NL text)     │
    │                                    │
    ▼                                    │
snowflake-arctic-embed-l-v2.0 ──► vector(1024)
```

Both sides embedded by the same model in the same vector space → directly comparable via cosine similarity.

### 9.2 Thresholds

|Cosine Similarity|Action             |Confidence|
|-----------------|-------------------|----------|
|> 0.85           |Auto-map           |medium    |
|0.70 – 0.85      |Manual review queue|low       |
|< 0.70           |Coverage gap       |unmatched |

### 9.3 Gap Severity Classification

```typescript
function classifySeverity(construct: CobolConstruct): string {
    const hasBusinessLogic = ['COMPUTE', 'EVALUATE', 'CALL']
        .some(v => construct.metadata.verbCounts?.[v] > 0);
    const isNonstopSpecific = construct.enterTal || construct.guardianIo || construct.tmf;

    if (isNonstopSpecific) return 'critical';  // always critical
    if (construct.loc > 50 && hasBusinessLogic) return 'critical';
    if (construct.loc > 20 || hasBusinessLogic) return 'major';
    return 'minor';
}
```

### 9.4 Exit Criteria

- [ ] All non-trivial constructs summarized and embedded (both sides)
- [ ] Cross-language matching complete in 1024-dim space
- [ ] coverage_gaps populated with severity
- [ ] Review queue generated

-----

## 10. Phase 5 — Graph Dependency Validation (Apache AGE)

### 10.1 Graph Schema

**COBOL nodes (including NonStop-specific):**

```cypher
(:CobolProgram    {name, file, guardian_file, construct_id})
(:CobolSection    {name, program, construct_id})
(:CobolParagraph  {name, program, section, construct_id, loc, complexity})
(:CobolCopybook   {name, file, construct_id})
(:CobolFile       {name, guardian_name, access_mode, organization})
(:CobolDataItem   {name, program, level, pic, usage, construct_id})
(:TalRoutine      {name, source_file, params})
(:CRoutine        {name, source_file, params})
(:PathwayServer   {name, server_class, construct_id})
```

**COBOL edges:**

```cypher
(:CobolParagraph)-[:PERFORMS]->(:CobolParagraph)
(:CobolProgram)-[:CALLS]->(:CobolProgram)
(:CobolProgram)-[:COPIES]->(:CobolCopybook)
(:CobolParagraph)-[:READS]->(:CobolFile)
(:CobolParagraph)-[:WRITES]->(:CobolFile)
(:CobolParagraph)-[:USES]->(:CobolDataItem)
(:CobolParagraph)-[:MODIFIES]->(:CobolDataItem)
(:CobolParagraph)-[:ENTER_TAL]->(:TalRoutine)
(:CobolParagraph)-[:ENTER_C]->(:CRoutine)
(:CobolParagraph)-[:RECEIVES_FROM]->(:PathwayServer)
```

**Java nodes and edges:** built from SCIP data (same as before).

**Cross-graph edges:**

```cypher
(:CobolParagraph)-[:CONVERTS_TO {confidence, mapping_type, similarity}]->(:JavaMethod)
(:TalRoutine)-[:REPLACED_BY {verified}]->(:JavaMethod)
(:CobolFile)-[:MIGRATED_TO]->(:JavaClass)
```

### 10.2 Validation Queries

Eight validation queries covering: call chains, fan-out/fan-in, data flow, ENTER TAL/C mapping, TMF transactions, Guardian file access, copybook dependencies, SQL equivalence.

### 10.3 Exit Criteria

- [ ] Both graphs fully populated (including NonStop nodes)
- [ ] CONVERTS_TO and REPLACED_BY edges created
- [ ] All 8 validation queries executed
- [ ] Broken chains added to coverage_gaps

-----

## 11. Phase 6 — Behavioral Validation (Golden File Parity)

### 11.1 Golden File Capture (on NonStop System)

Before decommission, run each COBOL program with known inputs on the live NonStop system and capture outputs:

```bash
# scripts/capture-golden-files.sh
# Run this ON the NonStop system
#
# For each program:
# 1. Prepare input file
# 2. Execute COBOL program
# 3. Capture: stdout, output files, return code, DB state
# 4. Package as golden file set

PROGRAM=$1
INPUT=$2
OUTPUT_DIR="/golden-files/$PROGRAM/$(date +%Y%m%d)"

mkdir -p "$OUTPUT_DIR"

# Capture DB state before
sqlci <<< "SELECT * FROM acct_table WHERE acct_id IN (SELECT acct_id FROM test_input);" > "$OUTPUT_DIR/db_before.txt"

# Run COBOL program
run $PROGRAM /IN $INPUT /OUT "$OUTPUT_DIR/stdout.txt"
echo $? > "$OUTPUT_DIR/return_code.txt"

# Capture output files
cp $DATA2.ACCTDB.OUTPUT "$OUTPUT_DIR/output_file.dat"

# Capture DB state after
sqlci <<< "SELECT * FROM acct_table WHERE acct_id IN (SELECT acct_id FROM test_input);" > "$OUTPUT_DIR/db_after.txt"

# Compute hashes
md5sum "$OUTPUT_DIR"/* > "$OUTPUT_DIR/checksums.md5"
```

### 11.2 Java Parity Test

Feed identical inputs to Java, compare output against golden files:

```
Golden file (NonStop output) ──► Diff Engine ◄── Java output
                                      │
                              ┌───────▼────────┐
                              │ Byte-level hash │
                              │ Line-by-line    │
                              │ COMP-3 epsilon  │
                              │ DB row diff     │
                              └────────────────┘
```

### 11.3 COMP-3 Precision Configuration

```typescript
export const PRECISION_CONFIG = {
    comp3Epsilon: 0.005,
    currencyEpsilon: 0.01,
    decimalPlaces: (pic: string): number => {
        const match = pic.match(/V9\((\d+)\)/);
        return match ? parseInt(match[1]) : 0;
    },
};
```

### 11.4 LLM Edge Case Generation

Focus on NonStop-specific edge cases: Guardian file status codes (00, 10, 23, 35, 39, 41, 42, 46, 47, 48), LOCKFILE timeout scenarios, TMF ABORT recovery, $RECEIVE message truncation, COMP-3 packed decimal boundary values.

### 11.5 Exit Criteria

- [ ] Golden files captured from NonStop for all critical programs
- [ ] Java parity tests executed
- [ ] COMP-3 precision tests passing within epsilon
- [ ] Edge cases generated and executed
- [ ] All failures documented

-----

## 12. Phase 7 — Gap Analysis Dashboard

Interactive React dashboard with three views: GAP LIST, PROGRAMS, LAYERS.

Each gap card expands to show: source code comparison (COBOL vs Java side-by-side), data items with PIC/USAGE, similarity analysis with score bar, graph validation issues, behavioral test pass/fail, ENTER TAL/C routine details with parameter signatures, COBOL verb analysis, and full audit log timeline.

Action buttons: Confirm Match, Reject Match, Escalate to SME, Flag Dead Code.

Dashboard served via Express API, queries materialized views in Postgres.

See `dashboard/sentinel-gap-detail.jsx` for the complete implementation.

-----

## 13. Phase 8 — Reporting & Review Workflow

### 13.1 Report Types

|Report                  |Audience         |Format          |Generation        |
|------------------------|-----------------|----------------|------------------|
|Executive Summary       |Leadership, Legal|PDF (via Pandoc)|Weekly cron       |
|Audit Trail             |Legal (Sarah)    |CSV export      |On demand         |
|Gap Report              |Tech Lead        |Dashboard + PDF |Real-time + weekly|
|Behavioral Report       |QA               |JSON + PDF      |Per test run      |
|NonStop Interop Report  |Platform team    |PDF             |On demand         |
|Decommission Certificate|All stakeholders |PDF (signed)    |Per program       |

### 13.2 Report Generation Pipeline

```
Postgres materialized views
        │
        ▼
  coverage-engine.ts  (computes metrics)
        │
        ▼
  report-generator.ts (formats as Markdown)
        │
        ▼
  markdown-to-pdf.ts  (Pandoc wrapper)
        │
        ▼
  reports/sentinel-{type}-{date}.pdf
```

### 13.3 Review Workflow — Three Tiers

**Tier 1 — Automated (no human):** Confidence = ‘high’ (exact name matches, Windsurf metadata, similarity > 0.85). Auto-mapped and logged. Covers ~70-80% of mappings.

**Tier 2 — Tech Lead Review:** Confidence = ‘medium’ or ‘low’. Dashboard shows review queue with COBOL source on left, Java on right, similarity score, and confirm/reject/reassign buttons. On confirm: `verified_by` and `verified_at` populated. All actions logged in `review_log`.

**Tier 3 — SME Escalation:** severity = ‘critical’ gaps, especially ENTER TAL/C and Guardian I/O. SME identifies Java equivalent, confirms genuinely missing, or declares dead code. Decision recorded with reasoning.

### 13.4 Sign-off Chain

For each program:

1. Tech lead reviews all Tier 2 items → signs off
1. Business owner confirms behavioral results → signs off
1. Legal (Sarah) reviews audit trail → signs off
1. All three + all four layers ≥ 95% → status = `ready`

-----

## 14. Phase 9 — Continuous Governance

### 14.1 Git Hook Integration

Post-commit on Java repo: re-parse changed files → update constructs → re-embed → re-validate graph.

### 14.2 Weekly Drift Detection

Re-embed all Java constructs modified in past 7 days, recompute similarity against mapped COBOL counterparts. Drop > 0.10 → warning. Drop > 0.20 → critical (block deployment).

### 14.3 Decommission Gate

See Appendix D for complete checklist.

-----

## 15. HP NonStop-Specific Handling

### 15.1 Construct Type Matrix

|NonStop Construct                    |construct_type                |Risk Level|Java Equivalent                |
|-------------------------------------|------------------------------|----------|-------------------------------|
|ENTER TAL “routine”                  |enter_tal                     |Critical  |Pure Java method or JNI wrapper|
|ENTER C “routine”                    |enter_c                       |Critical  |Pure Java method or JNI wrapper|
|Guardian file I/O ($DATA.SUBVOL.FILE)|file_descriptor               |Critical  |Standard Java I/O or JDBC      |
|Enscribe LOCKFILE                    |paragraph (with LOCKFILE verb)|Critical  |Java file locking / DB locking |
|TMF BEGIN/COMMIT/ABORT               |checkpoint                    |Critical  |JTA / Spring @Transactional    |
|Pathway server class                 |pathway_server                |Major     |REST API / message queue       |
|$RECEIVE IPC                         |receive_block                 |Major     |Message queue / gRPC           |
|SQL/MP embedded SQL                  |sql_block                     |Major     |Standard JDBC/JPA              |
|SCOBOL (Screen COBOL)                |Out of scope                  |—         |Web UI                         |

### 15.2 ENTER TAL/C Verification Process

For each ENTER TAL/C block:

1. Identify TAL/C routine source file (e.g., `$DATA1.TALLIB.RATELKP`)
1. Document parameters: names, types (INT, CHAR, FIXED), pass-by-ref/value
1. Capture I/O behavior as golden files from NonStop
1. Find or create Java equivalent
1. Run behavioral parity tests
1. Create REPLACED_BY edge in graph
1. Mark as verified in traceability_map

### 15.3 Guardian File Semantics

Guardian Enscribe files have record-level locking, keyed access, and status codes that differ from standard COBOL file status. Verify:

- File status code mapping (Guardian → Java exceptions)
- Record locking semantics (LOCKFILE with TIME LIMIT)
- Key-sequenced access patterns
- Alternate key access

-----

## 16. Configuration & Environment

```bash
# .env.example

# Database
SENTINEL_DB_HOST=localhost
SENTINEL_DB_PORT=5432
SENTINEL_DB_NAME=apollo
SENTINEL_DB_SCHEMA=sentinel

# Ollama
OLLAMA_BASE_URL=http://localhost:11434
OLLAMA_EMBED_MODEL_CODE=nomic-embed-code
OLLAMA_EMBED_MODEL_TEXT=snowflake-arctic-embed-l-v2.0
OLLAMA_LLM_MODEL=codellama:34b

# Paths
COBOL_SOURCE_DIR=/path/to/nonstop/cobol
COBOL_COPYBOOK_DIRS=/path/to/copybooks
JAVA_SOURCE_DIR=/path/to/converted/java
PROLEAP_JAR=$HOME/.local/sentinel/proleap/proleap-cobol-parser/target/proleap-cobol-parser.jar
SCIP_JAVA_BIN=$HOME/.local/sentinel/scip/scip-java
GOLDEN_FILES_DIR=/path/to/golden-files

# Graph
AGE_GRAPH_NAME=sentinel_graph

# Thresholds
SIMILARITY_AUTO_MAP=0.85
SIMILARITY_REVIEW=0.70
DRIFT_WARNING_DELTA=0.10
DRIFT_CRITICAL_DELTA=0.20
DECOMMISSION_MIN_COVERAGE=95
COMP3_EPSILON=0.005
```

-----

## 17. Testing Strategy

|Layer      |Tool    |Target   |Tests                                                                |
|-----------|--------|---------|---------------------------------------------------------------------|
|Unit       |Vitest 3|90%      |Parsers, matchers, normalizers, diff engine, summarizers             |
|Integration|Vitest 3|80%      |DB interactions, ProLeap invocation, SCIP parsing, embedding pipeline|
|E2E        |Scripts |Key flows|Full pipeline on fixture data                                        |

**Fixtures:** `nonstop-cobol/` contains sample NonStop COBOL with ENTER TAL, Guardian files, SQL/MP. `sample-java/` contains corresponding conversions. `golden-files/` contains captured NonStop outputs. `tal-routines/` contains TAL routine signatures.

-----

## 18. Timeline & Milestones

|Week|Phase  |Milestone                                        |
|----|-------|-------------------------------------------------|
|1   |Setup  |Repo, DB, ProLeap + SCIP installed               |
|1–2 |Phase 1|COBOL inventory complete (ProLeap, Tandem format)|
|2   |Phase 2|Java inventory complete (SCIP + Tree-sitter)     |
|2–3 |Phase 3|Deterministic mapping engine                     |
|3–4 |Phase 4|Dual-model embedding + cross-match               |
|4–5 |Phase 5|Graph validation (AGE)                           |
|3–6 |Phase 6|Golden file capture + behavioral testing         |
|6–7 |Phase 7|Gap analysis dashboard                           |
|7–8 |Phase 8|Reporting + review workflow                      |
|8+  |Phase 9|Continuous governance                            |

**Parallelism:** Phases 3+4 run in parallel. Phase 6 golden file capture from NonStop starts in Week 3 (independent of other phases). Dashboard development starts in Week 6.

-----

## 19. Risk Register

|#  |Risk                                                |Impact  |Mitigation                                                                                                                  |
|---|----------------------------------------------------|--------|----------------------------------------------------------------------------------------------------------------------------|
|R1 |ProLeap doesn’t handle all NonStop COBOL extensions |High    |Regex fallback for ENTER TAL/C, Guardian-specific syntax. ProLeap extracts EXEC blocks as text for post-processing.         |
|R2 |scip-java incompatible with Java version            |High    |Test in Week 1. Fallback: Tree-sitter-only Java parsing.                                                                    |
|R3 |COMP-3 packed decimal precision loss                |Critical|Targeted precision tests. BigDecimal enforcement check. Epsilon-based comparison.                                           |
|R4 |NonStop system unavailable for golden file capture  |Critical|Coordinate with ops team early (Week 2). Capture in batches. Use production-like test environment if production unavailable.|
|R5 |TAL/C routine source unavailable                    |High    |Document routine signatures from COBOL ENTER blocks. Capture I/O behavior as golden files even without source.              |
|R6 |snowflake-arctic embeddings weak for COBOL summaries|Medium  |ASG summaries are pure English text — text embedder handles well. Test with sample set first.                               |
|R7 |Dynamic CALL targets                                |High    |Flag as manual-review-required. Cannot trace statically.                                                                    |
|R8 |Windsurf conversion inconsistent                    |Medium  |Semantic layer catches variations. Normalize before matching.                                                               |
|R9 |Large codebase causes SCIP OOM                      |Medium  |Run with increased heap. Split by module if needed.                                                                         |
|R10|ProLeap JAR requires JDK 17+                        |Low     |User-space JDK install (RHEL has alternatives system).                                                                      |

-----

## 20. Appendix A — COBOL Verb Classification & Risk

|Category     |Verbs                                                |Conversion Risk          |
|-------------|-----------------------------------------------------|-------------------------|
|Arithmetic   |COMPUTE, ADD, SUBTRACT, MULTIPLY, DIVIDE             |High (COMP-3 precision)  |
|Control Flow |IF, EVALUATE, PERFORM, GO TO, STOP RUN, GOBACK       |Medium                   |
|Data Movement|MOVE, INITIALIZE, SET                                |Low                      |
|String       |STRING, UNSTRING, INSPECT, TALLYING                  |High (logic)             |
|File I/O     |OPEN, CLOSE, READ, WRITE, REWRITE, DELETE, START     |High (Guardian semantics)|
|Table        |SEARCH, SEARCH ALL                                   |Medium                   |
|Inter-program|CALL, CANCEL, ENTRY                                  |High (dependency)        |
|NonStop      |ENTER TAL, ENTER C, ENTER COBOL, LOCKFILE, CHECKPOINT|Critical                 |
|SQL          |EXEC SQL … END-EXEC                                  |Medium (SQL/MP → JDBC)   |
|Display      |ACCEPT, DISPLAY                                      |Low                      |

-----

## 21. Appendix B — SCIP Java Index Schema

```protobuf
message Index {
    Metadata metadata = 1;
    repeated Document documents = 2;
    repeated SymbolInformation external_symbols = 3;
}
message Document {
    string relative_path = 4;
    repeated Occurrence occurrences = 2;
    repeated SymbolInformation symbols = 3;
}
message SymbolInformation {
    string symbol = 1;         // "com/bank/CalcInterest#compute()."
    repeated string documentation = 3;
    repeated Relationship relationships = 4;
}
message Occurrence {
    repeated int32 range = 1;  // [startLine, startChar, endLine, endChar]
    string symbol = 2;
    int32 symbol_roles = 3;    // bitmask: Definition, Reference, etc.
}
```

-----

## 22. Appendix C — ProLeap ASG Node Types

Key ASG node types used by Sentinel:

|Node Type           |Information Extracted                      |
|--------------------|-------------------------------------------|
|ProgramUnit         |PROGRAM-ID, divisions                      |
|Section             |Section name, contained paragraphs         |
|Paragraph           |Name, start/end lines, contained statements|
|DataDescriptionEntry|Level, PIC, USAGE, REDEFINES, OCCURS       |
|FileDescriptionEntry|File name, access mode, organization       |
|PerformStatement    |Target paragraph, TIMES/UNTIL/VARYING      |
|CallStatement       |Target program name, USING parameters      |
|ComputeStatement    |Target variable, expression                |
|EvaluateStatement   |Subject, WHEN branches                     |
|ReadStatement       |File name, INTO, KEY, INVALID KEY          |
|WriteStatement      |File name, FROM, INVALID KEY               |
|CopyStatement       |Copybook name (pre-resolved)               |
|ExecSqlStatement    |SQL text (extracted as string)             |

-----

## 23. Appendix D — Decommission Gate Checklist

Before any COBOL program is retired, ALL must be satisfied:

### Structural Layer

- [ ] 100% of paragraphs mapped (confidence ≥ medium)
- [ ] 100% of copybooks mapped
- [ ] All data items accounted for
- [ ] No unmapped CALL targets

### Semantic Layer

- [ ] Average similarity ≥ 0.85
- [ ] No mapping with similarity < 0.70
- [ ] All review queue items resolved

### Graph Layer

- [ ] 0 broken call chains
- [ ] Fan-out/fan-in delta ≤ 2
- [ ] All file I/O patterns preserved
- [ ] All inter-program call paths verified

### Behavioral Layer

- [ ] All golden file parity tests passing
- [ ] All COMP-3 precision tests passing within epsilon
- [ ] All edge cases tested and passing

### NonStop-Specific

- [ ] All ENTER TAL calls have verified Java equivalents
- [ ] All ENTER C calls have verified Java equivalents
- [ ] All Guardian file access patterns verified
- [ ] TMF transaction boundaries verified
- [ ] Pathway IPC patterns verified (if applicable)
- [ ] $RECEIVE handlers verified (if applicable)
- [ ] SQL/MP to JDBC mapping verified

### Governance

- [ ] 0 open drift alerts
- [ ] Tech lead sign-off recorded
- [ ] Business owner sign-off recorded
- [ ] Legal (Sarah) sign-off recorded

### Documentation

- [ ] Traceability matrix exported and archived
- [ ] Golden files archived permanently
- [ ] Behavioral test results archived
- [ ] Decommission certificate generated and signed

-----

## 24. Appendix E — Report Templates

### Executive Summary Template

```markdown
# Sentinel Coverage Report — {date}

## Overall Status: {RED|AMBER|GREEN}
Composite Score: {score}%

## Layer Breakdown
| Layer | Coverage | Gaps | Status |
|-------|----------|------|--------|
| Structural | {pct}% | {n} | {RAG} |
| Semantic | {pct}% | {n} | {RAG} |
| Graph | {pct}% | {n} | {RAG} |
| Behavioral | {pct}% | {n} | {RAG} |

## NonStop-Specific
- ENTER TAL/C: {mapped}/{total} verified
- Guardian I/O: {mapped}/{total} verified
- TMF Transactions: {mapped}/{total} verified

## Programs Ready for Decommission
{list of programs with status=ready}

## Top Risks
{top 5 critical gaps sorted by LOC × complexity}

## Trend
{coverage % this week vs last week}
```

### Decommission Certificate Template

```markdown
# Decommission Certificate

## Program: {PROGRAM-ID}
## Guardian File: {$DATA.SUBVOL.FILE}
## Date: {date}

This certifies that COBOL program {PROGRAM-ID} has been
verified for decommission with the following scores:

| Layer | Score | Threshold | Status |
|-------|-------|-----------|--------|
| Structural | {pct}% | ≥ 95% | {PASS/FAIL} |
| Semantic | {pct}% | ≥ 95% | {PASS/FAIL} |
| Graph | {pct}% | ≥ 95% | {PASS/FAIL} |
| Behavioral | {pct}% | ≥ 95% | {PASS/FAIL} |

## NonStop Verification
- ENTER TAL/C: {n}/{n} verified ✓
- Guardian I/O: {n}/{n} verified ✓
- TMF: {n}/{n} verified ✓

## Sign-offs
- Tech Lead: {name} — {date}
- Business Owner: {name} — {date}
- Legal: {name} — {date}

## Recommendation: APPROVED FOR DECOMMISSION
```

-----

*Project Sentinel — Final Implementation Plan v2.0*
*HP NonStop COBOL → Java · ProLeap + SCIP + Dual Embedding + Golden File Parity*
*RHEL 128-core · Zero External API Dependencies*
*April 2026*

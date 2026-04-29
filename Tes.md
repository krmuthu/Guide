# Chunk Kind & Test Fixture Handling — Implementation Plan

> **Document Version:** 1.0 — 2026-04-29
> **Scope:** OME RAG (bootstrap + retrieval) and OME Agent (test generation phase)
> **Effort:** ~3 days OME RAG + ~1 day OME Agent
> **Risk:** Low — additive schema change, backward-compatible defaults
> **Goal:** Stop fixture pollution of vector search; enable phase-aware retrieval for the agent (production code in IMPLEMENT phase, test code in TEST phase) without compromising the agent’s ability to generate tests.

-----

## Table of Contents

1. [Problem Statement](#1-problem-statement)
1. [Design — Three Buckets, Two Mechanisms](#2-design)
1. [Schema Changes](#3-schema-changes)
1. [Bootstrap Pipeline Changes](#4-bootstrap-pipeline-changes)
1. [Hybrid Search Pipeline Changes](#5-hybrid-search-pipeline-changes)
1. [MCP Tool Changes — Existing Tools](#6-mcp-tool-changes)
1. [MCP Tool Additions — Two New Tools](#7-mcp-tool-additions)
1. [OME Agent — Test Generation Phase](#8-ome-agent-test-generation-phase)
1. [Backfill & Cleanup](#9-backfill-cleanup)
1. [Verification & Tests](#10-verification-tests)
1. [Implementation Schedule](#11-implementation-schedule)
1. [Rollout & Rollback](#12-rollout-rollback)

-----

## 1. Problem Statement

### 1.1 Current Behavior

The bootstrap pipeline chunks and embeds **all** Bitbucket source files indiscriminately. The skip list in `shouldGenerateHeader` only stops Qwen from generating contextual headers — fixtures still reach `chunks`, `vec_chunks`, and `fts_chunks`.

### 1.2 Concrete Failure Modes

|Failure                                        |Cause                                                                  |Impact                                                             |
|-----------------------------------------------|-----------------------------------------------------------------------|-------------------------------------------------------------------|
|Fixture JSON ranks above production code       |Mock data has high token density of business terms (`order`, `payment`)|Wrong chunks surface for “order processing” queries                |
|`.snap` snapshot files dominate retrieval      |Auto-generated, often >100KB, repeated structures                      |HNSW geometry distorted; recall degraded                           |
|Cross-repo test conventions mixed              |Test chunks retrievable across all 47 repos                            |Agent generates JUnit-Mockito hybrid with Spock syntax             |
|Test patterns leak into production prompts     |No way to filter `production` vs `test` at retrieval time              |OME Agent IMPLEMENT mode pulls test-only patterns into impl context|
|BM25 polluted with `TEST_USER_1`, `mock_data_*`|Fixture identifiers indexed                                            |Code identifier searches noisier                                   |

### 1.3 What Must Be Preserved

- The agent **must still generate tests**. Test files are valuable usage examples and convention sources.
- Test conventions are **repo-specific** (BMT has both JUnit and Spock repos). Cross-repo test retrieval is the trap.
- Fixtures must remain **discoverable by path** when the agent legitimately needs sample input data.

-----

## 2. Design

### 2.1 Three Buckets

|Bucket                  |Examples                                                                                                                                                                                        |Storage Policy                                                                                     |
|------------------------|------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|---------------------------------------------------------------------------------------------------|
|**Fixtures**            |`/fixtures/`, `*.snap`, `*.fixture.json`, `/cassettes/`, `/golden/`, `/__mocks__/data/`, large JSON > 50KB                                                                                      |Tracked in `files` table with `is_fixture=true`. **Not chunked**, **not embedded**, **not in FTS**.|
|**Tests**               |`*Test.java`, `*Tests.java`, `*Spec.groovy`, `*.test.ts`, `*.spec.ts`, `*_test.go`, `*_test.py`, `test_*.py`, files under `/test/`, `/tests/`, `/__tests__/`, `/spec/`, `/integration/`, `/e2e/`|Chunked + embedded normally. Tagged `chunk_kind='test'`.                                           |
|**Test utilities**      |files in `/test-utils/`, `/test_utils/`, `/testing/`, `/__mocks__/` (excluding `/data/` subdirs), test base classes, builders, factories                                                        |Chunked + embedded. Tagged `chunk_kind='test_util'`.                                               |
|**Production** (default)|Everything else                                                                                                                                                                                 |Tagged `chunk_kind='production'`.                                                                  |
|**Config**              |`package.json`, `pom.xml`, `build.gradle`, `application*.yml`, `tsconfig.json`, `Dockerfile`, etc.                                                                                              |Tagged `chunk_kind='config'`.                                                                      |
|**Doc**                 |`*.md`, `README*`, `CHANGELOG*`, files in `/docs/`                                                                                                                                              |Tagged `chunk_kind='doc'`.                                                                         |

### 2.2 Two Mechanisms

1. **`chunk_kind` column on `chunks`** — phase-aware retrieval filter for chunks that *are* embedded.
1. **`is_fixture` flag on `files`** — path-listable index of fixture files that are *not* embedded.

### 2.3 Why Not Embed Fixtures Behind a Flag

Considered and rejected. Two reasons:

- Fixture content is data, not code patterns. Embedding it gains nothing for code generation and pollutes retrieval for everyone.
- Fixtures are addressable by path. The agent that needs `order-sample.json` knows it needs *a sample order JSON* — `list_test_fixtures(repo, near='OrderService')` is more direct than vector search.

-----

## 3. Schema Changes

### 3.1 Migration SQL

```sql
-- Migration: 20260429_chunk_kind_and_fixture_flag.sql
SET search_path TO otak;

-- 1. Add chunk_kind to chunks table
ALTER TABLE chunks
  ADD COLUMN IF NOT EXISTS chunk_kind TEXT NOT NULL DEFAULT 'production'
  CHECK (chunk_kind IN ('production', 'test', 'test_util', 'config', 'doc'));

-- Partial indexes for fast filtering by kind in hybrid_search
CREATE INDEX IF NOT EXISTS idx_chunks_kind_repo
  ON chunks (chunk_kind, repo_slug)
  WHERE is_active = true;

CREATE INDEX IF NOT EXISTS idx_chunks_repo_kind_file
  ON chunks (repo_slug, chunk_kind, file_path)
  WHERE is_active = true;

-- 2. Add is_fixture and fixture_format to files table
ALTER TABLE files
  ADD COLUMN IF NOT EXISTS is_fixture BOOLEAN NOT NULL DEFAULT false,
  ADD COLUMN IF NOT EXISTS fixture_format TEXT,           -- 'json' | 'yaml' | 'snapshot' | 'csv' | 'xml' | NULL
  ADD COLUMN IF NOT EXISTS fixture_size_bytes INTEGER;

CREATE INDEX IF NOT EXISTS idx_files_fixture_repo
  ON files (repo_slug, is_fixture)
  WHERE is_fixture = true;

-- 3. Comments for clarity
COMMENT ON COLUMN chunks.chunk_kind IS
  'Classification of chunk source: production code, test, test_util, config, or doc. Used by hybrid_search and agent retrieval to filter by phase.';
COMMENT ON COLUMN files.is_fixture IS
  'True for files identified as test fixtures, snapshots, or mock data. These files are tracked but NOT chunked or embedded.';
```

### 3.2 Storage Impact

- `chunk_kind` adds ~16 bytes per chunk row (TEXT with short values gets TOAST-inlined). At 600K chunks, ~10 MB.
- `is_fixture` adds 1 byte per file row. Negligible.
- Two new indexes: ~50 MB combined for the chunk indexes at expected scale.

### 3.3 Default Behavior

`chunk_kind` defaults to `'production'`. Existing code that doesn’t filter by `chunk_kind` continues to work — but will see test/test_util chunks mixed in until the backfill runs (§9).

-----

## 4. Bootstrap Pipeline Changes

### 4.1 New Module — `src/ingestion/chunk_classifier.ts`

Single source of truth for fixture detection and chunk_kind classification.

```typescript
// src/ingestion/chunk_classifier.ts

const FIXTURE_PATH_PATTERNS = [
  /\/fixtures?\//i,
  /\/__fixtures__\//i,
  /\/__mocks__\/data\//i,
  /\/test[-_]?data\//i,
  /\/cassettes\//i,
  /\/golden\//i,
  /\/expected[-_]?outputs?\//i,
  /\/snapshots?\//i,
];

const FIXTURE_FILENAME_PATTERNS = [
  /\.snap$/i,
  /\.fixture\.(json|ts|js|yaml|yml)$/i,
  /\.golden$/i,
  /\.expected$/i,
];

const TEST_PATH_PATTERNS = [
  /\/tests?\//i,
  /\/__tests__\//i,
  /\/spec\//i,
  /\/integration\//i,
  /\/e2e\//i,
  /\/it\//i,
  /\/itest\//i,
  /src\/test\//i,
];

const TEST_FILENAME_PATTERNS = [
  /(^|\/)[^/]+Tests?\.(java|kt|scala|groovy)$/,
  /(^|\/)[^/]+Spec\.(groovy|scala|kt)$/,
  /(^|\/)[^/]+\.test\.(ts|tsx|js|jsx|mjs|cjs)$/,
  /(^|\/)[^/]+\.spec\.(ts|tsx|js|jsx|mjs|cjs)$/,
  /(^|\/)[^/]+_test\.(go|py)$/,
  /(^|\/)test_[^/]+\.py$/,
];

const TEST_UTIL_PATH_PATTERNS = [
  /\/test[-_]?utils?\//i,
  /\/testing\//i,                    // Java convention
  /\/__mocks__\/(?!data\/)/i,        // /__mocks__/ but NOT /__mocks__/data/
  /\/test[-_]?support\//i,
  /\/test[-_]?helpers?\//i,
];

const CONFIG_FILENAME_PATTERNS = [
  /^package\.json$/,
  /^pom\.xml$/,
  /^build\.gradle(\.kts)?$/,
  /^settings\.gradle(\.kts)?$/,
  /^tsconfig.*\.json$/,
  /^jest\.config\.(js|ts|mjs)$/,
  /^vitest\.config\.(js|ts|mjs)$/,
  /^webpack\.config\.(js|ts)$/,
  /^vite\.config\.(js|ts)$/,
  /^Dockerfile(\..+)?$/,
  /^docker-compose.*\.ya?ml$/,
  /^application(-[\w-]+)?\.(ya?ml|properties)$/,
  /^bootstrap(-[\w-]+)?\.(ya?ml|properties)$/,
  /\.env(\.[\w-]+)?$/,
];

const DOC_PATH_PATTERNS = [
  /\/docs?\//i,
  /\/documentation\//i,
];

const DOC_FILENAME_PATTERNS = [
  /\.md$/i,
  /^README(\.\w+)?$/i,
  /^CHANGELOG(\.\w+)?$/i,
  /^CONTRIBUTING(\.\w+)?$/i,
];

const FIXTURE_SIZE_THRESHOLD_BYTES = 50 * 1024;  // 50 KB

const ALLOWED_LARGE_JSON_NAMES = new Set([
  'package.json',
  'package-lock.json',     // tracked but not chunked anyway via existing skip
  'tsconfig.json',
  'openapi.json',
  'swagger.json',
]);

export type ChunkKind = 'production' | 'test' | 'test_util' | 'config' | 'doc';

export interface FixtureClassification {
  isFixture: boolean;
  format?: 'json' | 'yaml' | 'snapshot' | 'csv' | 'xml';
  sizeBytes?: number;
}

/**
 * Identifies fixture files BEFORE chunking. These files are tracked in `files`
 * with is_fixture=true, but never reach chunks/vec_chunks/fts_chunks.
 */
export function classifyAsFixture(filePath: string, sizeBytes: number): FixtureClassification {
  const basename = filePath.split('/').pop() ?? '';

  // Path-based detection
  if (FIXTURE_PATH_PATTERNS.some(re => re.test(filePath))) {
    return { isFixture: true, format: detectFixtureFormat(filePath), sizeBytes };
  }

  // Filename-based detection
  if (FIXTURE_FILENAME_PATTERNS.some(re => re.test(basename))) {
    return { isFixture: true, format: detectFixtureFormat(filePath), sizeBytes };
  }

  // Size-based detection for JSON/YAML — large data files are almost always fixtures
  if (sizeBytes > FIXTURE_SIZE_THRESHOLD_BYTES) {
    if (/\.(json|ya?ml)$/i.test(basename) && !ALLOWED_LARGE_JSON_NAMES.has(basename)) {
      return { isFixture: true, format: detectFixtureFormat(filePath), sizeBytes };
    }
  }

  return { isFixture: false };
}

function detectFixtureFormat(filePath: string): FixtureClassification['format'] {
  if (/\.snap$/i.test(filePath)) return 'snapshot';
  if (/\.json/i.test(filePath)) return 'json';
  if (/\.ya?ml/i.test(filePath)) return 'yaml';
  if (/\.csv$/i.test(filePath)) return 'csv';
  if (/\.xml$/i.test(filePath)) return 'xml';
  return undefined;
}

/**
 * Classifies a non-fixture file's chunks. Called per-chunk during ingestion.
 * The same file path always returns the same kind — chunks of one file
 * share a kind, so this can be computed once per file and cached.
 */
export function classifyChunkKind(filePath: string): ChunkKind {
  const basename = filePath.split('/').pop() ?? '';

  // Test utility — check BEFORE test, because /__mocks__/ files are utilities
  if (TEST_UTIL_PATH_PATTERNS.some(re => re.test(filePath))) {
    return 'test_util';
  }

  // Test files
  if (TEST_PATH_PATTERNS.some(re => re.test(filePath))) return 'test';
  if (TEST_FILENAME_PATTERNS.some(re => re.test(filePath))) return 'test';

  // Config
  if (CONFIG_FILENAME_PATTERNS.some(re => re.test(basename))) return 'config';

  // Doc
  if (DOC_PATH_PATTERNS.some(re => re.test(filePath))) return 'doc';
  if (DOC_FILENAME_PATTERNS.some(re => re.test(basename))) return 'doc';

  return 'production';
}
```

### 4.2 Wire Into Phase 2e (`sync_handler.ts`)

```typescript
// src/ingestion/sync_handler.ts — modified ingestion loop

import { classifyAsFixture, classifyChunkKind } from './chunk_classifier.js';

async function ingestFile(repo: string, filePath: string, content: Buffer): Promise<void> {
  const sizeBytes = content.length;

  // ─── EARLY EXIT — fixture detection BEFORE chunking ───
  const fixtureCheck = classifyAsFixture(filePath, sizeBytes);
  if (fixtureCheck.isFixture) {
    // Track in files table only — do NOT chunk, embed, or FTS-index
    await pool.query(
      `INSERT INTO otak.files (repo_slug, file_path, is_fixture, fixture_format, fixture_size_bytes, last_seen_at)
       VALUES ($1, $2, true, $3, $4, NOW())
       ON CONFLICT (repo_slug, file_path) DO UPDATE
         SET is_fixture = true,
             fixture_format = EXCLUDED.fixture_format,
             fixture_size_bytes = EXCLUDED.fixture_size_bytes,
             last_seen_at = NOW()`,
      [repo, filePath, fixtureCheck.format ?? null, fixtureCheck.sizeBytes ?? null]
    );
    metrics.fixturesSkipped++;
    return;
  }

  // ─── Normal flow — classify chunk kind once per file ───
  const chunkKind = classifyChunkKind(filePath);
  const chunks = await chunkViaCAST(filePath, content);

  for (const chunk of chunks) {
    chunk.chunkKind = chunkKind;          // attached to every chunk of this file
  }

  // ... existing flow: dedup, embed, write to chunks/vec_chunks/fts_chunks
  await writeQueue.enqueue(chunks);
}
```

### 4.3 WriteQueue — Persist `chunk_kind`

```typescript
// src/ingestion/write_queue.ts — modified INSERT

const INSERT_CHUNK_SQL = `
  INSERT INTO otak.chunks
    (id, repo_slug, file_path, content, content_hash, chunk_type, chunk_kind,
     start_line, end_line, breadcrumb, contextual_header, is_active, created_at)
  VALUES
    ($1, $2, $3, $4, $5, $6, $7, $8, $9, $10, $11, true, NOW())
  ON CONFLICT (id) DO UPDATE
    SET content = EXCLUDED.content,
        content_hash = EXCLUDED.content_hash,
        chunk_kind = EXCLUDED.chunk_kind,
        breadcrumb = EXCLUDED.breadcrumb,
        is_active = true
`;
```

### 4.4 Consolidate `shouldGenerateHeader`

The fixture path/snapshot logic moves out — fixtures never reach this function now. Simplify:

```typescript
// src/ingestion/header_skip_logic.ts — simplified

export function shouldGenerateHeader(chunk: {
  content: string;
  file_path: string;
  chunk_kind: ChunkKind;
  chunk_type: string;
}): boolean {
  // Fixtures never reach here (filtered at sync_handler).

  // Skip non-production kinds for header generation — embeddings handle them fine
  if (chunk.chunk_kind === 'config') return false;
  if (chunk.chunk_kind === 'doc') return false;
  if (chunk.chunk_kind === 'test_util') return false;

  // Tests get headers — they're valuable usage examples and headers help
  // disambiguate "test for X" queries from "X" queries.

  const lines = chunk.content.split('\n').filter(l => l.trim());
  if (lines.length < 5) return false;

  if (chunk.file_path.endsWith('.d.ts')) return false;
  if (chunk.file_path.includes('__generated__')) return false;
  if (chunk.file_path.includes('/generated/')) return false;
  if (chunk.file_path.includes('/dist/')) return false;
  if (/\.(lock|min\.(js|css))$/.test(chunk.file_path)) return false;
  if (chunk.chunk_type === 'full_file' && chunk.content.length < 200) return false;
  if (/\/(migrations?|flyway|liquibase|db\/migration)\//.test(chunk.file_path)) return false;

  return true;
}
```

### 4.5 Expected Volume Reduction

For a typical BMT repo (~12,000 raw files, ~8,000 currently chunked):

|Category                  |Before|After                           |Delta|
|--------------------------|------|--------------------------------|-----|
|Fixtures chunked          |~400  |0                               |−400 |
|`.snap` files chunked     |~150  |0                               |−150 |
|Large JSON > 50KB         |~80   |0                               |−80  |
|Test chunks (kept, tagged)|~1,800|~1,800 (now `chunk_kind='test'`)|0    |
|Production chunks         |~5,400|~5,400                          |0    |

**~630 fewer chunks per repo × 47 repos = ~30K fewer embeddings.** At nomic-embed-code’s ~3KB per halfvec, ~90 MB storage reclaimed; ~50 minutes of CPU embedding saved per full bootstrap.

-----

## 5. Hybrid Search Pipeline Changes

### 5.1 New Filter Parameter

Add `chunk_kinds?: ChunkKind[]` to `HybridSearchOptions`. Default behavior preserves backward compatibility:

```typescript
// src/lib/hybrid_search.ts — additions to HybridSearchOptions

export interface HybridSearchOptions {
  query: string;
  topK?: number;
  repoSlug?: string;
  fileExt?: string;
  astType?: string;
  source?: string;
  // ... existing fields ...

  // NEW
  chunkKinds?: ChunkKind[];               // Default: ['production', 'config', 'doc']
  excludeChunkKinds?: ChunkKind[];        // Subtract from chunkKinds
  sameRepoOnly?: boolean;                 // Force repo_slug filter on graph expansion + reranking
}
```

### 5.2 Default Filter

When `chunkKinds` is not provided, the pipeline defaults to:

```typescript
const DEFAULT_CHUNK_KINDS: ChunkKind[] = ['production', 'config', 'doc'];
```

This means: **by default, test and test_util chunks are excluded from semantic search**. Existing callers who don’t specify `chunkKinds` get cleaner production-only results automatically — this is the bug fix.

### 5.3 SQL Predicate Injection

Both the dense ANN branch and the BM25 FTS branch get the predicate:

```sql
-- Dense ANN branch
SELECT c.id, c.repo_slug, c.file_path, c.content,
       v.embedding <=> $1::halfvec AS distance
FROM otak.chunks c
JOIN otak.vec_chunks v ON v.chunk_id = c.id
WHERE c.is_active = true
  AND c.chunk_kind = ANY($2::text[])               -- NEW
  AND ($3::text IS NULL OR c.repo_slug = $3)
ORDER BY v.embedding <=> $1::halfvec
LIMIT $4;

-- BM25 FTS branch
SELECT c.id, ts_rank_cd(f.content_tsv, query) AS rank
FROM otak.chunks c
JOIN otak.fts_chunks f ON f.chunk_id = c.id,
     plainto_tsquery('english', $1) AS query
WHERE c.is_active = true
  AND c.chunk_kind = ANY($2::text[])               -- NEW
  AND f.content_tsv @@ query
ORDER BY rank DESC
LIMIT $3;
```

The new `idx_chunks_kind_repo` partial index makes these filters effectively free.

### 5.4 Source-Balanced Minimum Adjustment

`MIN_CODE_IN_CANDIDATES=5` was designed against the old mixed corpus. After the kind filter:

- Production-only queries: lower the floor to 3 (less prose contamination)
- Test-mode queries (kind=[‘test’]): introduce `MIN_TEST_IN_CANDIDATES=5` for the test branch

```typescript
const MIN_BY_KIND: Record<ChunkKind, number> = {
  production: 3,
  test: 5,
  test_util: 2,
  config: 1,
  doc: 1,
};
```

### 5.5 Same-Repo Enforcement for Test Mode

When `sameRepoOnly=true` (set automatically when `chunkKinds` includes `'test'`):

- Graph expansion (step 9 in pipeline) restricted to nodes with matching `repo_slug`
- Cross-repo reranking candidates dropped before bi-encoder rerank
- `priority_repo` boost set to 1.0 (full short-circuit) for the requesting repo

This prevents JUnit conventions leaking from `bmt-trading-core` into a Spock-based `bmt-pricing-engine` test generation.

-----

## 6. MCP Tool Changes — Existing Tools

### 6.1 `search_semantic_code` — New Parameter

Add to existing parameter list (`query, top_k, repo_slug, file_ext, ast_type, source, ...`):

|New Param       |Type      |Default                                     |Description                                        |
|----------------|----------|--------------------------------------------|---------------------------------------------------|
|`chunk_kinds`   |`string[]`|`['production', 'config', 'doc']`           |Filter chunks by kind                              |
|`same_repo_only`|`boolean` |`false` (auto-true if kinds includes ‘test’)|Restrict graph expansion + reranking to `repo_slug`|

### 6.2 Three Agent Retrieval Tools — New Parameter

|Tool                   |Existing Params                                |New Param               |
|-----------------------|-----------------------------------------------|------------------------|
|`assemble_file_context`|`repo_slug, file_path, matched_chunk_ids, role`|`chunk_kinds?: string[]`|
|`get_file_skeleton`    |`repo_slug, file_path`                         |`chunk_kinds?: string[]`|
|`get_chunks_for_symbol`|`symbol, repo_slug`                            |`chunk_kinds?: string[]`|

The `role` parameter on `assemble_file_context` should accept a new value:

|`role` value                |Resolves to `chunk_kinds`        |
|----------------------------|---------------------------------|
|`'implementation'` (default)|`['production', 'config']`       |
|`'test_generation'`         |`['test', 'test_util', 'config']`|
|`'documentation'`           |`['production', 'doc']`          |
|`'all'`                     |all kinds                        |

### 6.3 SCIP Tools — Unchanged

`get_scip_callers`, `get_scip_callees`, `get_scip_type_definition`, `get_scip_implementations`, `get_scip_references` operate on symbols, not chunks. No changes needed. SCIP already disambiguates test vs production via the file path on each occurrence.

### 6.4 `suggest_related_files` — Behavior Change

Current behavior returns mixed test + production files. Update to **structure** the response:

```typescript
{
  test_files: [{ file_path, score, source: 'naming_pattern' | 'co_change' | 'graph' }],
  co_changed_files: [{ file_path, score }],
  consumers: [{ file_path, score, edge_type }],
}
```

Test files always grouped separately so callers can ignore them when in implementation context.

-----

## 7. MCP Tool Additions — Two New Tools

### 7.1 `find_test_patterns`

**Purpose:** Find structurally similar test files in the same repo to use as convention reference for test generation.

**Why graph-based, not vector-based:** “Tests for similar code” is a structural question (same layer: controller → controller test, service → service test), not a semantic one. Graph relationships beat embedding similarity here.

```typescript
// MCP tool signature
{
  name: 'find_test_patterns',
  description: 'Find existing test files for code structurally similar to a given implementation file. Used during test generation to extract repo-specific test conventions.',
  inputSchema: {
    type: 'object',
    properties: {
      repo_slug: { type: 'string' },
      impl_file_path: { type: 'string', description: 'Production file the agent intends to write tests for' },
      max_results: { type: 'integer', default: 5, maximum: 10 },
    },
    required: ['repo_slug', 'impl_file_path'],
  },
}
```

**Algorithm:**

```typescript
async function findTestPatterns(repo: string, implFilePath: string, max = 5) {
  // 1. Classify the impl file's "layer" using project_type + path heuristics
  //    Examples: "controller", "service", "repository", "react_component", "react_hook"
  const implLayer = await detectLayer(repo, implFilePath);

  // 2. Find production files in the same layer in the same repo
  const peers = await pool.query(
    `SELECT DISTINCT file_path FROM otak.chunks
     WHERE repo_slug = $1
       AND chunk_kind = 'production'
       AND file_path != $2
       AND file_path ~ $3                 -- layer regex
     LIMIT 30`,
    [repo, implFilePath, layerRegex(implLayer)]
  );

  // 3. For each peer, find its mapped test file via test_coverage table
  //    (already populated by Phase 2i — Test Coverage Mapping)
  const testFiles = await pool.query(
    `SELECT tc.test_file, COUNT(*) AS strength
     FROM otak.test_coverage tc
     WHERE tc.repo_slug = $1
       AND tc.source_file = ANY($2::text[])
     GROUP BY tc.test_file
     ORDER BY strength DESC, tc.test_file
     LIMIT $3`,
    [repo, peers.rows.map(r => r.file_path), max]
  );

  // 4. For each test file, return chunks with chunk_kind='test'
  const results = [];
  for (const t of testFiles.rows) {
    const chunks = await pool.query(
      `SELECT id, file_path, content, breadcrumb, contextual_header
       FROM otak.chunks
       WHERE repo_slug = $1 AND file_path = $2 AND chunk_kind = 'test' AND is_active = true
       ORDER BY start_line`,
      [repo, t.test_file]
    );
    results.push({
      test_file: t.test_file,
      strength: t.strength,
      layer: implLayer,
      chunks: chunks.rows,
    });
  }

  return { impl_layer: implLayer, patterns: results };
}
```

**Layer detection** uses a small lookup keyed on `projects.tech_stack`:

|tech_stack   |Layer regex                                                                                         |
|-------------|----------------------------------------------------------------------------------------------------|
|`spring_boot`|`controller` → `Controller\.java$`; `service` → `Service\.java$`; `repository` → `Repository\.java$`|
|`react`      |`component` → `\.(tsx|jsx)$` outside `/hooks/`; `hook` → `/hooks/.+\.(ts|tsx)$`                     |

### 7.2 `list_test_fixtures`

**Purpose:** Path-listable index of fixtures (which are not embedded). Returns fixture paths the agent can read via existing `read_full_file`.

```typescript
{
  name: 'list_test_fixtures',
  description: 'List test fixtures, snapshots, and mock data files in a repo. Fixtures are tracked but not embedded — read them via read_full_file when needed.',
  inputSchema: {
    type: 'object',
    properties: {
      repo_slug: { type: 'string' },
      near_path: {
        type: 'string',
        description: 'Optional. Bias results toward fixtures near this directory.',
      },
      format: {
        type: 'string',
        enum: ['json', 'yaml', 'snapshot', 'csv', 'xml'],
        description: 'Optional format filter',
      },
      max_results: { type: 'integer', default: 20, maximum: 100 },
    },
    required: ['repo_slug'],
  },
}
```

**Implementation:**

```typescript
async function listTestFixtures(repo: string, opts: { near_path?: string; format?: string; max_results?: number }) {
  const max = opts.max_results ?? 20;

  if (opts.near_path) {
    // Bias by shared path prefix
    const prefix = path.dirname(opts.near_path);
    const result = await pool.query(
      `SELECT file_path, fixture_format, fixture_size_bytes,
              GREATEST(LENGTH(longest_common_prefix(file_path, $2)), 1) AS proximity
       FROM otak.files
       WHERE repo_slug = $1
         AND is_fixture = true
         AND ($3::text IS NULL OR fixture_format = $3)
       ORDER BY proximity DESC, file_path
       LIMIT $4`,
      [repo, prefix, opts.format ?? null, max]
    );
    return { fixtures: result.rows };
  }

  const result = await pool.query(
    `SELECT file_path, fixture_format, fixture_size_bytes
     FROM otak.files
     WHERE repo_slug = $1
       AND is_fixture = true
       AND ($2::text IS NULL OR fixture_format = $2)
     ORDER BY file_path
     LIMIT $3`,
    [repo, opts.format ?? null, max]
  );
  return { fixtures: result.rows };
}
```

`longest_common_prefix` is a simple SQL function:

```sql
CREATE OR REPLACE FUNCTION otak.longest_common_prefix(a text, b text) RETURNS text AS $$
DECLARE
  i int := 1;
  min_len int := LEAST(LENGTH(a), LENGTH(b));
BEGIN
  WHILE i <= min_len AND SUBSTRING(a FROM i FOR 1) = SUBSTRING(b FROM i FOR 1) LOOP
    i := i + 1;
  END LOOP;
  RETURN SUBSTRING(a FROM 1 FOR i - 1);
END;
$$ LANGUAGE plpgsql IMMUTABLE;
```

-----

## 8. OME Agent — Test Generation Phase

### 8.1 Phase Separation

The v5 plan already separates implementation from test generation. The retrieval policy now **flips per phase**:

|Phase    |`chunk_kinds`                    |Repo scope    |Tools used                                                                                                                             |
|---------|---------------------------------|--------------|---------------------------------------------------------------------------------------------------------------------------------------|
|IMPLEMENT|`['production', 'config', 'doc']`|Cross-repo OK |`search_semantic_code`, `assemble_file_context(role='implementation')`, `find_symbol`, SCIP tools                                      |
|TEST_GEN |`['test', 'test_util', 'config']`|Same repo only|`find_test_patterns` (NEW), `assemble_file_context(role='test_generation')`, `list_test_fixtures` (NEW), SCIP tools (for SUT signature)|

### 8.2 TEST_GEN Retrieval Sequence

```
Input: { repo: 'bmt-trading-core', impl_file: 'OrderService.java',
         changes: [CodeChange[]], generated by IMPLEMENT phase }

Step 1: find_test_patterns(repo, impl_file)
  → returns { impl_layer: 'service', patterns: [
      { test_file: 'PaymentServiceTest.java', strength: 8, chunks: [...] },
      { test_file: 'InventoryServiceTest.java', strength: 6, chunks: [...] },
    ] }
  → 3-5 sibling-layer test files. Agent extracts: framework (JUnit5),
    mocking style (Mockito @Mock + @InjectMocks), assertion library (AssertJ),
    SpringBoot annotations (@SpringBootTest vs @WebMvcTest), naming convention.

Step 2: get_chunks_for_symbol('OrderService', repo, chunk_kinds=['test_util'])
  → returns custom test base classes (AbstractServiceTest), test builders
    (OrderTestBuilder), custom matchers, fixture helpers.

Step 3: get_repo_config(repo, search='test')   [existing tool, no change]
  → Returns relevant slices of pom.xml/build.gradle test deps,
    application-test.yml, jest.config.ts, etc.

Step 4: get_scip_callees('OrderService.submitOrder', repo)   [existing]
  → SUT's collaborators — these need to be mocked.

Step 5: get_scip_type_definition(...)   [existing, per dependency]
  → Type signatures of mocked collaborators.

Step 6 (conditional): list_test_fixtures(repo, near=impl_file)
  → If a generated scenario needs sample input data, agent picks a fixture
    by path. Then read_full_file loads it. Fixtures stay out of vector search;
    they're addressable by path only.

Step 7: Generate test file using deep model with:
  - Convention summary from Step 1 patterns
  - Test util symbols from Step 2
  - Test framework config from Step 3
  - Mock/stub setup hints from Steps 4 & 5
  - Fixture paths (if any) from Step 6
  - The CodeChange[] from IMPLEMENT phase as the "what to test"
```

### 8.3 Prompt Template Anchor

The TEST_GEN prompt already exists in v5; the change is what gets injected. New anchors:

```
{TEST_PATTERNS}        ← from find_test_patterns
{TEST_UTILITIES}       ← from get_chunks_for_symbol(kind=test_util)
{TEST_FRAMEWORK_CONFIG}← from get_repo_config(search=test)
{SUT_COLLABORATORS}    ← from get_scip_callees + get_scip_type_definition
{AVAILABLE_FIXTURES}   ← from list_test_fixtures (paths only, content loaded on demand)
```

### 8.4 Agent Guard — Prevent Cross-Phase Bleed

In `gather_planner.ts`, hard-code:

```typescript
function planForPhase(phase: 'implement' | 'test_gen'): GatherPlan {
  if (phase === 'test_gen') {
    return {
      chunkKinds: ['test', 'test_util', 'config'],
      sameRepoOnly: true,
      ragTools: ['find_test_patterns', 'assemble_file_context', 'get_chunks_for_symbol',
                 'get_scip_callees', 'get_scip_type_definition', 'list_test_fixtures',
                 'get_repo_config'],
      // Excluded for test_gen: search_semantic_code (production-default conflicts),
      // analyze_blast_radius, get_ticket_context (already used in implement phase)
    };
  }

  return {
    chunkKinds: ['production', 'config', 'doc'],
    sameRepoOnly: false,
    ragTools: [/* existing implement set */],
  };
}
```

### 8.5 What This Buys

- **No JUnit/Spock cross-contamination.** Same-repo enforcement guarantees test conventions match the repo’s actual stack.
- **No fixture data treated as code patterns.** Fixtures don’t reach vector search; they’re read by path only when needed.
- **Test code still fully retrievable** — just under a different filter and tool path.
- **Production prompts stay clean.** IMPLEMENT phase never sees test-only assertion patterns or mock setups as if they were production logic.

-----

## 9. Backfill & Cleanup

### 9.1 Backfill Existing Chunks

For each already-ingested chunk, classify and update `chunk_kind`. This is a one-time migration over ~500K rows.

```typescript
// scripts/backfill_chunk_kind.ts

import { classifyChunkKind } from '../src/ingestion/chunk_classifier.js';

async function backfill() {
  const BATCH = 5000;
  let updated = 0;

  while (true) {
    const rows = await pool.query(
      `SELECT id, file_path FROM otak.chunks
       WHERE chunk_kind = 'production'                    -- only re-check defaults
         AND is_active = true
       LIMIT $1`,
      [BATCH]
    );
    if (!rows.rowCount) break;

    const updates = rows.rows.map(r => ({
      id: r.id,
      kind: classifyChunkKind(r.file_path),
    })).filter(u => u.kind !== 'production');

    if (updates.length) {
      // Group by kind for batched UPDATEs
      const byKind = new Map<string, string[]>();
      for (const u of updates) {
        if (!byKind.has(u.kind)) byKind.set(u.kind, []);
        byKind.get(u.kind)!.push(u.id);
      }
      for (const [kind, ids] of byKind) {
        await pool.query(
          `UPDATE otak.chunks SET chunk_kind = $1 WHERE id = ANY($2::uuid[])`,
          [kind, ids]
        );
      }
      updated += updates.length;
      console.log(`Reclassified ${updated} chunks so far...`);
    }

    if (rows.rows.length < BATCH) break;
  }

  console.log(`Done. Reclassified ${updated} chunks.`);
}
```

### 9.2 Delete Already-Embedded Fixtures

These are fixtures that slipped past `shouldGenerateHeader`’s old skip list and got chunked + embedded.

```typescript
// scripts/cleanup_fixture_chunks.ts

import { classifyAsFixture } from '../src/ingestion/chunk_classifier.js';
import { stat } from 'fs/promises';
import path from 'path';

async function cleanupFixtures() {
  // 1. Collect all distinct (repo_slug, file_path) currently in chunks
  const files = await pool.query(
    `SELECT DISTINCT repo_slug, file_path FROM otak.chunks WHERE is_active = true`
  );

  let toDelete: { repo: string; file: string; size: number; format?: string }[] = [];

  for (const row of files.rows) {
    const fullPath = path.join(REPOS_DIR, row.repo_slug, 'repo', row.file_path);
    let sizeBytes = 0;
    try {
      const s = await stat(fullPath);
      sizeBytes = s.size;
    } catch {
      continue;            // file no longer on disk; stale cleanup will handle
    }

    const check = classifyAsFixture(row.file_path, sizeBytes);
    if (check.isFixture) {
      toDelete.push({ repo: row.repo_slug, file: row.file_path, size: sizeBytes, format: check.format });
    }
  }

  console.log(`Found ${toDelete.length} fixture files currently embedded.`);
  console.log(`Total bytes: ${toDelete.reduce((a, b) => a + b.size, 0)}`);

  // 2. Cascade delete chunks (vec_chunks/fts_chunks delete via FK)
  for (const f of toDelete) {
    await pool.query(
      `DELETE FROM otak.chunks WHERE repo_slug = $1 AND file_path = $2`,
      [f.repo, f.file]
    );
    // Mark file as fixture in files table
    await pool.query(
      `INSERT INTO otak.files (repo_slug, file_path, is_fixture, fixture_format, fixture_size_bytes, last_seen_at)
       VALUES ($1, $2, true, $3, $4, NOW())
       ON CONFLICT (repo_slug, file_path) DO UPDATE
         SET is_fixture = true, fixture_format = EXCLUDED.fixture_format,
             fixture_size_bytes = EXCLUDED.fixture_size_bytes`,
      [f.repo, f.file, f.format ?? null, f.size]
    );
  }

  console.log(`Deleted fixture chunks for ${toDelete.length} files.`);

  // 3. Drop graph nodes for fixture files
  await pool.query(
    `DELETE FROM otak.graph_nodes
     WHERE (repo_slug, file_path) IN (
       SELECT repo_slug, file_path FROM otak.files WHERE is_fixture = true
     )`
  );

  // 4. Reclaim space
  await pool.query(`VACUUM ANALYZE otak.chunks`);
  await pool.query(`VACUUM ANALYZE otak.vec_chunks`);
  await pool.query(`VACUUM ANALYZE otak.fts_chunks`);
}
```

### 9.3 Run Order

```bash
# 1. Apply schema migration
psql -f migrations/20260429_chunk_kind_and_fixture_flag.sql

# 2. Deploy new code (still serving traffic; defaults preserve old behavior)
npm run build && systemctl restart ome-rag

# 3. Backfill chunk_kind in background (~10 min for 500K chunks)
node dist/scripts/backfill_chunk_kind.js

# 4. Cleanup orphaned fixture chunks (~5-15 min, depends on disk I/O)
node dist/scripts/cleanup_fixture_chunks.js

# 5. Flip default in hybrid_search to exclude test/test_util by default
#    (one config change, takes effect on next request)
```

-----

## 10. Verification & Tests

### 10.1 Unit Tests — Classifier

```typescript
// test/chunk_classifier.test.ts

describe('classifyAsFixture', () => {
  it('flags JSON in /fixtures/', () => {
    expect(classifyAsFixture('src/test/fixtures/order.json', 2000).isFixture).toBe(true);
  });
  it('flags .snap files', () => {
    expect(classifyAsFixture('src/__snapshots__/Foo.test.tsx.snap', 5000).isFixture).toBe(true);
  });
  it('flags large unknown JSON', () => {
    expect(classifyAsFixture('src/data/payload.json', 60_000).isFixture).toBe(true);
  });
  it('does NOT flag package.json regardless of size', () => {
    expect(classifyAsFixture('package.json', 200_000).isFixture).toBe(false);
  });
  it('does NOT flag openapi.json', () => {
    expect(classifyAsFixture('docs/openapi.json', 80_000).isFixture).toBe(false);
  });
  it('does NOT flag small JSON in src/', () => {
    expect(classifyAsFixture('src/lib/config.json', 1500).isFixture).toBe(false);
  });
});

describe('classifyChunkKind', () => {
  it('classifies *Test.java as test', () => {
    expect(classifyChunkKind('src/test/java/com/foo/OrderServiceTest.java')).toBe('test');
  });
  it('classifies *.test.ts as test', () => {
    expect(classifyChunkKind('src/lib/foo.test.ts')).toBe('test');
  });
  it('classifies /__mocks__/MyModule.ts as test_util', () => {
    expect(classifyChunkKind('src/__mocks__/MyModule.ts')).toBe('test_util');
  });
  it('classifies pom.xml as config', () => {
    expect(classifyChunkKind('pom.xml')).toBe('config');
  });
  it('classifies README.md as doc', () => {
    expect(classifyChunkKind('README.md')).toBe('doc');
  });
  it('defaults to production', () => {
    expect(classifyChunkKind('src/main/java/com/foo/OrderService.java')).toBe('production');
  });
  it('test_util takes precedence over test for /__mocks__/', () => {
    // /__mocks__/ matches both TEST_UTIL_PATH (without /data/) and TEST_PATH potentially
    expect(classifyChunkKind('src/__mocks__/UserApi.ts')).toBe('test_util');
  });
});
```

### 10.2 Integration — Hybrid Search Filter

```typescript
// test_script/test_chunk_kind_filter.mjs

const tests = [
  {
    name: 'Default search excludes test files',
    query: 'order processing',
    options: {},
    assert: (results) => results.every(r => r.chunk_kind !== 'test'),
  },
  {
    name: 'Explicit test kind returns only tests',
    query: 'order processing',
    options: { chunkKinds: ['test'] },
    assert: (results) => results.every(r => r.chunk_kind === 'test'),
  },
  {
    name: 'No fixture in results regardless of filter',
    query: 'order',
    options: { chunkKinds: ['production', 'test', 'test_util', 'config', 'doc'] },
    assert: (results) => results.every(r => !r.file_path.includes('/fixtures/') && !r.file_path.endsWith('.snap')),
  },
  {
    name: 'Same-repo enforcement on test mode',
    query: 'service test setup',
    options: { chunkKinds: ['test'], repoSlug: 'bmt-trading-core', sameRepoOnly: true },
    assert: (results) => results.every(r => r.repo_slug === 'bmt-trading-core'),
  },
];
```

### 10.3 Golden Queries — Test Phase

```typescript
const TEST_GEN_GOLDEN = [
  {
    repo: 'bmt-trading-core',
    impl_file: 'src/main/java/com/bmt/trading/OrderService.java',
    expectFramework: 'junit5',
    expectMockingStyle: 'mockito',
    expectAssertionLib: 'assertj',
    expectPatternsCount: '>= 3',
  },
  // Add 5-10 across different repos with different conventions
];
```

For each entry: run the TEST_GEN retrieval flow, extract conventions from the patterns, and assert they match the repo’s actual conventions.

### 10.4 Canary — Fixture Pollution Detector

Add to `test_script/diagnose_search.mjs`:

```typescript
async function checkFixturePollution() {
  const result = await pool.query(
    `SELECT repo_slug, file_path, COUNT(*) AS chunk_count
     FROM otak.chunks
     WHERE is_active = true
       AND (file_path ~ '/fixtures?/' OR file_path ~ '\\.snap$')
     GROUP BY repo_slug, file_path
     ORDER BY chunk_count DESC
     LIMIT 20`
  );
  if (result.rowCount && result.rows.length > 0) {
    console.warn(`⚠️ ${result.rows.length} fixture-like files still chunked:`);
    for (const r of result.rows) {
      console.warn(`  ${r.repo_slug}/${r.file_path} (${r.chunk_count} chunks)`);
    }
    return false;
  }
  console.log('✅ No fixture pollution detected');
  return true;
}
```

Run after every bootstrap.

### 10.5 Quantified Before/After

Run `diagnose_search.mjs` against a known repo before and after deploy. Expected:

|Metric                                         |Before                 |After (target)|
|-----------------------------------------------|-----------------------|--------------|
|Fixture chunks count                           |~600 (47 repos)        |0             |
|Test chunks in production query top-10         |1-3 typical            |0             |
|First production result rank for “OrderService”|4 (after fixture noise)|1             |
|Bootstrap full-run time                        |~8 hours               |~7h 10m       |
|Total chunks indexed                           |~500K                  |~470K         |

-----

## 11. Implementation Schedule

```
Day 1 — OME RAG: Schema + Classifier
  ├── Migration SQL (chunk_kind on chunks, is_fixture on files)
  ├── Apply migration to dev DB
  ├── Implement chunk_classifier.ts with full unit test suite
  ├── Verify classification on 200 sampled real file paths from BMT repos
  └── PR review

Day 2 — OME RAG: Bootstrap Wiring
  ├── Wire classifyAsFixture into sync_handler.ts (early exit)
  ├── Wire classifyChunkKind into chunk creation path
  ├── Update WriteQueue INSERT to persist chunk_kind
  ├── Simplify shouldGenerateHeader (remove now-redundant patterns)
  ├── Update suggest_related_files to structure response by kind
  ├── Smoke test: bootstrap one repo end-to-end, verify chunk_kind populated
  └── PR review

Day 3 — OME RAG: Search + MCP Tool Updates + Backfill
  ├── Add chunkKinds, excludeChunkKinds, sameRepoOnly to HybridSearchOptions
  ├── Inject SQL predicate in dense ANN + BM25 branches
  ├── Update MIN_BY_KIND source-balance constants
  ├── Add chunk_kinds parameter to search_semantic_code,
  │   assemble_file_context, get_file_skeleton, get_chunks_for_symbol
  ├── Implement role → chunk_kinds mapping in assemble_file_context
  ├── Implement find_test_patterns and list_test_fixtures
  ├── Run backfill script on staging
  ├── Run cleanup script on staging
  ├── Verify: diagnose_search.mjs shows zero fixture pollution
  └── PR review

Day 4 — OME Agent: Phase Awareness
  ├── Add phase parameter to gather_planner.ts
  ├── Hard-code chunk_kinds + sameRepoOnly per phase
  ├── Wire find_test_patterns + list_test_fixtures into RAGClient
  ├── Update TEST_GEN prompt template with new context anchors
  ├── Add agent golden tests (5+ test_gen tasks across repos)
  ├── Verify cross-repo bleed prevention
  └── PR review

Day 5 (Production rollout) — Coordinated Deploy
  ├── Apply schema migration to prod DB
  ├── Deploy ome-rag with kind-aware code (default chunkKinds preserves old behavior at first)
  ├── Run backfill on prod
  ├── Run cleanup on prod
  ├── Flip default chunkKinds to exclude test/test_util
  ├── Deploy ome-agent with phase-aware planner
  └── Monitor: query latency, retrieval quality, error rates
```

**Total: 4 engineering days + 1 rollout day.**

-----

## 12. Rollout & Rollback

### 12.1 Feature Flags

```bash
# In ome-rag .env
CHUNK_KIND_DEFAULT_FILTER=production,config,doc          # Empty = no filter (rollback)
SAME_REPO_TEST_ENFORCEMENT=true                          # false to disable
LIST_TEST_FIXTURES_ENABLED=true                          # false to disable new tools
FIND_TEST_PATTERNS_ENABLED=true

# In ome-agent .env
AGENT_PHASE_AWARE_RETRIEVAL=true                         # false to fall back to old single-mode
```

### 12.2 Rollback Plan

|Symptom                                 |Rollback                                                                                       |
|----------------------------------------|-----------------------------------------------------------------------------------------------|
|Production queries broken               |Set `CHUNK_KIND_DEFAULT_FILTER=` (empty) — disables filter, all kinds returned                 |
|Fixture cleanup deleted too aggressively|Re-bootstrap affected repos; chunks regenerate from disk                                       |
|Agent test gen quality regressed        |Set `AGENT_PHASE_AWARE_RETRIEVAL=false` — falls back to mixed retrieval                        |
|Schema migration fails                  |`chunk_kind` has DEFAULT, `is_fixture` has DEFAULT — migration is idempotent and safe to re-run|

### 12.3 What Cannot Be Rolled Back

- Deleted fixture chunks. Mitigation: cleanup script logs every deletion; re-bootstrap restores any chunk that should not have been a fixture (none expected, but the path is open).
- Reclassified chunks. Mitigation: classifier is deterministic; running backfill again with the same code produces the same kinds.

### 12.4 Monitoring

Add to existing dashboard (or `get_server_status` MCP tool output):

```typescript
{
  chunk_kind_distribution: {
    production: 470_000,
    test: 22_000,
    test_util: 4_500,
    config: 1_800,
    doc: 800,
  },
  fixture_files_tracked: 6_300,
  fixture_chunks_remaining: 0,            // canary — must stay at 0
  same_repo_test_queries_last_hour: 47,
  cross_repo_test_attempts_blocked: 3,    // any > 0 indicates an agent bug, not a tool bug
}
```

-----

## Appendix A — Decision Log

|Decision                                                   |Rationale                                                                                                                                                                                        |Alternatives Rejected                                                                                                                                                                                   |
|-----------------------------------------------------------|-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
|Don’t embed fixtures at all                                |Fixture content is data, not patterns — zero retrieval value, high pollution cost                                                                                                                |Embed behind a flag (rejected: still pollutes BM25 index even when filtered out at query time, since `tsvector` doesn’t filter cheaply); embed with low priority weight (rejected: complicates RRF math)|
|`chunk_kind` column vs separate tables                     |Single column keeps existing indexes simple; phase filter is just a `WHERE chunk_kind = ANY(...)`                                                                                                |Separate `test_chunks` table (rejected: requires duplicate HNSW index, doubles maintenance); kind in `files` table only (rejected: chunk-level filtering needs the column on chunks)                    |
|`is_fixture` flag on `files`, not a fixture-specific table |Fixtures need only metadata (path, format, size) — `files` already has the row                                                                                                                   |Separate `fixtures` table (rejected: redundant with `files`)                                                                                                                                            |
|Same-repo enforcement for test_gen                         |Test conventions are repo-specific; cross-repo retrieval mixes JUnit + Spock                                                                                                                     |Trust the cross-encoder reranker to sort it out (rejected: reranker scores semantic similarity, not convention compatibility)                                                                           |
|Two new tools instead of overloading existing ones         |`find_test_patterns` is graph-based (different access pattern from semantic search); `list_test_fixtures` returns paths only (no embedding) — neither fits cleanly into the existing tool surface|Add a `test_mode` flag to `search_semantic_code` (rejected: would force per-call branching logic; the patterns query is fundamentally graph-shaped, not vector-shaped)                                  |
|Tests get contextual headers, configs/docs/test_util do not|Tests are valuable retrieval targets when explicitly queried — headers help disambiguate “test for X” from “X”                                                                                   |Skip headers for all non-production (rejected: degrades test_gen retrieval quality)                                                                                                                     |

-----

## Appendix B — Fixture Detection Edge Cases

|File path                                  |Detected as       |Notes                                                                                                       |
|-------------------------------------------|------------------|------------------------------------------------------------------------------------------------------------|
|`src/test/resources/fixtures/orders.json`  |fixture           |Path match                                                                                                  |
|`src/__snapshots__/Component.test.tsx.snap`|fixture           |`.snap` extension                                                                                           |
|`cypress/fixtures/users.json`              |fixture           |`/fixtures/`                                                                                                |
|`tests/golden/expected_response.txt`       |fixture           |`/golden/`                                                                                                  |
|`src/data/large_dataset.json` (60 KB)      |fixture           |Size threshold + JSON                                                                                       |
|`src/data/config.json` (1 KB)              |NOT fixture       |Below size threshold, no path/name match                                                                    |
|`package.json` (200 KB monorepo root)      |NOT fixture       |Whitelisted name                                                                                            |
|`docs/openapi.json` (80 KB)                |NOT fixture       |Whitelisted name (also high signal for cross-repo API mapping)                                              |
|`src/test/java/com/foo/UserTest.java`      |test (not fixture)|Test file with logic, has assertions                                                                        |
|`src/test/resources/users.csv`             |fixture           |`/test/resources/` matches `/test[-_]?data/` after normalization — actually NO, this needs explicit handling|

**Edge case to monitor:** Java’s `src/test/resources/` is conventionally fixture-heavy but the path doesn’t match `test-data` or `fixtures`. Add an explicit pattern:

```typescript
// Add to FIXTURE_PATH_PATTERNS in chunk_classifier.ts
/src\/test\/resources\//i,
```

This catches Spring Boot’s standard test resources directory which is almost always fixtures (test YAMLs, JSON payloads, SQL seeds).

-----

**End of plan.**

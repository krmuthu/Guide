# SCIP Query-Time Verification Layer — Implementation Plan

> **Purpose:** Add compiler-verified symbol resolution at query time to ensure retrieval accuracy before agentic code modification  
> **Context:** OME Agent will autonomously apply code changes based on RAG results. Incorrect retrieval → silent bugs in production trading code. SCIP verification catches retrieval errors before they become code changes.  
> **Prerequisite:** SCIP bootstrap integration (Phase 2g.5) already indexes SCIP data during ingestion  
> **Stack:** scip-typescript · scip-java · PostgreSQL · IGraphClient · TypeScript ESM  
> **Effort:** ~5 days  
> **Risk:** Medium — SCIP CLI must be available on RHEL; graceful fallback if unavailable

-----

## Table of Contents

1. [Why Query-Time SCIP Verification](#1-why-query-time-scip-verification)
1. [Architecture — Where Verification Fits](#2-architecture--where-verification-fits)
1. [SCIP Index Storage](#3-scip-index-storage)
1. [Verification Engine](#4-verification-engine)
1. [Symbol Resolution Verifier](#5-symbol-resolution-verifier)
1. [Call Chain Verifier](#6-call-chain-verifier)
1. [Type Relationship Verifier](#7-type-relationship-verifier)
1. [Content Hash Verifier](#8-content-hash-verifier)
1. [Confidence Gating for Agentic Safety](#9-confidence-gating-for-agentic-safety)
1. [Integration with Query Pipeline](#10-integration-with-query-pipeline)
1. [Integration with MCP Tools](#11-integration-with-mcp-tools)
1. [Integration with ome_rag_guide IMPLEMENT Mode](#12-integration-with-ome_rag_guide-implement-mode)
1. [SCIP Index Refresh Strategy](#13-scip-index-refresh-strategy)
1. [Performance Budget](#14-performance-budget)
1. [Graceful Degradation](#15-graceful-degradation)
1. [Implementation Schedule](#16-implementation-schedule)

-----

## 1. Why Query-Time SCIP Verification

### 1.1 The Agentic Error Chain

```
Developer query: "Modify the order validation logic to add price range check"
  │
  ├── RAG retrieves OrderValidator.java (correct)
  ├── RAG retrieves OrderVerifier.java (wrong — name is similar but different class)
  ├── RAG retrieves PriceValidator.java (correct — related)
  │
  └── Agent acts on position 2 (OrderVerifier) because it matches "validation" semantically
      → Agent modifies OrderVerifier instead of OrderValidator
      → Silent bug: order verification now has price range checks, validation doesn't
      → Production trading system processes invalid orders
```

### 1.2 What SCIP Catches

SCIP (Source Code Intelligence Protocol) provides **compiler-verified** symbol resolution. It knows:

- The **exact fully-qualified name** of every symbol (`com.bmt.bonds.service.OrderValidator.validate`)
- The **exact type hierarchy** (`OrderValidator implements Validator<Order>`)
- The **exact call graph** (which methods call which, verified by the compiler)
- The **exact file and line** where each symbol is defined

When the RAG returns `OrderVerifier.java` for a query about “order validation”, SCIP verification checks:

1. Does `OrderVerifier` have a method related to “validation”? → Check SCIP symbol table
1. Is `OrderVerifier` in the same type hierarchy as `OrderValidator`? → Check SCIP type relations
1. Do callers of `OrderValidator.validate()` also call `OrderVerifier`? → Check SCIP call graph

If SCIP says “these are unrelated symbols that happen to have similar names,” the verification layer flags the result as low-confidence before the agent acts on it.

### 1.3 What SCIP Does NOT Do

SCIP verification is a **post-retrieval filter**, not a replacement for embedding search. It:

- Does NOT help find relevant code (that’s the embedding model’s job)
- Does NOT understand business logic (“is this the right validation?”)
- Does NOT work for dynamic/reflection-based code
- Does NOT cover all languages equally (TypeScript and Java have best support)

It verifies structural relationships — type hierarchies, call chains, symbol identity — that embedding models are weakest at distinguishing.

-----

## 2. Architecture — Where Verification Fits

```
Query Pipeline (existing 16 stages)
  │
  ├── Stages 1-12: Embed → Dense ANN → BM25 → RRF → Graph Expansion → Gating → Dedup
  │
  ├── Stage 13: Bi-encoder rerank (CodeRankEmbed 80→20)
  │
  ├── Stage 14: Cross-encoder rerank (Qwen3-Reranker 20→5)
  │
  ├── ★ NEW Stage 14.5: SCIP Verification (5 results)
  │     ├── Symbol resolution verification
  │     ├── Call chain verification
  │     ├── Type relationship verification
  │     ├── Content hash verification (file freshness)
  │     └── Confidence score annotation
  │
  ├── Stage 15: MMR diversification
  │
  └── Stage 16: Return to MCP/Windsurf with confidence annotations
                  │
                  └── Agent checks confidence before acting
```

SCIP verification runs on the **final 5 results only** — after all expensive reranking is done. This keeps the cost minimal (~20-50ms for 5 lookups against the SCIP index).

-----

## 3. SCIP Index Storage

### 3.1 Schema

SCIP index data is extracted during bootstrap (Phase 2g.5) via `scip print --json` or `scip expt-convert` (SQLite). Store in PostgreSQL for query-time access.

```sql
-- Compiler-verified symbol definitions
CREATE TABLE IF NOT EXISTS scip_symbols (
  id              SERIAL PRIMARY KEY,
  repo_slug       TEXT NOT NULL,
  symbol_uri      TEXT NOT NULL,          -- SCIP symbol URI (fully qualified)
  display_name    TEXT NOT NULL,          -- Human-readable: OrderValidator.validate
  kind            TEXT NOT NULL,          -- method, class, interface, field, variable, etc.
  file_path       TEXT NOT NULL,
  start_line      INTEGER NOT NULL,
  start_col       INTEGER NOT NULL,
  end_line        INTEGER,
  end_col         INTEGER,
  signature       TEXT,                   -- Full method/class signature
  enclosing_symbol TEXT,                  -- Parent class/module symbol URI
  documentation   TEXT,                   -- Extracted doc comment
  language        TEXT NOT NULL,          -- typescript, java
  created_at      TIMESTAMPTZ DEFAULT NOW(),
  updated_at      TIMESTAMPTZ DEFAULT NOW(),

  UNIQUE (repo_slug, symbol_uri)
);

-- Compiler-verified references (who uses what)
CREATE TABLE IF NOT EXISTS scip_references (
  id              SERIAL PRIMARY KEY,
  repo_slug       TEXT NOT NULL,
  symbol_uri      TEXT NOT NULL,          -- What symbol is referenced
  referencing_file TEXT NOT NULL,         -- File that contains the reference
  referencing_line INTEGER NOT NULL,
  reference_kind  TEXT NOT NULL,          -- definition, reference, implementation, type_definition
  enclosing_symbol TEXT,                  -- Symbol in which the reference appears
  created_at      TIMESTAMPTZ DEFAULT NOW()
);

-- Compiler-verified type relationships
CREATE TABLE IF NOT EXISTS scip_type_relations (
  id              SERIAL PRIMARY KEY,
  repo_slug       TEXT NOT NULL,
  child_symbol    TEXT NOT NULL,          -- Class that extends/implements
  parent_symbol   TEXT NOT NULL,          -- Superclass/interface
  relation_type   TEXT NOT NULL,          -- extends, implements
  created_at      TIMESTAMPTZ DEFAULT NOW(),

  UNIQUE (repo_slug, child_symbol, parent_symbol, relation_type)
);

-- Indexes for query-time lookups
CREATE INDEX IF NOT EXISTS idx_scip_symbols_name ON scip_symbols (display_name, repo_slug);
CREATE INDEX IF NOT EXISTS idx_scip_symbols_file ON scip_symbols (file_path, repo_slug);
CREATE INDEX IF NOT EXISTS idx_scip_symbols_uri ON scip_symbols (symbol_uri);
CREATE INDEX IF NOT EXISTS idx_scip_symbols_enclosing ON scip_symbols (enclosing_symbol);
CREATE INDEX IF NOT EXISTS idx_scip_refs_symbol ON scip_references (symbol_uri, repo_slug);
CREATE INDEX IF NOT EXISTS idx_scip_refs_file ON scip_references (referencing_file, repo_slug);
CREATE INDEX IF NOT EXISTS idx_scip_refs_enclosing ON scip_references (enclosing_symbol);
CREATE INDEX IF NOT EXISTS idx_scip_types_child ON scip_type_relations (child_symbol, repo_slug);
CREATE INDEX IF NOT EXISTS idx_scip_types_parent ON scip_type_relations (parent_symbol, repo_slug);
```

### 3.2 SCIP Index Population (Bootstrap Phase 2g.5)

This extends the existing SCIP bootstrap integration. The key addition is persisting SCIP data to PostgreSQL tables, not just using it for graph edge enrichment.

```typescript
// src/ingestion/scip_indexer.ts

export async function indexSCIPData(
  pool: Pool,
  repo: string,
  repoBasePath: string,
  language: 'typescript' | 'java'
): Promise<{ symbols: number; references: number; types: number }> {
  const scipIndexPath = path.join(repoBasePath, 'index.scip');

  // Check if SCIP index exists (generated by Phase 2g.5)
  try {
    await fs.access(scipIndexPath);
  } catch {
    console.log(`[SCIP] No index.scip found for ${repo} — skipping SCIP indexing`);
    return { symbols: 0, references: 0, types: 0 };
  }

  // Export SCIP to JSON for parsing
  const { stdout } = await execAsync(
    `scip print --json < "${scipIndexPath}"`,
    { maxBuffer: 100 * 1024 * 1024 }  // 100MB buffer for large repos
  );

  const scipData = JSON.parse(stdout);
  let symbols = 0, references = 0, types = 0;

  const client = await pool.connect();
  try {
    await client.query('BEGIN');

    // Clear existing SCIP data for this repo (full refresh per bootstrap)
    await client.query('DELETE FROM scip_symbols WHERE repo_slug = $1', [repo]);
    await client.query('DELETE FROM scip_references WHERE repo_slug = $1', [repo]);
    await client.query('DELETE FROM scip_type_relations WHERE repo_slug = $1', [repo]);

    for (const doc of scipData.documents ?? []) {
      const filePath = doc.relativePath ?? doc.relative_path;
      if (!filePath) continue;

      for (const occ of doc.occurrences ?? []) {
        if (!occ.symbol || occ.symbol.startsWith('local ')) continue;

        const range = occ.range ?? [];
        const startLine = range[0] ?? 0;
        const startCol = range[1] ?? 0;
        const endLine = range[2] ?? startLine;
        const endCol = range[3] ?? startCol;

        const isDefinition = (occ.symbol_roles ?? 0) & 1;  // Bit 0 = definition

        if (isDefinition) {
          // Extract display name from symbol URI
          const displayName = extractDisplayName(occ.symbol);
          const kind = extractSymbolKind(occ.symbol);
          const enclosing = extractEnclosingSymbol(occ.symbol);

          await client.query(`
            INSERT INTO scip_symbols (repo_slug, symbol_uri, display_name, kind, file_path,
                                      start_line, start_col, end_line, end_col,
                                      enclosing_symbol, language)
            VALUES ($1, $2, $3, $4, $5, $6, $7, $8, $9, $10, $11)
            ON CONFLICT (repo_slug, symbol_uri) DO UPDATE SET
              file_path = EXCLUDED.file_path,
              start_line = EXCLUDED.start_line,
              end_line = EXCLUDED.end_line,
              updated_at = NOW()
          `, [repo, occ.symbol, displayName, kind, filePath,
              startLine, startCol, endLine, endCol, enclosing, language]);
          symbols++;
        }

        // All occurrences (definitions + references)
        const refKind = isDefinition ? 'definition' : 'reference';
        await client.query(`
          INSERT INTO scip_references (repo_slug, symbol_uri, referencing_file,
                                       referencing_line, reference_kind, enclosing_symbol)
          VALUES ($1, $2, $3, $4, $5, $6)
        `, [repo, occ.symbol, filePath, startLine, refKind,
            extractEnclosingFromOccurrence(occ)]);
        references++;
      }

      // Extract type relationships from symbol information
      for (const sym of doc.symbols ?? []) {
        if (!sym.symbol) continue;
        for (const rel of sym.relationships ?? []) {
          if (!rel.symbol) continue;
          const relType = rel.is_implementation ? 'implements' :
                          rel.is_type_definition ? 'extends' : null;
          if (relType) {
            await client.query(`
              INSERT INTO scip_type_relations (repo_slug, child_symbol, parent_symbol, relation_type)
              VALUES ($1, $2, $3, $4)
              ON CONFLICT (repo_slug, child_symbol, parent_symbol, relation_type) DO NOTHING
            `, [repo, sym.symbol, rel.symbol, relType]);
            types++;
          }
        }
      }
    }

    await client.query('COMMIT');
  } catch (err) {
    await client.query('ROLLBACK');
    console.error(`[SCIP] Index failed for ${repo}:`, err);
    return { symbols: 0, references: 0, types: 0 };
  } finally {
    client.release();
  }

  console.log(`[SCIP] ${repo}: ${symbols} symbols, ${references} references, ${types} type relations`);
  return { symbols, references, types };
}

// Helper: Extract human-readable name from SCIP symbol URI
// e.g., "scip-java maven . com/bmt/bonds/OrderValidator#validate()." → "OrderValidator.validate"
function extractDisplayName(symbolUri: string): string {
  const parts = symbolUri.split(/[#./]/);
  const meaningful = parts.filter(p => p && !p.startsWith('scip-') && p !== 'maven' && p !== 'npm');
  return meaningful.slice(-2).join('.');
}

function extractSymbolKind(symbolUri: string): string {
  if (symbolUri.includes('#')) return 'method';
  if (symbolUri.includes('$')) return 'field';
  if (symbolUri.endsWith('/')) return 'module';
  if (symbolUri.endsWith('.')) return 'class';
  return 'unknown';
}

function extractEnclosingSymbol(symbolUri: string): string | null {
  const lastHash = symbolUri.lastIndexOf('#');
  if (lastHash > 0) return symbolUri.slice(0, lastHash + 1).replace(/#$/, '.');
  return null;
}

function extractEnclosingFromOccurrence(occ: any): string | null {
  return occ.enclosing_range ? occ.symbol : null;
}
```

-----

## 4. Verification Engine

### 4.1 Core Interface

```typescript
// src/lib/scip/verification_engine.ts

export interface VerificationResult {
  chunkId: string;
  filePath: string;
  repoSlug: string;

  // Verification outcomes
  symbolVerified: boolean;             // Symbol exists in SCIP at claimed location
  callChainVerified: boolean | null;   // null = not applicable (no call chain claim)
  typeRelationVerified: boolean | null; // null = not applicable
  contentFresh: boolean;               // File hash matches latest bootstrap

  // Aggregated confidence
  verificationConfidence: VerificationConfidence;
  verificationDetails: string[];       // Human-readable explanations

  // Original search score (from reranker)
  searchScore: number;

  // Final agentic confidence (search × verification)
  agenticConfidence: number;
}

export type VerificationConfidence =
  | 'scip_verified'     // SCIP confirms symbol identity — highest confidence
  | 'scip_consistent'   // SCIP data exists, no contradictions found
  | 'scip_mismatch'     // SCIP contradicts the retrieval — flag for agent
  | 'scip_unavailable'  // No SCIP data for this file/repo — fall back to search score
  | 'stale_content';    // File has changed since last indexing — warn agent

export interface VerificationRequest {
  chunkId: string;
  filePath: string;
  repoSlug: string;
  content: string;
  contentHash: string;
  searchScore: number;

  // Optional: extracted symbol names from the chunk
  symbolNames?: string[];

  // Optional: query context for call chain verification
  querySymbol?: string;
}
```

### 4.2 Engine Implementation

```typescript
// src/lib/scip/verification_engine.ts

export class SCIPVerificationEngine {
  constructor(
    private pool: Pool,
    private graphClient: IGraphClient
  ) {}

  async verify(request: VerificationRequest): Promise<VerificationResult> {
    const details: string[] = [];
    let symbolVerified = false;
    let callChainVerified: boolean | null = null;
    let typeRelationVerified: boolean | null = null;
    let contentFresh = true;

    // ── 1. Content freshness check ──
    contentFresh = await this.verifyContentFreshness(
      request.filePath, request.repoSlug, request.contentHash
    );
    if (!contentFresh) {
      details.push(`File ${request.filePath} has been modified since last indexing`);
    }

    // ── 2. Symbol resolution verification ──
    const symbolResult = await this.verifySymbolResolution(
      request.filePath, request.repoSlug, request.symbolNames
    );
    symbolVerified = symbolResult.verified;
    details.push(...symbolResult.details);

    // ── 3. Call chain verification (if query has a target symbol) ──
    if (request.querySymbol) {
      const callResult = await this.verifyCallChain(
        request.filePath, request.repoSlug, request.querySymbol, request.symbolNames
      );
      callChainVerified = callResult.verified;
      details.push(...callResult.details);
    }

    // ── 4. Type relationship verification ──
    if (request.symbolNames && request.symbolNames.length > 0) {
      const typeResult = await this.verifyTypeRelationships(
        request.repoSlug, request.symbolNames, request.querySymbol
      );
      typeRelationVerified = typeResult.verified;
      details.push(...typeResult.details);
    }

    // ── 5. Compute verification confidence ──
    const verificationConfidence = this.computeConfidence(
      symbolVerified, callChainVerified, typeRelationVerified, contentFresh
    );

    // ── 6. Compute agentic confidence ──
    const verificationMultiplier = this.getConfidenceMultiplier(verificationConfidence);
    const agenticConfidence = request.searchScore * verificationMultiplier;

    return {
      chunkId: request.chunkId,
      filePath: request.filePath,
      repoSlug: request.repoSlug,
      symbolVerified,
      callChainVerified,
      typeRelationVerified,
      contentFresh,
      verificationConfidence,
      verificationDetails: details,
      searchScore: request.searchScore,
      agenticConfidence,
    };
  }

  async verifyBatch(requests: VerificationRequest[]): Promise<VerificationResult[]> {
    // Run verifications in parallel (each is just DB lookups)
    return Promise.all(requests.map(r => this.verify(r)));
  }

  private computeConfidence(
    symbolVerified: boolean,
    callChainVerified: boolean | null,
    typeRelationVerified: boolean | null,
    contentFresh: boolean
  ): VerificationConfidence {
    if (!contentFresh) return 'stale_content';

    // If SCIP explicitly confirms the symbol exists at the claimed location
    if (symbolVerified && (callChainVerified === true || callChainVerified === null)) {
      return 'scip_verified';
    }

    // If SCIP data exists but doesn't confirm (e.g., symbol found but wrong type)
    if (symbolVerified && callChainVerified === false) {
      return 'scip_mismatch';
    }

    // If no SCIP data found for this file (no SCIP index for repo, or non-indexed file)
    if (!symbolVerified && callChainVerified === null && typeRelationVerified === null) {
      return 'scip_unavailable';
    }

    // SCIP data exists, no contradictions, but not a strong match
    return 'scip_consistent';
  }

  private getConfidenceMultiplier(confidence: VerificationConfidence): number {
    switch (confidence) {
      case 'scip_verified':     return 1.15;  // Boost: SCIP confirms retrieval
      case 'scip_consistent':   return 1.0;   // Neutral: no contradiction
      case 'scip_mismatch':     return 0.6;   // Penalize: SCIP contradicts retrieval
      case 'scip_unavailable':  return 0.9;   // Slight penalty: can't verify
      case 'stale_content':     return 0.5;   // Strong penalty: file changed
    }
  }
}
```

-----

## 5. Symbol Resolution Verifier

Checks whether symbols in the retrieved chunk actually exist in SCIP’s compiler-verified index at the claimed file and line range.

```typescript
// src/lib/scip/symbol_verifier.ts

export interface SymbolVerificationResult {
  verified: boolean;
  details: string[];
  matchedSymbols: string[];
  unmatchedSymbols: string[];
}

export class SymbolVerifier {
  constructor(private pool: Pool) {}

  async verifySymbolResolution(
    filePath: string,
    repoSlug: string,
    symbolNames?: string[]
  ): Promise<SymbolVerificationResult> {
    const details: string[] = [];
    const matched: string[] = [];
    const unmatched: string[] = [];

    // Check if SCIP has any symbols for this file
    const fileSymbols = await this.pool.query(`
      SELECT display_name, kind, start_line, end_line, symbol_uri
      FROM scip_symbols
      WHERE file_path = $1 AND repo_slug = $2
      ORDER BY start_line
    `, [filePath, repoSlug]);

    if (fileSymbols.rows.length === 0) {
      details.push(`No SCIP symbols found for ${filePath} — file may not be indexed`);
      return { verified: false, details, matchedSymbols: [], unmatchedSymbols: symbolNames ?? [] };
    }

    // If no specific symbols requested, just verify the file is in SCIP
    if (!symbolNames || symbolNames.length === 0) {
      details.push(`SCIP has ${fileSymbols.rows.length} symbols indexed for ${filePath}`);
      return { verified: true, details, matchedSymbols: [], unmatchedSymbols: [] };
    }

    // Verify each requested symbol exists in this file
    const scipNames = new Set(fileSymbols.rows.map(r => r.display_name.toLowerCase()));
    const scipNamesFull = new Map(fileSymbols.rows.map(r => [r.display_name.toLowerCase(), r]));

    for (const name of symbolNames) {
      const lower = name.toLowerCase();
      // Try exact match, then partial match (class.method → method)
      const exactMatch = scipNames.has(lower);
      const partialMatch = [...scipNames].some(s => s.includes(lower) || lower.includes(s));

      if (exactMatch) {
        matched.push(name);
        const sym = scipNamesFull.get(lower);
        details.push(`✓ '${name}' verified at ${filePath}:${sym?.start_line} (${sym?.kind})`);
      } else if (partialMatch) {
        matched.push(name);
        details.push(`~ '${name}' partially matched in SCIP index for ${filePath}`);
      } else {
        unmatched.push(name);
        details.push(`✗ '${name}' NOT found in SCIP index for ${filePath}`);
      }
    }

    const verified = matched.length > 0 && unmatched.length <= matched.length;
    return { verified, details, matchedSymbols: matched, unmatchedSymbols: unmatched };
  }
}
```

-----

## 6. Call Chain Verifier

Checks whether the retrieved file has a compiler-verified call relationship with the query’s target symbol.

```typescript
// src/lib/scip/callchain_verifier.ts

export class CallChainVerifier {
  constructor(private pool: Pool) {}

  async verifyCallChain(
    filePath: string,
    repoSlug: string,
    querySymbol: string,
    chunkSymbols?: string[]
  ): Promise<{ verified: boolean; details: string[] }> {
    const details: string[] = [];

    // Find the query symbol in SCIP
    const querySymbolResults = await this.pool.query(`
      SELECT symbol_uri, file_path, display_name
      FROM scip_symbols
      WHERE (display_name ILIKE $1 OR display_name ILIKE $2)
        AND repo_slug = $3
      LIMIT 5
    `, [`%${querySymbol}%`, `%.${querySymbol}`, repoSlug]);

    if (querySymbolResults.rows.length === 0) {
      details.push(`Query symbol '${querySymbol}' not found in SCIP — cannot verify call chain`);
      return { verified: false, details };
    }

    // Check if any symbol in the retrieved file references or is referenced by the query symbol
    for (const qs of querySymbolResults.rows) {
      // Does the retrieved file reference the query symbol?
      const forwardRef = await this.pool.query(`
        SELECT referencing_line, reference_kind
        FROM scip_references
        WHERE symbol_uri = $1
          AND referencing_file = $2
          AND repo_slug = $3
        LIMIT 5
      `, [qs.symbol_uri, filePath, repoSlug]);

      if (forwardRef.rows.length > 0) {
        details.push(
          `✓ File ${filePath} references '${qs.display_name}' ` +
          `at line(s) ${forwardRef.rows.map(r => r.referencing_line).join(', ')} (SCIP verified)`
        );
        return { verified: true, details };
      }

      // Does the query symbol's file reference symbols in the retrieved file?
      if (chunkSymbols && chunkSymbols.length > 0) {
        for (const cs of chunkSymbols) {
          const chunkSymResult = await this.pool.query(`
            SELECT symbol_uri FROM scip_symbols
            WHERE display_name ILIKE $1 AND file_path = $2 AND repo_slug = $3
            LIMIT 1
          `, [`%${cs}%`, filePath, repoSlug]);

          if (chunkSymResult.rows.length > 0) {
            const reverseRef = await this.pool.query(`
              SELECT referencing_line FROM scip_references
              WHERE symbol_uri = $1 AND referencing_file = $2 AND repo_slug = $3
              LIMIT 1
            `, [chunkSymResult.rows[0].symbol_uri, qs.file_path, repoSlug]);

            if (reverseRef.rows.length > 0) {
              details.push(
                `✓ '${qs.display_name}' in ${qs.file_path} calls '${cs}' in ${filePath} (SCIP verified)`
              );
              return { verified: true, details };
            }
          }
        }
      }
    }

    details.push(
      `✗ No SCIP-verified call relationship between '${querySymbol}' and ${filePath}`
    );
    return { verified: false, details };
  }
}
```

-----

## 7. Type Relationship Verifier

Checks whether the retrieved symbol is in the same type hierarchy as the query target — catches the `OrderValidator` vs `OrderVerifier` problem.

```typescript
// src/lib/scip/type_verifier.ts

export class TypeRelationshipVerifier {
  constructor(private pool: Pool) {}

  async verifyTypeRelationships(
    repoSlug: string,
    chunkSymbols: string[],
    querySymbol?: string
  ): Promise<{ verified: boolean; details: string[] }> {
    const details: string[] = [];

    if (!querySymbol) {
      return { verified: false, details: ['No query symbol for type verification'] };
    }

    // Find query symbol's type hierarchy
    const queryTypes = await this.pool.query(`
      SELECT child_symbol, parent_symbol, relation_type
      FROM scip_type_relations
      WHERE repo_slug = $1
        AND (child_symbol ILIKE $2 OR parent_symbol ILIKE $2)
    `, [repoSlug, `%${querySymbol}%`]);

    if (queryTypes.rows.length === 0) {
      details.push(`No type hierarchy found for '${querySymbol}' in SCIP`);
      return { verified: false, details };
    }

    // Build the query symbol's type tree (all ancestors + descendants)
    const queryTypeSet = new Set<string>();
    for (const row of queryTypes.rows) {
      queryTypeSet.add(row.child_symbol);
      queryTypeSet.add(row.parent_symbol);
    }

    // Check if any chunk symbol shares a type hierarchy
    for (const cs of chunkSymbols) {
      const chunkTypes = await this.pool.query(`
        SELECT child_symbol, parent_symbol, relation_type
        FROM scip_type_relations
        WHERE repo_slug = $1
          AND (child_symbol ILIKE $2 OR parent_symbol ILIKE $2)
      `, [repoSlug, `%${cs}%`]);

      for (const row of chunkTypes.rows) {
        if (queryTypeSet.has(row.child_symbol) || queryTypeSet.has(row.parent_symbol)) {
          details.push(
            `✓ '${cs}' shares type hierarchy with '${querySymbol}' ` +
            `via ${row.relation_type} (SCIP verified)`
          );
          return { verified: true, details };
        }
      }
    }

    // Check if chunk symbols and query symbol implement the same interface
    const queryInterfaces = queryTypes.rows
      .filter(r => r.relation_type === 'implements')
      .map(r => r.parent_symbol);

    for (const cs of chunkSymbols) {
      const chunkInterfaces = await this.pool.query(`
        SELECT parent_symbol FROM scip_type_relations
        WHERE repo_slug = $1
          AND child_symbol ILIKE $2
          AND relation_type = 'implements'
      `, [repoSlug, `%${cs}%`]);

      const sharedInterface = chunkInterfaces.rows.find(r =>
        queryInterfaces.includes(r.parent_symbol)
      );

      if (sharedInterface) {
        details.push(
          `✓ '${cs}' and '${querySymbol}' both implement ${extractDisplayName(sharedInterface.parent_symbol)}`
        );
        return { verified: true, details };
      }
    }

    details.push(
      `✗ No shared type hierarchy between chunk symbols and '${querySymbol}'`
    );
    return { verified: false, details };
  }
}

function extractDisplayName(symbolUri: string): string {
  const parts = symbolUri.split(/[#./]/);
  return parts.filter(p => p && !p.startsWith('scip-')).slice(-2).join('.');
}
```

-----

## 8. Content Hash Verifier

Checks whether the file on disk has changed since the last bootstrap indexing. If the file is stale, the agent should re-read from disk rather than trust the cached chunk.

```typescript
// src/lib/scip/content_verifier.ts

import { createHash } from 'crypto';
import { promises as fs } from 'fs';
import path from 'path';

export class ContentHashVerifier {
  constructor(
    private pool: Pool,
    private reposDir: string  // REPOS_DIR — where bootstrap cloned repos
  ) {}

  async verifyContentFreshness(
    filePath: string,
    repoSlug: string,
    storedHash: string
  ): Promise<boolean> {
    try {
      // Read current file from disk
      const fullPath = path.join(this.reposDir, repoSlug.replace('/', '/'), filePath);
      const content = await fs.readFile(fullPath, 'utf-8');
      const currentHash = createHash('sha256').update(content).digest('hex');

      return currentHash === storedHash;
    } catch {
      // File doesn't exist on disk (deleted?) or read error
      return false;
    }
  }

  /**
   * For IMPLEMENT mode: get the current file content and its hash
   * so the agent can verify before applying changes.
   */
  async getCurrentFileState(
    filePath: string,
    repoSlug: string
  ): Promise<{ content: string; hash: string; lineCount: number } | null> {
    try {
      const fullPath = path.join(this.reposDir, repoSlug.replace('/', '/'), filePath);
      const content = await fs.readFile(fullPath, 'utf-8');
      const hash = createHash('sha256').update(content).digest('hex');
      const lineCount = content.split('\n').length;
      return { content, hash, lineCount };
    } catch {
      return null;
    }
  }
}
```

-----

## 9. Confidence Gating for Agentic Safety

The confidence gate is the critical safety mechanism. It determines whether the agent should act on retrieval results or request human confirmation.

```typescript
// src/lib/scip/confidence_gate.ts

export interface AgenticDecision {
  action: 'proceed' | 'proceed_with_caution' | 'request_confirmation' | 'abort';
  reason: string;
  results: VerificationResult[];
}

export class ConfidenceGate {
  // Thresholds for agentic actions
  private readonly PROCEED_THRESHOLD = 0.80;          // All results above this → agent can act
  private readonly CAUTION_THRESHOLD = 0.60;          // Mixed results → act but flag uncertainty
  private readonly CONFIRMATION_THRESHOLD = 0.40;     // Low confidence → ask human
  // Below CONFIRMATION_THRESHOLD → abort

  evaluate(results: VerificationResult[]): AgenticDecision {
    if (results.length === 0) {
      return {
        action: 'abort',
        reason: 'No retrieval results to verify',
        results: [],
      };
    }

    const topResult = results[0];  // Highest-ranked after reranking
    const topConfidence = topResult.agenticConfidence;

    // Check for SCIP mismatches in top 3
    const top3 = results.slice(0, 3);
    const mismatches = top3.filter(r => r.verificationConfidence === 'scip_mismatch');
    const staleFiles = top3.filter(r => r.verificationConfidence === 'stale_content');
    const verified = top3.filter(r => r.verificationConfidence === 'scip_verified');

    // Any stale files in top 3 → request confirmation
    if (staleFiles.length > 0) {
      return {
        action: 'request_confirmation',
        reason: `${staleFiles.length} of top 3 results have stale content — ` +
                `files modified since last indexing: ${staleFiles.map(r => r.filePath).join(', ')}`,
        results,
      };
    }

    // SCIP-verified top result with high search score → proceed
    if (topResult.verificationConfidence === 'scip_verified' && topConfidence >= this.PROCEED_THRESHOLD) {
      return {
        action: 'proceed',
        reason: `Top result SCIP-verified with confidence ${topConfidence.toFixed(3)}. ` +
                `${verified.length} of top 3 verified.`,
        results,
      };
    }

    // Any SCIP mismatches in top 3 → request confirmation
    if (mismatches.length > 0) {
      return {
        action: 'request_confirmation',
        reason: `SCIP mismatch detected: ${mismatches.map(r =>
          `${r.filePath} (${r.verificationDetails.filter(d => d.startsWith('✗')).join('; ')})`
        ).join('; ')}`,
        results,
      };
    }

    // High search score but no SCIP verification available → proceed with caution
    if (topConfidence >= this.CAUTION_THRESHOLD) {
      return {
        action: 'proceed_with_caution',
        reason: `Search score ${topResult.searchScore.toFixed(3)} but SCIP verification ` +
                `${topResult.verificationConfidence}. Agent should verify file content before modifying.`,
        results,
      };
    }

    // Low confidence → request confirmation
    if (topConfidence >= this.CONFIRMATION_THRESHOLD) {
      return {
        action: 'request_confirmation',
        reason: `Low agentic confidence ${topConfidence.toFixed(3)}. ` +
                `Top result verification: ${topResult.verificationConfidence}`,
        results,
      };
    }

    // Very low confidence → abort
    return {
      action: 'abort',
      reason: `Agentic confidence ${topConfidence.toFixed(3)} below safety threshold. ` +
              `Retrieval likely incorrect.`,
      results,
    };
  }
}
```

-----

## 10. Integration with Query Pipeline

### 10.1 Stage 14.5 — Post-Reranking Verification

```typescript
// src/lib/hybrid_search.ts — add after cross-encoder reranking

import { SCIPVerificationEngine } from './scip/verification_engine.js';
import { ConfidenceGate } from './scip/confidence_gate.js';

// In the search function, after stage 14 (cross-encoder rerank):

// ── Stage 14.5: SCIP Verification (top 5 only) ──
const verificationEngine = new SCIPVerificationEngine(pool, graphClient);
const confidenceGate = new ConfidenceGate();

const verificationRequests: VerificationRequest[] = rerankedTop5.map(result => ({
  chunkId: result.id,
  filePath: result.filePath,
  repoSlug: result.repoSlug,
  content: result.content,
  contentHash: result.contentHash,
  searchScore: result.rerankerScore,
  symbolNames: extractSymbolNames(result.content, result.language),
  querySymbol: extractPrimarySymbol(query),  // Best-effort symbol extraction from query
}));

const verificationResults = await verificationEngine.verifyBatch(verificationRequests);

// Annotate results with verification data
for (let i = 0; i < rerankedTop5.length; i++) {
  rerankedTop5[i].verification = verificationResults[i];
}

// Re-sort by agentic confidence (verification-adjusted score)
rerankedTop5.sort((a, b) => b.verification.agenticConfidence - a.verification.agenticConfidence);

// Agentic decision (returned to MCP — agent decides how to handle)
const agenticDecision = confidenceGate.evaluate(verificationResults);
```

### 10.2 Symbol Name Extraction from Chunks

```typescript
// src/lib/scip/symbol_extractor.ts

/**
 * Extract likely symbol names from a code chunk.
 * Uses lightweight heuristics — NOT a full parser.
 * These names are used to look up in the SCIP index.
 */
export function extractSymbolNames(content: string, language?: string): string[] {
  const symbols: string[] = [];

  if (language === 'java' || language === 'kotlin') {
    // Class declarations
    const classMatch = content.match(/(?:class|interface|enum)\s+(\w+)/g);
    if (classMatch) {
      symbols.push(...classMatch.map(m => m.split(/\s+/)[1]));
    }

    // Method declarations
    const methodMatch = content.match(
      /(?:public|private|protected|static|final|\s)+[\w<>\[\]]+\s+(\w+)\s*\(/g
    );
    if (methodMatch) {
      symbols.push(...methodMatch.map(m => {
        const parts = m.trim().split(/\s+/);
        return parts[parts.length - 1].replace('(', '');
      }));
    }
  }

  if (language === 'typescript' || language === 'javascript') {
    // Function declarations and arrow functions
    const funcMatch = content.match(/(?:function|const|let|var|export)\s+(\w+)/g);
    if (funcMatch) {
      symbols.push(...funcMatch.map(m => m.split(/\s+/).pop()!));
    }

    // Class declarations
    const classMatch = content.match(/class\s+(\w+)/g);
    if (classMatch) {
      symbols.push(...classMatch.map(m => m.split(/\s+/)[1]));
    }

    // Interface declarations
    const ifaceMatch = content.match(/interface\s+(\w+)/g);
    if (ifaceMatch) {
      symbols.push(...ifaceMatch.map(m => m.split(/\s+/)[1]));
    }
  }

  // Deduplicate and filter common noise
  const noise = new Set(['if', 'else', 'for', 'while', 'return', 'new', 'this', 'super',
                         'const', 'let', 'var', 'function', 'class', 'interface', 'extends',
                         'implements', 'import', 'export', 'from', 'async', 'await',
                         'public', 'private', 'protected', 'static', 'final', 'void',
                         'string', 'number', 'boolean', 'any', 'null', 'undefined']);

  return [...new Set(symbols)].filter(s => !noise.has(s.toLowerCase()) && s.length > 1);
}

/**
 * Extract the primary symbol from a natural language query.
 * "How does OrderValidator.validate work?" → "OrderValidator.validate"
 * "Find the authentication middleware" → null (no specific symbol)
 */
export function extractPrimarySymbol(query: string): string | undefined {
  // PascalCase or camelCase identifiers
  const symbolMatch = query.match(/\b([A-Z][a-zA-Z0-9]*(?:\.[a-zA-Z][a-zA-Z0-9]*)*)\b/);
  if (symbolMatch) return symbolMatch[1];

  // Quoted identifiers
  const quotedMatch = query.match(/['"`]([a-zA-Z_]\w*(?:\.\w+)*)['"`]/);
  if (quotedMatch) return quotedMatch[1];

  // camelCase identifiers
  const camelMatch = query.match(/\b([a-z][a-zA-Z0-9]*[A-Z][a-zA-Z0-9]*)\b/);
  if (camelMatch) return camelMatch[1];

  return undefined;
}
```

-----

## 11. Integration with MCP Tools

### 11.1 Enhanced MCP Response Format

```typescript
// src/mcp/search_response.ts

export interface MCPSearchResponse {
  results: MCPSearchResult[];

  // NEW: Agentic safety metadata
  agenticDecision: {
    action: 'proceed' | 'proceed_with_caution' | 'request_confirmation' | 'abort';
    reason: string;
  };
}

export interface MCPSearchResult {
  content: string;
  filePath: string;
  repoSlug: string;
  startLine: number;
  endLine: number;
  relevanceScore: number;

  // NEW: SCIP verification metadata
  verification?: {
    confidence: VerificationConfidence;
    agenticConfidence: number;
    symbolVerified: boolean;
    contentFresh: boolean;
    details: string[];
  };
}
```

### 11.2 Updated ome_search Tool

```typescript
// src/mcp/tools/ome_search.ts

export async function omeSearch(
  query: string,
  options: SearchOptions
): Promise<MCPSearchResponse> {
  // Run existing 16-stage pipeline
  const results = await hybridSearch(query, options);

  // Stage 14.5: SCIP verification
  const verificationEngine = new SCIPVerificationEngine(pool, graphClient);
  const confidenceGate = new ConfidenceGate();

  const verificationRequests = results.map(r => ({
    chunkId: r.id,
    filePath: r.filePath,
    repoSlug: r.repoSlug,
    content: r.content,
    contentHash: r.contentHash,
    searchScore: r.score,
    symbolNames: extractSymbolNames(r.content, r.language),
    querySymbol: extractPrimarySymbol(query),
  }));

  const verifications = await verificationEngine.verifyBatch(verificationRequests);
  const decision = confidenceGate.evaluate(verifications);

  return {
    results: results.map((r, i) => ({
      content: r.content,  // Original content, NOT headers
      filePath: r.filePath,
      repoSlug: r.repoSlug,
      startLine: r.startLine,
      endLine: r.endLine,
      relevanceScore: r.score,
      verification: {
        confidence: verifications[i].verificationConfidence,
        agenticConfidence: verifications[i].agenticConfidence,
        symbolVerified: verifications[i].symbolVerified,
        contentFresh: verifications[i].contentFresh,
        details: verifications[i].verificationDetails,
      },
    })),
    agenticDecision: {
      action: decision.action,
      reason: decision.reason,
    },
  };
}
```

### 11.3 Updated ome_blast_radius Tool

Blast radius analysis benefits heavily from SCIP verification — the caller/callee chain can be cross-checked against compiler-verified references.

```typescript
// src/mcp/tools/ome_blast_radius.ts

export async function omeBlastRadius(
  symbol: string,
  repoSlug: string
): Promise<BlastRadiusResponse> {
  // Existing graph-based blast radius
  const graphResult = await graphClient.getReverseCallChain(symbol, repoSlug, 3);

  // NEW: Cross-reference with SCIP for accuracy
  const scipCallers = await pool.query(`
    SELECT DISTINCT r.referencing_file, r.referencing_line, r.enclosing_symbol
    FROM scip_references r
    JOIN scip_symbols s ON s.symbol_uri = r.symbol_uri AND s.repo_slug = r.repo_slug
    WHERE s.display_name ILIKE $1
      AND s.repo_slug = $2
      AND r.reference_kind = 'reference'
    ORDER BY r.referencing_file
  `, [`%${symbol}%`, repoSlug]);

  // Merge: graph callers + SCIP callers, annotate confidence
  const mergedCallers = mergeBlastRadiusResults(graphResult, scipCallers.rows);

  return {
    symbol,
    callers: mergedCallers.map(c => ({
      ...c,
      confidence: c.inGraph && c.inSCIP ? 'verified' :
                  c.inSCIP ? 'scip_only' :
                  c.inGraph ? 'graph_only' : 'unverified',
    })),
    totalAffectedFiles: mergedCallers.length,
  };
}
```

-----

## 12. Integration with ome_rag_guide IMPLEMENT Mode

The IMPLEMENT mode output format already includes line numbers and content hashes for agent verification. SCIP verification enhances this with compiler-verified symbol locations.

```typescript
// src/mcp/tools/ome_rag_guide.ts — IMPLEMENT mode enhancement

export async function omeRagGuideImplement(
  task: string,
  targetFiles: string[]
): Promise<ImplementModeResponse> {
  // Existing: retrieve relevant code, generate implementation plan
  const results = await omeSearch(task, { topK: 10 });

  // NEW: For each target file, provide SCIP-verified symbol locations
  const fileStates = await Promise.all(
    targetFiles.map(async (filePath) => {
      const repoSlug = inferRepoSlug(filePath);
      const contentVerifier = new ContentHashVerifier(pool, REPOS_DIR);
      const currentState = await contentVerifier.getCurrentFileState(filePath, repoSlug);

      if (!currentState) {
        return { filePath, exists: false };
      }

      // Get SCIP symbols in this file for precise modification targets
      const symbols = await pool.query(`
        SELECT display_name, kind, start_line, end_line, signature, symbol_uri
        FROM scip_symbols
        WHERE file_path = $1 AND repo_slug = $2
        ORDER BY start_line
      `, [filePath, repoSlug]);

      return {
        filePath,
        exists: true,
        contentHash: currentState.hash,
        lineCount: currentState.lineCount,
        symbols: symbols.rows.map(s => ({
          name: s.display_name,
          kind: s.kind,
          startLine: s.start_line,
          endLine: s.end_line,
          signature: s.signature,
        })),
      };
    })
  );

  return {
    task,
    retrievalConfidence: results.agenticDecision,
    targetFiles: fileStates,
    // Agent uses contentHash to verify file hasn't changed before applying modifications
    // Agent uses SCIP symbol locations for precise line-level modifications
  };
}
```

-----

## 13. SCIP Index Refresh Strategy

### 13.1 During Bootstrap (Incremental)

```typescript
// In bootstrap.ts — after Phase 2g knowledge graph

// Phase 2g.5: SCIP indexing
if (changedFiles.length > 0) {
  const language = project.tech_stack?.language?.includes('TypeScript') ? 'typescript' : 'java';

  try {
    // Generate SCIP index for changed files
    await generateSCIPIndex(repoBasePath, language);

    // Index into PostgreSQL
    const scipResult = await indexSCIPData(pool, repo, repoBasePath, language);
    console.log(
      `[SCIP] ${repo}: ${scipResult.symbols} symbols, ` +
      `${scipResult.references} references, ${scipResult.types} types`
    );
  } catch (err) {
    // SCIP failure is non-fatal — degrade gracefully
    console.warn(`[SCIP] ${repo}: indexing failed (non-fatal):`, err);
  }
}
```

### 13.2 SCIP Index Generation

```typescript
// src/ingestion/scip_generator.ts

export async function generateSCIPIndex(
  repoBasePath: string,
  language: 'typescript' | 'java'
): Promise<string> {
  const indexPath = path.join(repoBasePath, 'index.scip');

  if (language === 'typescript') {
    // scip-typescript indexes TypeScript/JavaScript projects
    await execAsync(
      `cd "${repoBasePath}" && npx @sourcegraph/scip-typescript index --output "${indexPath}"`,
      { timeout: 300_000 }  // 5 min timeout
    );
  } else if (language === 'java') {
    // scip-java indexes Maven/Gradle Java projects
    // Requires the project to be buildable
    await execAsync(
      `cd "${repoBasePath}" && scip-java index --output "${indexPath}"`,
      { timeout: 600_000 }  // 10 min timeout for Java builds
    );
  }

  return indexPath;
}
```

### 13.3 Air-Gap Consideration

`scip-typescript` requires `npx` (npm). On your air-gapped RHEL:

- Pre-install `@sourcegraph/scip-typescript` globally: `npm install -g @sourcegraph/scip-typescript`
- Or bundle it in your project’s `node_modules` before deploying to the server
- `scip-java` is a standalone binary — download once, copy to server

-----

## 14. Performance Budget

SCIP verification runs on only 5 results (post-reranking), so the cost is minimal:

|Operation                          |Time     |Notes                              |
|-----------------------------------|---------|-----------------------------------|
|Symbol resolution (5 files)        |~5ms     |5 indexed queries on scip_symbols  |
|Call chain verification (5 files)  |~10ms    |10-15 queries on scip_references   |
|Type relationship verification     |~5ms     |5-10 queries on scip_type_relations|
|Content hash verification (5 files)|~15ms    |5 file reads + SHA-256             |
|Confidence computation             |~1ms     |In-memory calculation              |
|**Total Stage 14.5**               |**~36ms**|**Within P95 latency budget**      |

For comparison, the existing pipeline stages:

- Dense ANN: ~80ms
- BM25: ~30ms
- Bi-encoder rerank: ~400ms
- Cross-encoder rerank: ~1,500ms
- **SCIP verification: ~36ms** (2.4% of total pipeline time)

### 14.1 Index Sizes

|Table              |Estimated Rows (47 repos)|Estimated Size|
|-------------------|-------------------------|--------------|
|scip_symbols       |~500K-1M                 |~200-400MB    |
|scip_references    |~2M-5M                   |~400MB-1GB    |
|scip_type_relations|~50K-100K                |~10-20MB      |

Well within your 1TB RAM PostgreSQL budget. The indexed lookups are sub-millisecond.

-----

## 15. Graceful Degradation

SCIP verification is **non-blocking** and **non-fatal**. If any part fails, the system falls back to search-score-only confidence.

```typescript
// src/lib/scip/verification_engine.ts — top-level try/catch

async verifyWithFallback(requests: VerificationRequest[]): Promise<VerificationResult[]> {
  try {
    // Check if SCIP tables have data
    const hasData = await this.pool.query(
      `SELECT EXISTS (SELECT 1 FROM scip_symbols LIMIT 1) AS has_data`
    );

    if (!hasData.rows[0].has_data) {
      console.debug('[SCIP] No SCIP data available — skipping verification');
      return requests.map(r => this.fallbackResult(r));
    }

    return await this.verifyBatch(requests);
  } catch (err) {
    console.warn('[SCIP] Verification failed (non-fatal):', err);
    return requests.map(r => this.fallbackResult(r));
  }
}

private fallbackResult(request: VerificationRequest): VerificationResult {
  return {
    chunkId: request.chunkId,
    filePath: request.filePath,
    repoSlug: request.repoSlug,
    symbolVerified: false,
    callChainVerified: null,
    typeRelationVerified: null,
    contentFresh: true,  // Assume fresh if we can't verify
    verificationConfidence: 'scip_unavailable',
    verificationDetails: ['SCIP verification unavailable — using search score only'],
    searchScore: request.searchScore,
    agenticConfidence: request.searchScore * 0.9,  // Slight penalty for unverified
  };
}
```

### 15.1 Per-Repo SCIP Coverage

Not all 47 repos will have SCIP indexes (some may fail to index, some may use unsupported languages). The system tracks coverage:

```typescript
// Query at health-check time
const coverage = await pool.query(`
  SELECT
    p.repo_slug,
    COUNT(DISTINCT s.file_path) AS scip_files,
    p.total_files,
    ROUND(COUNT(DISTINCT s.file_path)::numeric / GREATEST(p.total_files, 1) * 100, 1) AS coverage_pct
  FROM projects p
  LEFT JOIN scip_symbols s ON s.repo_slug = p.repo_slug
  GROUP BY p.repo_slug, p.total_files
  ORDER BY coverage_pct DESC
`);

// Expected: TypeScript repos ~80-90% coverage, Java repos ~70-80%
// Config/build files won't have SCIP data — that's expected
```

-----

## 16. Implementation Schedule

```
Day 1: SCIP Schema + Index Population
  ├── Create scip_symbols, scip_references, scip_type_relations tables
  ├── Create all indexes
  ├── Implement indexSCIPData() function
  ├── Implement generateSCIPIndex() for TypeScript and Java
  ├── Test on 2-3 representative repos (1 TypeScript MFE, 1 Java MS, 1 fullstack)
  ├── Verify symbol counts and reference counts are reasonable
  └── Integrate into bootstrap Phase 2g.5

Day 2: Verification Engine Core
  ├── Implement SCIPVerificationEngine
  ├── Implement SymbolVerifier
  ├── Implement CallChainVerifier
  ├── Implement TypeRelationshipVerifier
  ├── Implement ContentHashVerifier
  ├── Unit tests with mock SCIP data
  └── Verify graceful fallback when SCIP unavailable

Day 3: Confidence Gating + Query Pipeline Integration
  ├── Implement ConfidenceGate with thresholds
  ├── Implement extractSymbolNames() and extractPrimarySymbol()
  ├── Integrate as Stage 14.5 in hybrid_search.ts
  ├── Add verification metadata to search results
  ├── Test end-to-end with golden queries
  └── Verify latency impact (<50ms added)

Day 4: MCP Tool Integration
  ├── Update ome_search response format with verification metadata
  ├── Update ome_blast_radius with SCIP cross-reference
  ├── Update ome_rag_guide IMPLEMENT mode with SCIP symbol locations
  ├── Update MCPSearchResponse type definitions
  ├── Test MCP tools via Windsurf with verification annotations
  └── Verify agent receives confidence signals correctly

Day 5: Full Bootstrap Run + Tuning
  ├── Run full bootstrap with SCIP indexing on all 47 repos
  ├── Measure SCIP coverage per repo
  ├── Tune confidence thresholds on golden queries
  ├── Test edge cases: deleted files, renamed symbols, cross-repo references
  ├── Verify graceful degradation for repos without SCIP data
  └── Document coverage gaps and known limitations
```

### Configuration Reference

```bash
# ── SCIP Settings ──
SCIP_ENABLED=true                          # Master switch for SCIP verification
SCIP_TYPESCRIPT_CMD=scip-typescript        # Path to scip-typescript binary
SCIP_JAVA_CMD=scip-java                    # Path to scip-java binary
SCIP_INDEX_TIMEOUT=300000                  # 5 min timeout for index generation
SCIP_VERIFY_AT_QUERY=true                  # Enable Stage 14.5 verification

# ── Confidence Thresholds ──
SCIP_PROCEED_THRESHOLD=0.80               # Agent can act autonomously
SCIP_CAUTION_THRESHOLD=0.60               # Agent acts but flags uncertainty
SCIP_CONFIRMATION_THRESHOLD=0.40          # Agent requests human confirmation
# Below 0.40 → agent aborts

# ── Verification Multipliers ──
SCIP_VERIFIED_BOOST=1.15                  # Boost when SCIP confirms
SCIP_CONSISTENT_MULTIPLIER=1.0            # Neutral — no contradiction
SCIP_MISMATCH_PENALTY=0.6                 # Penalize SCIP contradictions
SCIP_UNAVAILABLE_PENALTY=0.9              # Slight penalty for unverifiable
SCIP_STALE_PENALTY=0.5                    # Strong penalty for stale files
```

### File Structure

```
src/lib/scip/
  verification_engine.ts       — Main engine orchestrating all verifiers
  symbol_verifier.ts           — SCIP symbol resolution verification
  callchain_verifier.ts        — SCIP call chain cross-reference
  type_verifier.ts             — SCIP type hierarchy verification
  content_verifier.ts          — Content hash freshness check
  confidence_gate.ts           — Agentic safety gate with thresholds
  symbol_extractor.ts          — Extract symbol names from chunks and queries

src/ingestion/
  scip_indexer.ts              — Populate SCIP tables from scip print --json
  scip_generator.ts            — Generate SCIP index via scip-typescript/scip-java

src/mcp/tools/
  ome_search.ts                — Updated with verification metadata
  ome_blast_radius.ts          — Updated with SCIP cross-reference
  ome_rag_guide.ts             — IMPLEMENT mode with SCIP symbol locations
```

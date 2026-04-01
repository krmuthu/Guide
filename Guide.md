# API Dependency Mapping — Implementation Plan

> **Replaces:** All previous MFE↔MS endpoint detection drafts
> **Goal:** Map every API dependency between MFE↔MS and MS↔MS repos via a single `gateway_prefix` column in the `projects` table, OpenAPI specs as primary endpoint source, and SCIP as source-location binder. Creates cross-repo `consumes_api` edges in the knowledge graph.
> **Based On:** Actual bootstrap pipeline (bootstrap.ts), existing IGraphClient, ProjectAnalyzer, projects table, REPOS_DIR convention
> **Stack:** OpenAPI specs · SCIP · tree-sitter · Qwen2.5-7B · IGraphClient · pgvector · 128-core RHEL
> **Effort:** ~8 days across 4 phases
> **Last Updated:** April 2026

-----

## Table of Contents

1. [Problem Statement](#1-problem-statement)
1. [Architecture — OpenAPI-First, gateway_prefix-Routed](#2-architecture)
1. [Extraction Source Hierarchy](#3-extraction-source-hierarchy)
1. [Schema Changes](#4-schema-changes)
1. [gateway_prefix — Auto-Population + Matching](#5-gateway_prefix)
1. [Phase A: MS Endpoint Extraction (OpenAPI + SCIP Binding)](#6-phase-a-ms-endpoint-extraction)
1. [Phase B: Consumer Extraction (MFE + MS-to-MS)](#7-phase-b-consumer-extraction)
1. [Phase C: Proxy Layer Resolution](#8-phase-c-proxy-layer-resolution)
1. [Phase D: Cross-Repo Matching + Graph Integration](#9-phase-d-cross-repo-matching--graph-integration)
1. [Endpoint Matching Engine — Disambiguation Chain](#10-endpoint-matching-engine)
1. [Shared / Platform Service Classification](#11-shared--platform-service-classification)
1. [Graph Schema — Nodes, Edges, Traversal](#12-graph-schema)
1. [Bootstrap Pipeline Integration](#13-bootstrap-pipeline-integration)
1. [MCP Tool Enhancements](#14-mcp-tool-enhancements)
1. [Health Score Integration — 14 Finding Types](#15-health-score-integration)
1. [Automatic PR Blast-Radius Comments](#16-automatic-pr-blast-radius-comments)
1. [Chatty Service Pair Detection](#17-chatty-service-pair-detection)
1. [Distributed Tracing Integration (Future)](#18-distributed-tracing-integration)
1. [Validation & Golden Endpoints](#19-validation--golden-endpoints)
1. [Implementation Schedule](#20-implementation-schedule)
1. [Configuration Reference](#21-configuration-reference)

-----

## 1. Problem Statement

### 1.1 The Missing Link

The knowledge graph captures intra-repo relationships — `calls`, `imports`, `extends` — but the MFE→MS and MS→MS dependencies are invisible. These connections cross repo boundaries via HTTP through a common API gateway:

```
mfe-bonds-trading (React)        Gateway              ms-bonds-trading (Spring Boot)
┌────────────────────┐     ┌──────────────┐     ┌──────────────────────────┐
│ OrderForm.tsx      │     │              │     │ OrderController.java     │
│  fetch('/api/bonds │ ──→ │ strips       │ ──→ │  @PostMapping("/orders") │
│    /orders', data) │     │ /api/bonds   │     │  createOrder(dto) {...}  │
└────────────────────┘     └──────────────┘     └──────────────────────────┘

ms-bonds-trading (Spring Boot)   Gateway              ms-auth (Spring Boot)
┌────────────────────┐     ┌──────────────┐     ┌──────────────────────────┐
│ OrderService.java  │     │              │     │ AuthController.java      │
│  authClient        │ ──→ │ strips       │ ──→ │  @GetMapping("/validate")│
│   .validate(token) │     │ /api/auth    │     │  validateToken(t) {...}  │
└────────────────────┘     └──────────────┘     └──────────────────────────┘
```

**Key insight:** Every MS has a unique gateway prefix (`/api/bonds`, `/api/auth`, `/api/equity`). This prefix is the disambiguator — it tells you exactly which MS a consumer is calling, even when multiple MSes have the same endpoint path like `/orders` or `/validate`.

### 1.2 What This Unlocks

|Capability      |Without                        |With                                                                |
|----------------|-------------------------------|--------------------------------------------------------------------|
|Blast radius    |Stops at repo boundary         |Crosses HTTP: controller change → all affected MFEs + MSes          |
|Code context    |In-repo callers only           |Full chain: React hook → gateway → controller → service → repository|
|Dead code       |Graph-based (in-repo)          |Orphan endpoints (MS exposes, nobody calls)                         |
|Health score    |MFE checks blind to MS deps    |Contract mismatches, unmatched calls, missing error handling        |
|Windsurf context|Can’t answer “what calls this?”|Full consumer map across all 47 repos                               |
|Change planning |Manual cross-repo assessment   |Automated: “changing /orders affects 3 MFEs and 2 MSes”             |

-----

## 2. Architecture

```
Bootstrap Phase 2g-api — runs after graph build, before contextual headers
  │
  │  ┌──────────────────────────────────────────────────────────────┐
  │  │  SETUP: gateway_prefix in projects table                    │
  │  │                                                              │
  │  │  Auto-populated from application.yml context-path or         │
  │  │  spring.application.name convention                          │
  │  │                                                              │
  │  │  ms-bonds-trading  → /api/bonds                             │
  │  │  ms-equity-trading → /api/equity                            │
  │  │  ms-auth           → /api/auth                              │
  │  │                                                              │
  │  │  THIS REPLACES: gateway-routes.json, gateway repo parsing   │
  │  └──────────────────────────────────────────────────────────────┘
  │
  │  ┌──────────────────────────────────────────────────────────────┐
  │  │  PHASE A: MS Endpoint Extraction (per-repo, parallel)       │
  │  │                                                              │
  │  │  OpenAPI Spec ──── authoritative contract ───┐  MERGE       │
  │  │    (most MSes)     paths, params, DTOs       ├──→ endpoint  │
  │  │                                               │    registry  │
  │  │  SCIP Binding ─── source file, line, ────────┘              │
  │  │    (adds location)  graph_node_id                            │
  │  │                                                              │
  │  │  SCIP-Only ────── repos without spec                        │
  │  │  AST Fallback ──── repos without SCIP                       │
  │  └──────────────────────────────────────────────────────────────┘
  │
  │  ┌──────────────────────────────────────────────────────────────┐
  │  │  PHASE B: Consumer Extraction (per-repo, parallel)          │
  │  │                                                              │
  │  │  MFE: axios, fetch, RTK Query, React Query, API wrappers   │
  │  │       → SCIP type-verified or AST heuristic                 │
  │  │       → client base URL → env var → target repo hint        │
  │  │                                                              │
  │  │  MS-to-MS: Feign (@FeignClient name → target service)      │
  │  │            RestTemplate (SCIP type + @Value → app.yml → URL)│
  │  │            WebClient (SCIP chain + .uri() argument)         │
  │  └──────────────────────────────────────────────────────────────┘
  │
  │  ┌──────────────────────────────────────────────────────────────┐
  │  │  PHASE C: Proxy Layer Resolution (MFE repos only)           │
  │  │                                                              │
  │  │  Layer 1: Next.js rewrites (next.config.js)                 │
  │  │  Layer 2: Next.js API routes (pages/api, app/api)           │
  │  │  Layer 3: Express BFF (http-proxy-middleware, manual proxy)  │
  │  │  Layer 4: Infra configs — httpd/nginx/k8s (LLM-parsed)     │
  │  │  Layer 5: Vite/Webpack dev proxy (supplementary signal)     │
  │  └──────────────────────────────────────────────────────────────┘
  │
  │  ┌──────────────────────────────────────────────────────────────┐
  │  │  PHASE D: Cross-Repo Matching (sequential, needs all data)  │
  │  │                                                              │
  │  │  1. Load gateway routes from projects.gateway_prefix         │
  │  │  2. Run disambiguation chain per consumer call               │
  │  │  3. Classify services: platform / domain / utility           │
  │  │  4. Create graph edges via IGraphClient                      │
  │  │  5. Refresh mfe_ms_dependencies materialized view            │
  │  │  6. Run chatty-pair analysis                                 │
  │  └──────────────────────────────────────────────────────────────┘
```

### 2.1 Why OpenAPI-First Changes Everything

Since most MSes already have OpenAPI/Swagger specs:

|Without OpenAPI                 |With OpenAPI (Your Case)                                   |
|--------------------------------|-----------------------------------------------------------|
|Parse every Java controller file|Parse one JSON/YAML spec per MS                            |
|Heuristic annotation matching   |Machine-generated from annotations — authoritative         |
|Partial type info               |Full schema: params, DTOs, response types, deprecated flags|
|SCIP needed for type extraction |SCIP only needed for source location binding               |
|Minutes per MS                  |Seconds per MS                                             |

### 2.2 Why gateway_prefix in projects Table

Every MS has a unique gateway prefix. Storing it in the `projects` table (which bootstrap already populates) eliminates:

- ❌ Static `gateway-routes.json` config file
- ❌ Parsing the gateway repo’s `application.yml`
- ❌ Manual route maintenance when MSes are added
- ❌ Separate `GatewayRoute[]` data structure

The prefix auto-populates from `application.yml` context-path or `spring.application.name` convention. One column, zero config files.

-----

## 3. Extraction Source Hierarchy

Sources ranked by accuracy. Higher sources win when conflicts arise.

```
 #   Source                          Confidence          When Used
 ─── ────────────────────────────── ─────────────────── ──────────────────────────────
 1   OpenAPI Spec + SCIP Binding     spec_verified       MS with spec + SCIP available
 2   OpenAPI Spec Only               spec_verified       MS with spec, SCIP unavailable
 3   SCIP-Only Extraction            scip_verified       MS without spec, SCIP available
 4   Integration Test Evidence       test_verified       Promotes heuristic findings
 5   Tree-Sitter AST Heuristic       heuristic           No spec, no SCIP
 6   LLM Config Parsing              llm_extracted       Infra proxy configs only
 ─── ────────────────────────────── ─────────────────── ──────────────────────────────
 Future:
 7   Runtime Traces (OTel/Jaeger)    runtime_verified    Would become #1 if available
```

When an endpoint appears in both OpenAPI spec and SCIP, take contract info (params, DTOs, response types) from the spec and source location (file, line, graph_node_id) from SCIP.

-----

## 4. Schema Changes

### 4.1 projects Table — Single Column Addition

```sql
ALTER TABLE projects ADD COLUMN IF NOT EXISTS gateway_prefix TEXT;
-- /api/bonds, /api/equity, /api/auth, etc.
-- What the common gateway uses to route to this MS
-- NULL for MFE repos and repos without a gateway route
```

Additional metadata columns on the existing `projects` table:

```sql
ALTER TABLE projects ADD COLUMN IF NOT EXISTS has_openapi_spec BOOLEAN DEFAULT false;
ALTER TABLE projects ADD COLUMN IF NOT EXISTS api_endpoint_count INTEGER DEFAULT 0;
ALTER TABLE projects ADD COLUMN IF NOT EXISTS api_consumer_count INTEGER DEFAULT 0;
ALTER TABLE projects ADD COLUMN IF NOT EXISTS service_role TEXT;
-- 'platform' (ms-auth, consumed by many), 'domain' (ms-bonds, paired to one MFE), 'utility'
```

### 4.2 New Tables

```sql
-- ═══════════════════════════════════════════════════════════════
-- MS endpoints — the API surface
-- ═══════════════════════════════════════════════════════════════

CREATE TABLE IF NOT EXISTS endpoint_registry (
  id                  SERIAL PRIMARY KEY,
  repo_slug           TEXT NOT NULL,
  http_method         TEXT NOT NULL,           -- GET, POST, PUT, DELETE, PATCH
  full_path           TEXT NOT NULL,           -- /orders/{id} (MS-local, gateway prefix stripped)

  -- Contract info (from OpenAPI spec)
  operation_id        TEXT,                    -- createOrder
  summary             TEXT,                    -- "Create a new order"
  tags                TEXT[],                  -- ["Orders", "Trading"]
  deprecated          BOOLEAN DEFAULT false,
  parameters          JSONB DEFAULT '[]',      -- [{name, location, type, required}]
  request_type        TEXT,                    -- CreateOrderRequest
  request_schema      JSONB,                   -- Full JSON Schema from spec
  response_type       TEXT,                    -- OrderResponse
  response_schema     JSONB,
  security_schemes    TEXT[],                  -- ["bearerAuth"]

  -- Source binding (from SCIP or AST)
  file_path           TEXT,                    -- src/main/java/.../OrderController.java
  class_name          TEXT,                    -- OrderController
  method_name         TEXT,                    -- createOrder
  start_line          INTEGER,
  end_line            INTEGER,
  graph_node_id       TEXT,                    -- link to existing function GraphNode
  binding_method      TEXT,                    -- scip_operationId, scip_annotation, ast_annotation, unbound

  -- Metadata
  extraction_source   TEXT NOT NULL,           -- openapi_plus_scip, openapi_only, scip, ast_heuristic
  confidence          TEXT NOT NULL,           -- spec_verified, scip_verified, test_verified, heuristic
  spec_path           TEXT,                    -- openapi.json (if from spec)
  undocumented        BOOLEAN DEFAULT false,   -- found by SCIP but not in spec
  tested_by           TEXT[],                  -- test files that hit this endpoint
  is_active           BOOLEAN DEFAULT true,
  created_at          TIMESTAMPTZ DEFAULT NOW(),
  updated_at          TIMESTAMPTZ DEFAULT NOW(),

  UNIQUE (repo_slug, http_method, full_path)
);

CREATE INDEX idx_ep_path ON endpoint_registry (full_path, http_method) WHERE is_active = true;
CREATE INDEX idx_ep_repo ON endpoint_registry (repo_slug) WHERE is_active = true;
CREATE INDEX idx_ep_opid ON endpoint_registry (operation_id) WHERE operation_id IS NOT NULL AND is_active = true;

-- ═══════════════════════════════════════════════════════════════
-- API calls — MFE + MS consumers
-- ═══════════════════════════════════════════════════════════════

CREATE TABLE IF NOT EXISTS api_calls (
  id                    SERIAL PRIMARY KEY,
  repo_slug             TEXT NOT NULL,
  file_path             TEXT NOT NULL,
  function_name         TEXT,                  -- enclosing function/hook name
  http_method           TEXT NOT NULL,
  raw_path              TEXT NOT NULL,          -- exact string from source: /api/bonds/orders
  normalized_path       TEXT NOT NULL,          -- gateway prefix stripped: /orders
  has_error_handling    BOOLEAN DEFAULT false,
  call_pattern          TEXT NOT NULL,          -- axios, fetch, rtk_query, feign_client, rest_template, web_client
  target_repo_hint      TEXT,                   -- resolved target MS repo slug
  target_resolution     TEXT,                   -- gateway_prefix, feign_name, client_binding, env_var, repo_pairing
  via_proxy             TEXT,                   -- proxy file path if through code proxy
  proxy_inbound_path    TEXT,
  start_line            INTEGER,
  end_line              INTEGER,
  graph_node_id         TEXT,

  -- Match result
  matched_endpoint_id   INTEGER REFERENCES endpoint_registry(id),
  match_confidence      TEXT,                   -- exact, high, medium, ambiguous
  match_method          TEXT,                   -- gateway_prefix, feign_name, unique_path, etc.

  -- Metadata
  extraction_source     TEXT NOT NULL,          -- scip, ast_heuristic
  confidence            TEXT NOT NULL,
  tested_by             TEXT[],
  is_active             BOOLEAN DEFAULT true,
  created_at            TIMESTAMPTZ DEFAULT NOW(),
  updated_at            TIMESTAMPTZ DEFAULT NOW()
);

CREATE INDEX idx_ac_path ON api_calls (normalized_path, http_method) WHERE is_active = true;
CREATE INDEX idx_ac_repo ON api_calls (repo_slug) WHERE is_active = true;
CREATE INDEX idx_ac_unmatched ON api_calls (repo_slug)
  WHERE matched_endpoint_id IS NULL AND is_active = true;

-- ═══════════════════════════════════════════════════════════════
-- DTO schemas — for cross-service contract validation
-- ═══════════════════════════════════════════════════════════════

CREATE TABLE IF NOT EXISTS api_dto_schemas (
  id              SERIAL PRIMARY KEY,
  repo_slug       TEXT NOT NULL,
  dto_name        TEXT NOT NULL,               -- OrderDTO
  fields          JSONB NOT NULL,              -- [{name, type, required, description}]
  used_as_request TEXT[],                      -- ["POST /orders"]
  used_as_response TEXT[],                     -- ["GET /orders/{id}"]
  spec_path       TEXT,
  is_active       BOOLEAN DEFAULT true,
  UNIQUE (repo_slug, dto_name)
);

-- ═══════════════════════════════════════════════════════════════
-- MFE proxy routes — resolved proxy configurations
-- ═══════════════════════════════════════════════════════════════

CREATE TABLE IF NOT EXISTS mfe_proxy_routes (
  id              SERIAL PRIMARY KEY,
  repo_slug       TEXT NOT NULL,               -- MFE repo
  inbound_prefix  TEXT NOT NULL,               -- /api/bonds (what MFE calls)
  target_repo     TEXT,                        -- ms-bonds-trading (where it goes)
  strip_prefix    BOOLEAN DEFAULT true,
  proxy_type      TEXT NOT NULL,               -- nextjs_rewrite, nextjs_api_route, http_proxy_middleware, llm_infra, dev_proxy
  proxy_file_path TEXT,                        -- next.config.js, server/proxy.ts
  confidence      TEXT NOT NULL,               -- high, medium, low, llm_extracted
  is_active       BOOLEAN DEFAULT true,
  UNIQUE (repo_slug, inbound_prefix, proxy_type)
);

-- ═══════════════════════════════════════════════════════════════
-- LLM infra proxy extraction cache
-- ═══════════════════════════════════════════════════════════════

CREATE TABLE IF NOT EXISTS infra_proxy_cache (
  repo_slug    TEXT NOT NULL,
  config_hash  TEXT NOT NULL,                  -- SHA-256 of config file content
  file_path    TEXT NOT NULL,
  routes       JSONB NOT NULL,
  created_at   TIMESTAMPTZ DEFAULT NOW(),
  PRIMARY KEY (repo_slug, config_hash)
);

-- ═══════════════════════════════════════════════════════════════
-- Cross-repo dependency materialized view
-- ═══════════════════════════════════════════════════════════════

CREATE MATERIALIZED VIEW IF NOT EXISTS mfe_ms_dependencies AS
SELECT
  ac.repo_slug AS consumer_repo,
  p_c.project_type AS consumer_type,
  er.repo_slug AS provider_repo,
  p_p.service_role AS provider_role,
  p_p.gateway_prefix AS provider_prefix,
  COUNT(*) AS call_count,
  COUNT(DISTINCT ac.file_path) AS consumer_files,
  COUNT(DISTINCT er.full_path) AS provider_endpoints,
  array_agg(DISTINCT er.http_method || ' ' || er.full_path
    ORDER BY er.http_method || ' ' || er.full_path) AS endpoints_used,
  MIN(ac.match_confidence) AS weakest_match
FROM api_calls ac
JOIN endpoint_registry er ON ac.matched_endpoint_id = er.id
JOIN projects p_c ON p_c.repo_slug = ac.repo_slug
JOIN projects p_p ON p_p.repo_slug = er.repo_slug
WHERE ac.is_active = true AND er.is_active = true
GROUP BY ac.repo_slug, p_c.project_type, er.repo_slug, p_p.service_role, p_p.gateway_prefix;

CREATE UNIQUE INDEX ON mfe_ms_dependencies (consumer_repo, provider_repo);
```

-----

## 5. gateway_prefix

### 5.1 Auto-Population During Bootstrap

Three sources tried in order during Phase 2b (where ProjectAnalyzer already runs):

```typescript
// src/lib/endpoint_extraction/gateway_prefix_resolver.ts

export async function resolveGatewayPrefix(
  repoSlug: string,
  repoBasePath: string,
  config: ProjectConfig
): Promise<{ prefix: string | null; source: string }> {

  // ── Source 1: application.yml server.servlet.context-path ──
  const contextPath = config.applicationYml?.server?.servlet?.['context-path']
    ?? config.applicationYml?.server?.['context-path']
    ?? null;

  if (contextPath && contextPath !== '/') {
    return { prefix: contextPath, source: 'application_yml_context_path' };
  }

  // ── Source 2: spring.application.name convention ──
  const appName = config.applicationYml?.spring?.application?.name;
  if (appName) {
    // ms-bonds-trading → /api/bonds-trading
    // bonds-trading-service → /api/bonds-trading
    const cleaned = appName
      .replace(/^ms-/, '')
      .replace(/-service$/, '')
      .replace(/-ms$/, '');
    return { prefix: `/api/${cleaned}`, source: 'app_name_convention' };
  }

  // ── Source 3: Repo slug convention ──
  // ms-bonds-trading → /api/bonds-trading
  if (repoSlug.startsWith('ms-')) {
    const cleaned = repoSlug.replace(/^ms-/, '');
    return { prefix: `/api/${cleaned}`, source: 'repo_slug_convention' };
  }

  return { prefix: null, source: 'none' };
}
```

Integration with bootstrap:

```typescript
// In bootstrap.ts, Phase 2b — after ProjectAnalyzer runs:

const config = await loadProjectConfig(pool, repo.slug, repoBasePath);
const { prefix, source } = await resolveGatewayPrefix(repo.slug, repoBasePath, config);

await pool.query(
  `UPDATE projects SET gateway_prefix = $1 WHERE repo_slug = $2`,
  [prefix, repo.slug]
);

if (prefix) {
  console.log(`[Bootstrap] ${repo.slug}: gateway_prefix=${prefix} (${source})`);
} else {
  console.log(`[Bootstrap] ${repo.slug}: no gateway_prefix detected`);
}
```

### 5.2 Manual Override

For MSes where auto-detection produces wrong results:

```json
// config/gateway-prefix-overrides.json (optional)
{
  "ms-settlement": "/api/settle",
  "ms-legacy-reports": "/api/reports"
}
```

```typescript
// Applied after auto-detection
async function applyGatewayPrefixOverrides(pool: Pool): Promise<void> {
  const overridesPath = process.env.GATEWAY_PREFIX_OVERRIDES || './config/gateway-prefix-overrides.json';
  try {
    const overrides = JSON.parse(await fs.readFile(overridesPath, 'utf-8'));
    for (const [repo, prefix] of Object.entries(overrides)) {
      await pool.query(
        `UPDATE projects SET gateway_prefix = $1 WHERE repo_slug = $2`,
        [prefix, repo]
      );
      console.log(`[Bootstrap] ${repo}: gateway_prefix=${prefix} (manual override)`);
    }
  } catch { /* No overrides file — fine */ }
}
```

### 5.3 Loading Gateway Routes for Matching

The entire gateway routing table comes from one query:

```typescript
// src/lib/endpoint_extraction/matching/gateway_loader.ts

export interface GatewayRoute {
  prefix: string;
  targetRepo: string;
}

export async function loadGatewayRoutes(pool: Pool): Promise<GatewayRoute[]> {
  const result = await pool.query(`
    SELECT repo_slug, gateway_prefix
    FROM projects
    WHERE project_type IN ('backend', 'fullstack')
      AND gateway_prefix IS NOT NULL
      AND active = true
  `);

  return result.rows.map(r => ({
    prefix: r.gateway_prefix,
    targetRepo: r.repo_slug,
  }));
}
```

### 5.4 How gateway_prefix Resolves Duplicate Paths

The core matching flow:

```
MFE calls: POST /api/bonds/orders
  │
  ├── projects table: gateway_prefix '/api/bonds' → ms-bonds-trading
  ├── Strip prefix: /api/bonds/orders → /orders
  ├── endpoint_registry: ms-bonds-trading has POST /orders
  └── Match ✅ confidence: exact

MFE calls: POST /api/equity/orders
  │
  ├── projects table: gateway_prefix '/api/equity' → ms-equity-trading
  ├── Strip prefix: /api/equity/orders → /orders
  ├── endpoint_registry: ms-equity-trading has POST /orders
  └── Match ✅ confidence: exact — no ambiguity despite same /orders path

MFE calls: POST /api/auth/validate
  │
  ├── projects table: gateway_prefix '/api/auth' → ms-auth
  ├── Strip prefix: /api/auth/validate → /validate
  ├── endpoint_registry: ms-auth has POST /validate
  └── Match ✅ confidence: exact — platform service resolved cleanly
```

-----

## 6. Phase A: MS Endpoint Extraction

For each MS repo, extract all exposed endpoints.

### 6.1 Strategy Per Repo

```
Input:  MS repo on disk at REPOS_DIR
Output: ExtractedEndpoint[] stored in endpoint_registry

Decision tree:
  Has OpenAPI spec?
    YES → Parse spec (authoritative contract: paths, params, DTOs, deprecated)
          Has SCIP index?
            YES → Bind spec endpoints to source locations (file, line, graph_node_id)
            NO  → Endpoints from spec only (no source location — binding_method='unbound')
    NO  → Has SCIP index?
            YES → Extract endpoints from SCIP-verified controller annotations
            NO  → Extract endpoints from tree-sitter AST heuristic
```

### 6.2 OpenAPI Spec Discovery + Parsing

```typescript
// src/lib/endpoint_extraction/openapi/spec_discovery.ts

const SPEC_LOCATIONS = [
  'src/main/resources/openapi.json',
  'src/main/resources/openapi.yml',
  'src/main/resources/openapi.yaml',
  'src/main/resources/static/openapi.json',
  'src/main/resources/swagger.json',
  'src/main/resources/swagger.yml',
  'docs/openapi.json',
  'docs/openapi.yml',
  'docs/api-spec.yml',
  'openapi.json',
  'openapi.yml',
  'swagger.json',
  'target/openapi.json',
  'build/openapi.json',
];

export async function discoverOpenAPISpec(
  repoBasePath: string
): Promise<{ spec: OpenAPISpec; path: string } | null> {
  for (const specPath of SPEC_LOCATIONS) {
    try {
      const content = await fs.readFile(path.join(repoBasePath, specPath), 'utf-8');
      const spec = specPath.endsWith('.json') ? JSON.parse(content) : yaml.parse(content);
      if (spec.paths && Object.keys(spec.paths).length > 0) {
        return { spec, path: specPath };
      }
    } catch { continue; }
  }
  return null;
}
```

```typescript
// src/lib/endpoint_extraction/openapi/spec_parser.ts

export function parseOpenAPISpec(spec: OpenAPISpec, repoSlug: string): SpecEndpoint[] {
  const endpoints: SpecEndpoint[] = [];
  const isV3 = spec.openapi?.startsWith('3.');

  // Resolve base path
  const basePath = isV3
    ? (spec.servers?.[0]?.url?.replace(/^https?:\/\/[^/]+/, '') ?? '')
    : (spec.basePath ?? '');

  for (const [pathTemplate, pathItem] of Object.entries(spec.paths ?? {})) {
    const pathParams = (pathItem as any).parameters ?? [];

    for (const method of ['get', 'post', 'put', 'delete', 'patch']) {
      const operation = (pathItem as any)[method];
      if (!operation) continue;

      const fullPath = normalizePath(combinePaths(basePath, pathTemplate));

      // Parameters with full type info
      const allParams = [...pathParams, ...(operation.parameters ?? [])];
      const parameters = allParams.map((p: any) => ({
        name: p.name,
        location: p.in,
        type: resolveSchemaType(p.schema ?? p, spec),
        required: p.required ?? (p.in === 'path'),
      }));

      // Request body (OpenAPI 3.x)
      let requestType: string | undefined;
      let requestSchema: any;
      if (isV3 && operation.requestBody) {
        const jsonContent = operation.requestBody.content?.['application/json'];
        if (jsonContent?.schema) {
          requestType = getSchemaTypeName(jsonContent.schema, spec);
          requestSchema = resolveRef(jsonContent.schema, spec);
        }
      }
      // Swagger 2.0: body parameter
      if (!isV3) {
        const bodyParam = allParams.find((p: any) => p.in === 'body');
        if (bodyParam?.schema) {
          requestType = getSchemaTypeName(bodyParam.schema, spec);
          requestSchema = resolveRef(bodyParam.schema, spec);
        }
      }

      // Response type
      const successResp = operation.responses?.['200'] ?? operation.responses?.['201'];
      let responseType: string | undefined;
      let responseSchema: any;
      if (isV3 && successResp?.content?.['application/json']?.schema) {
        responseType = getSchemaTypeName(successResp.content['application/json'].schema, spec);
        responseSchema = resolveRef(successResp.content['application/json'].schema, spec);
      } else if (!isV3 && successResp?.schema) {
        responseType = getSchemaTypeName(successResp.schema, spec);
        responseSchema = resolveRef(successResp.schema, spec);
      }

      endpoints.push({
        httpMethod: method.toUpperCase(),
        fullPath,
        operationId: operation.operationId,
        summary: operation.summary,
        tags: operation.tags ?? [],
        deprecated: operation.deprecated ?? false,
        parameters,
        requestType, requestSchema,
        responseType, responseSchema,
        securitySchemes: (operation.security ?? spec.security ?? [])
          .flatMap((s: any) => Object.keys(s)),
      });
    }
  }

  return endpoints;
}
```

### 6.3 DTO Schema Extraction

Since OpenAPI specs contain full request/response schemas, extract and store them for contract validation:

```typescript
// src/lib/endpoint_extraction/openapi/dto_extractor.ts

export function extractDTOsFromSpec(
  spec: OpenAPISpec,
  repoSlug: string,
  specPath: string
): ExtractedDTO[] {
  const dtos: ExtractedDTO[] = [];
  const schemas = spec.components?.schemas ?? spec.definitions ?? {};

  for (const [name, schema] of Object.entries(schemas as Record<string, any>)) {
    if (!schema.properties) continue;

    const fields = Object.entries(schema.properties).map(([fieldName, fieldSchema]: [string, any]) => ({
      name: fieldName,
      type: resolveSchemaType(fieldSchema, spec),
      required: (schema.required ?? []).includes(fieldName),
      description: fieldSchema.description,
      format: fieldSchema.format,
    }));

    // Track which endpoints use this DTO
    const asRequest: string[] = [];
    const asResponse: string[] = [];
    for (const [pt, pi] of Object.entries(spec.paths ?? {})) {
      for (const [m, op] of Object.entries(pi as Record<string, any>)) {
        if (!['get','post','put','delete','patch'].includes(m)) continue;
        const opStr = `${m.toUpperCase()} ${pt}`;
        const reqRef = op.requestBody?.content?.['application/json']?.schema?.$ref
          ?? op.parameters?.find((p: any) => p.in === 'body')?.schema?.$ref;
        if (reqRef?.endsWith(`/${name}`)) asRequest.push(opStr);
        const resRef = (op.responses?.['200'] ?? op.responses?.['201'])
          ?.content?.['application/json']?.schema?.$ref
          ?? (op.responses?.['200'] ?? op.responses?.['201'])?.schema?.$ref;
        if (resRef?.endsWith(`/${name}`)) asResponse.push(opStr);
      }
    }

    dtos.push({ name, repoSlug, specPath, fields, usedBy: { asRequest, asResponse } });
  }

  return dtos;
}
```

### 6.4 SCIP Source Binding

With OpenAPI as the contract source, SCIP binds spec endpoints to Java source locations:

```typescript
// src/lib/endpoint_extraction/scip/source_binder.ts

const SPRING_MAPPING_SYMBOLS: Record<string, string> = {
  'org/springframework/web/bind/annotation/GetMapping#': 'GET',
  'org/springframework/web/bind/annotation/PostMapping#': 'POST',
  'org/springframework/web/bind/annotation/PutMapping#': 'PUT',
  'org/springframework/web/bind/annotation/DeleteMapping#': 'DELETE',
  'org/springframework/web/bind/annotation/PatchMapping#': 'PATCH',
  'org/springframework/web/bind/annotation/RequestMapping#': '*',
};

export async function bindSpecEndpointsToSource(
  specEndpoints: SpecEndpoint[],
  repoSlug: string,
  repoBasePath: string,
  parser: Parser,
  scipIndex: SCIPIndex | null,
  graphClient: IGraphClient
): Promise<SourceBinding[]> {
  // Build lookup of all controller methods in this repo
  const controllerMethods = scipIndex
    ? await scanControllerMethodsWithSCIP(repoBasePath, parser, scipIndex)
    : await scanControllerMethodsWithAST(repoBasePath, parser);

  const bindings: SourceBinding[] = [];

  for (const specEp of specEndpoints) {
    // Strategy 1: Match by operationId → method name
    if (specEp.operationId) {
      const parts = specEp.operationId.split('_');
      const targetMethod = parts.length === 2 ? parts[1] : parts[0];
      const targetClass = parts.length === 2 ? parts[0] : null;

      const match = controllerMethods.find(cm =>
        cm.methodName === targetMethod &&
        (!targetClass || cm.className === targetClass) &&
        cm.httpMethod === specEp.httpMethod
      );

      if (match) {
        const graphNode = await graphClient.findSymbol(
          `${match.className}.${match.methodName}`, repoSlug
        );
        bindings.push({
          ...specEp,
          filePath: match.filePath,
          className: match.className,
          methodName: match.methodName,
          startLine: match.startLine,
          endLine: match.endLine,
          graphNodeId: graphNode?.[0]?.id,
          bindingMethod: scipIndex ? 'scip_operationId' : 'ast_operationId',
        });
        continue;
      }
    }

    // Strategy 2: Match by path + httpMethod
    const pathMatch = controllerMethods.find(cm =>
      pathToPattern(cm.fullPath) === pathToPattern(specEp.fullPath) &&
      cm.httpMethod === specEp.httpMethod
    );

    if (pathMatch) {
      const graphNode = await graphClient.findSymbol(
        `${pathMatch.className}.${pathMatch.methodName}`, repoSlug
      );
      bindings.push({
        ...specEp,
        filePath: pathMatch.filePath,
        className: pathMatch.className,
        methodName: pathMatch.methodName,
        startLine: pathMatch.startLine,
        endLine: pathMatch.endLine,
        graphNodeId: graphNode?.[0]?.id,
        bindingMethod: scipIndex ? 'scip_annotation' : 'ast_annotation',
      });
      continue;
    }

    // Strategy 3: Unbound — spec endpoint, no matching source
    bindings.push({
      ...specEp,
      filePath: undefined,
      className: specEp.tags[0] ?? 'unknown',
      methodName: specEp.operationId ?? `${specEp.httpMethod}_${specEp.fullPath}`,
      startLine: 0, endLine: 0,
      graphNodeId: undefined,
      bindingMethod: 'unbound',
    });
  }

  return bindings;
}
```

### 6.5 SCIP Controller Scanner

```typescript
// src/lib/endpoint_extraction/scip/controller_scanner.ts

const SPRING_CONTROLLER_SYMBOLS = new Set([
  'org/springframework/web/bind/annotation/RestController#',
  'org/springframework/stereotype/Controller#',
]);

async function scanControllerMethodsWithSCIP(
  repoBasePath: string,
  parser: Parser,
  scipIndex: SCIPIndex
): Promise<ControllerMethod[]> {
  const methods: ControllerMethod[] = [];

  const javaFiles = await findFiles(repoBasePath, ['.java', '.kt'], {
    excludePatterns: [/\/test\//],
  });

  for (const file of javaFiles) {
    const content = await fs.readFile(path.join(repoBasePath, file), 'utf-8');
    if (!/@(Rest)?Controller/.test(content)) continue;

    const tree = parser.parse(content);
    const scipFileData = scipIndex.getFileData(file);

    const classNodes = tree.rootNode.descendantsOfType('class_declaration');
    for (const classNode of classNodes) {
      // SCIP-verify this is a Spring controller
      const isController = getAnnotationNodes(classNode).some(ann => {
        const sym = scipFileData?.getSymbolAtPosition(ann.startPosition.row, ann.startPosition.column);
        return sym && SPRING_CONTROLLER_SYMBOLS.has(sym);
      });
      if (!isController) continue;

      const className = getClassName(classNode);
      const basePath = extractClassLevelPath(classNode);

      for (const method of classNode.descendantsOfType('method_declaration')) {
        for (const ann of getAnnotationNodes(method)) {
          const sym = scipFileData?.getSymbolAtPosition(ann.startPosition.row, ann.startPosition.column);
          if (!sym) continue;
          const httpMethod = SPRING_MAPPING_SYMBOLS[sym];
          if (!httpMethod) continue;

          const methodPath = extractAnnotationPath(ann);
          const resolved = httpMethod === '*' ? extractRequestMappingMethod(ann) ?? 'GET' : httpMethod;

          methods.push({
            className, methodName: getMethodName(method),
            filePath: file,
            startLine: method.startPosition.row + 1,
            endLine: method.endPosition.row + 1,
            httpMethod: resolved,
            basePath, methodPath,
            fullPath: normalizePath(combinePaths(basePath, methodPath)),
          });
        }
      }
    }
  }

  return methods;
}
```

-----

## 7. Phase B: Consumer Extraction

### 7.1 MFE API Call Extraction

```typescript
// src/lib/endpoint_extraction/mfe/mfe_extractor.ts

export async function extractMFEAPICalls(
  repoSlug: string,
  repoBasePath: string,
  parser: Parser,
  scipIndex: SCIPIndex | null
): Promise<ExtractedAPICall[]> {
  const allCalls: ExtractedAPICall[] = [];

  // Pre-resolve API client base paths
  const clientBases = await resolveAPIClientBasePaths(repoBasePath, parser, scipIndex);

  const tsFiles = await findFiles(repoBasePath, ['.ts', '.tsx', '.js', '.jsx'], {
    excludePatterns: [/node_modules/, /dist/, /build/, /\.next/, /\.(test|spec|stories|d)\./, /__mocks__/],
  });

  for (const file of tsFiles) {
    const content = await fs.readFile(path.join(repoBasePath, file), 'utf-8');
    if (!/\b(fetch|axios|api|http|client|query|mutation)\b/i.test(content)) continue;

    const tree = parser.parse(content);
    const scipFileData = scipIndex?.getFileData(file) ?? null;

    allCalls.push(
      ...extractAxiosCalls(tree.rootNode, scipFileData, repoSlug, file, clientBases),
      ...extractFetchCalls(tree.rootNode, scipFileData, repoSlug, file),
      ...extractRTKQueryEndpoints(tree.rootNode, scipFileData, repoSlug, file),
    );
  }

  return allCalls;
}
```

Axios extraction with SCIP type verification:

```typescript
function extractAxiosCalls(
  root: Parser.SyntaxNode,
  scipFileData: SCIPFileData | null,
  repoSlug: string,
  filePath: string,
  clientBases: Map<string, ClientBaseInfo>
): ExtractedAPICall[] {
  const calls: ExtractedAPICall[] = [];

  for (const call of root.descendantsOfType('call_expression')) {
    const callee = call.child(0);
    if (!callee || callee.type !== 'member_expression') continue;

    const object = callee.child(0);
    const method = callee.child(2);
    if (!object || !method) continue;

    const httpMethod = method.text?.toUpperCase();
    if (!httpMethod || !['GET','POST','PUT','DELETE','PATCH'].includes(httpMethod)) continue;

    // SCIP: verify object is an HTTP client
    let isHTTPClient = false;
    let confidence = 'heuristic';

    if (scipFileData) {
      const objectType = scipFileData.getTypeAtPosition(
        object.startPosition.row, object.startPosition.column
      );
      if (objectType) {
        const httpTypes = new Set(['AxiosInstance', 'AxiosStatic', 'Axios']);
        if (httpTypes.has(objectType.typeName)) {
          isHTTPClient = true;
          confidence = 'scip_verified';
        } else {
          continue; // Confirmed non-HTTP client — skip
        }
      } else {
        isHTTPClient = isLikelyHTTPClient(object.text);
      }
    } else {
      isHTTPClient = isLikelyHTTPClient(object.text);
    }

    if (!isHTTPClient) continue;

    // Extract URL
    const args = call.descendantsOfType('arguments')[0];
    const urlArg = args?.child(1);
    const rawPath = extractURLFromExpression(urlArg);
    if (!rawPath) continue;

    // Resolve client base URL
    const clientBase = clientBases.get(object.text);
    const fullRawPath = clientBase?.baseURL
      ? combinePaths(clientBase.baseURL, rawPath)
      : rawPath;

    calls.push({
      repoSlug, filePath,
      functionName: findEnclosingFunction(call),
      httpMethod,
      rawPath: fullRawPath,
      normalizedPath: normalizeAPIPath(fullRawPath),
      hasErrorHandling: isInsideTryCatch(call) || hasCatchChain(call),
      callPattern: 'axios',
      targetRepoHint: clientBase?.targetRepo ?? null,
      startLine: call.startPosition.row + 1,
      endLine: call.endPosition.row + 1,
      extractionSource: confidence === 'scip_verified' ? 'scip' : 'ast_heuristic',
      confidence,
    });
  }

  return calls;
}
```

API client base path resolution:

```typescript
// src/lib/endpoint_extraction/mfe/client_resolver.ts

export async function resolveAPIClientBasePaths(
  repoBasePath: string,
  parser: Parser,
  scipIndex: SCIPIndex | null
): Promise<Map<string, ClientBaseInfo>> {
  const clients = new Map<string, ClientBaseInfo>();

  const candidateFiles = await findFiles(repoBasePath, ['.ts', '.js'], {
    includePatterns: [/api/, /client/, /service/, /http/],
    excludePatterns: [/node_modules/, /test/, /dist/],
  });

  for (const file of candidateFiles) {
    const content = await fs.readFile(path.join(repoBasePath, file), 'utf-8');
    if (!content.includes('axios.create') && !content.includes('createApi')) continue;

    const tree = parser.parse(content);

    // Find axios.create({ baseURL: ... })
    for (const createCall of tree.rootNode.descendantsOfType('call_expression')) {
      const callee = createCall.child(0);
      if (callee?.type !== 'member_expression' || callee.child(2)?.text !== 'create') continue;

      const config = createCall.descendantsOfType('object')[0];
      if (!config) continue;

      const baseURLProp = findObjectProperty(config, 'baseURL');
      if (!baseURLProp) continue;

      let baseURL: string | null = extractStringValue(baseURLProp);
      let targetRepo: string | null = null;

      // Check for env var: process.env.REACT_APP_BONDS_API_URL
      if (!baseURL) {
        const envMatch = baseURLProp.text?.match(/(?:process\.env|import\.meta\.env)\.(\w+)/);
        if (envMatch) {
          targetRepo = inferRepoFromEnvVarName(envMatch[1]);
          baseURL = await resolveEnvVarFromDotEnv(repoBasePath, envMatch[1]);
        }
      }

      const exportedName = findExportedVariableName(createCall);
      if (exportedName) {
        clients.set(exportedName, { clientName: exportedName, baseURL, targetRepo, definitionFile: file });
      }
    }
  }

  return clients;
}

function inferRepoFromEnvVarName(envVarName: string): string | null {
  // REACT_APP_BONDS_API_URL → ms-bonds-trading
  // VITE_AUTH_SERVICE_URL → ms-auth
  const match = envVarName.match(
    /(?:REACT_APP_|VITE_|NEXT_PUBLIC_)?(\w+?)(?:_TRADING)?(?:_API|_SERVICE)?_(?:URL|BASE|HOST)/i
  );
  return match ? `ms-${match[1].toLowerCase().replace(/_/g, '-')}` : null;
}
```

### 7.2 MS-to-MS Call Extraction

Three Java HTTP client patterns, all SCIP-verified when available:

```typescript
// src/lib/endpoint_extraction/ms_to_ms/ms_consumer_extractor.ts

export async function extractMStoMSCalls(
  repoSlug: string,
  repoBasePath: string,
  parser: Parser,
  scipIndex: SCIPIndex | null,
  config: ProjectConfig
): Promise<ExtractedServiceCall[]> {
  const allCalls: ExtractedServiceCall[] = [];

  const javaFiles = await findFiles(repoBasePath, ['.java', '.kt'], {
    excludePatterns: [/\/test\//],
  });

  for (const file of javaFiles) {
    const content = await fs.readFile(path.join(repoBasePath, file), 'utf-8');
    if (!/FeignClient|RestTemplate|WebClient/.test(content)) continue;

    const tree = parser.parse(content);
    const scipFileData = scipIndex?.getFileData(file) ?? null;

    // ── Feign Clients ──
    if (content.includes('FeignClient')) {
      allCalls.push(...extractFeignCalls(tree.rootNode, scipFileData, repoSlug, file));
    }

    // ── RestTemplate ──
    if (/RestTemplate|restTemplate/.test(content)) {
      allCalls.push(...extractRestTemplateCalls(tree.rootNode, scipFileData, repoSlug, file, config));
    }

    // ── WebClient ──
    if (/WebClient|webClient/.test(content)) {
      allCalls.push(...extractWebClientCalls(tree.rootNode, scipFileData, repoSlug, file, config));
    }
  }

  return allCalls;
}
```

**Feign** — best signal because `@FeignClient(name = "ms-auth")` gives the target directly:

```typescript
function extractFeignCalls(
  root: Parser.SyntaxNode,
  scipFileData: SCIPFileData | null,
  repoSlug: string,
  filePath: string
): ExtractedServiceCall[] {
  const calls: ExtractedServiceCall[] = [];

  for (const iface of root.descendantsOfType('interface_declaration')) {
    const feignAnn = getAnnotationNodes(iface).find(ann => {
      if (scipFileData) {
        const sym = scipFileData.getSymbolAtPosition(ann.startPosition.row, ann.startPosition.column);
        return sym === 'org/springframework/cloud/openfeign/FeignClient#';
      }
      return ann.text?.includes('FeignClient');
    });
    if (!feignAnn) continue;

    const targetService = extractAnnotationParam(feignAnn, 'name')
      ?? extractAnnotationParam(feignAnn, 'value');
    const basePath = extractAnnotationParam(feignAnn, 'path') ?? '';

    for (const method of iface.descendantsOfType('method_declaration')) {
      const mapping = findMappingAnnotation(method, scipFileData);
      if (!mapping) continue;

      calls.push({
        repoSlug, filePath,
        methodName: getMethodName(method),
        httpMethod: mapping.httpMethod,
        rawPath: normalizePath(combinePaths(basePath, mapping.path)),
        normalizedPath: normalizePath(combinePaths(basePath, mapping.path)),
        callPattern: 'feign_client',
        targetServiceHint: targetService ?? undefined,
        targetResolutionMethod: 'feign_name',
        hasErrorHandling: true,
        startLine: method.startPosition.row + 1,
        endLine: method.endPosition.row + 1,
        extractionSource: scipFileData ? 'scip' : 'ast_heuristic',
        confidence: scipFileData ? 'scip_verified' : 'heuristic',
      });
    }
  }

  return calls;
}
```

**RestTemplate** — SCIP verifies the object type, `@Value` resolves the URL from `application.yml`:

```typescript
function extractRestTemplateCalls(
  root: Parser.SyntaxNode,
  scipFileData: SCIPFileData | null,
  repoSlug: string,
  filePath: string,
  config: ProjectConfig
): ExtractedServiceCall[] {
  const calls: ExtractedServiceCall[] = [];
  const REST_METHODS: Record<string, string> = {
    'getForObject': 'GET', 'getForEntity': 'GET',
    'postForObject': 'POST', 'postForEntity': 'POST',
    'put': 'PUT', 'delete': 'DELETE', 'patchForObject': 'PATCH',
    'exchange': '*',
  };

  for (const call of root.descendantsOfType('method_invocation')) {
    const objectNode = call.child(0);
    const methodNode = call.child(2);
    if (!objectNode || !methodNode) continue;
    if (!REST_METHODS[methodNode.text ?? '']) continue;

    // SCIP type verification
    if (scipFileData) {
      const objectType = scipFileData.getTypeAtPosition(
        objectNode.startPosition.row, objectNode.startPosition.column
      );
      if (objectType && objectType.qualifiedType !== 'org/springframework/web/client/RestTemplate#') {
        continue; // Not a RestTemplate — skip
      }
      if (!objectType && !objectNode.text?.toLowerCase().includes('resttemplate')) {
        continue; // Can't verify and name doesn't match — skip
      }
    } else {
      if (!objectNode.text?.toLowerCase().includes('resttemplate')) continue;
    }

    // Extract URL
    const args = call.descendantsOfType('argument_list')[0];
    const urlArg = args?.child(1);
    let rawPath = extractStringOrConcatValue(urlArg);
    let targetHint: string | null = null;

    // Resolve @Value injected URL variables via SCIP
    if (!rawPath && urlArg?.type === 'identifier' && scipFileData) {
      const def = scipFileData.getDefinition(urlArg.startPosition.row, urlArg.startPosition.column);
      if (def) {
        const valueAnno = scipFileData.getAnnotationOnSymbol(
          def.symbol, 'org/springframework/beans/factory/annotation/Value#'
        );
        if (valueAnno?.value) {
          const propMatch = valueAnno.value.match(/\$\{([^}]+)\}/);
          if (propMatch) {
            const resolved = resolveYamlProperty(config.applicationYml, propMatch[1]);
            if (resolved) {
              rawPath = valueAnno.value.replace(`\${${propMatch[1]}}`, resolved);
              targetHint = extractServiceFromURL(resolved);
            }
          }
        }
      }
    }

    if (!rawPath) continue;
    if (!targetHint) targetHint = extractServiceFromURL(rawPath);

    let httpMethod = REST_METHODS[methodNode.text!];
    if (httpMethod === '*') httpMethod = extractHttpMethodEnum(args) ?? 'GET';

    calls.push({
      repoSlug, filePath,
      methodName: findEnclosingMethodName(call),
      httpMethod, rawPath,
      normalizedPath: normalizeServiceCallPath(rawPath),
      callPattern: 'rest_template',
      targetServiceHint: targetHint ?? undefined,
      targetResolutionMethod: targetHint ? 'value_annotation' : 'unknown',
      hasErrorHandling: isInsideTryCatch(call),
      startLine: call.startPosition.row + 1,
      endLine: call.endPosition.row + 1,
      extractionSource: scipFileData ? 'scip' : 'ast_heuristic',
      confidence: scipFileData ? 'scip_verified' : 'heuristic',
    });
  }

  return calls;
}
```

**WebClient** — follows the reactive chain to find `.uri()` argument, SCIP verifies chain root type.

-----

## 8. Phase C: Proxy Layer Resolution

For MFE repos where the React component calls its own server (not the MS directly), resolve the proxy layer.

### 8.1 Five Proxy Layers

```typescript
// src/lib/endpoint_extraction/proxy/proxy_resolver.ts

export async function resolveProxyLayers(
  repoSlug: string,
  repoBasePath: string,
  parser: Parser,
  scipIndex: SCIPIndex | null,
  qwenClient: QwenClient,
  knownRepos: string[],
  pool: Pool
): Promise<ProxyResolution> {
  const proxyRoutes: ProxyRoute[] = [];
  const codeProxies: ExtractedProxy[] = [];

  // ── Layer 1: Next.js rewrites (transparent proxy, highest confidence) ──
  const nextRewrites = await extractNextJSRewrites(repoSlug, repoBasePath);
  for (const rw of nextRewrites) {
    proxyRoutes.push({
      inboundPrefix: rw.stripPrefix,
      targetRepo: rw.targetRepo,
      proxyType: 'nextjs_rewrite', confidence: 'high',
    });
  }

  // ── Layer 2: Next.js API routes (code proxy — extract outbound calls) ──
  const nextProxies = await extractNextJSProxies(repoSlug, repoBasePath, parser, scipIndex);
  codeProxies.push(...nextProxies);

  // ── Layer 3: Express BFF (http-proxy-middleware or manual proxy) ──
  const expressProxies = await extractExpressProxies(repoSlug, repoBasePath, parser, scipIndex);
  for (const ep of expressProxies) {
    if (ep.proxyType === 'http_proxy_middleware') {
      proxyRoutes.push({
        inboundPrefix: ep.inboundPath,
        targetRepo: ep.targetRepo!,
        proxyType: 'http_proxy_middleware', confidence: 'high',
      });
    } else {
      codeProxies.push(ep); // Manual proxy — extract outbound calls
    }
  }

  // ── Layer 4: Infrastructure configs — LLM-parsed (httpd/nginx/k8s) ──
  const infraRoutes = await extractInfraProxyRoutesViaLLM(
    repoSlug, repoBasePath, qwenClient, knownRepos, pool
  );
  for (const ir of infraRoutes) {
    if (!proxyRoutes.some(r => r.inboundPrefix === ir.inboundPrefix) && ir.targetRepo) {
      proxyRoutes.push({
        inboundPrefix: ir.inboundPrefix,
        targetRepo: ir.targetRepo,
        proxyType: ir.configType, confidence: 'llm_extracted',
      });
    }
  }

  // ── Layer 5: Vite/Webpack dev server proxy (supplementary) ──
  const devProxies = await extractDevServerProxyConfig(repoSlug, repoBasePath, parser);
  for (const dp of devProxies) {
    if (!proxyRoutes.some(r => r.inboundPrefix === dp.prefix) && dp.targetRepo) {
      proxyRoutes.push({
        inboundPrefix: dp.prefix,
        targetRepo: dp.targetRepo,
        proxyType: 'dev_proxy', confidence: 'low',
      });
    }
  }

  // ── Extract outbound calls from code proxies ──
  const proxyOutboundCalls: ExtractedAPICall[] = [];
  for (const proxy of codeProxies) {
    for (const call of proxy.outboundCalls) {
      call.viaProxy = proxy.proxyFilePath;
      call.proxyInboundPath = proxy.inboundPath;
      proxyOutboundCalls.push(call);
    }
  }

  return { proxyRoutes, codeProxies, proxyOutboundCalls };
}
```

### 8.2 LLM Infrastructure Config Parsing

For httpd, nginx, HAProxy, Kubernetes Ingress — deterministic parsers break on real-world config variations. Qwen handles them all:

```typescript
// src/lib/endpoint_extraction/proxy/llm_infra_parser.ts

async function extractInfraProxyRoutesViaLLM(
  repoSlug: string,
  repoBasePath: string,
  qwenClient: QwenClient,
  knownRepos: string[],
  pool: Pool
): Promise<InfraProxyRoute[]> {
  const infraFiles = await findInfraConfigFiles(repoBasePath);
  if (infraFiles.length === 0) return [];

  const allRoutes: InfraProxyRoute[] = [];

  for (const file of infraFiles) {
    // Content hash cache — skip if unchanged
    const contentHash = createHash('sha256').update(file.content).digest('hex');
    const cached = await pool.query(
      `SELECT routes FROM infra_proxy_cache WHERE config_hash = $1 AND repo_slug = $2`,
      [contentHash, repoSlug]
    );
    if (cached.rows[0]) { allRoutes.push(...cached.rows[0].routes); continue; }

    // LLM extraction with structured output prompt
    const prompt = buildInfraProxyPrompt(file.content, file.path, knownRepos);
    const response = await qwenClient.generate(prompt, { maxTokens: 2000, temperature: 0.1 });
    const parsed = parseAndValidateRoutes(response, knownRepos);

    // Regex sanity check
    const sanity = regexSanityCheck(file.content, detectConfigType(file.path));
    if (parsed.length > sanity.regexRouteCount * 3 && sanity.regexRouteCount > 0) {
      console.warn(`[InfraProxy] Possible hallucination: ${parsed.length} LLM vs ${sanity.regexRouteCount} regex`);
    }

    // Cache
    await pool.query(
      `INSERT INTO infra_proxy_cache (repo_slug, config_hash, file_path, routes, created_at)
       VALUES ($1, $2, $3, $4, NOW())
       ON CONFLICT (repo_slug, config_hash) DO UPDATE SET routes = $4, created_at = NOW()`,
      [repoSlug, contentHash, file.path, JSON.stringify(parsed)]
    );

    allRoutes.push(...parsed);
  }

  return allRoutes;
}
```

-----

## 9. Phase D: Cross-Repo Matching + Graph Integration

After all repos are processed, match consumers to providers.

### 9.1 Orchestration

```typescript
// src/lib/endpoint_extraction/pipeline.ts

export async function runAPIDependencyPipeline(
  pool: Pool,
  graphClient: IGraphClient,
  parser: Parser,
  qwenClient: QwenClient,
  repos: RepoInfo[]
): Promise<PipelineResult> {
  const msRepos = repos.filter(r => r.projectType === 'backend' || r.projectType === 'fullstack');
  const mfeRepos = repos.filter(r => r.projectType === 'frontend');
  const knownRepoSlugs = repos.map(r => r.slug);
  const limit = pLimit(parseInt(process.env.REPO_CONCURRENCY || '4', 10));

  // ═══ PHASE A: MS Endpoint Extraction (parallel) ═══
  const allEndpoints: ExtractedEndpoint[] = [];

  await Promise.all(msRepos.map(repo => limit(async () => {
    const repoBasePath = path.join(REPOS_DIR, repo.slug);
    const scipIndex = await loadSCIPIndex(repoBasePath);
    const config = await loadProjectConfig(pool, repo.slug, repoBasePath);

    // OpenAPI spec (primary)
    const specResult = await discoverOpenAPISpec(repoBasePath);
    let endpoints: ExtractedEndpoint[];

    if (specResult) {
      const specEndpoints = parseOpenAPISpec(specResult.spec, repo.slug);
      const bindings = await bindSpecEndpointsToSource(
        specEndpoints, repo.slug, repoBasePath, parser, scipIndex, graphClient
      );
      endpoints = mergeSpecWithBindings(specEndpoints, bindings, repo.slug, specResult.path);

      const dtos = extractDTOsFromSpec(specResult.spec, repo.slug, specResult.path);
      await storeDTOs(pool, dtos);

      console.log(`[API] ${repo.slug}: ${endpoints.length} endpoints from spec ` +
        `(${bindings.filter(b => b.bindingMethod !== 'unbound').length} source-bound)`);
    } else {
      endpoints = await extractEndpointsWithoutSpec(repo.slug, repoBasePath, parser, scipIndex);
      console.log(`[API] ${repo.slug}: ${endpoints.length} endpoints (${scipIndex ? 'SCIP' : 'AST'}, no spec)`);
    }

    await pool.query(
      `UPDATE projects SET has_openapi_spec = $1, api_endpoint_count = $2 WHERE repo_slug = $3`,
      [!!specResult, endpoints.length, repo.slug]
    );

    allEndpoints.push(...endpoints);
  })));

  // ═══ PHASE B: Consumer Extraction (parallel) ═══
  const allCalls: (ExtractedAPICall | ExtractedServiceCall)[] = [];
  const allProxyRoutes: ProxyRoute[] = [];

  // MS-to-MS
  await Promise.all(msRepos.map(repo => limit(async () => {
    const repoBasePath = path.join(REPOS_DIR, repo.slug);
    const scipIndex = await loadSCIPIndex(repoBasePath);
    const config = await loadProjectConfig(pool, repo.slug, repoBasePath);
    const calls = await extractMStoMSCalls(repo.slug, repoBasePath, parser, scipIndex, config);
    allCalls.push(...calls);
    console.log(`[API] ${repo.slug}: ${calls.length} outbound MS-to-MS calls`);
  })));

  // MFE + proxy
  await Promise.all(mfeRepos.map(repo => limit(async () => {
    const repoBasePath = path.join(REPOS_DIR, repo.slug);
    const scipIndex = await loadSCIPIndex(repoBasePath);

    // Phase C: proxy resolution
    const proxyRes = await resolveProxyLayers(
      repo.slug, repoBasePath, parser, scipIndex, qwenClient, knownRepoSlugs, pool
    );
    for (const route of proxyRes.proxyRoutes) {
      allProxyRoutes.push({ ...route, applicableToMFE: repo.slug });
    }
    allCalls.push(...proxyRes.proxyOutboundCalls);

    // MFE API calls
    const apiCalls = await extractMFEAPICalls(repo.slug, repoBasePath, parser, scipIndex);
    allCalls.push(...apiCalls);

    console.log(`[API] ${repo.slug}: ${apiCalls.length} API calls, ${proxyRes.proxyRoutes.length} proxy routes`);
  })));

  // ═══ PHASE D: Cross-Repo Matching (sequential) ═══

  // Load gateway routes from projects.gateway_prefix
  const gatewayRoutes = await loadGatewayRoutes(pool);
  console.log(`[API] Loaded ${gatewayRoutes.length} gateway routes from projects table`);

  // Build endpoint index
  const endpointIndex = new EndpointIndex(allEndpoints);

  // Match
  const matches: EndpointMatch[] = [];
  const unmatchedCalls: (ExtractedAPICall | ExtractedServiceCall)[] = [];

  for (const call of allCalls) {
    if (isExternalAPI(call.normalizedPath)) continue;
    const match = matchConsumerToProvider(
      call, endpointIndex, gatewayRoutes, allProxyRoutes, new Map()
    );
    if (match) matches.push(match);
    else unmatchedCalls.push(call);
  }

  // Orphan endpoints
  const matchedEpKeys = new Set(matches.map(m =>
    `${m.msEndpoint.httpMethod}|${m.msEndpoint.fullPath}|${m.msEndpoint.repoSlug}`
  ));
  const orphanEndpoints = allEndpoints.filter(e =>
    !matchedEpKeys.has(`${e.httpMethod}|${e.fullPath}|${e.repoSlug}`) &&
    !isInternalEndpoint(e)
  );

  // Store
  await storeEndpoints(pool, allEndpoints);
  await storeAPICalls(pool, allCalls, matches);
  await storeProxyRoutes(pool, allProxyRoutes);

  // Graph edges
  await createAPIGraphEdges(graphClient, pool, matches, allEndpoints, allCalls);

  // Classify services
  const classifications = await classifyServices(pool);
  for (const [repo, cls] of classifications.entries()) {
    await pool.query(
      `UPDATE projects SET service_role = $1, api_consumer_count = $2 WHERE repo_slug = $3`,
      [cls.role, cls.consumerCount, repo]
    );
  }

  // Materialized view
  await pool.query('REFRESH MATERIALIZED VIEW CONCURRENTLY mfe_ms_dependencies');

  const result: PipelineResult = {
    endpoints: allEndpoints.length,
    calls: allCalls.length,
    matches: matches.length,
    orphanEndpoints: orphanEndpoints.length,
    unmatchedCalls: unmatchedCalls.length,
    proxyRoutes: allProxyRoutes.length,
    platformServices: [...classifications.entries()]
      .filter(([,c]) => c.role === 'platform')
      .map(([r,c]) => ({ repo: r, consumers: c.consumerCount })),
  };

  console.log(`[API] Pipeline complete:`, JSON.stringify(result, null, 2));
  return result;
}
```

-----

## 10. Endpoint Matching Engine

### 10.1 Disambiguation Chain

Eight strategies tried in order. `gateway_prefix` from the `projects` table is the primary disambiguator.

```typescript
// src/lib/endpoint_extraction/matching/matcher.ts

export function matchConsumerToProvider(
  call: ExtractedAPICall | ExtractedServiceCall,
  index: EndpointIndex,
  gatewayRoutes: GatewayRoute[],
  mfeProxyRoutes: ProxyRoute[],
  classifications: Map<string, ServiceClassification>
): EndpointMatch | null {
  const callPath = call.normalizedPath;
  const callMethod = call.httpMethod;

  // ── 1. Feign name → exact repo (MS-to-MS only) ──
  if ('callPattern' in call && call.callPattern === 'feign_client' && call.targetServiceHint) {
    const match = findByRepoAndPattern(index, callPath, callMethod, call.targetServiceHint);
    if (match) return makeMatch(call, match, 'exact', 'feign_name');
  }

  // ── 2. gateway_prefix from projects table → exact repo ──
  // This is the PRIMARY disambiguator. MFE calls /api/bonds/orders,
  // projects.gateway_prefix='/api/bonds' → ms-bonds-trading, strip → /orders
  for (const route of gatewayRoutes) {
    if (!callPath.startsWith(route.prefix)) continue;
    const stripped = callPath.replace(route.prefix, '') || '/';
    const match = findByRepoAndPattern(index, stripped, callMethod, route.targetRepo);
    if (match) return makeMatch(call, match, 'exact', 'gateway_prefix');
  }

  // ── 3. MFE proxy route (Next.js rewrite, http-proxy-middleware) ──
  for (const route of mfeProxyRoutes) {
    if (route.applicableToMFE && route.applicableToMFE !== call.repoSlug) continue;
    if (!callPath.startsWith(route.inboundPrefix)) continue;
    const stripped = callPath.replace(route.inboundPrefix, '') || '/';
    const match = findByRepoAndPattern(index, stripped, callMethod, route.targetRepo);
    if (match) {
      const confidence = route.confidence === 'high' ? 'exact' : 'high';
      return makeMatch(call, match, confidence, `mfe_proxy_${route.proxyType}`);
    }
  }

  // ── 4. API client binding (SCIP-resolved baseURL → target repo) ──
  if (call.targetRepoHint) {
    const match = findByRepoAndPattern(index, callPath, callMethod, call.targetRepoHint);
    if (match) return makeMatch(call, match, 'high', 'client_binding');
  }

  // ── 5. Repo naming convention (domain services only) ──
  const pairedRepo = inferPairedMSRepo(call.repoSlug);
  if (pairedRepo) {
    const cls = classifications.get(pairedRepo);
    if (!cls || cls.role === 'domain') {
      const match = findByRepoAndPattern(index, callPath, callMethod, pairedRepo);
      if (match) return makeMatch(call, match, 'medium', 'repo_pairing');
    }
  }

  // ── 6. Unique path match (only one MS has this path+method) ──
  const candidates = index.getByPatternAndMethod(pathToPattern(callPath), callMethod);
  if (candidates.length === 1) {
    return makeMatch(call, candidates[0], 'high', 'unique_path');
  }

  // ── 7. Platform service preference ──
  if (candidates.length > 1) {
    const platforms = candidates.filter(e => classifications.get(e.repoSlug)?.role === 'platform');
    if (platforms.length === 1) {
      return makeMatch(call, platforms[0], 'medium', 'platform_preference');
    }
  }

  // ── 8. Ambiguous — multiple MSes, no resolution ──
  if (candidates.length > 1) {
    return makeMatch(call, candidates[0], 'ambiguous', 'ambiguous');
  }

  return null;
}
```

-----

## 11. Shared / Platform Service Classification

```typescript
// src/lib/endpoint_extraction/classification/service_classifier.ts

export async function classifyServices(pool: Pool): Promise<Map<string, ServiceClassification>> {
  const classifications = new Map<string, ServiceClassification>();

  const consumers = await pool.query(`
    SELECT er.repo_slug AS ms_repo, ac.repo_slug AS consumer_repo, p.project_type AS consumer_type
    FROM endpoint_registry er
    JOIN api_calls ac ON ac.matched_endpoint_id = er.id
    JOIN projects p ON p.repo_slug = ac.repo_slug
    WHERE er.is_active = true AND ac.is_active = true
    GROUP BY er.repo_slug, ac.repo_slug, p.project_type
  `);

  const byMS = new Map<string, { mfe: Set<string>; ms: Set<string> }>();
  for (const row of consumers.rows) {
    if (!byMS.has(row.ms_repo)) byMS.set(row.ms_repo, { mfe: new Set(), ms: new Set() });
    const entry = byMS.get(row.ms_repo)!;
    if (row.consumer_type === 'frontend') entry.mfe.add(row.consumer_repo);
    else entry.ms.add(row.consumer_repo);
  }

  for (const [msRepo, data] of byMS.entries()) {
    const total = data.mfe.size + data.ms.size;
    let role: ServiceRole;

    if (total >= 4 || data.ms.size >= 2) role = 'platform';
    else if (total <= 2 && data.mfe.size === 1 && data.ms.size === 0) role = 'domain';
    else role = 'utility';

    classifications.set(msRepo, {
      repoSlug: msRepo, role, consumerCount: total,
      mfeConsumers: [...data.mfe], msConsumers: [...data.ms],
    });
  }

  return classifications;
}
```

-----

## 12. Graph Schema

### 12.1 New Node Types

```
api_endpoint  — MS controller endpoint (from OpenAPI spec + SCIP binding)
api_call      — MFE or MS HTTP call site
```

### 12.2 New Edge Types

```
consumes_api  — Consumer api_call → Provider api_endpoint   (cross-repo HTTP dep)
exposes_api   — MS function node → api_endpoint              (same repo)
calls_api     — Function → api_call                          (same repo call site)
proxies_to    — proxy_route → api_call                       (code proxy two-hop)
```

### 12.3 Full-Flow Traversal

```
Trace: POST /api/bonds/orders

Upstream:
  mfe-bonds-trading/OrderForm.tsx:useCreateOrder
    ──calls_api──→ api_call: POST /api/bonds/orders
    ──(gateway_prefix: /api/bonds → ms-bonds-trading, strip)──
    ──consumes_api──→ api_endpoint: POST /orders (ms-bonds-trading)

Downstream:
  api_endpoint: POST /orders
    ──exposes_api──→ OrderController.createOrder
    ──calls──→ OrderService.create
    ──calls──→ OrderRepository.save

Cross-MS:
  OrderService.create
    ──calls_api──→ api_call: Feign POST /validate → ms-auth
    ──consumes_api──→ api_endpoint: POST /validate (ms-auth)
```

-----

## 13. Bootstrap Pipeline Integration

### 13.1 Phase Ordering

```
Phase 2b:       ProjectAnalyzer + gateway_prefix auto-population  (UPDATED)
Phase 2g:       Knowledge Graph (existing)
Phase 2g-scip:  SCIP indexing (existing)
Phase 2g-api:   API Dependency Mapping (NEW — this plan)
Phase 2h:       Contextual Headers (existing — can now include API context)
```

### 13.2 Incremental Update

```typescript
function needsEndpointReExtraction(changedFiles: string[], projectType: string): boolean {
  if (projectType === 'backend') {
    return changedFiles.some(f =>
      f.endsWith('Controller.java') || f.endsWith('Controller.kt') ||
      f.includes('FeignClient') || f.includes('Client.java') ||
      f.endsWith('application.yml') || f.endsWith('application.yaml') ||
      f.endsWith('openapi.json') || f.endsWith('swagger.yml')
    );
  }
  if (projectType === 'frontend') {
    return changedFiles.some(f =>
      /\.(ts|tsx)$/.test(f) && /(api|service|hook|client|query)/i.test(f)
    );
  }
  return false;
}

// If ANY repo's endpoints changed, re-run matching for ALL repos
// (new endpoint in ms-auth affects all consumers)
```

-----

## 14. MCP Tool Enhancements

### 14.1 New Tools

```
ome_get_api_dependencies
  → Given a repo, returns which endpoints it consumes or exposes
  → Parameters: repo (string), direction ("consumers" | "providers")

ome_trace_api_flow
  → Full chain: MFE caller → proxy → gateway → controller → service → repository
  → Parameters: endpoint_path (string), http_method (string)

ome_get_api_surface
  → Complete API surface of an MS with consumer counts + deprecated flags
  → Parameters: repo (string)
```

### 14.2 Enhanced Existing Tools

```
ome_analyze_blast_radius
  → Now includes cross_repo_api_impact section
  → Shows affected MFEs and MSes when a controller method changes

ome_get_code_context
  → Includes API endpoint/call info for the symbol

ome_suggest_related_files
  → Includes cross-repo API-connected files via consumes_api edges
```

-----

## 15. Health Score Integration

### 14 Finding Types

|# |Finding                                        |Dimension      |Severity|
|--|-----------------------------------------------|---------------|--------|
|1 |Orphan endpoint (exposed, nobody calls)        |API Design     |Minor   |
|2 |Unmatched API call (consumer, no provider)     |Reliability    |Moderate|
|3 |API call without error handling                |Reliability    |Moderate|
|4 |Undocumented endpoint (SCIP found, not in spec)|API Design     |Minor   |
|5 |Deprecated endpoint still consumed             |API Design     |Moderate|
|6 |High fan-in endpoint (>5 consumers)            |API Design     |Info    |
|7 |Missing API versioning (no /v1/)               |API Design     |Minor   |
|8 |Platform service call without circuit breaker  |Reliability    |Major   |
|9 |DTO field mismatch (consumer vs spec schema)   |API Design     |Major   |
|10|Missing @Valid on @RequestBody                 |API Design     |Moderate|
|11|Chatty service pair (>20 calls)                |Design Patterns|Moderate|
|12|Inconsistent error handling across consumers   |Reliability    |Minor   |
|13|Ambiguous dependency (unresolved match)        |Reliability    |Minor   |
|14|Stale OpenAPI spec (>90 days unchanged)        |API Design     |Minor   |

-----

## 16. Automatic PR Blast-Radius Comments

```typescript
// src/lib/pr_hooks/blast_radius_comment.ts

export async function generatePRBlastRadiusComment(
  repoSlug: string,
  changedFiles: string[],
  pool: Pool
): Promise<string | null> {
  // Find endpoints in changed controller files
  const affected = await pool.query(`
    SELECT er.http_method, er.full_path, er.method_name
    FROM endpoint_registry er
    WHERE er.repo_slug = $1 AND er.file_path = ANY($2) AND er.is_active = true
  `, [repoSlug, changedFiles]);

  if (affected.rows.length === 0) return null;

  // Find all consumers of those endpoints
  const consumers = await pool.query(`
    SELECT ac.repo_slug, ac.file_path, ac.function_name,
           er.http_method, er.full_path, p.project_type
    FROM api_calls ac
    JOIN endpoint_registry er ON ac.matched_endpoint_id = er.id
    JOIN projects p ON p.repo_slug = ac.repo_slug
    WHERE er.repo_slug = $1 AND er.file_path = ANY($2)
      AND ac.is_active = true AND er.is_active = true
    ORDER BY p.project_type, ac.repo_slug
  `, [repoSlug, changedFiles]);

  if (consumers.rows.length === 0) return null;

  let comment = `## API Impact Analysis\n\n`;
  comment += `This PR modifies **${affected.rows.length} endpoint(s)** consumed by `;
  comment += `**${new Set(consumers.rows.map(c => c.repo_slug)).size} repo(s)**.\n\n`;

  for (const ep of affected.rows) {
    const epConsumers = consumers.rows.filter(c =>
      c.http_method === ep.http_method && c.full_path === ep.full_path
    );
    comment += `**${ep.http_method} ${ep.full_path}** (${ep.method_name})\n`;
    for (const c of epConsumers) {
      const icon = c.project_type === 'frontend' ? '🖥️' : '⚙️';
      comment += `  ${icon} ${c.repo_slug} → ${c.file_path}`;
      if (c.function_name) comment += ` (${c.function_name})`;
      comment += `\n`;
    }
    comment += `\n`;
  }

  comment += `> Run E2E tests for affected consumers before merging.`;
  return comment;
}
```

-----

## 17. Chatty Service Pair Detection

```typescript
// src/lib/endpoint_extraction/analysis/chatty_pairs.ts

export async function detectChattyPairs(pool: Pool): Promise<HealthFinding[]> {
  const findings: HealthFinding[] = [];

  const pairs = await pool.query(`
    SELECT consumer_repo, provider_repo, call_count,
           consumer_type, provider_role, provider_endpoints
    FROM mfe_ms_dependencies
    ORDER BY call_count DESC
  `);

  for (const pair of pairs.rows) {
    if (pair.consumer_type === 'backend' && pair.call_count > 15) {
      findings.push({
        dimension: 'design_patterns',
        severity: pair.call_count > 30 ? 'major' : 'moderate',
        title: `Chatty MS pair: ${pair.consumer_repo} → ${pair.provider_repo}`,
        evidence: `${pair.call_count} API calls across ${pair.provider_endpoints} endpoints`,
        recommendation: 'Consider event-driven communication, caching, or merging if domain overlap is high',
      });
    }

    if (pair.consumer_type === 'frontend' && pair.call_count > 25) {
      findings.push({
        dimension: 'design_patterns',
        severity: 'minor',
        title: `High MFE→MS coupling: ${pair.consumer_repo} → ${pair.provider_repo}`,
        evidence: `${pair.call_count} calls across ${pair.provider_endpoints} endpoints`,
        recommendation: 'Consider BFF aggregation layer or API composition to reduce round trips',
      });
    }
  }

  return findings;
}
```

-----

## 18. Distributed Tracing Integration (Future)

If BMT deploys OpenTelemetry / Jaeger / Zipkin:

```
Runtime traces become the #1 confidence source:
  - Confirms endpoints are actually called in production
  - Provides call frequency, latency, error rates
  - Validates proxy resolution (actual routing observed)
  - Detects dead dependencies (code exists, zero traffic)

Integration path:
  New bootstrap phase: 2g-traces (optional)
  New edge type: runtime_calls (with frequency + latency metadata)
  New confidence tier: runtime_verified (highest)

  Cross-validate: if static says A→B but traces show no traffic → dead dependency
                  if traces show A→B but static missed it → extraction gap
```

-----

## 19. Validation & Golden Endpoints

### 19.1 Verification Queries

```sql
-- Gateway prefix coverage
SELECT repo_slug, gateway_prefix,
       CASE WHEN gateway_prefix IS NULL THEN '❌ MISSING' ELSE '✅' END AS status
FROM projects
WHERE project_type IN ('backend', 'fullstack') AND active = true
ORDER BY gateway_prefix IS NULL DESC, repo_slug;

-- Extraction confidence breakdown
SELECT extraction_source, confidence, COUNT(*) AS endpoints
FROM endpoint_registry WHERE is_active = true
GROUP BY extraction_source, confidence ORDER BY COUNT(*) DESC;

-- Dependency matrix
SELECT * FROM mfe_ms_dependencies ORDER BY call_count DESC;

-- Orphan endpoints
SELECT er.repo_slug, er.http_method, er.full_path, er.deprecated
FROM endpoint_registry er
LEFT JOIN api_calls ac ON ac.matched_endpoint_id = er.id AND ac.is_active = true
WHERE er.is_active = true AND ac.id IS NULL
  AND er.full_path NOT LIKE '/actuator%' AND er.full_path NOT LIKE '/health%';

-- Unmatched consumer calls
SELECT ac.repo_slug, ac.http_method, ac.raw_path, ac.normalized_path, ac.call_pattern
FROM api_calls ac
WHERE ac.is_active = true AND ac.matched_endpoint_id IS NULL;

-- Platform services ranked
SELECT repo_slug, service_role, api_consumer_count, gateway_prefix
FROM projects WHERE service_role = 'platform' ORDER BY api_consumer_count DESC;

-- Source binding success rate
SELECT binding_method, COUNT(*) FROM endpoint_registry
WHERE is_active = true GROUP BY binding_method;
```

### 19.2 Golden Endpoint Test Suite

```typescript
const GOLDEN_TESTS = [
  { repo: 'ms-bonds-trading', method: 'POST', path: '/orders',
    mustHaveConsumer: 'mfe-bonds-trading' },
  { repo: 'ms-auth', method: 'POST', path: '/validate',
    minConsumers: 3 },
  { repo: 'ms-bonds-trading', calledBy: 'ms-settlement',
    method: 'GET', path: '/orders/:id' },
];

// Run as CI check after pipeline changes
```

-----

## 20. Implementation Schedule

```
Day 1: Schema + gateway_prefix + OpenAPI Parser
  ├── ALTER TABLE projects ADD COLUMN gateway_prefix
  ├── Create endpoint_registry, api_calls, api_dto_schemas tables
  ├── gateway_prefix auto-population (application.yml, app name, repo slug)
  ├── Manual override support
  ├── OpenAPI spec discovery + parser (v2 + v3)
  ├── DTO schema extraction
  └── Test: verify all MS repos get correct gateway_prefix

Day 2: SCIP Source Binding + Fallback Extractors
  ├── Controller method scanner (SCIP-verified)
  ├── Source binding: spec endpoint → Java method + file + line + graph_node_id
  ├── AST fallback for repos without SCIP
  ├── Merge logic: spec + SCIP bindings → ExtractedEndpoint[]
  └── Test: run on all MS repos, compare spec vs SCIP counts

Day 3: MFE API Call Extraction
  ├── Axios / fetch extraction (SCIP type-verified + AST fallback)
  ├── Client base path resolver (axios.create → baseURL → target repo)
  ├── RTK Query / React Query endpoint extraction
  ├── Env var name → target repo inference
  └── Test: run on all MFE repos

Day 4: MS-to-MS Call Extraction
  ├── Feign client extraction (SCIP-verified + AST fallback)
  ├── RestTemplate extraction (SCIP type + @Value → application.yml)
  ├── WebClient extraction (SCIP chain + .uri())
  └── Test: run on all MS repos, verify cross-MS calls detected

Day 5: Proxy Layer Resolution
  ├── Next.js rewrites + API routes
  ├── Express http-proxy-middleware + manual proxy
  ├── LLM infra config parsing (httpd/nginx/k8s) with cache
  ├── Dev server proxy config
  └── Test: verify proxy routes resolve correctly per MFE

Day 6: Matching Engine + Service Classification
  ├── Load gateway routes from projects.gateway_prefix
  ├── 8-strategy disambiguation chain
  ├── Platform / Domain / Utility classification
  ├── mfe_ms_dependencies materialized view
  └── Test: verify duplicate /orders paths resolve correctly via prefix

Day 7: Graph Integration + MCP Tools + Health Score
  ├── api_endpoint + api_call graph nodes via IGraphClient
  ├── consumes_api + exposes_api + calls_api + proxies_to edges
  ├── Enhanced ome_analyze_blast_radius
  ├── ome_get_api_dependencies + ome_trace_api_flow + ome_get_api_surface
  ├── 14 health score finding types
  ├── Chatty service pair detection
  └── PR blast-radius comment generation

Day 8: Full Test + Calibration
  ├── Full pipeline run across all 47 repos
  ├── Review gateway_prefix coverage (all MSes covered?)
  ├── Review ambiguous matches — tune disambiguation
  ├── Golden endpoint regression suite
  ├── Accuracy report (spec vs SCIP vs AST)
  ├── Calibrate chatty-pair thresholds
  └── Contextual header enrichment with API context
```

-----

## 21. Configuration Reference

```bash
# ═══ Feature flags ═══
EXTRACT_API_DEPENDENCIES=true             # Enable Phase 2g-api
EXTRACT_OPENAPI_SPECS=true                # Parse OpenAPI/Swagger specs
EXTRACT_MS_TO_MS_CALLS=true              # Feign / RestTemplate / WebClient
EXTRACT_PROXY_LAYERS=true                # Resolve MFE proxy configs
LLM_INFRA_PROXY_PARSING=true            # Use Qwen for httpd/nginx/k8s configs

# ═══ Gateway prefix ═══
GATEWAY_PREFIX_OVERRIDES=./config/gateway-prefix-overrides.json  # Optional manual overrides

# ═══ Matching ═══
MIN_MATCH_CONFIDENCE=medium              # Drop matches below this
FLAG_AMBIGUOUS_MATCHES=true              # Store ambiguous for review

# ═══ Classification ═══
PLATFORM_SERVICE_MIN_CONSUMERS=4         # ≥4 total consumers → platform
PLATFORM_SERVICE_MIN_MS_CONSUMERS=2      # ≥2 MS consumers → platform

# ═══ Health score ═══
CHATTY_PAIR_MS_THRESHOLD=15              # MS-to-MS calls → chatty finding
CHATTY_PAIR_MFE_THRESHOLD=25             # MFE-to-MS calls → chatty finding
ENDPOINT_HEALTH_ENABLED=true

# ═══ PR hooks ═══
PR_BLAST_RADIUS_COMMENTS=true
```

### File Structure

```
src/lib/endpoint_extraction/
  pipeline.ts                        — Phase 2g-api orchestrator
  types.ts                           — Shared interfaces
  gateway_prefix_resolver.ts         — Auto-populate from application.yml

  openapi/
    spec_discovery.ts                — Find spec files in repo
    spec_parser.ts                   — Parse OpenAPI v2/v3
    dto_extractor.ts                 — Extract DTO schemas
    spec_merger.ts                   — Merge spec + SCIP bindings

  scip/
    source_binder.ts                 — Bind spec endpoints to Java source
    controller_scanner.ts            — SCIP-verified controller scanner

  mfe/
    mfe_extractor.ts                 — Unified MFE API call extractor
    client_resolver.ts               — axios.create → baseURL → target repo

  ms_to_ms/
    ms_consumer_extractor.ts         — Unified MS-to-MS extractor (Feign/RT/WC)

  proxy/
    proxy_resolver.ts                — 5-layer proxy resolution
    nextjs_extractor.ts              — rewrites + API routes
    express_extractor.ts             — http-proxy-middleware + manual
    llm_infra_parser.ts              — Qwen for httpd/nginx/k8s
    devserver_extractor.ts           — Vite/Webpack dev proxy

  matching/
    matcher.ts                       — 8-strategy disambiguation
    endpoint_index.ts                — Path+method lookup
    gateway_loader.ts                — Load routes from projects.gateway_prefix
    path_utils.ts                    — Normalization, pattern matching

  classification/
    service_classifier.ts            — Platform / Domain / Utility

  graph/
    api_graph_edges.ts               — Node/edge creation via IGraphClient

  analysis/
    chatty_pairs.ts                  — Chatty service pair detection

  health/
    endpoint_health.ts               — 14 finding types

  pr_hooks/
    blast_radius_comment.ts          — Bitbucket PR comments

  validation/
    golden_endpoints.ts              — Regression test suite
```

import { useState } from “react”;

const phases = [
{
id: 1,
title: “Inventory & Catalog”,
subtitle: “Parse → Register → Baseline”,
icon: “📋”,
color: “#2D6A4F”,
duration: “Week 1–2”,
goal: “Build a complete structural inventory of both COBOL and Java codebases into Postgres tables.”,
steps: [
{
name: “COBOL Source Parsing”,
detail:
“Parse every COBOL source file using Tree-sitter (tree-sitter-cobol grammar). Extract: PROGRAM-ID, SECTIONs, PARAGRAPHs, COPY statements, data items (WORKING-STORAGE, LINKAGE, FILE SECTION), PERFORM targets, CALL targets, file descriptors (SELECT/ASSIGN), and embedded SQL (EXEC SQL blocks).”,
output: “cobol_constructs table in Postgres”,
schema: `CREATE TABLE cobol_constructs ( id SERIAL PRIMARY KEY, file_path TEXT NOT NULL, program_id TEXT, construct_type TEXT NOT NULL, -- 'program','section','paragraph','copybook', -- 'data_item','file_descriptor','sql_block', -- 'call_target','perform_target' construct_name TEXT NOT NULL, start_line INT, end_line INT, loc INT, parent_construct_id INT REFERENCES cobol_constructs(id), raw_source TEXT, content_hash TEXT,  -- MD5 for dedup metadata JSONB,     -- verb counts, complexity created_at TIMESTAMPTZ DEFAULT NOW() );`,
},
{
name: “Java Target Parsing”,
detail:
“Parse every generated Java file using Tree-sitter (tree-sitter-java). Extract: packages, classes, methods (with signatures), fields, imports, annotations, exception handlers, and SQL/JDBC calls.”,
output: “java_constructs table in Postgres”,
schema: `CREATE TABLE java_constructs ( id SERIAL PRIMARY KEY, file_path TEXT NOT NULL, package_name TEXT, class_name TEXT, construct_type TEXT NOT NULL, -- 'class','method','field','import', -- 'annotation','exception_handler', -- 'jdbc_call','inner_class' construct_name TEXT NOT NULL, signature TEXT,    -- full method signature start_line INT, end_line INT, loc INT, parent_construct_id INT REFERENCES java_constructs(id), raw_source TEXT, content_hash TEXT, metadata JSONB, created_at TIMESTAMPTZ DEFAULT NOW() );`,
},
{
name: “Baseline Metrics Snapshot”,
detail:
“Compute aggregate metrics: total COBOL programs, sections, paragraphs, copybooks, LOC, verb distribution (COMPUTE, EVALUATE, PERFORM, READ, WRITE, CALL). Same for Java: classes, methods, fields, LOC. Store as a versioned snapshot for trend tracking.”,
output: “conversion_baseline table”,
schema: `CREATE TABLE conversion_baseline ( id SERIAL PRIMARY KEY, snapshot_date DATE DEFAULT CURRENT_DATE, side TEXT NOT NULL,  -- 'cobol' or 'java' metric_name TEXT NOT NULL, metric_value NUMERIC, breakdown JSONB     -- per-program detail );`,
},
],
},
{
id: 2,
title: “Deterministic Mapping”,
subtitle: “Rule-Based Traceability”,
icon: “🔗”,
color: “#1B4332”,
duration: “Week 2–3”,
goal: “Establish hard links between COBOL constructs and Java constructs using naming conventions, file patterns, and Windsurf conversion metadata.”,
steps: [
{
name: “Convention-Based Matching”,
detail:
“Windsurf typically follows naming conventions during conversion. Build matchers for: PROGRAM-ID → Java class name (e.g., CALC-INTEREST → CalcInterest.java), SECTION/PARAGRAPH → method name (e.g., 100-INIT → init() or initialize()), COPYBOOK → Java DTO/record class, FILE-SECTION → I/O handler class. Use normalized fuzzy name matching (strip hyphens, underscores, apply camelCase↔KEBAB-CASE conversion).”,
output: “Populated traceability_map with confidence=‘high’”,
schema: `CREATE TABLE traceability_map ( id SERIAL PRIMARY KEY, cobol_construct_id INT REFERENCES cobol_constructs(id), java_construct_id INT REFERENCES java_constructs(id), mapping_type TEXT NOT NULL, -- 'name_match','structure_match','semantic_match', -- 'manual','windsurf_metadata' confidence TEXT NOT NULL, -- 'high','medium','low','unmatched' similarity_score NUMERIC, notes TEXT, verified_by TEXT,       -- manual reviewer verified_at TIMESTAMPTZ, created_at TIMESTAMPTZ DEFAULT NOW() );`,
},
{
name: “Structural Pattern Matching”,
detail:
“Beyond names, match by structural signatures: COBOL paragraph with 3 PERFORMs + 1 COMPUTE → Java method with 3 method calls + 1 arithmetic expression. Use AST node-type sequences as fingerprints. Match COPY→import chains, CALL→dependency chains, and file I/O patterns (OPEN/READ/WRITE/CLOSE → stream/reader patterns).”,
output: “Additional traceability_map rows with confidence=‘medium’”,
},
{
name: “Windsurf Metadata Extraction”,
detail:
“If Windsurf left any conversion comments (// Converted from: CALC-INTEREST.COMPUTE-RATE), markers, or mapping files, parse those as the highest-confidence source. Check for: inline comments referencing COBOL source, any .windsurf or conversion log files, git commit messages from the conversion session.”,
output: “traceability_map rows with mapping_type=‘windsurf_metadata’”,
},
],
},
{
id: 3,
title: “Semantic Embedding Layer”,
subtitle: “Vector-Based Gap Detection”,
icon: “🧠”,
color: “#40916C”,
duration: “Week 3–4”,
goal: “Use nomic-embed-code to catch what deterministic matching missed — fuzzy matches, refactored logic, and semantic gaps.”,
steps: [
{
name: “Embed COBOL Constructs”,
detail:
“For every COBOL paragraph/section with >5 LOC, generate an embedding using nomic-embed-code (already deployed via Ollama). Prefix each chunk with ’search_document: COBOL paragraph <name> in <program>: ’ followed by the source. Store in pgvector.”,
output: “cobol_embeddings table with vector(3584) column”,
schema: `CREATE TABLE cobol_embeddings ( id SERIAL PRIMARY KEY, construct_id INT REFERENCES cobol_constructs(id), embedding vector(3584), chunk_text TEXT, created_at TIMESTAMPTZ DEFAULT NOW() ); CREATE INDEX ON cobol_embeddings USING ivfflat (embedding vector_cosine_ops) WITH (lists = 100);`,
},
{
name: “Embed Java Constructs”,
detail:
“Same process for Java methods/classes with >5 LOC. Prefix: ’search_document: Java method <class>.<method>: ’. Store in pgvector with same dimensionality.”,
output: “java_embeddings table with vector(3584) column”,
schema: `CREATE TABLE java_embeddings ( id SERIAL PRIMARY KEY, construct_id INT REFERENCES java_constructs(id), embedding vector(3584), chunk_text TEXT, created_at TIMESTAMPTZ DEFAULT NOW() ); CREATE INDEX ON java_embeddings USING ivfflat (embedding vector_cosine_ops) WITH (lists = 100);`,
},
{
name: “Cross-Language Nearest Neighbor Search”,
detail:
“For every COBOL construct NOT yet mapped (confidence != ‘high’), find its top-3 nearest Java neighbors via cosine similarity. Apply thresholds: >0.85 → auto-map as ‘medium’ confidence, 0.70–0.85 → flag for manual review, <0.70 → mark as ‘unmatched’ (potential coverage gap). Also run reverse: find Java constructs with no nearby COBOL neighbor (potentially dead/added code).”,
output: “Additional traceability_map rows + coverage_gaps table”,
schema: `CREATE TABLE coverage_gaps ( id SERIAL PRIMARY KEY, side TEXT NOT NULL,        -- 'cobol_unmapped' or 'java_orphan' construct_id INT NOT NULL, nearest_match_id INT, nearest_similarity NUMERIC, gap_type TEXT, -- 'missing_conversion','partial_conversion', -- 'logic_drift','dead_code' severity TEXT,             -- 'critical','major','minor' resolution TEXT, resolved_by TEXT, resolved_at TIMESTAMPTZ, created_at TIMESTAMPTZ DEFAULT NOW() );`,
},
],
},
{
id: 4,
title: “Graph-Based Dependency Validation”,
subtitle: “Apache AGE Call-Chain Verification”,
icon: “🕸️”,
color: “#52B788”,
duration: “Week 4–5”,
goal: “Verify that COBOL call chains and data flows are preserved in Java using your existing Apache AGE graph.”,
steps: [
{
name: “Build COBOL Call Graph”,
detail:
“In Apache AGE, create nodes for every COBOL program, section, paragraph, and copybook. Create edges: PERFORMS (paragraph→paragraph), CALLS (program→program), COPIES (program→copybook), READS/WRITES (paragraph→file). This gives you the complete COBOL dependency graph.”,
output: “COBOL subgraph in Apache AGE”,
schema: `– Apache AGE Cypher
SELECT * FROM cypher(‘conversion_graph’, $$
CREATE (:CobolProgram {
name: ‘CALC-INTEREST’,
file: ‘CALC-INTEREST.cbl’,
construct_id: 42
})
$$) AS (v agtype);

SELECT * FROM cypher(‘conversion_graph’, $$
MATCH (a:CobolParagraph {name: ‘100-INIT’}),
(b:CobolParagraph {name: ‘200-COMPUTE’})
CREATE (a)-[:PERFORMS]->(b)
$$) AS (e agtype);`, }, { name: "Build Java Call Graph", detail: "Same for Java: nodes for classes and methods, edges for CALLS (method→method), IMPORTS (class→class), EXTENDS/IMPLEMENTS, USES_FIELD. Build from Tree-sitter AST by resolving method invocations to their declarations.", output: "Java subgraph in Apache AGE", }, { name: "Cross-Graph Isomorphism Check", detail: "Using the traceability_map, create CONVERTS_TO edges between COBOL and Java nodes. Then validate: (1) Every COBOL call chain A→B→C should have a corresponding Java chain A'→B'→C'. (2) Fan-out/fan-in should be preserved (if COBOL paragraph calls 5 sub-paragraphs, Java method should call ~5 methods). (3) Data flow: if COBOL reads FILE-A in paragraph X and writes FILE-B in paragraph Y, Java should have equivalent I/O in mapped methods.", output: "Graph validation report with broken chain alerts", schema: `– Find broken call chains
SELECT * FROM cypher(‘conversion_graph’, $$
MATCH (cp:CobolParagraph)-[:PERFORMS]->(ct:CobolParagraph)
WHERE NOT EXISTS {
MATCH (cp)-[:CONVERTS_TO]->(jp:JavaMethod),
(ct)-[:CONVERTS_TO]->(jt:JavaMethod),
(jp)-[:CALLS]->(jt)
}
RETURN cp.name AS cobol_caller,
ct.name AS cobol_callee,
‘BROKEN_CHAIN’ AS issue
$$) AS (caller agtype, callee agtype, issue agtype);`, }, ], }, { id: 5, title: "Behavioral Validation", subtitle: "I/O Parity Testing", icon: "✅", color: "#74C69D", duration: "Week 5–7", goal: "Prove functional equivalence by running identical inputs through both COBOL and Java and comparing outputs byte-for-byte.", steps: [ { name: "Test Case Harvesting", detail: "Gather existing test data: production input files, JCL job streams with sample data, copybook-based test fixtures, database seed scripts, and any QA regression packs. For each COBOL program, document: input files/formats, expected output files/formats, return codes, and database side effects.", output: "test_cases table with input/output pairs per program", }, { name: "Parallel Execution Harness", detail: "Build a test runner that: (1) feeds identical input to the COBOL program (via GnuCOBOL or existing mainframe) and the Java equivalent, (2) captures stdout, output files, DB changes, and return codes from both, (3) diffs the outputs. Use your Vitest 3 setup for the Java side. Flag any divergence as a behavioral gap.", output: "Automated comparison pipeline", schema: `CREATE TABLE behavioral_results (
id SERIAL PRIMARY KEY,
test_case_id INT,
cobol_program TEXT,
java_class TEXT,
input_hash TEXT,
cobol_output_hash TEXT,
java_output_hash TEXT,
match BOOLEAN,
diff_summary TEXT,
diff_detail JSONB,
run_at TIMESTAMPTZ DEFAULT NOW()
);`, }, { name: "Edge Case Generation", detail: "Use Ollama (your local LLM) to generate edge cases from COBOL source: boundary values for PIC 9(n) fields, SPACES/ZEROS/HIGH-VALUES handling, REDEFINES overlaps, signed vs unsigned numerics, COMP-3 packed decimal edge cases, and leap year / date arithmetic. Feed these through the parallel harness.", output: "Expanded test suite covering conversion-sensitive logic", }, ], }, { id: 6, title: "Coverage Dashboard & Reporting", subtitle: "Single Pane of Glass", icon: "📊", color: "#95D5B2", duration: "Week 7–8", goal: "Build a real-time dashboard showing conversion coverage, gaps, and verification status across all layers.", steps: [ { name: "Coverage Metrics Engine", detail: "Compute and store: Structural Coverage (% of COBOL constructs with a mapped Java construct), Semantic Coverage (% with similarity >0.85), Graph Coverage (% of call chains preserved), Behavioral Coverage (% of test cases passing parity), and Overall Composite Score (weighted: Structural 20%, Semantic 20%, Graph 25%, Behavioral 35%).", output: "Materialized views in Postgres refreshed on demand", schema: `CREATE MATERIALIZED VIEW coverage_summary AS
SELECT
‘structural’ AS layer,
COUNT(*) FILTER (
WHERE confidence IN (‘high’,‘medium’)
)::NUMERIC / NULLIF(COUNT(*), 0) * 100
AS coverage_pct,
COUNT(*) FILTER (
WHERE confidence = ‘unmatched’
) AS gap_count
FROM traceability_map
UNION ALL
SELECT
‘behavioral’,
COUNT(*) FILTER (WHERE match = true)::NUMERIC
/ NULLIF(COUNT(*), 0) * 100,
COUNT(*) FILTER (WHERE match = false)
FROM behavioral_results;`,
},
{
name: “Dashboard UI”,
detail:
“Extend your existing nautical-themed Apollo dashboard. Add panels: (1) Conversion Coverage Donut (overall %), (2) Gap Heatmap by COBOL program, (3) Drill-down: click a program → see its paragraphs → see mapped Java methods → see test results, (4) Trend line: coverage % over time as gaps get resolved, (5) Risk register: unmapped constructs sorted by LOC and complexity.”,
output: “HTML dashboard with live Postgres queries”,
},
{
name: “Stakeholder Reports”,
detail:
“Auto-generate reports for Sarah (Legal) and other stakeholders: Executive Summary (overall % with RAG status), Audit Trail (every mapping decision with who verified it), Risk Log (unmapped items with severity), and Sign-off Tracker (per-program verification status).”,
output: “PDF/DOCX reports on demand”,
},
],
},
{
id: 7,
title: “Continuous Governance”,
subtitle: “Prevent Regression”,
icon: “🔒”,
color: “#B7E4C7”,
duration: “Ongoing”,
goal: “Ensure traceability stays valid as code evolves post-conversion.”,
steps: [
{
name: “Git Hook Integration”,
detail:
“On every commit to the Java codebase, re-parse changed files, update java_constructs, recompute affected embeddings, and re-run graph validation for touched methods. Alert if a mapped Java method is deleted or significantly refactored (content_hash change).”,
output: “Pre-commit and post-commit hooks”,
},
{
name: “Drift Detection Job”,
detail:
“Weekly cron: re-embed all Java constructs modified in the past 7 days, recompute similarity against their mapped COBOL counterparts. If similarity drops below 0.70 for any pair, raise a drift alert. This catches gradual logic divergence as developers refactor the Java code.”,
output: “drift_alerts table + notification pipeline”,
},
{
name: “Decommission Gate”,
detail:
“Before any COBOL program can be decommissioned, require: (1) 100% structural mapping at paragraph level, (2) all behavioral tests passing, (3) graph validation clean, (4) manual sign-off recorded. This becomes your formal go/no-go checkpoint.”,
output: “Decommission readiness checklist per program”,
},
],
},
];

const techStack = [
{ component: “Tree-sitter”, role: “AST parsing for COBOL and Java”, status: “Existing (Apollo)” },
{ component: “Postgres + pgvector”, role: “Construct catalog, embeddings, traceability tables”, status: “Existing” },
{ component: “Apache AGE”, role: “Call-graph validation, cross-language isomorphism”, status: “Existing” },
{ component: “nomic-embed-code”, role: “Code embeddings for semantic matching (3584 dims)”, status: “Existing (Ollama)” },
{ component: “Ollama LLM”, role: “Edge case generation, conversion comment analysis”, status: “Existing” },
{ component: “Vitest 3”, role: “Java-side behavioral test execution”, status: “Existing” },
{ component: “Node.js orchestrator”, role: “Pipeline coordination, dashboard backend”, status: “Existing” },
{ component: “GnuCOBOL (or mainframe)”, role: “COBOL-side test execution for parity”, status: “To confirm” },
];

export default function TraceabilityPlan() {
const [activePhase, setActivePhase] = useState(0);
const [activeStep, setActiveStep] = useState(0);
const [showSchema, setShowSchema] = useState(null);
const [view, setView] = useState(“phases”); // ‘phases’ | ‘stack’ | ‘flow’

const phase = phases[activePhase];
const step = phase.steps[activeStep];

return (
<div style={{
fontFamily: “‘IBM Plex Mono’, ‘Fira Code’, monospace”,
background: “#0A1F12”,
color: “#D8F3DC”,
minHeight: “100vh”,
padding: “0”,
}}>
{/* Header */}
<div style={{
background: “linear-gradient(135deg, #1B4332 0%, #2D6A4F 50%, #1B4332 100%)”,
padding: “24px 20px 16px”,
borderBottom: “2px solid #40916C”,
}}>
<div style={{ fontSize: “10px”, letterSpacing: “4px”, color: “#95D5B2”, marginBottom: “4px” }}>
PROJECT APOLLO — MODERNIZATION ASSURANCE
</div>
<div style={{ fontSize: “20px”, fontWeight: “700”, color: “#D8F3DC”, marginBottom: “4px” }}>
COBOL → Java Traceability
</div>
<div style={{ fontSize: “11px”, color: “#74C69D” }}>
7-Phase Fool-Proof Coverage & Verification Plan
</div>
</div>

```
  {/* View Tabs */}
  <div style={{
    display: "flex",
    gap: "0",
    borderBottom: "1px solid #2D6A4F",
    background: "#0D2818",
  }}>
    {[
      { key: "phases", label: "PHASES" },
      { key: "stack", label: "TECH STACK" },
      { key: "flow", label: "DATA FLOW" },
    ].map((tab) => (
      <button
        key={tab.key}
        onClick={() => setView(tab.key)}
        style={{
          flex: 1,
          padding: "10px 8px",
          background: view === tab.key ? "#1B4332" : "transparent",
          color: view === tab.key ? "#B7E4C7" : "#52B788",
          border: "none",
          borderBottom: view === tab.key ? "2px solid #95D5B2" : "2px solid transparent",
          fontSize: "11px",
          fontWeight: "600",
          letterSpacing: "1.5px",
          cursor: "pointer",
          fontFamily: "inherit",
        }}
      >
        {tab.label}
      </button>
    ))}
  </div>

  {view === "phases" && (
    <div>
      {/* Phase Selector */}
      <div style={{
        display: "flex",
        overflowX: "auto",
        gap: "6px",
        padding: "12px 12px 8px",
        background: "#0D2818",
      }}>
        {phases.map((p, i) => (
          <button
            key={p.id}
            onClick={() => { setActivePhase(i); setActiveStep(0); setShowSchema(null); }}
            style={{
              minWidth: "40px",
              height: "40px",
              borderRadius: "8px",
              border: activePhase === i ? "2px solid #95D5B2" : "1px solid #2D6A4F",
              background: activePhase === i ? "#2D6A4F" : "#132A1C",
              color: "#D8F3DC",
              fontSize: "16px",
              cursor: "pointer",
              display: "flex",
              alignItems: "center",
              justifyContent: "center",
              flexShrink: 0,
              position: "relative",
            }}
          >
            {p.icon}
            <span style={{
              position: "absolute",
              bottom: "-2px",
              right: "4px",
              fontSize: "8px",
              color: "#74C69D",
              fontFamily: "inherit",
            }}>{p.id}</span>
          </button>
        ))}
      </div>

      {/* Phase Header */}
      <div style={{
        padding: "16px 16px 12px",
        borderBottom: "1px solid #1B4332",
      }}>
        <div style={{ display: "flex", alignItems: "baseline", gap: "8px", marginBottom: "4px" }}>
          <span style={{ fontSize: "22px" }}>{phase.icon}</span>
          <span style={{ fontSize: "16px", fontWeight: "700", color: "#B7E4C7" }}>
            Phase {phase.id}: {phase.title}
          </span>
        </div>
        <div style={{ fontSize: "11px", color: "#52B788", marginBottom: "8px" }}>
          {phase.subtitle} · {phase.duration}
        </div>
        <div style={{
          fontSize: "12px",
          color: "#95D5B2",
          lineHeight: "1.5",
          background: "#132A1C",
          padding: "10px 12px",
          borderRadius: "6px",
          borderLeft: `3px solid ${phase.color}`,
        }}>
          {phase.goal}
        </div>
      </div>

      {/* Step Tabs */}
      <div style={{
        display: "flex",
        gap: "4px",
        padding: "10px 12px",
        overflowX: "auto",
      }}>
        {phase.steps.map((s, i) => (
          <button
            key={i}
            onClick={() => { setActiveStep(i); setShowSchema(null); }}
            style={{
              padding: "6px 12px",
              borderRadius: "14px",
              border: "none",
              background: activeStep === i ? "#40916C" : "#1B4332",
              color: activeStep === i ? "#D8F3DC" : "#74C69D",
              fontSize: "11px",
              fontWeight: "600",
              cursor: "pointer",
              whiteSpace: "nowrap",
              fontFamily: "inherit",
            }}
          >
            {i + 1}. {s.name}
          </button>
        ))}
      </div>

      {/* Step Detail */}
      <div style={{ padding: "8px 16px 20px" }}>
        <div style={{
          fontSize: "14px",
          fontWeight: "700",
          color: "#B7E4C7",
          marginBottom: "8px",
        }}>
          {step.name}
        </div>
        <div style={{
          fontSize: "12px",
          color: "#95D5B2",
          lineHeight: "1.6",
          marginBottom: "12px",
        }}>
          {step.detail}
        </div>

        {step.output && (
          <div style={{
            display: "flex",
            alignItems: "center",
            gap: "6px",
            fontSize: "11px",
            color: "#74C69D",
            marginBottom: "10px",
          }}>
            <span style={{
              background: "#2D6A4F",
              padding: "2px 8px",
              borderRadius: "4px",
              fontSize: "10px",
              letterSpacing: "1px",
            }}>OUTPUT</span>
            {step.output}
          </div>
        )}

        {step.schema && (
          <button
            onClick={() => setShowSchema(showSchema === activeStep ? null : activeStep)}
            style={{
              background: "#1B4332",
              border: "1px solid #40916C",
              color: "#95D5B2",
              padding: "8px 14px",
              borderRadius: "6px",
              fontSize: "11px",
              cursor: "pointer",
              fontFamily: "inherit",
              width: "100%",
              textAlign: "left",
            }}
          >
            {showSchema === activeStep ? "▼" : "▶"} View SQL / Schema
          </button>
        )}

        {showSchema === activeStep && step.schema && (
          <pre style={{
            background: "#081710",
            border: "1px solid #1B4332",
            borderRadius: "6px",
            padding: "12px",
            fontSize: "10px",
            lineHeight: "1.5",
            color: "#74C69D",
            overflow: "auto",
            marginTop: "8px",
            whiteSpace: "pre-wrap",
            wordBreak: "break-word",
          }}>
            {step.schema}
          </pre>
        )}
      </div>
    </div>
  )}

  {view === "stack" && (
    <div style={{ padding: "16px" }}>
      <div style={{
        fontSize: "13px",
        fontWeight: "700",
        color: "#B7E4C7",
        marginBottom: "12px",
      }}>
        Infrastructure — 100% Reuse from Apollo
      </div>
      <div style={{
        fontSize: "11px",
        color: "#74C69D",
        marginBottom: "16px",
        lineHeight: "1.5",
      }}>
        Every component already exists in your stack.
        No new dependencies required except optionally GnuCOBOL for behavioral testing.
      </div>
      {techStack.map((t, i) => (
        <div key={i} style={{
          background: "#132A1C",
          borderRadius: "8px",
          padding: "12px",
          marginBottom: "8px",
          borderLeft: t.status === "To confirm" ? "3px solid #E76F51" : "3px solid #40916C",
        }}>
          <div style={{ display: "flex", justifyContent: "space-between", alignItems: "center", marginBottom: "4px" }}>
            <span style={{ fontSize: "12px", fontWeight: "700", color: "#B7E4C7" }}>
              {t.component}
            </span>
            <span style={{
              fontSize: "9px",
              padding: "2px 8px",
              borderRadius: "10px",
              background: t.status === "To confirm" ? "#5C2E1A" : "#1B4332",
              color: t.status === "To confirm" ? "#E9C46A" : "#74C69D",
              letterSpacing: "0.5px",
            }}>
              {t.status.toUpperCase()}
            </span>
          </div>
          <div style={{ fontSize: "11px", color: "#95D5B2" }}>{t.role}</div>
        </div>
      ))}
    </div>
  )}

  {view === "flow" && (
    <div style={{ padding: "16px" }}>
      <div style={{
        fontSize: "13px",
        fontWeight: "700",
        color: "#B7E4C7",
        marginBottom: "16px",
      }}>
        Coverage Layers — Defense in Depth
      </div>
      {[
        {
          layer: "Layer 1: Structural",
          weight: "20%",
          method: "Tree-sitter AST → naming & pattern matching",
          catches: "Direct 1:1 mappings, naming convention matches",
          color: "#2D6A4F",
        },
        {
          layer: "Layer 2: Semantic",
          weight: "20%",
          method: "nomic-embed-code → pgvector cosine similarity",
          catches: "Refactored logic, renamed constructs, fuzzy matches",
          color: "#40916C",
        },
        {
          layer: "Layer 3: Graph",
          weight: "25%",
          method: "Apache AGE → call-chain isomorphism",
          catches: "Broken call chains, missing dependencies, data flow gaps",
          color: "#52B788",
        },
        {
          layer: "Layer 4: Behavioral",
          weight: "35%",
          method: "Parallel execution → byte-level output diff",
          catches: "Logic errors, edge case failures, numeric precision drift",
          color: "#74C69D",
        },
      ].map((l, i) => (
        <div key={i} style={{
          background: "#132A1C",
          borderRadius: "8px",
          padding: "14px",
          marginBottom: "10px",
          borderLeft: `4px solid ${l.color}`,
        }}>
          <div style={{ display: "flex", justifyContent: "space-between", marginBottom: "6px" }}>
            <span style={{ fontSize: "12px", fontWeight: "700", color: "#B7E4C7" }}>
              {l.layer}
            </span>
            <span style={{
              fontSize: "11px",
              color: "#95D5B2",
              background: "#1B4332",
              padding: "1px 8px",
              borderRadius: "8px",
            }}>
              {l.weight}
            </span>
          </div>
          <div style={{ fontSize: "11px", color: "#74C69D", marginBottom: "4px" }}>
            {l.method}
          </div>
          <div style={{ fontSize: "11px", color: "#52B788" }}>
            Catches: {l.catches}
          </div>
          {i < 3 && (
            <div style={{
              textAlign: "center",
              color: "#40916C",
              fontSize: "14px",
              margin: "6px 0 -8px",
            }}>↓ feeds gaps to ↓</div>
          )}
        </div>
      ))}

      <div style={{
        background: "#1B4332",
        borderRadius: "8px",
        padding: "14px",
        marginTop: "16px",
        textAlign: "center",
        border: "1px solid #52B788",
      }}>
        <div style={{ fontSize: "10px", letterSpacing: "2px", color: "#74C69D", marginBottom: "4px" }}>
          COMPOSITE SCORE
        </div>
        <div style={{ fontSize: "11px", color: "#95D5B2", lineHeight: "1.6" }}>
          0.20 × Structural + 0.20 × Semantic + 0.25 × Graph + 0.35 × Behavioral
        </div>
        <div style={{ fontSize: "10px", color: "#52B788", marginTop: "6px" }}>
          Decommission gate: ALL layers must individually pass ≥ 95%
        </div>
      </div>
    </div>
  )}

  {/* Footer */}
  <div style={{
    padding: "12px 16px",
    borderTop: "1px solid #1B4332",
    fontSize: "10px",
    color: "#40916C",
    textAlign: "center",
    letterSpacing: "1px",
  }}>
    APOLLO TRACEABILITY · RHEL 128-CORE · ZERO EXTERNAL DEPS
  </div>
</div>
```

);
}

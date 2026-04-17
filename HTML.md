# Project Sentinel — Dashboard Implementation Plan

## Gap Analysis & Review Workflow UI

**Version:** 1.0
**Date:** April 15, 2026
**Status:** Ready for Development
**Depends On:** Sentinel core pipeline (Phases 1–6), Postgres schema, Express API

-----

## Table of Contents

1. [Overview](#1-overview)
1. [Architecture](#2-architecture)
1. [API Layer — Express Backend](#3-api-layer--express-backend)
1. [Dashboard Views](#4-dashboard-views)
1. [Gap Detail Panel](#5-gap-detail-panel)
1. [Review Workflow UI](#6-review-workflow-ui)
1. [Stub Detection View](#7-stub-detection-view)
1. [Reporting & Export](#8-reporting--export)
1. [Component Structure](#9-component-structure)
1. [Database Queries — Complete Reference](#10-database-queries--complete-reference)
1. [State Management](#11-state-management)
1. [Real-Time Updates](#12-real-time-updates)
1. [Configuration](#13-configuration)
1. [Build & Deployment](#14-build--deployment)
1. [Testing](#15-testing)
1. [Timeline](#16-timeline)

-----

## 1. Overview

### 1.1 Purpose

The Sentinel Dashboard is the primary interface for the engineering team, tech lead, and stakeholders (including Legal) to monitor conversion coverage, review gaps, verify stub detections, and sign off on decommission readiness. It is the single pane of glass for Project Sentinel.

### 1.2 Users & Their Needs

|User         |Primary Need                          |Key Views                     |
|-------------|--------------------------------------|------------------------------|
|Tech Lead    |Know which gaps need review today     |Gap List → Review queue       |
|Engineer     |Understand what to implement          |Gap Detail → Source comparison|
|QA           |Track behavioral test failures        |Behavioral results per program|
|SME          |Review escalated NonStop-specific gaps|Stub + TAL/C views            |
|Legal (Sarah)|Sign off on decommission              |Decommission gate per program |
|Management   |Know overall progress                 |Summary + trend               |

### 1.3 Technology

|Layer     |Choice                               |Rationale                                                          |
|----------|-------------------------------------|-------------------------------------------------------------------|
|Frontend  |React 18 (JSX)                       |Reusable components, hooks, no build tooling for dashboard delivery|
|Styling   |Tailwind CSS (utility classes only)  |No compiler needed in this environment                             |
|API       |Express 4 + TypeScript               |Consistent with Sentinel’s Node.js orchestrator                    |
|DB queries|Postgres (pg driver) + recursive CTEs|All data in one place                                              |
|Real-time |Server-Sent Events (SSE)             |No WebSocket server needed, works over HTTP                        |
|PDF export|Pandoc CLI wrapper                   |Already available on RHEL, zero deps                               |
|Charts    |Recharts (via CDN in dashboard)      |Works without build step                                           |

### 1.4 Hosting

Served locally on the RHEL server — `http://localhost:3001`. No internet required. No external API calls. Engineers access via SSH tunnel or internal network.

-----

## 2. Architecture

```
Browser (React SPA)
        │
        │  HTTP / SSE
        ▼
Express API Server (port 3001)
  src/dashboard/api-server.ts
        │
        ├──► Coverage Engine (materialized views)
        │    src/dashboard/coverage-engine.ts
        │
        ├──► Gap API
        │    src/dashboard/gap-api.ts
        │
        ├──► Stub API
        │    src/dashboard/stub-api.ts
        │
        ├──► Review API
        │    src/dashboard/review-api.ts
        │
        ├──► Report Generator
        │    src/reporting/report-generator.ts
        │
        └──► SSE Event Emitter
             src/dashboard/event-emitter.ts
                    │
                    └──► Postgres LISTEN/NOTIFY
```

### 2.1 Data Flow

```
Pipeline runs (cron / manual)
        │
        ▼
Postgres tables updated
        │
        ▼
Postgres NOTIFY on sentinel_changes channel
        │
        ▼
Express SSE endpoint picks up notification
        │
        ▼
Browser receives SSE event
        │
        ▼
React invalidates relevant query
        │
        ▼
Dashboard refreshes affected panel only
```

-----

## 3. API Layer — Express Backend

### 3.1 Server Bootstrap

```typescript
// src/dashboard/api-server.ts
import express from 'express';
import path from 'path';
import { coverageRoutes } from './routes/coverage.js';
import { gapRoutes } from './routes/gaps.js';
import { stubRoutes } from './routes/stubs.js';
import { reviewRoutes } from './routes/review.js';
import { reportRoutes } from './routes/reports.js';
import { sseRoutes } from './routes/sse.js';
import { decommissionRoutes } from './routes/decommission.js';

const app = express();
app.use(express.json());

// Serve dashboard SPA
app.use(express.static(path.join(process.cwd(), 'dashboard/dist')));

// API routes
app.use('/api/coverage',      coverageRoutes);
app.use('/api/gaps',          gapRoutes);
app.use('/api/stubs',         stubRoutes);
app.use('/api/review',        reviewRoutes);
app.use('/api/reports',       reportRoutes);
app.use('/api/decommission',  decommissionRoutes);
app.use('/api/events',        sseRoutes);

// SPA fallback
app.get('*', (_, res) =>
    res.sendFile(path.join(process.cwd(), 'dashboard/dist/index.html'))
);

const PORT = process.env.SENTINEL_DASHBOARD_PORT ?? 3001;
app.listen(PORT, () => console.log(`[Sentinel] Dashboard: http://localhost:${PORT}`));
```

### 3.2 Complete API Endpoints

#### Coverage Endpoints

|Method|Path                        |Description               |Response                                                    |
|------|----------------------------|--------------------------|------------------------------------------------------------|
|GET   |`/api/coverage/summary`     |Layer-by-layer coverage % |`{structural, semantic?, dependency, behavioral, composite}`|
|GET   |`/api/coverage/programs`    |Per-program coverage table|Array of program records                                    |
|GET   |`/api/coverage/programs/:id`|Single program detail     |Full program coverage object                                |
|GET   |`/api/coverage/trend`       |Coverage % over time      |Array of `{date, structural, dependency, behavioral}`       |
|GET   |`/api/coverage/nonstop`     |NonStop construct coverage|`{enterTal, enterC, guardianIo, tmf, pathway}`              |
|POST  |`/api/coverage/refresh`     |Refresh materialized views|`{ok, duration_ms}`                                         |

#### Gap Endpoints

|Method|Path                          |Description                                  |
|------|------------------------------|---------------------------------------------|
|GET   |`/api/gaps`                   |Paginated gap list with filters              |
|GET   |`/api/gaps/:id`               |Full gap detail with source code             |
|GET   |`/api/gaps/program/:programId`|All gaps for a program                       |
|GET   |`/api/gaps/stats`             |Aggregate stats (total, by severity, by type)|
|PATCH |`/api/gaps/:id/resolve`       |Mark gap as resolved                         |
|PATCH |`/api/gaps/:id/escalate`      |Escalate to SME                              |
|PATCH |`/api/gaps/:id/flag-dead-code`|Flag as dead code                            |

#### Stub Endpoints

|Method|Path                           |Description                        |
|------|-------------------------------|-----------------------------------|
|GET   |`/api/stubs`                   |All detected stubs with filters    |
|GET   |`/api/stubs/:id`               |Full stub detail                   |
|GET   |`/api/stubs/program/:programId`|Stubs by program                   |
|GET   |`/api/stubs/stats`             |Stub counts by severity, confidence|
|PATCH |`/api/stubs/:id/confirm`       |Confirm as real stub, needs fixing |
|PATCH |`/api/stubs/:id/dismiss`       |Dismiss as false positive          |
|POST  |`/api/stubs/rescan`            |Re-run stub detection on program   |

#### Review Endpoints

|Method|Path                                   |Description                         |
|------|---------------------------------------|------------------------------------|
|GET   |`/api/review/queue`                    |Items pending review (Tier 2)       |
|GET   |`/api/review/queue/count`              |Count of pending items              |
|POST  |`/api/review/mapping/:id/confirm`      |Confirm a traceability mapping      |
|POST  |`/api/review/mapping/:id/reject`       |Reject a mapping                    |
|POST  |`/api/review/mapping/:id/reassign`     |Reassign to different Java construct|
|GET   |`/api/review/log/:entityType/:entityId`|Audit log for an entity             |
|GET   |`/api/review/log/recent`               |Recent review actions (all)         |

#### Decommission Endpoints

|Method|Path                                      |Description                          |
|------|------------------------------------------|-------------------------------------|
|GET   |`/api/decommission`                       |All programs with decommission status|
|GET   |`/api/decommission/:programId`            |Gate checklist for a program         |
|POST  |`/api/decommission/:programId/signoff`    |Record a sign-off                    |
|GET   |`/api/decommission/:programId/certificate`|Generate PDF certificate             |

#### Report Endpoints

|Method|Path                      |Description                   |
|------|--------------------------|------------------------------|
|GET   |`/api/reports/executive`  |Generate executive summary PDF|
|GET   |`/api/reports/audit-trail`|Download audit trail CSV      |
|GET   |`/api/reports/gaps`       |Generate gap report PDF       |
|GET   |`/api/reports/behavioral` |Generate behavioral report PDF|
|GET   |`/api/reports/stubs`      |Generate stub report PDF      |

### 3.3 Query Parameter Standards

All list endpoints accept:

```
GET /api/gaps?
    program=CALC-INTEREST
    &severity=critical,major
    &gapType=missing_conversion,stub_conversion
    &side=cobol_unmapped
    &isEnterTal=true
    &unresolvedOnly=true
    &search=COMPUTE
    &sortBy=loc
    &sortDir=desc
    &page=1
    &pageSize=25
```

### 3.4 Response Envelope

All endpoints return:

```json
{
  "ok": true,
  "data": { ... },
  "meta": {
    "total": 42,
    "page": 1,
    "pageSize": 25,
    "computedAt": "2026-04-15T09:00:00Z"
  }
}
```

Errors:

```json
{
  "ok": false,
  "error": "Program not found",
  "code": "NOT_FOUND"
}
```

-----

## 4. Dashboard Views

The dashboard has five primary views, accessible via the top navigation:

```
┌──────────────────────────────────────────────────────────┐
│  SENTINEL                               [Refresh] [Export]│
├────────────┬────────────┬────────────┬────────────┬──────┤
│  OVERVIEW  │    GAPS    │   STUBS    │  PROGRAMS  │ SIGN │
│            │            │  🔴 14     │            │  OFF │
└────────────┴────────────┴────────────┴────────────┴──────┘
```

### 4.1 Overview View

**Purpose:** Executive-level health at a glance. First thing you see on load.

**Layout:**

```
┌────────────────────────────────────────────────────────┐
│  COMPOSITE SCORE                                        │
│  ████████████████████░░░░  76.4%   AMBER               │
│                                                         │
│  Structural  ████████░  78.4%      Dependency ██████░ 65.3% │
│  Behavioral  █████████  81.2%      Semantic   ███████ 72.1% │
│  (optional)                                             │
└────────────────────────────────────────────────────────┘

┌────────────┬────────────┬────────────┬────────────┐
│  18 Gaps   │ 14 Stubs   │ 29 Failing │ 4 Ready to │
│  Critical  │  Detected  │   Tests    │ Decommission│
└────────────┴────────────┴────────────┴────────────┘

┌────────────────────────────────────┐  ┌──────────────────┐
│  Coverage Trend (8 weeks)          │  │  NonStop Status  │
│  100% ┤                            │  │  TAL/C  8/11 ✓   │
│   80% ┤         ▗▄▄▄▄              │  │  Guardian 5/7 ✓  │
│   60% ┤   ▗▄▄▄▀▀                   │  │  TMF    2/3 ✓    │
│   40% ┤▄▄▀                         │  │  SQL/MP 6/6 ✓    │
│       └──────────────────          │  └──────────────────┘
└────────────────────────────────────┘
```

**Data sources:**

- Composite score: weighted formula from `coverage_summary` view
- Trend: snapshot history from `conversion_baseline`
- NonStop status: `nonstop_coverage` view

### 4.2 Gaps View

**Purpose:** Primary working view for engineers and tech lead. Full filterable, sortable gap list.

**Layout:**

```
┌──────────────────────────────────────────────────────┐
│  [Program ▼] [Severity ▼] [Type ▼] [TAL/C] [Open]   │
│  🔍 Search constructs...              18 gaps shown   │
├──────────────────────────────────────────────────────┤
│  ● CRITICAL  CALC-INTEREST                            │
│  300-COMPUTE-DAILY-RATE    87 LOC  C:12  62% sim     │
│  Missing conversion · COBOL→Java  ▸                  │
├──────────────────────────────────────────────────────┤
│  ◉ CRITICAL  CALC-INTEREST             TAL/C          │
│  ENTER-TAL-RATE-LOOKUP     12 LOC  C:3   0% sim     │
│  Missing conversion · ENTER TAL  ▸                   │
├──────────────────────────────────────────────────────┤
│  ● MAJOR     ACCT-MAINT                               │
│  510-ENSCRIBE-UPDATE       38 LOC  C:5   0% sim     │
│  Missing conversion  ▸                               │
└──────────────────────────────────────────────────────┘
```

Tap any row → expands to full [Gap Detail Panel](#5-gap-detail-panel).

**Sort options:** Severity, LOC, Complexity, Similarity score, Program, Type

**Filter chips (active filters shown as removable pills above list):**

```
Program: CALC-INTEREST  ×    Severity: Critical  ×    TAL/C only  ×
```

### 4.3 Stubs View

**Purpose:** Shows all Java methods detected as stubs — mapped but not implemented.

**Layout:**

```
┌─────────────────────────────────────────────────────┐
│  14 stubs detected  ·  9 critical  ·  4 major  ·  1 minor  │
│  [Program ▼] [Severity ▼] [Confidence ▼]            │
├─────────────────────────────────────────────────────┤
│  ● CRITICAL  91%  PAYMENT-PROC                       │
│  PaymentValidator.validateAmount()                   │
│  Signals: missing_arithmetic, missing_branching,     │
│  return_false, unused_parameters  ▸                  │
├─────────────────────────────────────────────────────┤
│  ● CRITICAL  88%  CALC-INTEREST                      │
│  CalcInterest.calculateDailyRate()                   │
│  Signals: return_bigdecimal_zero, missing_branching, │
│  extreme_loc_mismatch, missing_arithmetic  ▸         │
└─────────────────────────────────────────────────────┘
```

Each stub row expands to show [Stub Detail Panel](#7-stub-detection-view).

### 4.4 Programs View

**Purpose:** Per-program health dashboard for program-level sign-off tracking.

**Layout:**

```
┌──────────────────────────────────────────────────────┐
│  7 programs  ·  0 ready  ·  5 in-progress  ·  2 blocked │
├──────────────────────────────────────────────────────┤
│  CALC-INTEREST                              78% AMBER │
│  ████████████████░░░░░░  42 constructs                │
│  3 critical gaps  ·  2 stubs  ·  5 failing tests     │
│  Structural ██░ 71%  Dependency █░ 60%  Behavioral ██████ 80% │
├──────────────────────────────────────────────────────┤
│  PAYMENT-PROC                               82% AMBER │
│  ████████████████████░░  55 constructs               │
│  2 critical gaps  ·  1 stub  ·  10 failing tests     │
└──────────────────────────────────────────────────────┘
```

Tap program → drill into [Program Detail](#program-detail-modal).

### 4.5 Sign-Off View

**Purpose:** Decommission gate status and sign-off management for Legal and management.

**Layout:**

```
┌──────────────────────────────────────────────────────┐
│  Decommission Gate Status                            │
│                                                      │
│  Program         Struct  Dep  Behav  TL  BO  Legal  │
│  ─────────────────────────────────────────────────  │
│  FX-CONVERT      100%   100% 100%   ✓   ✓   —       │
│  CUST-LOOKUP     100%    98%  95%   ✓   —   —       │
│  CALC-INTEREST    78%    65%  80%   —   —   —       │
│  PAYMENT-PROC     85%    70%  72%   —   —   —       │
└──────────────────────────────────────────────────────┘
```

Tap program → opens sign-off checklist modal.

-----

## 5. Gap Detail Panel

When a gap row is tapped, it expands in-place with collapsible sections. Each section is independently toggled.

### 5.1 Panel Sections (in order)

```
▼ SOURCE CODE
▼ DATA ITEMS
▼ SIMILARITY ANALYSIS
▼ GRAPH / DEPENDENCY ISSUES
▼ BEHAVIORAL TEST RESULTS
▼ ENTER TAL/C DETAIL          (only if gap.isEnterTal)
▼ VERB ANALYSIS
▼ AUDIT LOG
  [Action buttons]
```

### 5.2 Source Code Section

Shows COBOL source and nearest Java match side by side.

**COBOL side:**

- Header: program name + guardian file path
- Line range shown in corner
- Syntax colour: keywords in blue, data names in teal, literals in amber

**Java side:**

- Header: fully-qualified class.method + line number
- Link to file path
- Shows actual Java code from `java_constructs.raw_source`
- If no match: “No Java equivalent found in codebase”

**Visual diff mode (toggle):** When both sides present, show a semantic diff highlighting which COBOL branches exist in Java and which are missing.

### 5.3 Data Items Section

Table of COBOL data items used by this paragraph:

```
NAME                 LEVEL  PIC              USAGE        RISK
─────────────────────────────────────────────────────────────
WS-DAILY-RATE          05  S9(5)V9(8)       COMP-3       ⚠ Precision
WS-PRINCIPAL           05  S9(11)V9(2)      COMP-3       ⚠ Precision
WS-ANNUAL-RATE         05  S9(3)V9(6)       COMP-3       ⚠ Precision
WS-DAY-COUNT           05  9(3)             DISPLAY      —
ACCT-FILE              FD  —                Guardian     ⚠ NonStop IO
```

COMP-3 items are highlighted in purple with precision warning. Guardian file items in blue with NonStop IO warning.

### 5.4 Similarity Analysis Section

Only shown when a nearest Java match exists:

```
Nearest match: com.bank.interest.CalcInterest.calculateRate()
File: src/main/java/com/bank/interest/CalcInterest.java:142

Cosine Similarity:   62%   ████████████░░░░░░░░░░
                     0%      70% review    85% auto   100%

Embedding model: snowflake-arctic-embed-l-v2.0
COBOL summary embedded:  "Performs 4 arithmetic computations..."
Java summary embedded:   "Calculates rate by dividing annual rate by 360..."
```

When embeddings are disabled (flag off), show: “Semantic similarity not available — embeddings disabled. Enable SENTINEL_ENABLE_EMBEDDINGS to activate.”

### 5.5 Dependency Issues Section

List of broken chain findings from `dependency_validation_results`:

```
✗  PERFORMS 310-CALC-30-360 → no Java equivalent found
     COBOL: 300-COMPUTE-DAILY-RATE PERFORMS 310-CALC-30-360
     Java:  CalcInterest.calculateRate() has no call to equivalent

✗  Fan-out mismatch: COBOL calls 3 sub-paragraphs, Java calls 1 method
     Expected JAVA_CALLS edges: ~3
     Found JAVA_CALLS edges: 1

⚠  310-CALC-30-360 is mapped (confidence=low) but not in Java call chain
```

### 5.6 Behavioral Test Results Section

```
Total   Pass   Fail   Not Run
  12       5      7        0

[Show failing tests ▾]

  ✗  test_act360_rate_20pct      Expected: 0.00055556  Got: 0.00000000
  ✗  test_act365_rate_5pct       Expected: 0.00013699  Got: 0.00000000
  ✗  test_30_360_boundary        Not run — ENTER TAL blocked
```

### 5.7 ENTER TAL/C Detail Section (NonStop only)

Only rendered when `gap.isEnterTal = true`:

```
TAL Routine: RATE_TABLE_LOOKUP
Source File: $DATA1.TALLIB.RATELKP

Parameters:
  rate_table_ptr    INT:ref       (output — rate result)
  currency_code     CHAR:8        (input)
  effective_date    INT:32        (input — Julian date)
  rate_result       FIXED(8):ref  (output)
  return_status     INT:ref       (output — 0=ok, non-zero=error)

Required actions:
  1. Capture I/O pairs from NonStop system (golden files)
  2. Implement Java equivalent or JNI wrapper
  3. Verify parity against golden files
  4. Add REPLACED_BY edge in graph_edges

Status: ○ Not started
```

Status cycle: Not started → Golden files captured → Java implemented → Parity verified → Done.

### 5.8 Verb Analysis Section

Color-coded verb frequency chart:

```
COMPUTE  ████████  4    ← High risk (precision)
EVALUATE ████      2    ← High risk (branch coverage)
PERFORM  ██████    3    ← Medium risk (call chain)
MOVE     ████████████████  8  ← Low risk
IF       ██████    3    ← Medium risk
```

### 5.9 Audit Log Section

Timeline view of all actions taken on this gap:

```
○──────────────────────────────────────────────────
│  FLAGGED           2026-04-05 · System
│  Auto-detected: similarity 0.62, below threshold
│
○──────────────────────────────────────────────────
│  REVIEWED          2026-04-07 · RJ
│  Checked source comparison — ACT/365 branch missing
│
○──────────────────────────────────────────────────
│  ESCALATED         2026-04-07 · RJ
│  Escalated to SME: interest calculation domain logic
○──────────────────────────────────────────────────
```

### 5.10 Action Buttons

Context-aware based on current gap state:

**Unresolved gap:**

```
[✓ Confirm Match]  [✗ Reject Match]  [↑ Escalate to SME]  [⊘ Flag Dead Code]
```

**Rejected gap (needs manual mapping):**

```
[🔗 Assign to Java Method...]  [⊘ Flag Dead Code]  [↑ Escalate]
```

**Resolved gap:**

```
[↺ Reopen]                              Resolved by MK · 2026-04-10
```

-----

## 6. Review Workflow UI

### 6.1 Review Queue Badge

Navigation shows pending review count:

```
GAPS  ·  [17 pending review]
```

### 6.2 Review Queue View

Filtered view showing only items needing Tier 2 human review (confidence = ‘low’, similarity 0.70–0.85, or manually flagged):

```
┌────────────────────────────────────────────────────┐
│  Review Queue  ·  17 items pending                  │
│  [Assign to me]  [Filter: unassigned ▼]            │
├────────────────────────────────────────────────────┤
│  PAYMENT-PROC  ·  200-VALIDATE-AMOUNT  ·  72% sim  │
│  PaymentValidator.validateAmount()                 │
│  Similarity: 0.72  ·  Mapping: structure_match     │
│  [✓ Confirm]  [✗ Reject]  [View source ▸]         │
├────────────────────────────────────────────────────┤
│  CALC-INTEREST  ·  400-ADJ-COMPOUND  ·  74% sim   │
│  CalcInterest.adjustCompound()                     │
│  Similarity: 0.74  ·  Mapping: name_match          │
│  [✓ Confirm]  [✗ Reject]  [View source ▸]         │
└────────────────────────────────────────────────────┘
```

### 6.3 Inline Confirmation Flow

When “Confirm” is tapped on a review queue item:

```
┌─────────────────────────────────────────┐
│  Confirm Mapping                         │
│                                         │
│  COBOL:  400-ADJ-COMPOUND               │
│  Java:   CalcInterest.adjustCompound()  │
│                                         │
│  Add a note (optional):                 │
│  ┌─────────────────────────────────┐   │
│  │ Logic matches — quarterly branch│   │
│  │ not in Java but minor impact    │   │
│  └─────────────────────────────────┘   │
│                                         │
│  [Cancel]          [Confirm Mapping]    │
└─────────────────────────────────────────┘
```

On confirm: POST `/api/review/mapping/:id/confirm` → row updates in place, item removed from queue, `review_log` entry created.

### 6.4 Reject Flow

When “Reject” is tapped:

```
┌─────────────────────────────────────────┐
│  Reject Mapping                          │
│                                         │
│  Why are you rejecting this mapping?    │
│  ○ Wrong method — different semantics   │
│  ○ Wrong class entirely                 │
│  ○ This is dead code                    │
│  ○ Manual assignment needed             │
│  ○ Other (specify below)               │
│                                         │
│  Notes:                                 │
│  ┌─────────────────────────────────┐   │
│  └─────────────────────────────────┘   │
│                                         │
│  [Cancel]          [Reject Mapping]     │
└─────────────────────────────────────────┘
```

### 6.5 Manual Assignment Flow

When a mapping is rejected with “Manual assignment needed”:

```
┌──────────────────────────────────────────────────────┐
│  Assign Java Equivalent                               │
│  COBOL: 400-ADJ-COMPOUND                             │
│                                                      │
│  Search Java methods:                                │
│  ┌────────────────────────────────────────────────┐ │
│  │ adjustCompound                                  │ │
│  └────────────────────────────────────────────────┘ │
│                                                      │
│  Results:                                            │
│  ○  CalcInterest.adjustCompound()        74% match  │
│  ○  InterestHelper.applyCompounding()   61% match   │
│  ○  [Enter manually...]                              │
│                                                      │
│  [Cancel]               [Assign Selected]            │
└──────────────────────────────────────────────────────┘
```

-----

## 7. Stub Detection View

### 7.1 Stub List

```
┌─────────────────────────────────────────────────────┐
│  14 stubs detected  ·  ● 9 critical  ● 4 major  ● 1 minor │
│  [Re-scan all]  [Export stub report]                │
├─────────────────────────────────────────────────────┤
│  ● CRITICAL  91%  PAYMENT-PROC                       │
│  PaymentValidator.validateAmount()                   │
│  4 signals: missing_arithmetic, missing_branching,  │
│  return_false, unused_parameters                    │
│  COBOL: 200-VALIDATE-AMOUNT  ·  65 LOC  ·  C:9      │
│  [Confirm stub]  [Dismiss]  [View detail ▸]         │
└─────────────────────────────────────────────────────┘
```

### 7.2 Stub Detail Panel

Expands from the list row:

**Signal breakdown table:**

```
Signal                      Weight  Evidence
─────────────────────────────────────────────────────
return_false                   45%  return false;
missing_arithmetic             55%  COBOL COMPUTE×2, Java has no arithmetic
missing_branching              50%  COBOL EVALUATE×1, Java has no if/switch
unused_parameters              50%  amount, payType never used in body
comment_heavy_minimal_code     45%  3 comment lines vs 1 code line
extreme_loc_mismatch           55%  Java 6 LOC vs COBOL 65 LOC (9.2%)
─────────────────────────────────────────────────────
Total confidence:              91%  (capped at 100%)
```

**Source comparison:**

```
COBOL — 200-VALIDATE-AMOUNT (PAYMENT-PROC)        Java — validateAmount()
Lines 145–209                                     Line 88

EVALUATE WS-PAY-TYPE                              public boolean validateAmount(
  WHEN "WIRE"                                         BigDecimal amount,
    MOVE 1000000.00 TO WS-MAX-LIMIT                   String payType) {
    MOVE 0.0015 TO WS-FEE-PCT                     // Validates payment amount
  WHEN "ACH"                                         // against type limits
    MOVE 100000.00 TO WS-MAX-LIMIT                    return false;
    MOVE 0.0005 TO WS-FEE-PCT                     }
  WHEN "CHECK"
    MOVE 50000.00 TO WS-MAX-LIMIT
    MOVE ZEROS TO WS-FEE-PCT
  WHEN OTHER
    PERFORM 900-ERROR-HANDLER
END-EVALUATE.

IF WS-PAY-AMOUNT > WS-MAX-LIMIT
  PERFORM 900-ERROR-HANDLER
ELSE
  COMPUTE WS-FEE-AMOUNT ROUNDED =
    WS-PAY-AMOUNT * WS-FEE-PCT
END-IF.
```

**Recommended fix (LLM-generated, on demand):**

```
[Generate implementation hint ▶]

Based on the COBOL source, the Java method should:
1. Switch on payType: WIRE (limit $1M, fee 0.15%), ACH ($100K, fee
   0.05%), CHECK ($50K, no fee)
2. Return false + set error if amount exceeds limit
3. Compute fee = amount × fee_pct and store in result
4. Use BigDecimal throughout — COBOL used COMP-3 S9(11)V9(2)

Suggested signature:
  public ValidationResult validateAmount(BigDecimal amount, String payType)
```

**Actions:**

```
[Confirm as stub]  [Dismiss (false positive)]  [Escalate to SME]
```

On “Confirm”: creates entry in `coverage_gaps` with `gap_type='stub_conversion'`, `severity='critical'`.

On “Dismiss”: sets `is_stub=false`, `stub_confidence` kept for audit, `review_log` entry created.

-----

## 8. Reporting & Export

### 8.1 Report Generation Panel

Accessible from any view via the Export button in the header:

```
┌──────────────────────────────────────┐
│  Generate Report                      │
│                                      │
│  Type:                               │
│  ○ Executive Summary (PDF)           │
│  ○ Audit Trail (CSV)                 │
│  ○ Gap Report (PDF)                  │
│  ○ Stub Report (PDF)                 │
│  ○ Behavioral Report (PDF)           │
│  ○ Decommission Certificate (PDF)    │
│                                      │
│  Program: [All programs ▼]           │
│  Date range: [Last 30 days ▼]        │
│                                      │
│  [Cancel]       [Generate & Download]│
└──────────────────────────────────────┘
```

Generation calls `GET /api/reports/{type}?program=...` which:

1. Queries Postgres for current data
1. Renders Markdown template
1. Converts to PDF via Pandoc: `pandoc report.md -o report.pdf --pdf-engine=wkhtmltopdf`
1. Returns file as download

### 8.2 Inline Coverage Badge

Shareable badge showing current composite score. Embed in internal wiki:

```
GET /api/coverage/badge  →  SVG image
```

-----

## 9. Component Structure

```
dashboard/
├── index.html                       # Entry point
├── src/
│   ├── main.jsx                     # React root
│   ├── App.jsx                      # Router + layout
│   │
│   ├── components/
│   │   ├── layout/
│   │   │   ├── Header.jsx           # Nav + score + export button
│   │   │   ├── NavTabs.jsx          # Primary navigation tabs
│   │   │   └── StatusBar.jsx        # Pipeline last-run status
│   │   │
│   │   ├── coverage/
│   │   │   ├── CompositeScore.jsx   # Big number + RAG indicator
│   │   │   ├── LayerBars.jsx        # 4 progress bars
│   │   │   ├── CoverageTrend.jsx    # Line chart (Recharts)
│   │   │   └── NonstopStatus.jsx    # TAL/C, Guardian, TMF status
│   │   │
│   │   ├── gaps/
│   │   │   ├── GapList.jsx          # Filterable gap list
│   │   │   ├── GapRow.jsx           # Collapsed gap row
│   │   │   ├── GapDetail.jsx        # Expanded detail panel
│   │   │   ├── GapFilters.jsx       # Filter bar + active pills
│   │   │   ├── GapStats.jsx         # Stat cards (total, critical, etc.)
│   │   │   └── sections/
│   │   │       ├── SourceCode.jsx   # Side-by-side COBOL/Java
│   │   │       ├── DataItems.jsx    # Data item table
│   │   │       ├── Similarity.jsx   # Score bar + model info
│   │   │       ├── GraphIssues.jsx  # Broken chains list
│   │   │       ├── BehavioralTests.jsx  # Pass/fail counts
│   │   │       ├── EnterTalDetail.jsx   # TAL/C routine panel
│   │   │       ├── VerbAnalysis.jsx     # Verb bar chart
│   │   │       └── AuditLog.jsx         # Timeline
│   │   │
│   │   ├── stubs/
│   │   │   ├── StubList.jsx         # Stub list with confidence
│   │   │   ├── StubRow.jsx          # Collapsed stub row
│   │   │   ├── StubDetail.jsx       # Signal breakdown + source
│   │   │   └── StubStats.jsx        # Summary counts
│   │   │
│   │   ├── programs/
│   │   │   ├── ProgramList.jsx      # Program cards
│   │   │   ├── ProgramCard.jsx      # Per-program health card
│   │   │   └── ProgramDetail.jsx    # Drill-down modal
│   │   │
│   │   ├── review/
│   │   │   ├── ReviewQueue.jsx      # Pending items list
│   │   │   ├── ConfirmModal.jsx     # Confirm mapping dialog
│   │   │   ├── RejectModal.jsx      # Reject reason dialog
│   │   │   └── AssignModal.jsx      # Manual assignment search
│   │   │
│   │   ├── decommission/
│   │   │   ├── DecommissionTable.jsx  # Gate status table
│   │   │   ├── GateChecklist.jsx      # Per-program checklist
│   │   │   └── SignoffModal.jsx       # Record sign-off dialog
│   │   │
│   │   └── shared/
│   │       ├── SectionToggle.jsx    # Collapsible section header
│   │       ├── SeverityBadge.jsx    # Critical/Major/Minor pill
│   │       ├── ProgressBar.jsx      # Reusable progress bar
│   │       ├── CodeBlock.jsx        # Syntax-highlighted code
│   │       ├── LoadingSpinner.jsx
│   │       ├── ErrorBoundary.jsx
│   │       └── Toast.jsx            # Action feedback
│   │
│   ├── hooks/
│   │   ├── useApi.js                # Fetch wrapper with loading/error
│   │   ├── useGaps.js               # Gap list with filters
│   │   ├── useStubs.js              # Stub list
│   │   ├── useCoverage.js           # Coverage metrics
│   │   ├── useReviewQueue.js        # Review queue
│   │   └── useSSE.js                # Server-Sent Events listener
│   │
│   └── utils/
│       ├── api.js                   # Base fetch with envelope
│       ├── severity.js              # Color/label helpers
│       ├── format.js                # LOC, %, date formatting
│       └── filters.js               # Filter param serialization
│
└── public/
    └── favicon.ico
```

-----

## 10. Database Queries — Complete Reference

### 10.1 Coverage Summary

```sql
-- /api/coverage/summary
SELECT
    layer,
    ROUND(coverage_pct, 1) AS coverage_pct,
    gap_count
FROM sentinel.coverage_summary;
```

### 10.2 Per-Program Coverage

```sql
-- /api/coverage/programs
SELECT
    pc.program_id,
    pc.guardian_file_name,
    pc.total_constructs,
    pc.total_paragraphs,
    pc.enter_tal_c_count,
    pc.high_confidence,
    pc.medium_confidence,
    pc.critical_gaps,
    pc.major_gaps,
    ROUND(pc.structural_coverage_pct, 1) AS structural_coverage_pct,
    -- Stub count
    COUNT(jc.id) FILTER (WHERE jc.is_stub = TRUE) AS stub_count,
    -- Behavioral pass rate
    ROUND(
        COUNT(br.id) FILTER (WHERE br.match = TRUE)::NUMERIC
        / NULLIF(COUNT(br.id), 0) * 100, 1
    ) AS behavioral_pass_pct,
    -- Decommission status
    COALESCE(dl.status, 'pending') AS decommission_status
FROM sentinel.program_coverage pc
LEFT JOIN sentinel.traceability_map tm ON TRUE  -- join for stub count
LEFT JOIN sentinel.java_constructs jc ON jc.id = tm.java_construct_id
    AND jc.class_name ILIKE '%' || pc.program_id || '%'
LEFT JOIN sentinel.behavioral_results br
    ON br.cobol_program = pc.program_id
    AND br.run_id = (
        SELECT run_id FROM sentinel.behavioral_results ORDER BY run_at DESC LIMIT 1
    )
LEFT JOIN sentinel.decommission_log dl ON dl.cobol_program = pc.program_id
GROUP BY
    pc.program_id, pc.guardian_file_name, pc.total_constructs,
    pc.total_paragraphs, pc.enter_tal_c_count, pc.high_confidence,
    pc.medium_confidence, pc.critical_gaps, pc.major_gaps,
    pc.structural_coverage_pct, dl.status
ORDER BY pc.critical_gaps DESC, pc.structural_coverage_pct ASC;
```

### 10.3 Gap List with Filters

```sql
-- /api/gaps  (parameterized)
SELECT
    cg.id,
    cg.side,
    cg.construct_type,
    cg.construct_name,
    cg.program_or_class,
    cg.loc,
    cg.complexity,
    cg.gap_type,
    cg.severity,
    cg.nearest_match_name,
    cg.nearest_similarity,
    cg.is_enter_tal,
    cg.is_enter_c,
    cg.is_guardian_io,
    cg.is_tmf,
    cg.resolution,
    cg.resolved_by,
    cg.resolved_at,
    -- COBOL source (brief)
    LEFT(cc.raw_source, 200) AS cobol_source_preview,
    -- Java source (brief)
    LEFT(jc.raw_source, 200) AS java_source_preview,
    -- Review count
    COUNT(rl.id) AS review_actions
FROM sentinel.coverage_gaps cg
LEFT JOIN sentinel.cobol_constructs cc ON cc.id = cg.construct_id
    AND cg.side = 'cobol_unmapped'
LEFT JOIN sentinel.traceability_map tm ON tm.cobol_construct_id = cg.construct_id
LEFT JOIN sentinel.java_constructs jc ON jc.id = tm.java_construct_id
LEFT JOIN sentinel.review_log rl ON rl.entity_type = 'gap'
    AND rl.entity_id = cg.id
WHERE
    ($1::TEXT IS NULL OR cg.program_or_class = $1)           -- program filter
    AND ($2::TEXT[] IS NULL OR cg.severity = ANY($2::TEXT[]))  -- severity filter
    AND ($3::TEXT[] IS NULL OR cg.gap_type = ANY($3::TEXT[]))  -- type filter
    AND ($4::TEXT IS NULL OR cg.side = $4)                    -- side filter
    AND ($5::BOOLEAN IS NULL OR cg.is_enter_tal = $5)         -- tal filter
    AND ($6::BOOLEAN IS NULL OR cg.is_enter_c = $6)           -- enter_c filter
    AND ($7::BOOLEAN IS NULL OR (cg.resolution IS NULL) = $7) -- unresolved filter
    AND ($8::TEXT IS NULL OR
         cg.construct_name ILIKE '%' || $8 || '%'
         OR cg.program_or_class ILIKE '%' || $8 || '%')       -- search filter
GROUP BY
    cg.id, cc.raw_source, jc.raw_source
ORDER BY
    CASE cg.severity WHEN 'critical' THEN 1 WHEN 'major' THEN 2 ELSE 3 END,
    cg.loc DESC
LIMIT $9 OFFSET $10;
```

### 10.4 Full Gap Detail

```sql
-- /api/gaps/:id
SELECT
    cg.*,
    cc.raw_source        AS cobol_source,
    cc.asg_summary,
    cc.metadata          AS cobol_metadata,
    cc.nonstop_metadata,
    cc.start_line        AS cobol_start_line,
    cc.end_line          AS cobol_end_line,
    cc.parent_construct_id,
    jc.raw_source        AS java_source,
    jc.pseudocode_summary,
    jc.file_path         AS java_file_path,
    jc.class_name,
    jc.signature         AS java_signature,
    tm.mapping_type,
    tm.confidence        AS mapping_confidence,
    tm.similarity_score,
    tm.mapping_detail,
    tm.verified_by,
    tm.verified_at,
    -- Data items from ASG
    (
        SELECT json_agg(di ORDER BY di->>'level', di->>'name')
        FROM jsonb_array_elements(cc.metadata->'dataItems') di
    ) AS data_items,
    -- Graph issues
    (
        SELECT json_agg(
            json_build_object('type', dvr.validation_type, 'detail', dvr.detail)
        )
        FROM sentinel.dependency_validation_results dvr
        WHERE dvr.cobol_source = cg.construct_name
        AND dvr.status = 'fail'
        AND dvr.run_id = (
            SELECT run_id FROM sentinel.dependency_validation_results ORDER BY run_at DESC LIMIT 1
        )
    ) AS graph_issues,
    -- Behavioral test results
    (
        SELECT json_build_object(
            'total',   COUNT(*),
            'pass',    COUNT(*) FILTER (WHERE br.match = TRUE),
            'fail',    COUNT(*) FILTER (WHERE br.match = FALSE),
            'notRun',  COUNT(*) FILTER (WHERE br.cobol_program IS NULL)
        )
        FROM sentinel.behavioral_results br
        JOIN sentinel.test_cases tc ON tc.id = br.test_case_id
        WHERE tc.cobol_program = cg.program_or_class
        AND br.run_id = (
            SELECT run_id FROM sentinel.behavioral_results ORDER BY run_at DESC LIMIT 1
        )
    ) AS behavioral_results,
    -- Audit log
    (
        SELECT json_agg(
            json_build_object(
                'action', rl.action, 'by', rl.performed_by,
                'at', rl.created_at, 'notes', rl.notes
            ) ORDER BY rl.created_at
        )
        FROM sentinel.review_log rl
        WHERE rl.entity_type = 'gap' AND rl.entity_id = cg.id
    ) AS review_history
FROM sentinel.coverage_gaps cg
LEFT JOIN sentinel.cobol_constructs cc ON cc.id = cg.construct_id
    AND cg.side = 'cobol_unmapped'
LEFT JOIN sentinel.traceability_map tm ON tm.cobol_construct_id = cg.construct_id
LEFT JOIN sentinel.java_constructs jc ON jc.id = tm.java_construct_id
WHERE cg.id = $1;
```

### 10.5 Review Queue

```sql
-- /api/review/queue
SELECT
    tm.id           AS mapping_id,
    cc.construct_name AS cobol_name,
    cc.program_id,
    cc.loc,
    cc.complexity,
    jc.construct_name AS java_name,
    jc.class_name,
    jc.file_path,
    tm.mapping_type,
    tm.confidence,
    tm.similarity_score,
    tm.created_at
FROM sentinel.traceability_map tm
JOIN sentinel.cobol_constructs cc ON cc.id = tm.cobol_construct_id
JOIN sentinel.java_constructs jc ON jc.id = tm.java_construct_id
WHERE tm.confidence IN ('low', 'medium')
AND tm.verified_at IS NULL
ORDER BY
    CASE tm.confidence WHEN 'low' THEN 1 ELSE 2 END,
    cc.loc DESC;
```

### 10.6 Stub List

```sql
-- /api/stubs
SELECT
    jc.id,
    jc.construct_name,
    jc.class_name,
    jc.file_path,
    jc.loc           AS java_loc,
    jc.stub_confidence,
    jc.stub_severity,
    jc.stub_signals,
    jc.stub_summary,
    jc.raw_source,
    cc.construct_name AS cobol_name,
    cc.program_id,
    cc.loc           AS cobol_loc,
    cc.complexity    AS cobol_complexity,
    cc.metadata      AS cobol_metadata,
    cc.raw_source    AS cobol_source,
    tm.confidence    AS mapping_confidence
FROM sentinel.java_constructs jc
JOIN sentinel.traceability_map tm ON tm.java_construct_id = jc.id
JOIN sentinel.cobol_constructs cc ON cc.id = tm.cobol_construct_id
WHERE jc.is_stub = TRUE
ORDER BY jc.stub_confidence DESC, cc.loc DESC;
```

### 10.7 Decommission Gate Status

```sql
-- /api/decommission/:programId
WITH program_metrics AS (
    SELECT
        cc.program_id,
        COUNT(DISTINCT cc.id) AS total,
        COUNT(DISTINCT cc.id) FILTER (
            WHERE cc.construct_type IN ('paragraph', 'section', 'enter_tal', 'enter_c')
            AND cc.id IN (
                SELECT cobol_construct_id FROM sentinel.traceability_map
                WHERE confidence IN ('high', 'medium') AND verified_at IS NOT NULL
            )
        ) AS fully_verified
    FROM sentinel.cobol_constructs cc
    WHERE cc.program_id = $1
    GROUP BY cc.program_id
)
SELECT
    pm.program_id,
    pm.total,
    pm.fully_verified,
    ROUND(pm.fully_verified::NUMERIC / NULLIF(pm.total, 0) * 100, 1) AS structural_pct,
    (SELECT ROUND(AVG(similarity_score) * 100, 1)
     FROM sentinel.traceability_map tm
     JOIN sentinel.cobol_constructs cc ON cc.id = tm.cobol_construct_id
     WHERE cc.program_id = $1) AS semantic_avg,
    (SELECT COUNT(*) FROM sentinel.dependency_validation_results
     WHERE cobol_source LIKE $1 || '%' AND status = 'fail'
     AND run_id = (SELECT run_id FROM sentinel.dependency_validation_results
                   ORDER BY run_at DESC LIMIT 1)) AS broken_chains,
    (SELECT ROUND(COUNT(*) FILTER (WHERE match = TRUE)::NUMERIC
                  / NULLIF(COUNT(*), 0) * 100, 1)
     FROM sentinel.behavioral_results WHERE cobol_program = $1
     AND run_id = (SELECT run_id FROM sentinel.behavioral_results
                   ORDER BY run_at DESC LIMIT 1)) AS behavioral_pct,
    (SELECT COUNT(*) FROM sentinel.java_constructs jc
     JOIN sentinel.traceability_map tm ON tm.java_construct_id = jc.id
     JOIN sentinel.cobol_constructs cc ON cc.id = tm.cobol_construct_id
     WHERE cc.program_id = $1 AND jc.is_stub = TRUE) AS stub_count,
    dl.tech_lead_signoff,
    dl.tech_lead_signoff_at,
    dl.business_owner_signoff,
    dl.business_owner_signoff_at,
    dl.legal_signoff,
    dl.legal_signoff_at,
    dl.status
FROM program_metrics pm
LEFT JOIN sentinel.decommission_log dl ON dl.cobol_program = $1;
```

-----

## 11. State Management

No external state library. React hooks with local component state + custom fetch hooks:

```javascript
// hooks/useApi.js
export function useApi(url, options = {}) {
    const [data, setData] = useState(null);
    const [loading, setLoading] = useState(true);
    const [error, setError] = useState(null);
    const [refreshKey, setRefreshKey] = useState(0);

    useEffect(() => {
        if (!url) return;
        setLoading(true);
        fetch(`/api${url}`)
            .then(r => r.json())
            .then(body => { setData(body.data); setLoading(false); })
            .catch(err => { setError(err.message); setLoading(false); });
    }, [url, refreshKey]);

    const refresh = useCallback(() => setRefreshKey(k => k + 1), []);
    return { data, loading, error, refresh };
}

// hooks/useGaps.js
export function useGaps(filters) {
    const params = new URLSearchParams(serializeFilters(filters)).toString();
    return useApi(`/gaps?${params}`);
}
```

### 11.1 Filter State

Filters live in URL query params so they survive page reload and are shareable:

```javascript
// hooks/useGapFilters.js
export function useGapFilters() {
    const [searchParams, setSearchParams] = useSearchParams();

    const filters = {
        program: searchParams.get('program') ?? 'all',
        severity: searchParams.getAll('severity'),
        gapType: searchParams.getAll('gapType'),
        unresolvedOnly: searchParams.get('unresolvedOnly') !== 'false',
        talOnly: searchParams.get('talOnly') === 'true',
    };

    const setFilter = (key, value) => {
        const next = new URLSearchParams(searchParams);
        if (value === null || value === 'all') next.delete(key);
        else if (Array.isArray(value)) { next.delete(key); value.forEach(v => next.append(key, v)); }
        else next.set(key, value);
        setSearchParams(next, { replace: true });
    };

    return { filters, setFilter };
}
```

-----

## 12. Real-Time Updates

### 12.1 Postgres NOTIFY

After pipeline writes to Postgres, trigger a NOTIFY:

```sql
-- Add to pipeline write triggers
CREATE OR REPLACE FUNCTION sentinel.notify_change()
RETURNS TRIGGER AS $$
BEGIN
    PERFORM pg_notify(
        'sentinel_changes',
        json_build_object(
            'table', TG_TABLE_NAME,
            'operation', TG_OP,
            'program', COALESCE(NEW.program_id, NEW.program_or_class, NEW.cobol_program)
        )::text
    );
    RETURN NEW;
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER notify_coverage_gaps
    AFTER INSERT OR UPDATE ON sentinel.coverage_gaps
    FOR EACH ROW EXECUTE FUNCTION sentinel.notify_change();

CREATE TRIGGER notify_behavioral_results
    AFTER INSERT ON sentinel.behavioral_results
    FOR EACH ROW EXECUTE FUNCTION sentinel.notify_change();

CREATE TRIGGER notify_java_constructs
    AFTER UPDATE OF is_stub ON sentinel.java_constructs
    FOR EACH ROW EXECUTE FUNCTION sentinel.notify_change();
```

### 12.2 SSE Endpoint

```typescript
// src/dashboard/routes/sse.ts
import { Router } from 'express';
import { pool } from '../../config/database.js';

export const sseRoutes = Router();

sseRoutes.get('/', async (req, res) => {
    res.setHeader('Content-Type', 'text/event-stream');
    res.setHeader('Cache-Control', 'no-cache');
    res.setHeader('Connection', 'keep-alive');
    res.flushHeaders();

    // Dedicated connection for LISTEN
    const client = await pool.connect();
    await client.query('LISTEN sentinel_changes');

    const send = (data: object) => {
        res.write(`data: ${JSON.stringify(data)}\n\n`);
    };

    send({ type: 'connected', timestamp: new Date().toISOString() });

    client.on('notification', (msg) => {
        if (msg.channel === 'sentinel_changes') {
            send({ type: 'change', payload: JSON.parse(msg.payload ?? '{}') });
        }
    });

    // Heartbeat every 30s
    const hb = setInterval(() => res.write(': heartbeat\n\n'), 30_000);

    req.on('close', () => {
        clearInterval(hb);
        client.query('UNLISTEN sentinel_changes').finally(() => client.release());
    });
});
```

### 12.3 Browser SSE Hook

```javascript
// hooks/useSSE.js
export function useSSE(onMessage) {
    useEffect(() => {
        const es = new EventSource('/api/events');
        es.onmessage = (e) => {
            try { onMessage(JSON.parse(e.data)); }
            catch { /* ignore parse errors */ }
        };
        es.onerror = () => {
            // Reconnect automatically (browser default)
        };
        return () => es.close();
    }, []);
}

// Usage in App.jsx
useSSE((event) => {
    if (event.type === 'change') {
        // Invalidate relevant queries based on changed table
        if (['coverage_gaps', 'java_constructs'].includes(event.payload.table)) {
            gapHook.refresh();
            stubHook.refresh();
            coverageHook.refresh();
        }
    }
});
```

-----

## 13. Configuration

```bash
# .env additions for dashboard

SENTINEL_DASHBOARD_PORT=3001
SENTINEL_DASHBOARD_HOST=localhost    # bind address

# PDF generation
PANDOC_BIN=/usr/bin/pandoc
PDF_ENGINE=wkhtmltopdf               # or pdflatex, weasyprint

# Session (for review sign-off)
SENTINEL_SESSION_SECRET=<random-32-bytes>

# Report output directory
SENTINEL_REPORTS_DIR=/var/sentinel/reports
```

-----

## 14. Build & Deployment

### 14.1 Development

```bash
# Start API server with hot reload
npx tsx watch src/dashboard/api-server.ts

# Dashboard served from Express static in dev
# (no separate dev server needed)
```

### 14.2 Production

The dashboard is pure HTML + CDN-loaded React. No build step required:

```html
<!-- dashboard/index.html -->
<!DOCTYPE html>
<html>
<head>
    <title>Project Sentinel</title>
    <script src="https://unpkg.com/react@18/umd/react.production.min.js"></script>
    <script src="https://unpkg.com/react-dom@18/umd/react-dom.production.min.js"></script>
    <script src="https://unpkg.com/recharts@2/umd/Recharts.js"></script>
    <link href="https://cdn.tailwindcss.com" rel="stylesheet">
</head>
<body>
    <div id="root"></div>
    <script src="app.js" type="module"></script>
</body>
</html>
```

However, if the server has no internet access (common on RHEL in secure environments), bundle dependencies first:

```bash
# scripts/bundle-dashboard.sh
# Run this once on an internet-connected machine,
# then copy to RHEL server

npm install react react-dom recharts
mkdir -p dashboard/vendor
cp node_modules/react/umd/react.production.min.js dashboard/vendor/
cp node_modules/react-dom/umd/react-dom.production.min.js dashboard/vendor/
cp node_modules/recharts/umd/Recharts.js dashboard/vendor/
```

### 14.3 Process Management

```bash
# Run as user-space process (no sudo)
# Start
nohup node dist/dashboard/api-server.js > /var/log/sentinel/dashboard.log 2>&1 &
echo $! > /var/run/sentinel/dashboard.pid

# Stop
kill $(cat /var/run/sentinel/dashboard.pid)

# Or use PM2 in user-space
pm2 start dist/dashboard/api-server.js --name sentinel-dashboard
pm2 save
pm2 startup  # follow instructions for user-space systemd
```

### 14.4 SSH Tunnel (for remote access)

Since dashboard runs on localhost:3001, access from a developer’s laptop:

```bash
ssh -L 3001:localhost:3001 user@rhel-server
# Then open http://localhost:3001 in browser
```

-----

## 15. Testing

### 15.1 API Tests (Vitest 3)

```typescript
// tests/dashboard/coverage-api.test.ts
import { describe, it, expect, beforeAll } from 'vitest';
import request from 'supertest';
import { app } from '../../src/dashboard/api-server.js';

describe('GET /api/coverage/summary', () => {
    it('returns all required layers', async () => {
        const res = await request(app).get('/api/coverage/summary');
        expect(res.status).toBe(200);
        expect(res.body.ok).toBe(true);
        expect(res.body.data).toHaveProperty('structural');
        expect(res.body.data).toHaveProperty('dependency');
        expect(res.body.data).toHaveProperty('behavioral');
        expect(res.body.data).toHaveProperty('composite');
    });

    it('composite is weighted correctly', async () => {
        const res = await request(app).get('/api/coverage/summary');
        const { structural, dependency, behavioral, composite } = res.body.data;
        const expected = (0.25 * structural.pct + 0.30 * dependency.pct + 0.45 * behavioral.pct);
        expect(composite).toBeCloseTo(expected, 1);
    });
});

describe('GET /api/gaps', () => {
    it('respects severity filter', async () => {
        const res = await request(app).get('/api/gaps?severity=critical');
        expect(res.status).toBe(200);
        const gaps = res.body.data;
        expect(gaps.every(g => g.severity === 'critical')).toBe(true);
    });

    it('respects unresolvedOnly filter', async () => {
        const res = await request(app).get('/api/gaps?unresolvedOnly=true');
        const gaps = res.body.data;
        expect(gaps.every(g => g.resolution === null)).toBe(true);
    });

    it('paginates correctly', async () => {
        const page1 = await request(app).get('/api/gaps?page=1&pageSize=5');
        const page2 = await request(app).get('/api/gaps?page=2&pageSize=5');
        expect(page1.body.data).toHaveLength(5);
        expect(page1.body.data[0].id).not.toBe(page2.body.data[0].id);
    });
});
```

### 15.2 Component Tests (Vitest + jsdom)

```typescript
// tests/dashboard/GapRow.test.tsx
import { render, screen, fireEvent } from '@testing-library/react';
import { GapRow } from '../../dashboard/src/components/gaps/GapRow.jsx';

const mockGap = {
    id: 1, severity: 'critical', constructName: '300-COMPUTE-DAILY-RATE',
    programOrClass: 'CALC-INTEREST', loc: 87, complexity: 12,
    gapType: 'missing_conversion', isEnterTal: false, resolution: null,
};

it('shows severity badge', () => {
    render(<GapRow gap={mockGap} />);
    expect(screen.getByText('CRITICAL')).toBeInTheDocument();
});

it('expands on tap', () => {
    render(<GapRow gap={mockGap} />);
    fireEvent.click(screen.getByText('300-COMPUTE-DAILY-RATE'));
    expect(screen.getByText('SOURCE CODE')).toBeInTheDocument();
});

it('shows TAL/C badge for ENTER TAL gaps', () => {
    render(<GapRow gap={{ ...mockGap, isEnterTal: true }} />);
    expect(screen.getByText('TAL')).toBeInTheDocument();
});
```

### 15.3 E2E (manual checklist)

Before each release, manually verify:

- [ ] Coverage summary loads and shows correct composite score
- [ ] Gap filters work and persist in URL
- [ ] Gap detail expands and shows source code
- [ ] Review confirm/reject flow records to audit log
- [ ] Stub detection view shows signals correctly
- [ ] Decommission sign-off records correctly
- [ ] SSE updates reflected without page reload
- [ ] PDF export generates successfully
- [ ] Dashboard works without internet (vendor bundle)

-----

## 16. Timeline

|Week|Milestone                                                                                              |
|----|-------------------------------------------------------------------------------------------------------|
|1   |API server bootstrap, all route skeletons, DB queries for coverage + gaps                              |
|1-2 |Overview view: composite score, layer bars, trend chart, NonStop status                                |
|2   |Gaps view: list, filters, basic row display                                                            |
|2-3 |Gap detail panel: all 8 sections (source, data items, similarity, graph, behavioral, TAL, verbs, audit)|
|3   |Stubs view: list + stub detail panel with signal breakdown                                             |
|3-4 |Review workflow: queue, confirm/reject/assign modals                                                   |
|4   |Programs view + decommission gate + sign-off flow                                                      |
|4-5 |SSE real-time updates                                                                                  |
|5   |Reporting: PDF generation, CSV export                                                                  |
|5-6 |Polish: mobile layout, loading states, error boundaries, accessibility                                 |
|6   |Testing + bug fixes                                                                                    |

**Total:** 6 weeks

-----

*Project Sentinel — Dashboard Implementation Plan v1.0*
*HP NonStop COBOL → Java · React + Express + Postgres*
*April 2026*

# OME Agent Phase 2 — Jira-to-PR Pipeline with Human Checkpoints

> **Document Version:** 1.0 — 2026-04-05
> **Project:** OME Agent Phase 2 (extends Phase 1 — ome_rag_guide)
> **Prerequisite:** Phase 1 complete (UNDERSTAND / IMPLEMENT / DIAGNOSE modes operational)
> **GPU:** NVIDIA H100 — 95,830 MB (93.6 GB) VRAM
> **Server:** RHEL 9.7, 128 cores, 1TB RAM
> **Pipeline:** Jira Ticket → Plan → Human Checkpoint → Execute → Test → PR → Review
> **Effort:** ~15-18 days

-----

## Table of Contents

1. [Model Analysis — Qwen3-235B vs Qwen3.5-122B](#1-model-analysis)
1. [GPU Memory Layout — Revised](#2-gpu-memory-layout)
1. [Model Deployment — Revised](#3-model-deployment)
1. [Pipeline Architecture](#4-pipeline-architecture)
1. [Human Checkpoint System](#5-human-checkpoint-system)
1. [Jira Webhook Listener](#6-jira-webhook-listener)
1. [Ticket-to-Plan Pipeline](#7-ticket-to-plan-pipeline)
1. [Plan Execution Engine](#8-plan-execution-engine)
1. [Test Runner Integration](#9-test-runner-integration)
1. [PR Creation Pipeline](#10-pr-creation-pipeline)
1. [Database Schema — Phase 2 Extensions](#11-database-schema)
1. [Confidence Gate System](#12-confidence-gate-system)
1. [Observability Dashboard](#13-observability-dashboard)
1. [Rollback & Safety](#14-rollback--safety)
1. [Implementation Schedule](#15-implementation-schedule)
1. [Configuration](#16-configuration)
1. [File Structure — Phase 2 Additions](#17-file-structure)

-----

## 1. Model Analysis — Qwen3-235B-A22B vs Qwen3.5-122B-A10B

### 1.1 Architecture Comparison

|Property         |Qwen3-235B-A22B|Qwen3.5-122B-A10B                  |
|-----------------|---------------|-----------------------------------|
|Total Parameters |235B           |122B                               |
|Active Parameters|22B            |10B                                |
|Architecture     |Standard MoE   |Gated Delta Networks + MoE         |
|Expert Count     |128            |256                                |
|Native Context   |128K           |262K (extensible to 1M)            |
|Multimodal       |Text only      |Text + Image + Video               |
|License          |Apache 2.0     |Apache 2.0                         |
|Release          |2025-07        |2026-02                            |
|Thinking Mode    |Yes            |Yes (improved)                     |
|Tool Calling     |Yes            |Yes (BFCL-V4: 72.2 — best-in-class)|

### 1.2 Benchmark Comparison (Reasoning-Enabled)

|Benchmark           |Qwen3-235B-A22B|Qwen3.5-122B-A10B|Winner         |
|--------------------|---------------|-----------------|---------------|
|SWE-bench Verified  |~65-68         |72.4             |**122B**       |
|BFCL-V4 (Tool Use)  |~55            |72.2             |**122B** (+30%)|
|GPQA Diamond        |~80            |85.5             |**122B**       |
|MMLU-Pro            |~82            |86.1             |**122B**       |
|Terminal-Bench 2.0  |~22            |41.6             |**122B** (+90%)|
|Tau2-Bench (Agentic)|~75            |~82              |**122B**       |
|Coding Index (AA)   |~28            |34.7             |**122B**       |

### 1.3 Hardware Fit on Single H100 (93.6 GB VRAM)

|Property              |Qwen3-235B Q4_K_XL   |Qwen3.5-122B Q4_K_M    |
|----------------------|---------------------|-----------------------|
|Model Weight on GPU   |~35 GB (non-MoE only)|~73 GB (full model)    |
|MoE Expert Offload    |~82 GB → CPU RAM     |**None — fully on GPU**|
|KV Cache (32K ctx)    |~12 GB               |~6 GB                  |
|Total GPU Usage       |~47 GB               |~79 GB                 |
|CPU RAM for Offload   |~82 GB               |0 GB                   |
|Inference Speed       |~15-25 tok/s         |**~50-80 tok/s**       |
|Parallel Slots        |1                    |**3-4**                |
|CPU→GPU Bus Bottleneck|Yes (PCIe for MoE)   |**No**                 |

### 1.4 Recommendation — Replace Qwen3-235B with Qwen3.5-122B

**Qwen3.5-122B-A10B is the clear winner for OME Agent Phase 2.** The reasoning:

**Performance:** Qwen3.5-122B surpasses Qwen3-235B on every coding and agentic benchmark that matters for Jira-to-PR: SWE-bench (+7 points), tool calling (+30%), terminal operations (+90%). The newer model’s improved RL training and hybrid architecture produce genuinely better code generation.

**Speed:** 3-4x faster inference (50-80 tok/s vs 15-25 tok/s) because the entire model fits on GPU without CPU offload. No PCIe bus bottleneck for expert layers. This matters enormously for Phase 2 where IMPLEMENT mode runs multi-step code generation — a 60-second task becomes 15-20 seconds.

**Concurrency:** 3-4 parallel slots (vs 1) means the pipeline can handle simultaneous ticket processing without queueing. Critical for a webhook-driven Jira pipeline where multiple tickets may trigger concurrently.

**Reliability:** Eliminating CPU↔GPU offload removes the single biggest source of latency variance and potential OOM on CPU side. Deterministic, predictable inference.

**Context Window:** 262K native (vs 128K) — Phase 2 needs large contexts for multi-file code generation with full reference files, similar PRs, and test suites.

### 1.5 Dual Model Strategy — Revised

|Role              |Phase 1 (Current)       |Phase 2 (Revised)                   |
|------------------|------------------------|------------------------------------|
|Fast Model (:8090)|Qwen3-Coder-Next 80B-A3B|Qwen3-Coder-Next 80B-A3B (unchanged)|
|Deep Model (:8091)|Qwen3-235B-A22B         |**Qwen3.5-122B-A10B**               |

The fast model (Coder-Next) remains unchanged — it handles intent classification, convention detection, and UNDERSTAND mode excellently. The deep model upgrade gives us better code generation, faster inference, and higher concurrency for the autonomous pipeline.

### 1.6 GPU Memory Layout — Revised

```
H100 — 95,830 MB (93.6 GB)
──────────────────────────────
Qwen3-Coder-Next Q5_K_M         ~30 GB  (full on GPU, unchanged)
                                         [3 parallel slots, 64K ctx]

  ── OR (time-shared via systemd) ──

Qwen3.5-122B-A10B Q4_K_M        ~73 GB  (full on GPU — NO CPU offload)
                                         [3 parallel slots, 32K ctx]
──────────────────────────────

Strategy: Time-share GPU between models.
  Phase 2 pipeline runs 122B exclusively.
  IDE queries can use either model via routing.
  Both cannot be loaded simultaneously (73+30 = 103 > 93.6).
```

**Important change from Phase 1:** The original plan loaded both models simultaneously (35 GB non-MoE + 30 GB Coder-Next = ~65 GB on GPU). With 122B at 73 GB, simultaneous loading is not possible. Solution: **time-shared GPU with model swap orchestration** (see §3).

### 1.7 Model Swap Orchestration

```typescript
// src/llm/model_swap.ts

export class ModelSwapManager {
  private currentModel: 'fast' | 'deep' | 'none' = 'none';
  private swapLock = new AsyncMutex();

  // Average swap time: ~15-25 seconds (model load from NVMe)
  // Swap is infrequent — pipeline batches work per model

  async ensureModel(needed: 'fast' | 'deep'): Promise<void> {
    if (this.currentModel === needed) return;

    await this.swapLock.runExclusive(async () => {
      if (this.currentModel === needed) return; // Double-check after lock

      // Stop current model
      if (this.currentModel !== 'none') {
        await this.stopModel(this.currentModel);
      }

      // Start needed model
      await this.startModel(needed);
      await this.waitForHealth(needed, 60_000);
      this.currentModel = needed;
    });
  }

  // Pipeline strategy: batch all UNDERSTAND queries → swap → batch IMPLEMENT
  // Avoid ping-pong swapping
}
```

**Alternative — Keep 235B for dual-load:** If simultaneous model availability is critical for IDE responsiveness, keep Qwen3-235B (35 GB non-MoE on GPU) and accept the slower inference. The swap strategy is recommended because Phase 2 pipeline runs are batch-oriented (process ticket end-to-end), not interactive.

-----

## 2. Model Deployment — Revised

### 2.1 Deep Model — Qwen3.5-122B-A10B (:8091)

```bash
./llama-server \
  --model /models/qwen35-122b/Qwen3.5-122B-A10B-Q4_K_M.gguf \
  --host 127.0.0.1 --port 8091 \
  --n-gpu-layers 99 --ctx-size 32768 --parallel 3 \
  --threads 8 --flash-attn --jinja \
  --reasoning-format deepseek \
  --temp 0.6 --top-p 0.95 --top-k 20 --min-p 0.0
```

Key differences from 235B deployment:

- No `-ot ".ffn_.*_exps.=CPU"` — all layers fit on GPU
- `--parallel 3` (up from 1) — 3 concurrent requests
- `--threads 8` (down from 32) — GPU does all compute, CPU threads only for I/O
- Faster cold start: ~15s load (vs ~45s for 235B with CPU offload)

### 2.2 Download via aria2c (Corporate Nexus Proxy)

```bash
# Qwen3.5-122B-A10B Q4_K_M from Unsloth — single file, ~73 GB
aria2c -x 16 -s 16 -k 1M --auto-file-renaming=false \
  --continue=true --max-tries=0 --retry-wait=30 \
  --connect-timeout=60 --timeout=600 \
  -d /models/qwen35-122b/ \
  "https://huggingface.co/unsloth/Qwen3.5-122B-A10B-GGUF/resolve/main/Qwen3.5-122B-A10B-Q4_K_M.gguf"

# Verify download
sha256sum /models/qwen35-122b/Qwen3.5-122B-A10B-Q4_K_M.gguf
```

### 2.3 Systemd Service — Phase 2 Pipeline

```ini
# /etc/systemd/system/ome-agent-deep.service (revised)
[Unit]
Description=OME Agent Deep Model (Qwen3.5-122B-A10B)
After=network.target

[Service]
Type=simple
User=ome
ExecStart=/opt/llama.cpp/llama-server \
  --model /models/qwen35-122b/Qwen3.5-122B-A10B-Q4_K_M.gguf \
  --host 127.0.0.1 --port 8091 \
  --n-gpu-layers 99 --ctx-size 32768 --parallel 3 \
  --threads 8 --flash-attn --jinja \
  --reasoning-format deepseek \
  --temp 0.6 --top-p 0.95 --top-k 20 --min-p 0.0
Restart=always
RestartSec=10
Environment=CUDA_VISIBLE_DEVICES=0

[Install]
WantedBy=multi-user.target
```

-----

## 3. Pipeline Architecture

### 3.1 End-to-End Flow

```
                        ┌──────────────────────────────────────┐
                        │  Jira Cloud / Server                 │
                        │  Webhook: issue.updated              │
                        │  Trigger: status → "Ready for Dev"   │
                        └──────────────┬───────────────────────┘
                                       │ POST /webhook/jira
                                       ▼
┌──────────────────────────────────────────────────────────────────────┐
│  PHASE A — INTAKE (Automatic)                                        │
│                                                                      │
│  1. Validate webhook signature                                       │
│  2. Extract ticket key, type, priority                               │
│  3. Check eligibility (type, size, complexity estimate)              │
│  4. Sparse ticket enrichment (climb Epic→Story, linked PRs/docs)    │
│  5. Insert into pipeline_runs table (status: intake_complete)        │
└──────────────────────────┬───────────────────────────────────────────┘
                           ▼
┌──────────────────────────────────────────────────────────────────────┐
│  PHASE B — PLAN (Automatic, via Deep Model)                          │
│                                                                      │
│  1. Call ome_rag_guide(mode='implement', ticket_id=...)              │
│  2. Receive: change plan, blast radius, affected tests               │
│  3. Confidence assessment (file/line/symbol verification)            │
│  4. Risk scoring (blast radius size, critical path, cross-repo)     │
│  5. Insert into change_plans table (status: plan_generated)          │
└──────────────────────────┬───────────────────────────────────────────┘
                           ▼
┌──────────────────────────────────────────────────────────────────────┐
│  ══════════════════ HUMAN CHECKPOINT 1 ══════════════════            │
│                                                                      │
│  PLAN REVIEW — Developer/TL reviews:                                 │
│  • Change summary & rationale                                        │
│  • Files to modify (with current vs proposed diffs)                  │
│  • Blast radius (affected callers, downstream services)              │
│  • New files to create                                               │
│  • Config changes                                                    │
│  • Test strategy                                                     │
│                                                                      │
│  Actions: APPROVE | REJECT | MODIFY (with notes)                     │
│  Channel: Slack notification + Web UI + email                        │
│  SLA: Plan expires after 48 hours (auto-expire, no silent execution) │
│  Confidence gate: if confidence < 0.7, REQUIRE human review          │
│                    if confidence ≥ 0.9 AND risk=low, allow AUTO       │
└──────────────────────────┬───────────────────────────────────────────┘
                           ▼ (only on APPROVE)
┌──────────────────────────────────────────────────────────────────────┐
│  PHASE C — EXECUTE (Automatic, guarded)                              │
│                                                                      │
│  1. Create git worktree from latest main/develop                     │
│  2. Content hash verification — confirm files unchanged since plan   │
│  3. Apply changes in execution_order sequence                        │
│  4. Each change: verify currentCode matches → apply proposedCode     │
│  5. If ANY hash mismatch: ABORT → re-plan required                  │
│  6. Run linter (eslint / checkstyle) on changed files                │
│  7. Run TypeScript/Java compilation on changed modules               │
│  8. Status: changes_applied                                          │
└──────────────────────────┬───────────────────────────────────────────┘
                           ▼
┌──────────────────────────────────────────────────────────────────────┐
│  PHASE D — TEST (Automatic)                                          │
│                                                                      │
│  1. Identify affected test suites (test_coverage table + blast)      │
│  2. Run unit tests on changed modules                                │
│  3. Run integration tests if cross-service changes                   │
│  4. Capture: pass/fail/skip counts, coverage delta, output logs      │
│  5. If tests fail:                                                   │
│     a. Attempt auto-fix (1 retry via DIAGNOSE mode)                  │
│     b. If still failing: ABORT → rollback → human notification       │
│  6. Status: tests_passed or tests_failed                             │
└──────────────────────────┬───────────────────────────────────────────┘
                           ▼
┌──────────────────────────────────────────────────────────────────────┐
│  ══════════════════ HUMAN CHECKPOINT 2 ══════════════════            │
│                                                                      │
│  PR REVIEW — Standard Bitbucket PR review process:                   │
│  • Auto-generated PR title from Jira + change summary                │
│  • Auto-generated PR description (changes, rationale, test results)  │
│  • Diff is standard git diff — reviewable by any developer           │
│  • Linked to Jira ticket (bidirectional)                             │
│  • Blast radius summary in PR description                            │
│  • Test results attached                                             │
│                                                                      │
│  Actions: APPROVE + MERGE | REQUEST CHANGES | DECLINE                │
│  This is a normal code review — humans own the merge decision        │
└──────────────────────────────────────────────────────────────────────┘
```

### 3.2 Pipeline State Machine

```
intake_received
  → enrichment_in_progress
    → enrichment_complete
      → plan_generating
        → plan_generated
          → checkpoint_1_pending        ← HUMAN CHECKPOINT 1
            → plan_approved             (human approved)
            → plan_rejected             (human rejected — terminal)
            → plan_modification_needed  (human wants changes → re-plan)
            → plan_expired              (48h SLA exceeded — terminal)
              → execution_starting
                → execution_in_progress
                  → execution_hash_mismatch  (abort — re-plan needed)
                  → execution_lint_failed    (abort — auto-fix or escalate)
                  → changes_applied
                    → tests_running
                      → tests_passed
                        → pr_creating
                          → checkpoint_2_pending  ← HUMAN CHECKPOINT 2 (PR review)
                            → pr_merged           (terminal — success)
                            → pr_declined         (terminal)
                            → pr_changes_requested (→ re-plan or manual)
                      → tests_failed
                        → auto_fix_attempting
                          → tests_passed (retry succeeded)
                          → auto_fix_failed (terminal — escalate to human)
```

-----

## 4. Human Checkpoint System

### 4.1 Checkpoint 1 — Plan Approval

The plan review is the critical safety gate. The system NEVER executes code changes without explicit human approval (except for auto-approved low-risk changes when confidence ≥ 0.9).

```typescript
// src/execution/checkpoint.ts

export interface CheckpointDecision {
  action: 'approve' | 'reject' | 'modify';
  reviewer: string;              // Bitbucket username
  notes?: string;                // Rejection reason or modification instructions
  modifications?: {
    remove_changes?: string[];   // Change IDs to remove
    add_instructions?: string;   // Natural language for re-plan
  };
  decided_at: Date;
}

export interface PlanCheckpointPayload {
  pipeline_run_id: string;
  ticket_key: string;
  ticket_summary: string;
  enriched_description: string;

  // What will change
  change_summary: string;
  changes: CodeChange[];
  new_files: NewFile[];
  config_changes: ConfigChange[];

  // Impact assessment
  blast_radius: {
    directly_affected: string[];
    transitively_affected: string[];
    cross_repo_impact: string[];
    affected_tests: string[];
  };

  // Quality signals
  confidence: {
    level: 'high' | 'medium' | 'low';
    score: number;
    reasons: string[];
  };
  risk_level: 'low' | 'medium' | 'high' | 'critical';
  risk_factors: string[];

  // Context
  similar_prs: PRReference[];
  conventions_applied: string[];
  reference_files: string[];

  // Auto-approval eligibility
  auto_approve_eligible: boolean;
  auto_approve_reason?: string;
}
```

### 4.2 Auto-Approval Criteria (Conservative)

Auto-approval is ONLY enabled when ALL conditions are met:

```typescript
export function isAutoApproveEligible(payload: PlanCheckpointPayload): {
  eligible: boolean;
  reason: string;
} {
  // Hard requirements — ALL must pass
  const checks = [
    {
      pass: payload.confidence.score >= 0.9,
      reason: `Confidence ${payload.confidence.score} < 0.9`,
    },
    {
      pass: payload.risk_level === 'low',
      reason: `Risk level '${payload.risk_level}' is not 'low'`,
    },
    {
      pass: payload.changes.length <= 3,
      reason: `${payload.changes.length} changes exceeds limit of 3`,
    },
    {
      pass: payload.new_files.length === 0,
      reason: 'Creates new files — requires human review',
    },
    {
      pass: payload.config_changes.length === 0,
      reason: 'Modifies config — requires human review',
    },
    {
      pass: payload.blast_radius.cross_repo_impact.length === 0,
      reason: 'Cross-repo impact — requires human review',
    },
    {
      pass: payload.blast_radius.directly_affected.length <= 5,
      reason: `Blast radius ${payload.blast_radius.directly_affected.length} > 5`,
    },
    {
      pass: !payload.changes.some(c =>
        c.file.includes('/security/') ||
        c.file.includes('/auth/') ||
        c.file.includes('application.yml') ||
        c.file.includes('pom.xml')
      ),
      reason: 'Touches security/auth/config files — requires human review',
    },
  ];

  const failures = checks.filter(c => !c.pass);

  if (failures.length === 0) {
    return { eligible: true, reason: 'All auto-approval criteria met' };
  }

  return {
    eligible: false,
    reason: failures.map(f => f.reason).join('; '),
  };
}
```

### 4.3 Review Web UI

```
GET /api/pipeline/review/:pipeline_run_id
```

Serves a dark-theme review page showing:

- Ticket context (enriched description, parent epic, linked docs)
- Change plan with syntax-highlighted diffs (side-by-side)
- Blast radius visualization (expandable tree)
- Confidence breakdown (per-change confidence scores)
- Similar PRs that the model referenced
- Three action buttons: **Approve** / **Request Changes** / **Reject**
- Notes field for modification instructions

### 4.4 Notification Channels

```typescript
// src/execution/notifications.ts

export async function notifyCheckpointPending(
  payload: PlanCheckpointPayload,
  assignees: string[]
): Promise<void> {
  const reviewUrl = `${AGENT_BASE_URL}/api/pipeline/review/${payload.pipeline_run_id}`;

  // 1. Slack notification (primary)
  await slackClient.postMessage({
    channel: process.env.PIPELINE_SLACK_CHANNEL,
    blocks: [
      {
        type: 'section',
        text: {
          type: 'mrkdwn',
          text: [
            `*Plan Review Required* — \`${payload.ticket_key}\``,
            `> ${payload.ticket_summary}`,
            ``,
            `*Changes:* ${payload.changes.length} file modifications`,
            `*Risk:* ${payload.risk_level} | *Confidence:* ${(payload.confidence.score * 100).toFixed(0)}%`,
            `*Blast Radius:* ${payload.blast_radius.directly_affected.length} direct, ` +
            `${payload.blast_radius.cross_repo_impact.length} cross-repo`,
          ].join('\n'),
        },
      },
      {
        type: 'actions',
        elements: [
          { type: 'button', text: { type: 'plain_text', text: 'Review Plan' },
            url: reviewUrl, style: 'primary' },
        ],
      },
    ],
  });

  // 2. Email digest (for TLs not on Slack)
  // 3. Jira comment with review link
  await jiraClient.addComment(payload.ticket_key,
    `[OME Agent] Implementation plan generated. ` +
    `[Review Plan](${reviewUrl}) — ${payload.changes.length} changes, ` +
    `risk: ${payload.risk_level}, confidence: ${(payload.confidence.score * 100).toFixed(0)}%`
  );
}
```

### 4.5 Checkpoint SLA & Expiry

```typescript
// Plans expire after 48 hours if not reviewed
// This prevents stale plans from being executed on changed code

const PLAN_EXPIRY_HOURS = parseInt(process.env.PLAN_EXPIRY_HOURS || '48', 10);

// Cron job: expire stale plans every hour
cron.schedule('0 * * * *', async () => {
  const expired = await agentDb.query(`
    UPDATE ome_agent.pipeline_runs
    SET status = 'plan_expired',
        completed_at = NOW(),
        expiry_reason = 'Plan not reviewed within SLA'
    WHERE status = 'checkpoint_1_pending'
      AND checkpoint_1_notified_at < NOW() - INTERVAL '${PLAN_EXPIRY_HOURS} hours'
    RETURNING ticket_key, pipeline_run_id
  `);

  for (const run of expired.rows) {
    await notifyPlanExpired(run.ticket_key, run.pipeline_run_id);
  }
});
```

-----

## 5. Jira Webhook Listener

### 5.1 Webhook Registration

```
POST https://your-jira.atlassian.net/rest/api/3/webhook
{
  "name": "OME Agent Pipeline",
  "url": "https://ome-agent.internal:3100/webhook/jira",
  "events": ["jira:issue_updated"],
  "filters": {
    "issue-related-events-section": "project = BMT AND type in (Story, Task, Bug)"
  }
}
```

### 5.2 Webhook Handler

```typescript
// src/pipeline/jira_webhook.ts

export async function handleJiraWebhook(req: Request, res: Response): Promise<void> {
  // 1. Verify webhook signature
  const signature = req.headers['x-hub-signature'];
  if (!verifyJiraSignature(req.body, signature)) {
    res.status(401).json({ error: 'Invalid signature' });
    return;
  }

  const event = req.body;
  const issue = event.issue;
  const changelog = event.changelog;

  // 2. Only trigger on status transition to "Ready for Dev"
  const statusChange = changelog?.items?.find(
    (item: any) => item.field === 'status' && item.toString === 'Ready for Dev'
  );

  if (!statusChange) {
    res.status(200).json({ ignored: true, reason: 'Not a qualifying status change' });
    return;
  }

  // 3. Eligibility check
  const eligibility = await checkEligibility(issue);
  if (!eligibility.eligible) {
    await jiraClient.addComment(issue.key,
      `[OME Agent] Ticket not eligible for automated implementation: ${eligibility.reason}`
    );
    res.status(200).json({ ignored: true, reason: eligibility.reason });
    return;
  }

  // 4. Create pipeline run
  const runId = await createPipelineRun(issue);

  // 5. Start async pipeline (don't block webhook response)
  setImmediate(() => runPipeline(runId).catch(err => {
    console.error(`[Pipeline] ${issue.key} failed:`, err);
    markPipelineFailed(runId, err.message);
  }));

  res.status(202).json({ pipeline_run_id: runId, ticket: issue.key });
}
```

### 5.3 Eligibility Criteria

```typescript
export async function checkEligibility(issue: JiraIssue): Promise<{
  eligible: boolean;
  reason: string;
  complexity: 'small' | 'medium' | 'large' | 'too_large';
}> {
  const type = issue.fields.issuetype.name;
  const storyPoints = issue.fields.story_points ?? issue.fields.customfield_10016;

  // Reject: Epics, Sub-tasks without parent context, specific labels
  if (['Epic', 'Initiative'].includes(type)) {
    return { eligible: false, reason: 'Epics are too broad for automation', complexity: 'too_large' };
  }

  if (issue.fields.labels?.includes('no-automation')) {
    return { eligible: false, reason: 'Ticket has no-automation label', complexity: 'medium' };
  }

  // Estimate complexity from story points, description length, linked tickets
  const descLen = (issue.fields.description || '').length;
  const linkedCount = issue.fields.issuelinks?.length ?? 0;

  let complexity: 'small' | 'medium' | 'large' | 'too_large' = 'medium';

  if (storyPoints && storyPoints > 8) {
    complexity = 'too_large';
  } else if (storyPoints && storyPoints <= 2) {
    complexity = 'small';
  } else if (storyPoints && storyPoints <= 5) {
    complexity = 'medium';
  } else if (storyPoints && storyPoints > 5) {
    complexity = 'large';
  }

  if (complexity === 'too_large') {
    return {
      eligible: false,
      reason: `Story points (${storyPoints}) exceeds automation limit of 8`,
      complexity,
    };
  }

  // Check if repos are indexed in OME RAG
  const mentionedRepos = extractRepoReferences(issue);
  if (mentionedRepos.length > 0) {
    const indexed = await ragClient.getProjectInfo(mentionedRepos[0]);
    if (!indexed) {
      return { eligible: false, reason: `Repo ${mentionedRepos[0]} not indexed`, complexity };
    }
  }

  return { eligible: true, reason: 'Eligible', complexity };
}
```

-----

## 6. Ticket-to-Plan Pipeline

### 6.1 Enrichment + Plan Generation

```typescript
// src/pipeline/plan_generator.ts

export async function generatePlan(runId: string): Promise<PlanCheckpointPayload> {
  const run = await getPipelineRun(runId);
  const ticketKey = run.ticket_key;

  // ── Phase A: Enrich ticket ──
  await updateStatus(runId, 'enrichment_in_progress');

  // Sparse ticket enrichment (from Phase 1 — §19 of Agent Plan)
  const ticketContext = await ragClient.getTicketContext(ticketKey);
  let enrichedTicket = ticketContext;

  if (isSparseTicket(ticketContext)) {
    enrichedTicket = await enrichSparseTicket(ticketContext, agentContext);
  }

  await updateStatus(runId, 'enrichment_complete');

  // ── Phase B: Generate implementation plan ──
  await updateStatus(runId, 'plan_generating');

  // This calls ome_rag_guide in IMPLEMENT mode
  // Which internally: gathers RAG context, reads files, calls deep model
  const implementResponse = await orchestrate({
    query: buildImplementQuery(enrichedTicket),
    ticket_id: ticketKey,
    mode: 'implement',
    context: JSON.stringify({
      enriched_description: enrichedTicket.inferredDescription ?? enrichedTicket.summary,
      acceptance_criteria: enrichedTicket.acceptanceCriteria,
      parent_context: enrichedTicket.parentContext,
      linked_pr_context: enrichedTicket.linkedPRContext,
      linked_documentation: enrichedTicket.linkedDocumentation,
    }),
  }, agentContext);

  // ── Phase B2: Confidence & risk assessment ──
  const confidence = computeConfidence(implementResponse);
  const risk = assessRisk(implementResponse, enrichedTicket);
  const autoApproval = isAutoApproveEligible({
    ...buildCheckpointPayload(implementResponse, enrichedTicket, confidence, risk),
  });

  // ── Store plan ──
  const payload = buildCheckpointPayload(
    implementResponse, enrichedTicket, confidence, risk
  );
  payload.auto_approve_eligible = autoApproval.eligible;
  payload.auto_approve_reason = autoApproval.reason;

  await storePlan(runId, payload);
  await updateStatus(runId, 'plan_generated');

  return payload;
}

function buildImplementQuery(ticket: EnrichedTicket): string {
  const parts = [
    `Implement Jira ticket ${ticket.ticketId}: "${ticket.summary}"`,
  ];

  if (ticket.inferredDescription) {
    parts.push(`Description: ${ticket.inferredDescription}`);
  }

  if (ticket.acceptanceCriteria) {
    parts.push(`Acceptance criteria: ${ticket.acceptanceCriteria}`);
  }

  if (ticket.parentContext) {
    parts.push(`Parent story context: ${ticket.parentContext.summary}`);
  }

  return parts.join('\n');
}
```

### 6.2 Confidence Computation

```typescript
// src/pipeline/confidence.ts

export function computeConfidence(response: ImplementResponse): {
  level: 'high' | 'medium' | 'low';
  score: number;
  reasons: string[];
} {
  let score = 1.0;
  const reasons: string[] = [];

  // Deductions
  for (const change of response.changes) {
    // File not verified on disk
    if (!change._verified_on_disk) {
      score -= 0.15;
      reasons.push(`${change.file}: not verified on disk`);
    }

    // Line content mismatch (fuzzy match < 90%)
    if (change._line_match_score && change._line_match_score < 0.9) {
      score -= 0.1;
      reasons.push(`${change.file}:${change.currentCode?.startLine}: line match ${(change._line_match_score * 100).toFixed(0)}%`);
    }

    // Symbol not found in graph
    for (const sym of change.affectedSymbols) {
      if (!change._symbols_verified?.includes(sym)) {
        score -= 0.05;
        reasons.push(`Symbol '${sym}' not found in knowledge graph`);
      }
    }
  }

  // Boost: similar PRs found and referenced
  if (response.similarPRs.length > 0) {
    score += 0.05;
  }

  // Boost: conventions detected and applied
  if (response.conventions.length > 0) {
    score += 0.03;
  }

  score = Math.max(0, Math.min(1, score));

  return {
    level: score >= 0.85 ? 'high' : score >= 0.6 ? 'medium' : 'low',
    score,
    reasons,
  };
}
```

### 6.3 Risk Assessment

```typescript
export function assessRisk(
  response: ImplementResponse,
  ticket: EnrichedTicket
): { level: 'low' | 'medium' | 'high' | 'critical'; factors: string[] } {
  const factors: string[] = [];
  let riskScore = 0;

  // Cross-repo impact
  if (response.blastRadius?.cross_repo_impact?.length > 0) {
    riskScore += 3;
    factors.push(`Cross-repo impact: ${response.blastRadius.cross_repo_impact.join(', ')}`);
  }

  // Security/auth file changes
  const sensitiveFiles = response.changes.filter(c =>
    /\/(security|auth|crypto|ssl|tls)\//i.test(c.file) ||
    /application\.(yml|yaml|properties)$/i.test(c.file) ||
    /(pom\.xml|build\.gradle|package\.json)$/i.test(c.file)
  );
  if (sensitiveFiles.length > 0) {
    riskScore += 2;
    factors.push(`Sensitive files: ${sensitiveFiles.map(f => f.file).join(', ')}`);
  }

  // High centrality functions affected
  const highCentrality = response.changes.filter(c =>
    c.affectedSymbols.some(s => response._centralityScores?.[s] > 0.01)
  );
  if (highCentrality.length > 0) {
    riskScore += 2;
    factors.push(`High-traffic functions affected`);
  }

  // Many files changed
  if (response.changes.length > 5) {
    riskScore += 1;
    factors.push(`${response.changes.length} files modified`);
  }

  // New files (harder to verify correctness)
  if (response.newFiles.length > 0) {
    riskScore += 1;
    factors.push(`${response.newFiles.length} new files`);
  }

  // Database migration involved
  const hasMigration = response.changes.some(c =>
    /\/(migrations?|flyway|liquibase)\//.test(c.file)
  );
  if (hasMigration) {
    riskScore += 3;
    factors.push('Database migration included');
  }

  const level = riskScore >= 6 ? 'critical' :
                riskScore >= 4 ? 'high' :
                riskScore >= 2 ? 'medium' : 'low';

  return { level, factors };
}
```

-----

## 7. Plan Execution Engine

### 7.1 Git Worktree Manager

```typescript
// src/execution/worktree_manager.ts

export class WorktreeManager {
  private worktreeBase: string;

  constructor() {
    this.worktreeBase = process.env.WORKTREE_BASE || '/tmp/ome-agent-worktrees';
  }

  async createWorktree(
    repoSlug: string,
    branchName: string,
    baseBranch: string = 'develop'
  ): Promise<WorktreeContext> {
    const repoPath = path.join(process.env.REPOS_DIR || '/repos', repoSlug);
    const worktreePath = path.join(this.worktreeBase, `${repoSlug}--${branchName}`);

    // Fetch latest
    await execAsync(`git -C ${repoPath} fetch origin ${baseBranch}`);

    // Create worktree with new branch
    await execAsync(
      `git -C ${repoPath} worktree add ${worktreePath} -b ${branchName} origin/${baseBranch}`
    );

    // Record base commit for rollback
    const baseCommit = (await execAsync(
      `git -C ${worktreePath} rev-parse HEAD`
    )).stdout.trim();

    return {
      worktreePath,
      repoPath,
      repoSlug,
      branchName,
      baseBranch,
      baseCommit,
    };
  }

  async removeWorktree(ctx: WorktreeContext): Promise<void> {
    await execAsync(`git -C ${ctx.repoPath} worktree remove ${ctx.worktreePath} --force`);
    await execAsync(`git -C ${ctx.repoPath} branch -D ${ctx.branchName}`).catch(() => {});
  }
}
```

### 7.2 Change Applicator

```typescript
// src/execution/change_applicator.ts

export async function applyChangePlan(
  ctx: WorktreeContext,
  plan: PlanCheckpointPayload,
  agentDb: AgentDB
): Promise<ApplyResult> {
  const results: ChangeResult[] = [];
  let aborted = false;

  // Sort by execution order
  const ordered = plan.changes.sort((a, b) =>
    (a.priority ?? 99) - (b.priority ?? 99)
  );

  for (const change of ordered) {
    const filePath = path.join(ctx.worktreePath, change.file);

    // ── Step 1: Content hash verification ──
    try {
      const currentContent = await fs.readFile(filePath, 'utf-8');
      const lines = currentContent.split('\n');

      if (change.currentCode) {
        const actualLines = lines
          .slice(change.currentCode.startLine - 1, change.currentCode.endLine)
          .join('\n');
        const actualHash = md5(actualLines.trim());

        if (actualHash !== change.currentCode.contentHash) {
          // Content has changed since plan was generated
          // Try fuzzy match — content might have shifted lines
          const fuzzyMatch = fuzzyFindContent(
            currentContent, change.currentCode.content, 0.75
          );

          if (!fuzzyMatch) {
            results.push({
              changeId: change.id,
              status: 'hash_mismatch',
              error: `File changed since plan: expected hash ${change.currentCode.contentHash}, ` +
                     `got ${actualHash} at lines ${change.currentCode.startLine}-${change.currentCode.endLine}`,
            });
            aborted = true;
            break;
          }

          // Content found at different line numbers — correct and proceed
          change.currentCode.startLine = fuzzyMatch.startLine;
          change.currentCode.endLine = fuzzyMatch.endLine;
          console.log(
            `[Apply] ${change.file}: content shifted to lines ` +
            `${fuzzyMatch.startLine}-${fuzzyMatch.endLine} (was ${change.currentCode.startLine}-${change.currentCode.endLine})`
          );
        }
      }
    } catch (err) {
      if (change.action === 'insert' || !change.currentCode) {
        // New file or insert — OK if file doesn't exist
      } else {
        results.push({
          changeId: change.id,
          status: 'file_not_found',
          error: `${change.file} not found in worktree`,
        });
        aborted = true;
        break;
      }
    }

    // ── Step 2: Apply change ──
    try {
      switch (change.action) {
        case 'modify':
        case 'replace':
          await applyModification(filePath, change);
          break;
        case 'insert':
          await applyInsertion(filePath, change);
          break;
        case 'delete':
          await applyDeletion(filePath, change);
          break;
      }

      results.push({ changeId: change.id, status: 'applied' });

    } catch (err) {
      results.push({
        changeId: change.id,
        status: 'apply_failed',
        error: (err as Error).message,
      });
      aborted = true;
      break;
    }
  }

  // ── Step 3: Apply new files ──
  if (!aborted && plan.new_files) {
    for (const newFile of plan.new_files) {
      const filePath = path.join(ctx.worktreePath, newFile.path);
      await fs.mkdir(path.dirname(filePath), { recursive: true });
      await fs.writeFile(filePath, newFile.content, 'utf-8');
      results.push({ changeId: `new-${newFile.path}`, status: 'created' });
    }
  }

  // ── Step 4: Lint check ──
  let lintPassed = true;
  if (!aborted) {
    const lintResult = await runLinter(ctx, plan.changes.map(c => c.file));
    if (!lintResult.passed) {
      lintPassed = false;
    }
  }

  // ── Step 5: Compile check ──
  let compilePassed = true;
  if (!aborted && lintPassed) {
    const compileResult = await runCompilation(ctx);
    if (!compileResult.passed) {
      compilePassed = false;
    }
  }

  if (aborted) {
    // Rollback: reset worktree to base commit
    await execAsync(`git -C ${ctx.worktreePath} reset --hard ${ctx.baseCommit}`);
  }

  return {
    success: !aborted && lintPassed && compilePassed,
    aborted,
    lintPassed,
    compilePassed,
    changes: results,
  };
}
```

-----

## 8. Test Runner Integration

### 8.1 Test Discovery

```typescript
// src/execution/test_runner.ts

export async function discoverAffectedTests(
  pool: Pool,
  graphClient: IGraphClient,
  repoSlug: string,
  changedFiles: string[]
): Promise<TestSuite[]> {
  const testFiles = new Set<string>();

  // Source 1: test_coverage table (bootstrap Phase 2i)
  for (const file of changedFiles) {
    const coverage = await pool.query(
      `SELECT test_file FROM test_coverage
       WHERE repo_slug = $1 AND source_file = $2`,
      [repoSlug, file]
    );
    coverage.rows.forEach(r => testFiles.add(r.test_file));
  }

  // Source 2: Naming convention fallback
  for (const file of changedFiles) {
    const base = path.basename(file, path.extname(file));
    const dir = path.dirname(file);

    const patterns = [
      `${dir}/${base}.test${path.extname(file)}`,
      `${dir}/${base}.spec${path.extname(file)}`,
      `${dir}/__tests__/${base}${path.extname(file)}`,
      `src/test/java/${dir.replace('src/main/java/', '')}/${base}Test.java`,
    ];

    for (const pattern of patterns) {
      try {
        await fs.access(path.join(process.env.REPOS_DIR!, repoSlug, pattern));
        testFiles.add(pattern);
      } catch {}
    }
  }

  // Source 3: Blast radius — tests for transitively affected files
  for (const file of changedFiles) {
    const fileNode = await graphClient.queryNodes({
      file_path: file, repo_slug: repoSlug, node_type: 'file',
    });

    if (fileNode.length > 0) {
      const callers = await graphClient.getReverseCallChain(
        fileNode[0].name, repoSlug, 1
      );
      for (const caller of callers) {
        if (caller.file_path) {
          const tests = await pool.query(
            `SELECT test_file FROM test_coverage
             WHERE repo_slug = $1 AND source_file = $2`,
            [repoSlug, caller.file_path]
          );
          tests.rows.forEach(r => testFiles.add(r.test_file));
        }
      }
    }
  }

  return Array.from(testFiles).map(f => ({
    testFile: f,
    type: inferTestType(f),
  }));
}
```

### 8.2 Test Execution

```typescript
export async function runTests(
  ctx: WorktreeContext,
  tests: TestSuite[]
): Promise<TestResult> {
  const project = await getProjectInfo(ctx.repoSlug);
  const isJava = project.project_type === 'backend';
  const isNode = project.project_type === 'frontend';

  let command: string;
  let timeout = 300_000; // 5 minutes default

  if (isJava) {
    // Run specific test classes via Maven
    const testClasses = tests
      .filter(t => t.type === 'unit')
      .map(t => path.basename(t.testFile, '.java'))
      .join(',');

    command = testClasses
      ? `cd ${ctx.worktreePath} && mvn test -pl . -Dtest=${testClasses} -DfailIfNoTests=false`
      : `cd ${ctx.worktreePath} && mvn test -pl .`;
    timeout = 600_000; // 10 minutes for Java
  } else {
    // Run specific test files via npm/vitest
    const testFiles = tests
      .filter(t => t.type === 'unit')
      .map(t => t.testFile)
      .join(' ');

    command = testFiles
      ? `cd ${ctx.worktreePath} && npx vitest run ${testFiles} --reporter=json`
      : `cd ${ctx.worktreePath} && npm test -- --watchAll=false`;
  }

  try {
    const { stdout, stderr } = await execAsync(command, { timeout });
    return {
      passed: true,
      output: stdout,
      errorOutput: stderr,
      testCount: parseTestCount(stdout, isJava),
    };
  } catch (err: any) {
    return {
      passed: false,
      output: err.stdout || '',
      errorOutput: err.stderr || err.message,
      testCount: parseTestCount(err.stdout || '', isJava),
      exitCode: err.code,
    };
  }
}
```

### 8.3 Auto-Fix on Test Failure (1 Retry)

```typescript
export async function attemptAutoFix(
  ctx: WorktreeContext,
  testResult: TestResult,
  plan: PlanCheckpointPayload
): Promise<{ fixed: boolean; newTestResult?: TestResult }> {
  // Only attempt once — no retry loops
  if (testResult.passed) return { fixed: true };

  console.log(`[AutoFix] Attempting DIAGNOSE mode for test failure`);

  // Call ome_rag_guide in DIAGNOSE mode with test output
  const diagnoseResponse = await orchestrate({
    query: `Fix test failures after implementing ${plan.ticket_key}`,
    context: [
      `Test output:\n${testResult.errorOutput?.slice(0, 3000)}`,
      `Changes applied:\n${plan.changes.map(c => `${c.action} ${c.file}`).join('\n')}`,
    ].join('\n\n'),
    mode: 'diagnose',
    current_repo: ctx.repoSlug,
  }, agentContext);

  if (!diagnoseResponse.suggestedFix || diagnoseResponse.suggestedFix.length === 0) {
    return { fixed: false };
  }

  // Apply fix
  for (const fix of diagnoseResponse.suggestedFix) {
    try {
      await applyModification(path.join(ctx.worktreePath, fix.file), fix);
    } catch {
      return { fixed: false };
    }
  }

  // Re-run tests
  const retryResult = await runTests(ctx, await discoverAffectedTests(
    pool, graphClient, ctx.repoSlug, plan.changes.map(c => c.file)
  ));

  return { fixed: retryResult.passed, newTestResult: retryResult };
}
```

-----

## 9. PR Creation Pipeline

### 9.1 PR Description Generator

```typescript
// src/execution/pr_creator.ts

export async function createPullRequest(
  ctx: WorktreeContext,
  plan: PlanCheckpointPayload,
  testResult: TestResult
): Promise<PRCreateResult> {
  // Commit changes
  await execAsync(`git -C ${ctx.worktreePath} add -A`);

  const commitMsg = buildCommitMessage(plan);
  await execAsync(
    `git -C ${ctx.worktreePath} commit -m ${escapeShell(commitMsg)}`
  );

  // Push branch
  await execAsync(
    `git -C ${ctx.worktreePath} push origin ${ctx.branchName}`
  );

  // Generate PR description via fast model
  const description = await generatePRDescription(plan, testResult);

  // Create PR via Bitbucket API
  const pr = await bitbucketClient.createPullRequest({
    repoSlug: ctx.repoSlug,
    title: `[${plan.ticket_key}] ${plan.ticket_summary}`,
    description,
    sourceBranch: ctx.branchName,
    targetBranch: ctx.baseBranch,
    reviewers: await getDefaultReviewers(ctx.repoSlug),
  });

  // Link PR to Jira ticket
  await jiraClient.addRemoteLink(plan.ticket_key, {
    url: pr.links.html,
    title: pr.title,
    icon: { url16x16: 'https://bitbucket.org/favicon.ico' },
  });

  // Transition Jira ticket to "In Review"
  await jiraClient.transitionIssue(plan.ticket_key, 'In Review');

  return {
    prId: pr.id,
    prNumber: pr.id,
    prUrl: pr.links.html,
    branchName: ctx.branchName,
  };
}

function buildCommitMessage(plan: PlanCheckpointPayload): string {
  const lines = [
    `${plan.ticket_key}: ${plan.ticket_summary}`,
    '',
    plan.change_summary,
    '',
    `Generated by OME Agent (confidence: ${(plan.confidence.score * 100).toFixed(0)}%)`,
    '',
    `Changes:`,
    ...plan.changes.map(c => `- ${c.action} ${c.file}: ${c.description}`),
  ];

  if (plan.new_files && plan.new_files.length > 0) {
    lines.push('', 'New files:');
    plan.new_files.forEach(f => lines.push(`- ${f.path}`));
  }

  return lines.join('\n');
}

async function generatePRDescription(
  plan: PlanCheckpointPayload,
  testResult: TestResult
): Promise<string> {
  const sections = [
    `## ${plan.ticket_key}: ${plan.ticket_summary}`,
    '',
    `### Summary`,
    plan.change_summary,
    '',
    `### Changes`,
    ...plan.changes.map(c => `- **${c.action}** \`${c.file}\`: ${c.description}`),
  ];

  if (plan.new_files && plan.new_files.length > 0) {
    sections.push('', '### New Files');
    plan.new_files.forEach(f => sections.push(`- \`${f.path}\``));
  }

  sections.push(
    '',
    '### Blast Radius',
    `- Directly affected: ${plan.blast_radius.directly_affected.length} files`,
    `- Transitively affected: ${plan.blast_radius.transitively_affected.length} files`,
  );

  if (plan.blast_radius.cross_repo_impact.length > 0) {
    sections.push(
      `- **Cross-repo impact:** ${plan.blast_radius.cross_repo_impact.join(', ')}`
    );
  }

  sections.push(
    '',
    '### Test Results',
    testResult.passed ? '✅ All tests passed' : '⚠️ Some tests required auto-fix',
    `- Tests run: ${testResult.testCount?.total ?? 'N/A'}`,
    `- Passed: ${testResult.testCount?.passed ?? 'N/A'}`,
    `- Failed: ${testResult.testCount?.failed ?? 'N/A'}`,
    '',
    '### Agent Metadata',
    `- Confidence: ${(plan.confidence.score * 100).toFixed(0)}%`,
    `- Risk: ${plan.risk_level}`,
    `- Model: Qwen3.5-122B-A10B`,
    `- Conventions applied: ${plan.conventions_applied.join(', ') || 'none detected'}`,
    `- Similar PRs referenced: ${plan.similar_prs.map(p => `#${p.pr_number}`).join(', ') || 'none'}`,
    '',
    '---',
    '*This PR was generated by OME Agent. Plan was reviewed and approved by a human before execution.*',
  );

  return sections.join('\n');
}
```

-----

## 10. Database Schema — Phase 2 Extensions

```sql
-- Extend ome_agent schema for Phase 2

CREATE TABLE ome_agent.pipeline_runs (
  id                      UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  ticket_key              TEXT NOT NULL,
  ticket_summary          TEXT NOT NULL,
  ticket_type             TEXT NOT NULL,         -- Story, Task, Bug
  ticket_priority         TEXT,
  repo_slug               TEXT,
  status                  TEXT NOT NULL DEFAULT 'intake_received',
  complexity              TEXT,                  -- small, medium, large

  -- Enrichment
  enriched_description    TEXT,
  parent_ticket_key       TEXT,
  epic_key                TEXT,
  linked_pr_count         INTEGER DEFAULT 0,
  linked_doc_count        INTEGER DEFAULT 0,

  -- Plan
  plan_data               JSONB,                 -- Full PlanCheckpointPayload
  confidence_score        NUMERIC(4,3),
  risk_level              TEXT,
  auto_approve_eligible   BOOLEAN DEFAULT false,

  -- Checkpoint 1
  checkpoint_1_notified_at TIMESTAMPTZ,
  checkpoint_1_decided_at  TIMESTAMPTZ,
  checkpoint_1_decision    TEXT,                  -- approve, reject, modify
  checkpoint_1_reviewer    TEXT,
  checkpoint_1_notes       TEXT,

  -- Execution
  worktree_path           TEXT,
  branch_name             TEXT,
  base_commit             TEXT,
  execution_started_at    TIMESTAMPTZ,
  execution_completed_at  TIMESTAMPTZ,
  changes_applied         INTEGER DEFAULT 0,
  changes_failed          INTEGER DEFAULT 0,

  -- Testing
  tests_started_at        TIMESTAMPTZ,
  tests_completed_at      TIMESTAMPTZ,
  tests_passed            BOOLEAN,
  test_count_total        INTEGER,
  test_count_passed       INTEGER,
  test_count_failed       INTEGER,
  auto_fix_attempted      BOOLEAN DEFAULT false,
  auto_fix_succeeded      BOOLEAN,

  -- PR
  pr_id                   INTEGER,
  pr_url                  TEXT,
  pr_status               TEXT,                  -- open, merged, declined

  -- Metadata
  model_used              TEXT DEFAULT 'qwen3.5-122b-a10b',
  total_llm_tokens        INTEGER,
  total_rag_calls         INTEGER,
  total_ms                INTEGER,
  error_message           TEXT,
  created_at              TIMESTAMPTZ DEFAULT NOW(),
  completed_at            TIMESTAMPTZ
);

CREATE INDEX idx_pipeline_status ON ome_agent.pipeline_runs (status);
CREATE INDEX idx_pipeline_ticket ON ome_agent.pipeline_runs (ticket_key);
CREATE INDEX idx_pipeline_created ON ome_agent.pipeline_runs (created_at DESC);

-- Detailed change tracking
CREATE TABLE ome_agent.pipeline_changes (
  id                UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  pipeline_run_id   UUID REFERENCES ome_agent.pipeline_runs(id) ON DELETE CASCADE,
  change_id         TEXT NOT NULL,              -- "change-001"
  file_path         TEXT NOT NULL,
  action            TEXT NOT NULL,              -- modify, insert, replace, delete
  description       TEXT,
  status            TEXT DEFAULT 'pending',     -- pending, applied, hash_mismatch, failed
  content_hash      TEXT,                       -- MD5 of currentCode for verification
  error_message     TEXT,
  applied_at        TIMESTAMPTZ
);

CREATE INDEX idx_changes_run ON ome_agent.pipeline_changes (pipeline_run_id);

-- Audit log — every state transition
CREATE TABLE ome_agent.pipeline_audit (
  id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  pipeline_run_id UUID REFERENCES ome_agent.pipeline_runs(id) ON DELETE CASCADE,
  from_status     TEXT,
  to_status       TEXT NOT NULL,
  actor           TEXT NOT NULL,                -- 'system', 'webhook', reviewer username
  details         JSONB,
  created_at      TIMESTAMPTZ DEFAULT NOW()
);

CREATE INDEX idx_audit_run ON ome_agent.pipeline_audit (pipeline_run_id, created_at);

-- Pipeline metrics aggregation
CREATE TABLE ome_agent.pipeline_metrics (
  id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  date            DATE NOT NULL,
  tickets_received    INTEGER DEFAULT 0,
  plans_generated     INTEGER DEFAULT 0,
  plans_approved      INTEGER DEFAULT 0,
  plans_rejected      INTEGER DEFAULT 0,
  plans_expired       INTEGER DEFAULT 0,
  executions_succeeded INTEGER DEFAULT 0,
  executions_failed   INTEGER DEFAULT 0,
  prs_created         INTEGER DEFAULT 0,
  prs_merged          INTEGER DEFAULT 0,
  avg_confidence      NUMERIC(4,3),
  avg_plan_to_approve_minutes INTEGER,
  avg_total_pipeline_minutes  INTEGER,
  UNIQUE (date)
);
```

-----

## 11. Confidence Gate System

The confidence gate drives the entire pipeline’s safety posture.

```
Confidence ≥ 0.9 AND risk = low:
  → Auto-approve eligible (if enabled via ALLOW_AUTO_APPROVE=true)
  → Still creates PR for human review (Checkpoint 2)

Confidence 0.7 - 0.89:
  → Requires human plan approval (Checkpoint 1)
  → Proceed normally after approval

Confidence 0.5 - 0.69:
  → Requires human plan approval
  → WARNING banner in review UI
  → Suggests specific areas to scrutinize

Confidence < 0.5:
  → Pipeline ABORTS
  → Jira comment: "Ticket too complex for automated implementation"
  → Recommend manual implementation with ome_rag_guide in IDE
```

-----

## 12. Observability Dashboard

```
GET /api/pipeline/dashboard
```

Dark-theme dashboard showing:

- Pipeline funnel: tickets received → plans generated → approved → executed → PRs merged
- Success rate by week (bar chart)
- Average confidence score trend (line chart)
- Time-in-stage breakdown (plan generation, human review wait, execution, testing)
- Failed pipelines with root cause (hash mismatch, test failure, model error)
- Top repositories by pipeline volume
- Model performance metrics (tok/s, latency p50/p95, token usage)

-----

## 13. Rollback & Safety

### 13.1 Automatic Rollback Triggers

|Trigger                          |Action                                             |
|---------------------------------|---------------------------------------------------|
|Content hash mismatch on any file|Reset worktree, status → `execution_hash_mismatch` |
|Lint failure                     |Reset worktree, status → `execution_lint_failed`   |
|Compilation failure              |Reset worktree, status → `execution_compile_failed`|
|Test failure after auto-fix retry|Reset worktree, status → `auto_fix_failed`         |
|Exception during execution       |Reset worktree, status → `execution_error`         |

### 13.2 Rollback Implementation

```typescript
async function rollback(ctx: WorktreeContext, runId: string, reason: string): Promise<void> {
  // 1. Reset worktree to base commit
  await execAsync(`git -C ${ctx.worktreePath} reset --hard ${ctx.baseCommit}`);
  await execAsync(`git -C ${ctx.worktreePath} clean -fd`);

  // 2. Remove worktree
  await worktreeManager.removeWorktree(ctx);

  // 3. Update pipeline status
  await updateStatus(runId, 'rolled_back');

  // 4. Notify
  await notifyRollback(runId, reason);

  // 5. Audit log
  await auditLog(runId, 'rollback', 'system', { reason });
}
```

### 13.3 Safety Invariants

1. **No direct push to main/develop** — always via PR branch
1. **No force push** — ever
1. **No execution without plan approval** (unless auto-approve criteria met AND feature flag enabled)
1. **No merge without human PR review** — always requires at least 1 Bitbucket approval
1. **48-hour plan expiry** — prevents stale plans from executing on changed code
1. **Content hash verification** — prevents applying changes to modified files
1. **Single retry limit** — auto-fix attempts exactly once, then escalates
1. **Audit trail** — every state transition logged with actor and timestamp
1. **Feature flag kill switch** — `PIPELINE_ENABLED=false` stops all new pipeline runs

-----

## 14. Implementation Schedule

```
Week 1: Infrastructure (Days 1-5)

  Day 1: Model Migration
    ├── Download Qwen3.5-122B-A10B Q4_K_M via aria2c
    ├── Deploy on :8091 (replace 235B)
    ├── Verify inference speed (target: 50+ tok/s)
    ├── Verify 3 parallel slots work
    ├── Update systemd service
    ├── Benchmark: IMPLEMENT mode latency comparison
    └── Update DualLLMClient model_router for 122B

  Day 2: Model Swap Orchestrator (if dual-load not feasible)
    ├── ModelSwapManager implementation
    ├── Health check + readiness probe
    ├── Batch-oriented swap strategy
    ├── Fallback: if swap fails, run deep-only mode
    └── Test: swap cycle under load

  Day 3: Database Schema + Jira Webhook
    ├── Phase 2 schema migration (pipeline_runs, pipeline_changes, pipeline_audit)
    ├── Jira webhook handler
    ├── Webhook signature verification
    ├── Eligibility checker
    └── Test: webhook → intake_received flow

  Day 4: Git Worktree Manager
    ├── WorktreeManager class
    ├── Create/remove worktree lifecycle
    ├── Branch naming convention (ome-agent/{ticket_key})
    ├── Base commit recording for rollback
    └── Test: create worktree, modify file, rollback

  Day 5: Notification System
    ├── Slack integration (webhook or API)
    ├── Jira comment posting
    ├── Email digest (optional)
    ├── Notification templates
    └── Test: end-to-end notification flow

Week 2: Core Pipeline (Days 6-10)

  Day 6: Ticket-to-Plan Pipeline
    ├── Plan generator (calls ome_rag_guide IMPLEMENT)
    ├── Confidence computation
    ├── Risk assessment
    ├── Auto-approval eligibility check
    └── Test: 3 golden tickets → plan generation

  Day 7: Human Checkpoint System
    ├── Checkpoint 1 state machine
    ├── Plan review REST API
    ├── Review Web UI (dark theme, diff viewer)
    ├── Approval/rejection/modification handlers
    ├── Plan expiry cron job (48h SLA)
    └── Test: approve → execution flow, reject → terminal

  Day 8: Change Applicator
    ├── Content hash verification
    ├── Fuzzy line matching for shifted content
    ├── Change application (modify/insert/replace/delete)
    ├── Lint runner (ESLint for TS, Checkstyle for Java)
    ├── Compilation runner (tsc, mvn compile)
    └── Test: apply 5 changes to test repo, verify diffs

  Day 9: Test Runner + Auto-Fix
    ├── Affected test discovery (test_coverage + naming + blast radius)
    ├── Test execution (Jest/Vitest for MFE, Maven for MS)
    ├── Test result parser
    ├── Auto-fix via DIAGNOSE mode (1 retry)
    ├── Rollback on persistent failure
    └── Test: introduce intentional test failure, verify auto-fix

  Day 10: PR Creation
    ├── Bitbucket API integration (create PR, add reviewers)
    ├── PR description generator (via fast model)
    ├── Commit message builder
    ├── Jira ticket linking (remote link + transition)
    ├── Worktree cleanup after PR creation
    └── Test: end-to-end ticket → PR on test repo

Week 3: Quality & Safety (Days 11-15)

  Day 11: Rollback & Safety
    ├── Automatic rollback on all failure conditions
    ├── Audit trail (every state transition)
    ├── Kill switch (PIPELINE_ENABLED feature flag)
    ├── Content hash mismatch handling
    └── Test: simulate every failure mode → verify rollback

  Day 12: Observability Dashboard
    ├── Pipeline metrics aggregation
    ├── Dashboard REST API
    ├── HTML dashboard (funnel, trends, failures)
    ├── Model performance metrics
    └── Slack daily digest (pipelines run, success rate)

  Day 13: Integration Testing — MFE Repos
    ├── Pick 2 React MFE repos
    ├── Create test Jira tickets (small, medium complexity)
    ├── Run end-to-end pipeline
    ├── Verify: plan quality, code quality, test execution
    └── Tune: confidence thresholds, risk factors

  Day 14: Integration Testing — MS Repos
    ├── Pick 2 Spring Boot MS repos
    ├── Create test Jira tickets (bug fix, feature, refactor)
    ├── Run end-to-end pipeline
    ├── Verify: Maven build, Java test execution
    └── Tune: eligibility criteria, auto-fix effectiveness

  Day 15: Polish + Documentation
    ├── Edge cases (empty repo, no tests, huge file, concurrent tickets)
    ├── Performance profiling (target: plan in <90s, execute in <60s)
    ├── Pipeline monitoring alerts
    ├── Documentation: runbook, architecture, configuration
    └── Team walkthrough + demo

Week 4 (Buffer): Stabilization

  Days 16-18:
    ├── Address issues found in Week 3 testing
    ├── Refine Slack notification templates
    ├── A/B test: auto-approve vs always-review
    ├── Collect metrics from first 10 real tickets
    └── Calibrate confidence thresholds from real-world data
```

-----

## 15. Configuration

```bash
# ═══ Phase 2 Pipeline ═══
PIPELINE_ENABLED=true                      # Kill switch
ALLOW_AUTO_APPROVE=false                   # Start with false, enable after calibration
PLAN_EXPIRY_HOURS=48                       # Plans expire after 48h
AUTO_FIX_MAX_RETRIES=1                     # Single retry on test failure
MIN_CONFIDENCE_FOR_EXECUTION=0.5           # Below this, abort pipeline
MIN_CONFIDENCE_FOR_AUTO_APPROVE=0.9        # Must be this high for auto-approve
MAX_STORY_POINTS_FOR_AUTOMATION=8          # Tickets above this are ineligible

# ═══ Jira Integration ═══
JIRA_WEBHOOK_SECRET=<shared-secret>
JIRA_BASE_URL=https://your-jira.atlassian.net
JIRA_API_TOKEN=<api-token>
JIRA_TRIGGER_STATUS=Ready for Dev          # Status transition that triggers pipeline
JIRA_REVIEW_STATUS=In Review               # Status set after PR creation
PIPELINE_SLACK_CHANNEL=#ome-agent-pipeline

# ═══ Bitbucket Integration ═══
BITBUCKET_API_URL=https://bitbucket.your-org.com/rest/api/1.0
BITBUCKET_API_TOKEN=<api-token>
BITBUCKET_DEFAULT_REVIEWERS=user1,user2    # Comma-separated

# ═══ Git Worktree ═══
WORKTREE_BASE=/tmp/ome-agent-worktrees
WORKTREE_BASE_BRANCH=develop               # Default target branch

# ═══ Deep Model (Revised) ═══
DEEP_MODEL_URL=http://127.0.0.1:8091       # Now Qwen3.5-122B-A10B
DEEP_MODEL_NAME=qwen3.5-122b-a10b
DEEP_MODEL_TIMEOUT_MS=120000

# ═══ Test Execution ═══
TEST_TIMEOUT_MS=600000                     # 10 minutes for Java
LINT_TIMEOUT_MS=60000                      # 1 minute
COMPILE_TIMEOUT_MS=300000                  # 5 minutes
```

-----

## 16. File Structure — Phase 2 Additions

```
ome-agent/src/
  ├── pipeline/                            # NEW — Phase 2
  │   ├── jira_webhook.ts                  # Webhook handler + eligibility
  │   ├── plan_generator.ts                # Enrichment → plan via IMPLEMENT mode
  │   ├── confidence.ts                    # Confidence computation
  │   ├── risk.ts                          # Risk assessment
  │   ├── state_machine.ts                 # Pipeline status transitions
  │   └── pipeline_orchestrator.ts         # End-to-end pipeline coordinator
  │
  ├── execution/                           # NEW — Phase 2
  │   ├── worktree_manager.ts              # Git worktree lifecycle
  │   ├── change_applicator.ts             # Apply CodeChange[] to files
  │   ├── content_verifier.ts              # Hash verification + fuzzy match
  │   ├── lint_runner.ts                   # ESLint / Checkstyle
  │   ├── compile_runner.ts                # tsc / mvn compile
  │   ├── test_runner.ts                   # Discover + run affected tests
  │   ├── auto_fixer.ts                    # DIAGNOSE mode retry
  │   ├── pr_creator.ts                    # Bitbucket PR creation
  │   ├── rollback.ts                      # Worktree rollback
  │   └── checkpoint.ts                    # Human checkpoint logic
  │
  ├── integrations/                        # NEW — Phase 2
  │   ├── jira_client.ts                   # Jira API (comments, transitions, links)
  │   ├── bitbucket_client.ts              # Bitbucket API (PRs, reviewers)
  │   └── slack_client.ts                  # Slack notifications
  │
  ├── llm/
  │   ├── model_swap.ts                    # NEW — GPU model swap orchestrator
  │   ├── dual_client.ts                   # Updated for 122B
  │   └── model_router.ts                  # Updated routing
  │
  ├── api/
  │   ├── pipeline_routes.ts               # NEW — review UI, dashboard
  │   └── pipeline_dashboard.ts            # NEW — HTML dashboard renderer
  │
  ├── db/
  │   └── migrations/
  │       ├── 001_initial.sql              # Existing
  │       └── 002_phase2_pipeline.sql      # NEW — pipeline tables
  │
  └── jobs/
      ├── plan_expiry.ts                   # NEW — 48h expiry cron
      └── daily_metrics.ts                 # NEW — aggregate pipeline metrics
```

-----

## Summary — Why This Approach Produces Accurate Results

1. **Model upgrade (Qwen3.5-122B):** +7 points SWE-bench, +30% tool calling, 3x faster — better code generation with lower latency.
1. **Dual human checkpoints:** Plan review catches bad plans before any code is touched. PR review catches execution errors. No code reaches main/develop without human eyes.
1. **Content hash verification:** Every change is verified against the actual file state. Hash mismatch → abort and re-plan. Prevents applying changes to stale code.
1. **Confidence-gated execution:** Pipeline aborts if confidence < 0.5. Auto-approve only when confidence ≥ 0.9 AND risk is low AND changes are small. Conservative by default.
1. **Sparse ticket enrichment:** Climbs Epic→Story hierarchy, references linked PRs and Confluence docs. Gives the model actual context instead of “implement as discussed in standup.”
1. **Test-driven verification:** Discovers affected tests via test_coverage table + naming conventions + blast radius graph traversal. Runs tests in isolated worktree. Auto-fix gets exactly one retry.
1. **Full audit trail:** Every state transition logged. Every rollback recorded. Dashboard shows funnel metrics. Daily digest to Slack.
1. **Kill switch:** `PIPELINE_ENABLED=false` stops everything. No silent execution. No surprises.

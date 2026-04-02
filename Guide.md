# Reducing Token Usage in Windsurf AI via `.windsurfrules`

Windsurf charges per message, so every unnecessary back-and-forth, verbose explanation, or confirmation loop costs real money. A well-crafted `.windsurfrules` file can cut your per-task message count by **40–60%** by eliminating wasteful Cascade behaviors at the source.

-----

## 1. Constrain Response Verbosity

The single biggest token drain is Cascade explaining things you didn’t ask for.

```yaml
# .windsurfrules
response_style:
  - Never repeat the question or restate context back to me
  - Skip preambles, summaries, and sign-offs
  - Only show changed lines, not entire files
  - Use diff format for code modifications
  - Maximum explanation length: 3 sentences unless I ask for detail
  - No markdown formatting in terminal commands
```

**Impact:** Saves 30–50% of output tokens per message.

-----

## 2. Pre-Load Persistent Context

Every time you re-explain your stack in chat, you’re burning input tokens. Embed it once in rules.

```yaml
project_context:
  language: TypeScript
  runtime: Node.js
  database: Postgres with pgvector
  os: RHEL (no sudo access)
  package_manager: npm
  ide: Windsurf (Cascade)
  inference: llama.cpp server (local, CPU-only)
  vector_store: pgvector (not ChromaDB, not Pinecone)
  test_framework: vitest  # or your actual framework
  style: ESLint + Prettier

assumptions:
  - Always assume this stack unless explicitly told otherwise
  - Never suggest cloud-hosted AI services or external APIs
  - Never suggest Docker if the task can be done without it
```

**Impact:** Eliminates 1–2 setup messages per conversation.

-----

## 3. Enforce Tool-First, Talk-Later

Cascade defaults to explaining what it *would* do. Force it to just do it.

```yaml
execution_policy:
  - Prefer using tools (terminal, file edit, search) over explaining what you would do
  - Do not ask for confirmation before making changes unless the operation is destructive (delete, drop, reset)
  - Never list multiple approaches — pick the best one and execute immediately
  - If you need to read a file, read it — don't tell me you're going to read it
  - Execute first, explain only if the result is unexpected
```

**Impact:** Cuts 2–4 confirmation round-trips per task.

-----

## 4. Batch Operations

One-at-a-time fixes are the most expensive anti-pattern.

```yaml
batch_policy:
  - When fixing lint/type errors, fix ALL errors in a single pass
  - When creating related files, create all of them in one step
  - Combine related changes into a single edit operation
  - When running commands, chain them with && rather than separate messages
  - When updating imports across files, do all files in one pass
```

**Impact:** Turns 5–10 messages into 1–2 for multi-file changes.

-----

## 5. Suppress Unnecessary Behaviors

Cascade has helpful defaults that become expensive at scale.

```yaml
suppress:
  - Do not explain code you just wrote unless I ask "explain" or "why"
  - Do not suggest next steps unless I ask "what next"
  - Do not repeat tool output back to me — I can see it
  - Do not show the full file after a small edit
  - Do not add comments to code unless the logic is non-obvious
  - Do not create README or documentation files unless asked
  - Do not run tests unless I ask or the task explicitly requires validation
```

**Impact:** Eliminates 1–3 unnecessary messages per task.

-----

## 6. Short-Circuit Common Patterns

Teach Cascade to handle routine issues autonomously.

```yaml
auto_fix:
  - For import errors: fix them silently without reporting
  - For TypeScript type errors: infer the correct type, don't ask me
  - For missing dependencies: install them and continue
  - If a file doesn't exist and is needed: create it with sensible defaults
  - For ESLint warnings: auto-fix on save, don't report unless unfixable
  - For missing environment variables: check .env.example, copy the pattern

auto_resolve:
  - Merge conflicts in lock files: accept incoming
  - Port already in use: kill the process and retry
  - Module not found after install: clear node_modules/.cache and retry once
```

**Impact:** Saves 1–2 messages per routine error.

-----

## 7. Scope File Reading

Large file reads burn context tokens fast.

```yaml
file_reading:
  - When reading files for context, read only the relevant function or class
  - Use grep/ripgrep to locate code before reading full files
  - Maximum context per file read: 200 lines around the target
  - For large files (>500 lines): always use line-range reads, never read the whole file
  - Prefer AST-based navigation over full-file scanning
  - Cache file structure mentally — don't re-read files you've already seen in this session
```

**Impact:** Reduces input token consumption by 20–40% on large codebases.

-----

## 8. Smart MCP and Tool Failure Handling

Don’t waste a message just to report a failure.

```yaml
error_handling:
  on_tool_failure: retry once with adjusted parameters before reporting
  on_mcp_failure: log to .windsurf/errors.log and halt silently
  on_network_timeout: retry with exponential backoff (max 2 retries)
  on_ambiguous_error: include the full error message in your report (don't summarize it)

  never:
    - Don't send a message that only says "I encountered an error"
    - Don't ask "should I retry?" — just retry once automatically
```

**Impact:** Eliminates 1–2 error-reporting round-trips.

-----

## 9. Message-Efficient Communication Protocol

Define how you want to interact with Cascade.

```yaml
communication:
  - If I send a one-liner command like "fix types" or "add tests", treat it as a complete instruction
  - Don't ask clarifying questions if you can make a reasonable assumption — state the assumption and proceed
  - When I paste an error, fix it. Don't analyze it first unless the fix isn't obvious
  - Use inline comments in code for explanations, not separate chat messages
  - If a task has multiple steps, do all steps in one message, not sequentially

shorthand:
  "ft": "fix all TypeScript errors in the current file"
  "lint": "run eslint --fix on changed files"
  "test": "run tests for the file I'm currently editing"
  "clean": "remove unused imports and variables"
```

**Impact:** Reduces average messages per task from 5–8 to 2–3.

-----

## 10. Phase-Aware Token Budgets

For multi-phase workflows, set token expectations per phase.

```yaml
phase_budgets:
  planning:
    max_messages: 3
    output_format: bullet points only
    no_code_in_this_phase: true

  implementation:
    max_messages_per_file: 2
    show_only: diffs
    auto_proceed: true  # don't wait for confirmation between files

  review:
    max_messages: 2
    format: "table of issues with severity and line numbers"

  debugging:
    max_retries: 3
    on_max_retries: "stop and summarize what you've tried"
```

-----

## 11. Smart Ask vs Auto-Decide Policy

Every question Cascade asks costs a message. But eliminating *all* questions is risky for destructive operations. The optimal strategy is a **hybrid approach** — auto-decide for predictable choices, ask only when the decision is costly to reverse, and when you do ask, minimize the reply tokens.

### Why “Ask with Options” Beats “Ask Open-Ended”

|Pattern                           |Messages Used                   |Your Reply Tokens|
|----------------------------------|--------------------------------|-----------------|
|Cascade asks open-ended question  |2 (question + your typed answer)|20–50 tokens     |
|Cascade presents numbered options |2 (question + your “1”)         |1–2 tokens       |
|Cascade auto-decides (no question)|0                               |0 tokens         |

**Auto-decide saves the most**, but numbered options are the next best — your reply is just a number instead of a sentence.

### Recommended Rules

```yaml
ask_policy:
  # HIGH-RISK: Always ask before these (costly to reverse)
  always_ask:
    - Before deleting files or dropping tables
    - When multiple valid architectures exist (e.g., REST vs GraphQL)
    - Before changes affecting >10 files
    - Before modifying shared/core modules used by other teams
    - Before altering database schemas in production

  # LOW-RISK: Never ask for these (just decide and go)
  never_ask:
    - Import style or formatting choices (follow existing patterns)
    - Which test framework (use the one already in package.json)
    - Type inference decisions
    - Error fix approaches (just pick the best one)
    - Variable naming conventions (follow existing codebase style)
    - File placement (follow existing project structure)
    - Whether to add error handling (always add it)

  # WHEN you must ask, ask efficiently
  ask_format:
    - Present choices as a numbered list (1, 2, 3)
    - Maximum 3 options — never more
    - Mark your recommendation with ★
    - Keep each option to one line
    - Example format: |
        Pick an approach:
        1. ★ Add index on user_id (fastest, no schema change)
        2. Denormalize the table (faster reads, migration needed)
        3. Add Redis cache layer (most scalable, new dependency)

  # AFTER the user picks, don't echo the choice back — just execute
  post_ask:
    - Do not confirm the selection ("You chose option 2...")
    - Do not re-explain the chosen option
    - Immediately execute the chosen approach
    - If the user replies with just a number, treat it as the full instruction
```

### Anti-Patterns to Avoid

```yaml
# BAD: Cascade asks vague open-ended questions (costs you typing tokens)
# "How would you like me to handle the authentication? Should I use JWT
#  or session-based auth? And what about refresh tokens?"

# GOOD: Cascade presents concise numbered options
# "Auth approach:
#  1. ★ JWT + refresh tokens (stateless, fits your API pattern)
#  2. Session-based (simpler, needs Redis)
#  Pick one:"

# BEST: Cascade auto-decides based on codebase context
# *silently picks JWT because the existing codebase already uses it,
#  implements it, and moves on*
```

**Impact:** Saves 1–3 messages per decision point. For a typical feature implementation with 3–4 decision points, this saves 4–12 messages.

-----

## Summary: Expected Savings

|Strategy                |Token Reduction|Message Reduction      |
|------------------------|---------------|-----------------------|
|Constrain verbosity     |30–50% output  |—                      |
|Pre-load context        |10–15% input   |1–2 per conversation   |
|Tool-first execution    |—              |2–4 per task           |
|Batch operations        |—              |3–8 per multi-file task|
|Suppress explanations   |20–30% output  |1–3 per task           |
|Short-circuit fixes     |—              |1–2 per error          |
|Scope file reads        |20–40% input   |—                      |
|Smart error handling    |—              |1–2 per failure        |
|Communication protocol  |—              |2–5 per task           |
|Phase budgets           |15–25% overall |varies                 |
|Smart ask vs auto-decide|5–10% input    |1–3 per decision point |

**Combined effect:** A well-optimized `.windsurfrules` can reduce your total Windsurf spend by **40–60%** while maintaining output quality.

-----

> **Tip:** Start with sections 1, 3, and 5 — they deliver the highest impact with the least effort. Layer in the rest as you identify specific patterns that waste messages in your workflow.

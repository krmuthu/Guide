# Financial Integrity Scan — Windsurf Cascade Prompt

> **Usage:** Copy the prompt below and paste it into Windsurf AI Cascade.
> **Prerequisites:** OME RAG MCP server running on :3000 with all tools available. The 4 new MCP tools (`start_financial_scan`, `store_financial_finding`, `complete_financial_scan`, `query_financial_findings`) registered and DB schema created.
> **If MCP storage tools are NOT yet available:** Use the “Lite Mode” prompt at the bottom — it generates a markdown report directly without DB storage.

-----

## Full Prompt (with DB storage)

```
Run a comprehensive financial integrity scan across all backend repositories.

## Your Task

You are a senior financial software auditor. Scan every backend/fullstack repository for financial transaction safety issues. Work through repos one at a time, file by file. Store every finding in the database via MCP tools. Generate a final report.

## Phase 1: Discover Repos and Initialize

1. Call `get_project_info` (no params) to list ALL repositories
2. Filter to repos where project_type is "backend" or "fullstack"
3. For each repo, you will run a separate scan

## Phase 2: Per-Repo Scan

For each repo:

### Step 2a: Initialize scan
- Call `get_project_info` with the repo_slug to get tech_stack, dependencies
- Note: Does this repo use Spring TX (`spring-boot-starter-data-jpa` or `spring-tx` in dependencies)? Does it have `resilience4j` or `spring-retry`? Does it have a business-day library (`jollyday`, `time4j`)?
- Call `get_repo_config` with the repo_slug to check application.yml for TX isolation level, scheduler config, retry config
- Call `start_financial_scan` with the repo_slug to get a scan_id

### Step 2b: Discover financial files
- Call `search_files` with pattern="*Service.java" and the repo_slug
- Call `search_files` with pattern="*Processor.java"
- Call `search_files` with pattern="*Handler.java"
- Call `search_files` with pattern="*Controller.java"
- Combine results. Exclude files containing: test, Test, spec, mock, Mock, fixture, generated, __test__, __spec__, config/Config (unless it's a financial config), dto/DTO, entity/Entity (unless name contains Transaction/Ledger/Account)

### Step 2c: Pre-screen each file
For each candidate file:
- Call `get_file_skeleton` with repo_slug and file_path
- Check method names and annotations
- KEEP the file only if ANY of these are true:
  - Method names contain: debit, credit, sell, buy, disburse, repay, place, withdraw, subscribe, redeem, rollover, maturity, settle, transfer, charge, refund, allocate, close, open, exercise, coupon, accrue, capitalize, drawdown, pledge, release, waive, forfeit, break, terminate, renew, payout
  - Method names contain: process + (Transaction|Payment|Settlement|Order|Trade|Loan|Deposit|Fund|Bond|Position|Maturity|Rollover|Coupon|Repayment|Disbursement|Redemption)
  - Annotations include: @Transactional on financial-named methods
  - Annotations include: @Scheduled on methods with maturity/expiry/coupon/rollover/repayment in name
  - Class name contains: Transaction, Ledger, Account, Payment, Settlement, Trade, Order, Position, Loan, Deposit, Fund, Bond, Maturity, Rollover
- SKIP all other files

### Step 2d: Analyze each financial file

For each financial file:

1. Call `read_full_file` with repo_slug and file_path

2. If file is >500 lines: use `get_file_skeleton` to identify financial methods, then use `get_chunks_for_symbol` for each financial method individually instead of analyzing the whole file at once

3. Analyze the code for ALL 5 checks below. For each potential finding, verify before storing.

---

## The 5 Checks

### CHECK 1: Unbalanced Paired Operations
**Category:** debit_credit
**Check types:** single_legged_debit, single_legged_credit, loop_debit_credit, duplicate_debit

Look for methods that perform an OUTFLOW operation without a matching INFLOW (or vice versa):

OUTFLOW: debit, withdraw, sell, shortSell, closePosition, deallocate, exerciseOption, disposeHolding, reducePosition, disburse, disburseLoan, drawdown, releaseCollateral, writeOff, waiveFee, forgivePrincipal, place, placeDeposit, breakDeposit, earlyWithdraw, terminateDeposit, closeDeposit, redeem, cancelUnits, collectFee, distributeDividend, surrenderUnits, payoutCoupon, payInterest, redeemBond, transferOut, settlePayable, remit, payOut

INFLOW: credit, deposit, buy, purchase, openPosition, allocate, assignOption, coverShort, addHolding, increasePosition, repay, repayLoan, paydown, pledgeCollateral, collectRepayment, accrue, accrueInterest, capitalize, collectInstallment, creditInterest, renewDeposit, subscribe, allocateUnits, rebate, reinvestDividend, issueUnits, receiveCoupon, collectInterest, purchaseBond, transferIn, settleReceivable, receive, payIn

Also flag:
- Same debit/credit method called twice in same method without being in mutually exclusive branches → duplicate_debit
- Debit/credit inside a for/while/forEach/stream loop without idempotency check (processedIds, transactionId, contains, computeIfAbsent, ON CONFLICT, MERGE INTO) → loop_debit_credit

**Verification before storing:**
- Call `get_scip_callers` for the method — does any caller provide the missing counterpart?
- Call `get_scip_callees` — does the method internally call the counterpart?
- Call `suggest_related_files` — is there a co-change partner that's the counterpart service?

**DO NOT flag these (intentional single-leg):**
- Fee collection (collectFee without matching rebate)
- Interest accrual (accrueInterest as standalone)
- Dividend payout from fund NAV
- Saga/choreography pattern where counterpart is in a different service (confirmed by SCIP callers or suggest_related_files showing paired service)
- Methods named process* that delegate to sub-methods which have the counterpart

---

### CHECK 2: Transaction Rollback Integrity
**Category:** transaction
**Check types:** multi_write_no_transaction, multi_write_single_repo_no_transaction, commit_without_rollback, save_then_throw, readonly_transaction_with_writes, cross_service_partial_commit

Look for:
- Public method with 2+ calls to repository.save/saveAll/saveAndFlush/delete/deleteAll/deleteById/update/insert/persist/merge/executeUpdate on different repository objects WITHOUT @Transactional → multi_write_no_transaction (MAJOR)
- Same but on same repository, 2+ writes → multi_write_single_repo_no_transaction (MODERATE)
- Manual .commit() without .rollback() in catch path → commit_without_rollback (MAJOR)
- repository.save() followed by throw in same method without @Transactional → save_then_throw (MAJOR if unconditional throw, MODERATE if conditional)
- @Transactional(readOnly = true) on method that calls save/delete/update → readonly_transaction_with_writes (MAJOR)
- Method writes to local DB AND makes HTTP call (RestTemplate, WebClient, FeignClient) to downstream service → cross_service_partial_commit (MAJOR)

**Also flag:**
- @Transactional on a private method (Spring proxy won't intercept)
- Self-invocation: this.otherTransactionalMethod() inside a @Transactional method (proxy bypass)

**DO NOT flag if:**
- @Transactional is on the CLASS (applies to all public methods)
- @Transactional is inherited from parent class — call `get_scip_implementations` or check extends/implements
- The save calls are on non-DB objects (cache, logger, audit, session, file, preference)
- Project does NOT have Spring TX in dependencies (not a Spring app)

---

### CHECK 3: Unclosed Conditions
**Category:** conditions
**Check types:** if_chain_no_else, switch_no_default, enum_switch_missing_cases

Look for:
- if / else if / else if chain (2+ branches) WITHOUT a final else block → if_chain_no_else
- switch statement with 2+ cases WITHOUT a default case → switch_no_default
- switch on an enum type that doesn't handle all enum values → enum_switch_missing_cases

For enum switches: call `get_scip_type_definition` with the enum type name to get ALL values. Compare against handled cases.

**Severity escalation to MAJOR** when the condition variable is financial-sensitive:
type, transactionType, txnType, entryType, side, direction, action, operationType, movementType, status, product, productType, instrumentType, assetClass, assetType, securityType, tenor, maturity, rate, rateType, interestType, currency, ccy, amount, notional, principal, category, tier, channel, segment, tranche, facility, loanType, depositType, fundType, accountType

Otherwise MODERATE.

**DO NOT flag:**
- Simple if (without else-if) — only chains with 2+ branches
- equals(), hashCode(), toString(), compareTo(), clone() methods
- Builder patterns, fluent API chains
- Validation-only methods that return boolean

---

### CHECK 4: Unhandled Errors / Sudden Termination
**Category:** errors
**Check types:** empty_catch, catch_log_no_rethrow, catch_returns_null, system_exit_in_service, resource_no_finally, async_void_no_handler, broad_catch_in_financial

Look for:
- catch block with no statements (ignoring comments) → empty_catch (MAJOR)
- catch block that only logs (log.error/warn) but does NOT rethrow AND the method is in a financial file → catch_log_no_rethrow (MODERATE)
- catch block that returns null in a financial method → catch_returns_null (MAJOR)
- System.exit() or process.exit() in a non-main method → system_exit_in_service (MAJOR)
- try block that acquires resource (getConnection, openSession, acquire, lock, createStatement, prepareStatement) without finally block and NOT using try-with-resources → resource_no_finally (MODERATE)
- @Async method returning void without AsyncUncaughtExceptionHandler in the class → async_void_no_handler (MAJOR)
- catch(Exception) or catch(Throwable) in financial method that does NOT rethrow → broad_catch_in_financial (MODERATE)

**DO NOT flag:**
- catch blocks that delegate to handleError(), onError(), reportError(), errorHandler(), notifyError()
- catch blocks in utility/helper methods that are not financial
- @Async methods returning CompletableFuture/Future (exception preserved)
- main() methods or @SpringBootApplication classes for System.exit

---

### CHECK 5: Maturity / Expiry Without Handler
**Category:** maturity
**Check types:** maturity_no_equals_check, maturity_no_business_day, rollover_missing_open_leg, rollover_missing_close_leg, scheduled_maturity_no_retry, scheduled_maturity_no_alert

Look for:
- Method compares a maturity/expiry/settlement/coupon/rollover/repayment date using .isBefore() or .isAfter() but NEVER checks .isEqual() or .equals() and does NOT use !isAfter() (which is inclusive) → maturity_no_equals_check (MAJOR)
- Method compares maturity dates but has NO business-day adjustment anywhere in the method, class, or imports → maturity_no_business_day (MAJOR)
- Rollover method that closes/terminates old instrument but does NOT create/open new one → rollover_missing_open_leg (MAJOR)
- Rollover method that creates new instrument but does NOT close old one → rollover_missing_close_leg (MAJOR)
- @Scheduled method processing maturities/coupons/rollovers/repayments without retry mechanism (no @Retryable, no retry template, no dead letter queue) → scheduled_maturity_no_retry (MAJOR)
- @Scheduled maturity processor without alerting on failure (no alert/notify/email/slack pattern) → scheduled_maturity_no_alert (MODERATE)

**CRITICAL — DO NOT flag these (this was a known false positive source):**
- Helper methods: isSettlementHoliday(), isWeekend(), isBusinessDay(), isHoliday(), isWorkingDay(), isValid(), isActive(), isEmpty() — these are NOT date comparisons
- ANY method whose name starts with is, has, get, derive, calculate, compute, check, validate AND returns boolean — these are utility methods
- Classes that CONTAIN isWeekend/isHoliday/isBusinessDay/isSettlementHoliday helper methods — business-day handling EXISTS in the class
- .isBefore()/.isAfter() calls where the receiver is NOT a date variable (maturityDate, expiryDate, settlementDate, valueDate, today, now, LocalDate, etc.)
- Use of !isAfter() — this IS inclusive (equivalent to isBeforeOrEqual)

**Verification:**
- Call `get_file_dependencies` — is CalendarService, BusinessDayService, or HolidayCalendar imported?
- Call `check_dependency_usage` for jollyday/time4j/business-calendar — does the project have a BD library?

---

## Phase 2d (continued): Verification & Storage

For each finding that passes the checks above:

4. **Severity boosting** — call these tools and boost if applicable:
   - `analyze_blast_radius` on methods with major findings — if blast radius is high (5+ consumers), mention in boost_reasons
   - `get_file_history` — if file has 10+ changes in recent months AND is untested, add "frequently modified, untested" to boost_reasons
   - `get_file_pr_context` — if a past PR title contains "fix" + the finding type (e.g., "fix debit", "fix transaction"), add "previous bug fix for same issue" to boost_reasons

5. **Store each finding** by calling `store_financial_finding` with:
   - scan_id (from step 2a)
   - repo_slug
   - file_path
   - line_number (exact line where the issue is)
   - function_name (method name)
   - check_type (exact enum value from the checks above)
   - check_category (debit_credit | transaction | conditions | errors | maturity)
   - severity (minor | moderate | major)
   - confidence (possible | likely)
   - evidence (1-2 sentences: what's wrong and what's the financial risk)
   - recommendation (1 sentence: how to fix it)
   - boost_reasons (array of strings, empty if no boosting)

6. After each file, print a one-line summary:
   `✅ FileName.java — X findings (Y major, Z moderate)`
   or `✅ FileName.java — clean`

## Phase 3: Report

After all files in a repo are scanned:

1. Call `complete_financial_scan` with the scan_id
2. Call `query_financial_findings` with the scan_id

3. Generate a report in this format:

---

# Financial Integrity Report: {repo_slug}

**Score:** {score}/10 ({rating})
**Date:** {date}
**Files scanned:** {files_scanned} of {total_candidates} candidates

## Summary by Category

| Category | 🔴 Major | 🟡 Moderate | ⚪ Minor | Total |
|---|---|---|---|---|
| Debit/Credit Balance | X | X | X | X |
| Transaction Integrity | X | X | X | X |
| Unclosed Conditions | X | X | X | X |
| Error Handling | X | X | X | X |
| Maturity/Expiry | X | X | X | X |
| **Total** | **X** | **X** | **X** | **X** |

## Critical Findings (Major Severity)

For each major finding:

### {N}. 🔴 {check_type readable name}
**File:** `{file_path}:{line_number}` — `{function_name}()`
**Evidence:** {evidence}
**Recommendation:** {recommendation}
{boost_reasons if any}

## Moderate Findings

(Same format but with 🟡)

## Files with Most Findings

List top 5 files by finding count.

---

Then move to the next repo.

## After All Repos

Print a combined summary:

# Financial Integrity — All Repos Summary

| Repo | Score | Rating | Major | Moderate | Minor | Total |
|---|---|---|---|---|---|---|
| BMT/ms-bonds-trading | 6.5 | Watch | 4 | 5 | 3 | 12 |
| BMT/ms-equity-trading | 8.5 | Healthy | 1 | 2 | 1 | 4 |
| ... | | | | | | |

## Important Reminders

- Work through files methodically — one at a time
- ALWAYS verify single-legged findings with get_scip_callers before storing
- NEVER flag helper methods (is*, get*, derive*) returning boolean as maturity issues
- If a file has 0 findings, just print clean and move on — don't explain why each check passed
- If you hit context limits, tell me which repo/file you stopped at so I can ask you to resume
- Focus on accuracy over speed — one false positive is worse than missing a true positive
```

-----

## Lite Mode Prompt (No DB — Markdown Report Only)

Use this if the 4 new MCP tools are not yet deployed. Findings go into a markdown report instead of PostgreSQL.

```
Run a financial integrity scan on the repository: {REPO_SLUG}

You are a senior financial software auditor for OCBC banking systems (Bonds, Equity, Loans, Deposits, Funds). Analyze every service file for 5 categories of financial transaction safety issues. Generate a markdown report.

## Process

1. Call `get_project_info` for the repo to understand tech stack and dependencies
2. Call `search_files` with patterns *Service.java, *Processor.java, *Handler.java to find candidate files
3. For each candidate, call `get_file_skeleton` — skip files with no financial method names
4. For each financial file, call `read_full_file` and analyze for all 5 checks:

### Check 1 — Debit/Credit Balance (debit_credit)
Find methods with outflow operations (debit, sell, disburse, withdraw, redeem, closePosition, etc.) that lack a matching inflow (credit, buy, repay, deposit, subscribe, openPosition, etc.) — or vice versa. Also flag debit/credit inside loops without idempotency guards, and duplicate calls in same method.
Before flagging single-legged: call `get_scip_callers` — does a caller provide the counterpart? Call `suggest_related_files` — is there a paired service?
DO NOT flag: fee collection, interest accrual, saga patterns, helper methods.

### Check 2 — Transaction Integrity (transaction)
Find methods with multiple repository writes (save, delete, persist, merge) without @Transactional. Flag manual commit without rollback, save-then-throw, readOnly TX with writes, @Transactional on private methods, and local DB write + HTTP call (cross-service partial commit).
DO NOT flag: class-level @Transactional, inherited TX, non-Spring repos.

### Check 3 — Unclosed Conditions (conditions)
Find if/else-if chains (2+ branches) without final else. Find switch without default. Find enum switches missing cases — call `get_scip_type_definition` to verify all enum values.
Escalate to MAJOR when condition variable is financial-sensitive (transactionType, side, currency, productType, loanType, etc.)
DO NOT flag: simple if, equals/hashCode/toString, builders.

### Check 4 — Error Handling (errors)
Find empty catch blocks, catch-log-no-rethrow in financial methods, catch returning null, System.exit in service code, try without finally for resource acquisition, @Async void without exception handler, broad catch(Exception/Throwable) without rethrow.
DO NOT flag: catch blocks delegating to error handlers, utility methods, @Async returning Future.

### Check 5 — Maturity/Expiry (maturity)
Find maturity/expiry date comparisons (isBefore/isAfter) without equals check. Find date comparisons without business-day adjustment. Find rollover with missing leg. Find @Scheduled maturity processors without retry.
CRITICAL — DO NOT flag: isSettlementHoliday(), isWeekend(), isBusinessDay() — these are helpers, NOT date comparisons. Skip any method returning boolean whose name starts with is/has/get/derive/check. Skip classes that already have business-day helper methods.

## Verification

For each potential finding, verify with MCP tools before including in report:
- `get_scip_callers` / `get_scip_callees` for call chain verification
- `get_scip_type_definition` for enum completeness
- `get_file_dependencies` for imported services (CalendarService, TX manager)
- `analyze_blast_radius` for severity boosting on major findings

## Output

Generate a markdown report with:
1. Score (start at 10, deduct: minor -0.5, moderate -1.0, major -2.0)
2. Summary table by category
3. All findings grouped by severity (major first), with: file:line, method, evidence, recommendation
4. List of files with most findings

Work through files one at a time. Print a one-line status after each file.
If no findings for a file, just say "clean" and move on.
Focus on accuracy — one false positive is worse than missing a true positive.
```

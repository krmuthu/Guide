# Financial Code Integrity Checks — Implementation Plan

> **Scope:** 5 checks across all OCBC banking business lines — Bonds, Equity, Loans, Deposits, Funds
> **Integration:** Layer 1 AST analyzers in health score OR standalone CLI
> **Stack:** tree-sitter AST · IGraphClient · REPOS_DIR files · Spring Boot (Java) + React (TypeScript)
> **Effort:** ~5 days

-----

## Table of Contents

1. [Overview — The 5 Checks](#1-overview)
1. [Paired Operations Reference — All Business Lines](#2-paired-operations-reference)
1. [Check 1 — Duplicate / Single-Legged / Looping Debit-Credit](#3-check-1)
1. [Check 2 — Transaction Rollback Integrity](#4-check-2)
1. [Check 3 — Unclosed Conditions (Missing Catch-All)](#5-check-3)
1. [Check 4 — Unhandled Errors / Sudden Termination](#6-check-4)
1. [Check 5 — Maturity / Expiry Without Handler](#7-check-5)
1. [Shared Infrastructure](#8-shared-infrastructure)
1. [Integration with Health Score](#9-integration-with-health-score)
1. [Standalone CLI Mode](#10-standalone-cli-mode)
1. [Layer 3 LLM Augmentation](#11-layer-3-llm-augmentation)
1. [Implementation Schedule](#12-implementation-schedule)

-----

## 1. Overview

These checks target **financial transaction safety** across all OCBC banking business lines — Bonds, Equity, Loans, Deposits, and Funds. A missed edge case in any of these domains can cause real monetary loss, regulatory breach, or ledger imbalance. They complement the existing 8 health score dimensions by adding a **9th dimension: Financial Integrity** (or they can run standalone).

|#|Check                                           |What It Catches                                                                                                                                                                                                                 |Severity          |
|-|------------------------------------------------|--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|------------------|
|1|Duplicate / Single-Legged / Looping Debit-Credit|Double-charging, unbalanced ledger entries, any paired financial operation without its counterpart. Covers: debit/credit (bonds), buy/sell (equity), disburse/repay (loans), place/withdraw (deposits), subscribe/redeem (funds)|**major**         |
|2|Transaction Rollback                            |Missing `@Transactional`, manual commits without rollback, partial state changes outside TX boundary, save-then-throw without rollback                                                                                          |**major**         |
|3|Unclosed Conditions                             |`if/else if` chains without final `else`, `switch` without `default`, enum switches missing cases — escalated to major when condition involves financial-sensitive variables                                                    |**moderate–major**|
|4|Unhandled Errors / Sudden Termination           |Empty catch blocks, `System.exit()` / `process.exit()` in service code, swallowed exceptions, missing `finally` for resource cleanup, `@Async` without error handler                                                            |**moderate–major**|
|5|Maturity / Expiry Without Handler               |Date-driven events (loan maturity, deposit rollover, bond coupon, option expiry) that check a maturity/expiry date but don’t handle non-business day, auto-rollover, or the “matured today” case                                |**major**         |

-----

## 2. Paired Operations Reference — All Business Lines

Every financial operation that moves money, units, or positions has a **counterpart** — the Check 1 single-legged detection must know all of them.

### 2.1 Paired Operations by Business Line

|Business Line |Outflow (Debit-side)                                                                                            |Inflow (Credit-side)                                                                                                               |Notes                                             |
|--------------|----------------------------------------------------------------------------------------------------------------|-----------------------------------------------------------------------------------------------------------------------------------|--------------------------------------------------|
|**Bonds**     |`debit`, `sell`, `redeemBond`, `payoutCoupon`, `payInterest`                                                    |`credit`, `buy`, `purchaseBond`, `receiveCoupon`, `collectInterest`                                                                |Coupon payments are paired with accrual           |
|**Equity**    |`sell`, `closePosition`, `deallocate`, `exerciseOption`, `shortSell`, `disposeHolding`, `reducePosition`        |`buy`, `purchase`, `openPosition`, `allocate`, `assignOption`, `coverShort`, `addHolding`, `increasePosition`                      |Position open/close must be balanced              |
|**Loans**     |`disburse`, `disburseLoan`, `drawdown`, `releaseCollateral`, `writeOff`, `waiveFee`, `forgivePrincipal`         |`repay`, `repayLoan`, `paydown`, `pledgeCollateral`, `collectFee`, `accrue`, `capitalize`, `collectRepayment`, `collectInstallment`|Disbursement without repayment schedule is suspect|
|**Deposits**  |`place`, `placeDeposit`, `breakDeposit`, `earlyWithdraw`, `terminateDeposit`, `closeDeposit`, rollover close-leg|`withdraw`, `accrueInterest`, `creditInterest`, `renewDeposit`, rollover open-leg                                                  |Rollover is two-legged: close old + open new      |
|**Funds**     |`redeem`, `cancelUnits`, `collectFee`, `distributeDividend`, `surrenderUnits`                                   |`subscribe`, `allocateUnits`, `rebate`, `reinvestDividend`, `issueUnits`                                                           |NAV-driven — units and cash must balance          |
|**Cross-line**|`transferOut`, `settlePayable`, `remit`, `payOut`                                                               |`transferIn`, `settleReceivable`, `receive`, `payIn`                                                                               |Internal transfers are always two-legged          |

### 2.2 Financial-Sensitive Variable Names

These variable/method names in conditions or switch expressions indicate financial-critical branching logic. Used by Check 3 (unclosed conditions) and Check 4 (error handling) to escalate severity.

|Category                      |Variable/field names                                                                                                                                                 |
|------------------------------|---------------------------------------------------------------------------------------------------------------------------------------------------------------------|
|**Transaction classification**|`type`, `transactionType`, `txnType`, `entryType`, `side`, `direction`, `action`, `operationType`, `movementType`                                                    |
|**Product identifiers**       |`product`, `productType`, `instrumentType`, `assetClass`, `assetType`, `securityType`                                                                                |
|**Time/schedule**             |`tenor`, `tenorUnit`, `maturity`, `maturityDate`, `expiryDate`, `valueDate`, `settlementDate`, `frequency`, `schedule`, `couponDate`, `rolloverDate`, `repaymentDate`|
|**Rate/amount**               |`rate`, `rateType`, `interestType`, `fixedOrFloat`, `currency`, `ccy`, `amount`, `notional`, `principal`                                                             |
|**Classification**            |`status`, `category`, `tier`, `channel`, `segment`, `tranche`, `facility`, `loanType`, `depositType`, `fundType`, `accountType`                                      |

-----

## 3. Check 1 — Duplicate / Single-Legged / Looping Debit-Credit

### 3.1 What We’re Looking For

Across all business lines, every financial operation that moves money, units, or positions MUST have a matching counterpart (double-entry bookkeeping). This applies to bonds (debit/credit), equity (buy/sell, open/close position), loans (disburse/repay), deposits (place/withdraw), and funds (subscribe/redeem). Violations:

**Single-legged entry:** A method calls `debit()` but never calls `credit()` (or vice versa) — money leaves one account but never arrives in another.

**Duplicate entry:** Same debit/credit call executed twice for the same transaction — double-charge risk. Especially dangerous inside retry loops or when idempotency keys are absent.

**Loop without guard:** A debit or credit inside a `for`/`while` loop without an idempotency check or accumulator pattern — can process the same entry N times.

### 3.2 Detection Strategy

```
Layer 1 (AST): Scope-aware function body analysis
  → Find functions containing any paired financial operation call
  → Check if counterpart exists in same function or call chain
  → Check if operation is inside a loop body
  → Check for idempotency guards (dedup key, processed set, conditional check)

Layer 2 (Graph): Call chain verification
  → If a function calls debit() but not credit(), trace callers via IGraphClient
  → Check if any caller provides the matching leg
  → If no caller in the chain has the counterpart → single-legged finding

Layer 3 (LLM): Context verification for ambiguous cases
  → Qwen reviews full file for business intent
  → Distinguishes intentional single-leg (fee collection, interest accrual) from bugs
```

### 3.3 AST Implementation

```typescript
// src/lib/health_score/analyzers/financial_integrity/debit_credit.ts

interface DebitCreditCall {
  type: 'debit' | 'credit';
  methodName: string;
  line: number;
  inLoop: boolean;
  loopType?: 'for' | 'while' | 'forEach' | 'stream';
  hasIdempotencyGuard: boolean;
  enclosingFunction: string;
  filePath: string;
}

// ══════════════════════════════════════════════════════════════════
// Paired operation patterns — ALL business lines
// ══════════════════════════════════════════════════════════════════

// ── OUTFLOW / DEBIT-SIDE operations ──
const DEBIT_PATTERNS = [
  // Generic ledger
  /\bdebit\s*\(/i,
  /\bwithdraw\s*\(/i,
  /\bsubtractBalance\s*\(/i,
  /\bcharge\s*\(/i,
  /\bdecrease(?:Balance|Amount|Funds)\s*\(/i,
  /\.setTransactionType\s*\(\s*["'](?:DEBIT|DR|WITHDRAWAL)["']\s*\)/i,
  /TransactionType\s*\.\s*(?:DEBIT|DR|WITHDRAWAL)/i,
  /\.setEntryType\s*\(\s*["']D["']\s*\)/i,
  /\.setMovementType\s*\(\s*["'](?:OUT|OUTFLOW|DEBIT)["']\s*\)/i,

  // Equity
  /\bsell\s*\(/i,
  /\bshortSell\s*\(/i,
  /\bclosePosition\s*\(/i,
  /\bdeallocate\s*\(/i,
  /\bexerciseOption\s*\(/i,
  /\bdisposeHolding\s*\(/i,
  /\breducePosition\s*\(/i,
  /OrderSide\s*\.\s*SELL/i,
  /TradeSide\s*\.\s*SELL/i,

  // Loans
  /\bdisburse\s*\(/i,
  /\bdisburseLoan\s*\(/i,
  /\bdrawdown\s*\(/i,
  /\breleaseCollateral\s*\(/i,
  /\bwriteOff\s*\(/i,
  /\bwaiveFee\s*\(/i,
  /\bforgivePrincipal\s*\(/i,

  // Deposits
  /\bplace(?:Deposit)?\s*\(/i,
  /\bbreakDeposit\s*\(/i,
  /\bearlyWithdraw\s*\(/i,
  /\bterminateDeposit\s*\(/i,
  /\bcloseDeposit\s*\(/i,

  // Funds
  /\bredeem\s*\(/i,
  /\bcancelUnits?\s*\(/i,
  /\bcollectFee\s*\(/i,
  /\bdistributeDividend\s*\(/i,
  /\bsurrenderUnits?\s*\(/i,

  // Bonds
  /\bpayoutCoupon\s*\(/i,
  /\bpayInterest\s*\(/i,
  /\bredeemBond\s*\(/i,

  // Cross-line
  /\btransferOut\s*\(/i,
  /\bsettlePayable\s*\(/i,
  /\bremit\s*\(/i,
  /\bpayOut\s*\(/i,
];

// ── INFLOW / CREDIT-SIDE operations ──
const CREDIT_PATTERNS = [
  // Generic ledger
  /\bcredit\s*\(/i,
  /\bdeposit\s*\(/i,
  /\baddBalance\s*\(/i,
  /\bincrease(?:Balance|Amount|Funds)\s*\(/i,
  /\.setTransactionType\s*\(\s*["'](?:CREDIT|CR|DEPOSIT)["']\s*\)/i,
  /TransactionType\s*\.\s*(?:CREDIT|CR|DEPOSIT)/i,
  /\.setEntryType\s*\(\s*["']C["']\s*\)/i,
  /\.setMovementType\s*\(\s*["'](?:IN|INFLOW|CREDIT)["']\s*\)/i,

  // Equity
  /\bbuy\s*\(/i,
  /\bpurchase\s*\(/i,
  /\bopenPosition\s*\(/i,
  /\ballocate(?:Shares|Units|Position)?\s*\(/i,
  /\bassignOption\s*\(/i,
  /\bcoverShort\s*\(/i,
  /\baddHolding\s*\(/i,
  /\bincreasePosition\s*\(/i,
  /OrderSide\s*\.\s*BUY/i,
  /TradeSide\s*\.\s*BUY/i,

  // Loans
  /\brepay\s*\(/i,
  /\brepayLoan\s*\(/i,
  /\bpaydown\s*\(/i,
  /\bpledgeCollateral\s*\(/i,
  /\bcollectRepayment\s*\(/i,
  /\baccrue(?:Interest)?\s*\(/i,
  /\bcapitalize(?:Interest)?\s*\(/i,
  /\bcollectInstallment\s*\(/i,

  // Deposits
  /\bplaceDeposit\s*\(/i,
  /\baccrueInterest\s*\(/i,
  /\bcreditInterest\s*\(/i,
  /\brenewDeposit\s*\(/i,

  // Funds
  /\bsubscribe\s*\(/i,
  /\ballocateUnits?\s*\(/i,
  /\brebate\s*\(/i,
  /\breinvestDividend\s*\(/i,
  /\bissueUnits?\s*\(/i,

  // Bonds
  /\breceiveCoupon\s*\(/i,
  /\bcollectInterest\s*\(/i,
  /\bpurchaseBond\s*\(/i,

  // Cross-line
  /\btransferIn\s*\(/i,
  /\bsettleReceivable\s*\(/i,
  /\breceive\s*\(/i,
  /\bpayIn\s*\(/i,
];

const IDEMPOTENCY_PATTERNS = [
  /idempoten/i,
  /dedup(?:lication)?/i,
  /alreadyProcessed/i,
  /\.contains\s*\(/,           // Set.contains() check before operation
  /processedIds/i,
  /transactionId/i,            // Transaction ID check
  /ifAbsent/i,                 // putIfAbsent pattern
  /computeIfAbsent/i,
  /INSERT\s+.*\s+ON\s+CONFLICT/i,  // Postgres upsert
  /INSERT\s+IGNORE/i,              // MySQL idempotent insert
  /MERGE\s+INTO/i,                 // Oracle/SQL Server merge
  /referenceNumber/i,              // Payment reference dedup
  /requestId/i,                    // API idempotency key
];

export function analyzeDebitCredit(
  files: LoadedFile[],
  graphClient: IGraphClient,
  repo: string
): FinancialFinding[] {
  const findings: FinancialFinding[] = [];

  for (const file of files) {
    if (file.isTestFile) continue;
    if (!isFinancialFile(file)) continue;

    const tree = file.tree;
    const functionNodes = findFunctionNodes(tree, file.language);

    for (const funcNode of functionNodes) {
      const funcName = extractFunctionName(funcNode, file.language);
      const funcBody = funcNode.text;
      const funcLines = funcBody.split('\n');

      const debits = findPatternCalls(funcBody, DEBIT_PATTERNS, 'debit', funcNode, file);
      const credits = findPatternCalls(funcBody, CREDIT_PATTERNS, 'credit', funcNode, file);

      // ── Check 1a: Single-legged entry ──
      if (debits.length > 0 && credits.length === 0) {
        findings.push({
          check: 'single_legged_debit',
          severity: 'major',
          filePath: file.path,
          line: debits[0].line,
          function: funcName,
          evidence: `Function '${funcName}' contains ${debits.length} outflow operation(s) but no matching inflow. ` +
                    `Methods: ${debits.map(d => d.methodName).join(', ')}. ` +
                    `Risk: Unbalanced ledger — money/units leave one account but never arrive in another.`,
          recommendation: 'Verify this is intentional (e.g., fee collection, interest accrual, dividend payout). ' +
                          'If not, add matching inflow operation or ensure the caller provides the counterpart leg.',
          needsGraphVerification: true,
          confidence: 'possible',
        });
      }

      if (credits.length > 0 && debits.length === 0) {
        findings.push({
          check: 'single_legged_credit',
          severity: 'major',
          filePath: file.path,
          line: credits[0].line,
          function: funcName,
          evidence: `Function '${funcName}' contains ${credits.length} inflow operation(s) but no matching outflow. ` +
                    `Methods: ${credits.map(c => c.methodName).join(', ')}. ` +
                    `Risk: Unbalanced ledger — money/units credited without corresponding debit entry.`,
          recommendation: 'Verify this is intentional (e.g., interest accrual, subscription allocation). ' +
                          'If not, add matching outflow operation or ensure the caller provides the counterpart leg.',
          needsGraphVerification: true,
          confidence: 'possible',
        });
      }

      // ── Check 1b: Inside loop without idempotency guard ──
      const allOps = [...debits, ...credits];
      for (const op of allOps) {
        if (op.inLoop && !op.hasIdempotencyGuard) {
          findings.push({
            check: 'loop_debit_credit',
            severity: 'major',
            filePath: file.path,
            line: op.line,
            function: funcName,
            evidence: `${op.type}() call '${op.methodName}' is inside a ${op.loopType ?? 'loop'} ` +
                      `without an idempotency guard. ` +
                      `Risk: Same transaction could be processed multiple times if the loop iterates over ` +
                      `duplicate items or retries on failure.`,
            recommendation: 'Add idempotency protection: check a processed-ID set, use a database-level ' +
                            'unique constraint on transaction/reference ID, or use INSERT ON CONFLICT / MERGE.',
            needsGraphVerification: false,
            confidence: 'likely',
          });
        }
      }

      // ── Check 1c: Duplicate operation in same function ──
      const debitNames = debits.map(d => d.methodName);
      const duplicateDebits = debitNames.filter((name, i) => debitNames.indexOf(name) !== i);
      for (const dupName of new Set(duplicateDebits)) {
        const dupCalls = debits.filter(d => d.methodName === dupName);
        if (!areInDifferentBranches(dupCalls, funcNode, file.language)) {
          findings.push({
            check: 'duplicate_debit',
            severity: 'major',
            filePath: file.path,
            line: dupCalls[1].line,
            function: funcName,
            evidence: `'${dupName}' called ${dupCalls.length} times in '${funcName}' at lines ` +
                      `${dupCalls.map(d => d.line).join(', ')} — not in mutually exclusive branches. ` +
                      `Risk: Double debit/charge on the same transaction.`,
            recommendation: 'If intentional (multi-leg debit), add a comment. Otherwise, remove the duplicate ' +
                            'call or guard with a conditional.',
            needsGraphVerification: false,
            confidence: 'likely',
          });
        }
      }
    }
  }

  return findings;
}

// ── Helper: Detect if a call site is inside a loop ──
function isInsideLoop(node: SyntaxNode, language: string): { inLoop: boolean; loopType?: string } {
  let current = node.parent;
  while (current) {
    switch (current.type) {
      case 'for_statement':
      case 'for_in_statement':
        return { inLoop: true, loopType: 'for' };
      case 'while_statement':
      case 'do_statement':
        return { inLoop: true, loopType: 'while' };
      case 'enhanced_for_statement':
        return { inLoop: true, loopType: 'forEach' };
      case 'for_of_statement':
        return { inLoop: true, loopType: 'forEach' };
    }
    if (current.type === 'method_invocation') {
      const methodName = current.childForFieldName('name')?.text;
      if (['forEach', 'map', 'flatMap', 'forEachOrdered'].includes(methodName ?? '')) {
        return { inLoop: true, loopType: 'stream' };
      }
    }
    current = current.parent;
  }
  return { inLoop: false };
}

// ── Helper: Check if an idempotency guard exists in scope ──
function hasIdempotencyGuard(funcBody: string, callLine: number, funcLines: string[]): boolean {
  const precedingCode = funcLines.slice(0, callLine).join('\n');
  return IDEMPOTENCY_PATTERNS.some(p => p.test(precedingCode));
}

// ── Pre-filter: Is this file likely to contain financial operations? ──
function isFinancialFile(file: LoadedFile): boolean {
  const financialPathTerms = [
    // Generic
    'transaction', 'ledger', 'account', 'payment', 'settlement', 'balance',
    'booking', 'journal', 'posting', 'reconcil', 'clearing',
    // Bonds
    'bond', 'coupon', 'fixed-income', 'fixedincome', 'yield',
    // Equity
    'trade', 'order', 'position', 'portfolio', 'holding', 'execution',
    'fill', 'quote', 'stock', 'share', 'equity', 'broker', 'market-data',
    'dividend', 'option', 'derivative',
    // Loans
    'loan', 'disbursement', 'repayment', 'amortiz', 'installment',
    'principal', 'collateral', 'facility', 'drawdown', 'prepayment',
    'mortgage', 'credit-line', 'creditline',
    // Deposits
    'deposit', 'maturity', 'rollover', 'tenor', 'placement',
    'fixed-deposit', 'fixeddeposit', 'time-deposit', 'timedeposit', 'accrual',
    // Funds
    'fund', 'nav', 'subscription', 'redemption', 'unit', 'allotment',
    'valuation', 'custody', 'trustee', 'asset-management', 'assetmanagement',
  ];

  const pathLower = file.path.toLowerCase();
  if (financialPathTerms.some(term => pathLower.includes(term))) return true;

  const head = file.lines.slice(0, 80).join('\n');
  const financialClassTerms = [
    /\b(debit|credit|TransactionType|LedgerEntry|AccountBalance|JournalEntry|PostingEntry)\b/,
    /\b(SettlementService|ReconciliationService|ClearingService)\b/,
    /\b(TradeService|OrderService|PositionService|PortfolioService|HoldingService)\b/,
    /\b(ExecutionService|MarketDataService|BrokerService|DividendService)\b/,
    /\b(OrderSide|TradeSide|TradeType|PositionType)\b/,
    /\b(LoanService|DisbursementService|RepaymentService|AmortizationService)\b/,
    /\b(CollateralService|FacilityService|DrawdownService|InstallmentService)\b/,
    /\b(LoanType|FacilityType|RepaymentType)\b/,
    /\b(DepositService|PlacementService|RolloverService|MaturityService)\b/,
    /\b(DepositType|TenorType|AccrualService)\b/,
    /\b(FundService|SubscriptionService|RedemptionService|NAVService)\b/,
    /\b(UnitAllocationService|CustodyService|ValuationService)\b/,
    /\b(FundType|SubscriptionType|RedemptionType)\b/,
  ];

  if (financialClassTerms.some(p => p.test(head))) return true;

  return false;
}
```

### 3.4 Graph Verification for Single-Legged Findings

```typescript
// src/lib/health_score/analyzers/financial_integrity/debit_credit_graph.ts

export async function verifyWithCallChain(
  findings: FinancialFinding[],
  graphClient: IGraphClient,
  repo: string
): Promise<FinancialFinding[]> {
  const verified: FinancialFinding[] = [];

  for (const finding of findings) {
    if (!finding.needsGraphVerification) {
      verified.push(finding);
      continue;
    }

    const missingType = finding.check === 'single_legged_debit' ? 'credit' : 'debit';
    const patterns = missingType === 'credit' ? CREDIT_PATTERNS : DEBIT_PATTERNS;

    const callers = await graphClient.getReverseCallChain(
      finding.function, repo, /* maxDepth */ 3
    );

    let counterpartFoundInCaller = false;

    for (const caller of callers) {
      if (!caller.file_path) continue;
      try {
        const callerContent = await fs.readFile(
          path.join(process.env.REPOS_DIR || '/repos', repo.replace('/', '/'), caller.file_path),
          'utf-8'
        );
        if (patterns.some(p => p.test(callerContent))) {
          counterpartFoundInCaller = true;
          break;
        }
      } catch { continue; }
    }

    if (counterpartFoundInCaller) {
      finding.confidence = 'unlikely';
      finding.severity = 'minor';
      finding.evidence += ` (Note: A caller in the call chain does contain the matching ` +
                          `${missingType} operation — this may be an intentional architectural split.)`;
    } else {
      finding.confidence = 'likely';
      finding.evidence += ` (Verified: no caller in the 3-level call chain provides a matching ` +
                          `${missingType} operation.)`;
    }

    verified.push(finding);
  }

  return verified;
}
```

-----

## 4. Check 2 — Transaction Rollback Integrity

### 4.1 What We’re Looking For

**Missing `@Transactional`:** Service methods that perform multiple DB writes without a transaction boundary — partial failure leaves data inconsistent.

**Manual commit without rollback:** Code using `EntityManager.getTransaction().commit()` or `connection.commit()` without a corresponding rollback in the catch block.

**Save-then-throw pattern:** `repository.save(entity)` followed by a condition that throws an exception — the save persists but the caller thinks the operation failed.

**Multiple repository calls without TX:** A method calling `repoA.save()` and `repoB.save()` without `@Transactional` — if `repoB.save()` fails, `repoA`’s change is already committed.

### 4.2 AST Implementation — Java

```typescript
// src/lib/health_score/analyzers/financial_integrity/transaction_rollback.ts

export function analyzeTransactionRollback(
  files: LoadedFile[],
  config: ProjectConfig
): FinancialFinding[] {
  const findings: FinancialFinding[] = [];

  for (const file of files) {
    if (file.isTestFile) continue;
    if (file.language !== 'java') continue;

    const tree = file.tree;

    const classNode = findClassDeclaration(tree);
    const classAnnotations = extractAnnotations(classNode);
    const hasClassTransactional = classAnnotations.some(a =>
      a === 'Transactional' || a === 'javax.transaction.Transactional' ||
      a === 'org.springframework.transaction.annotation.Transactional'
    );

    const methods = findMethodDeclarations(tree);

    for (const method of methods) {
      const methodName = method.childForFieldName('name')?.text ?? 'unknown';
      const methodBody = method.text;
      const methodAnnotations = extractAnnotations(method);
      const hasMethodTransactional = methodAnnotations.some(a => a === 'Transactional');
      const isTransactional = hasClassTransactional || hasMethodTransactional;
      const isPublic = hasModifier(method, 'public');

      // ── Check 2a: Multiple repository writes without @Transactional ──
      const saveCallLines = findMethodCallLines(method, [
        'save', 'saveAll', 'saveAndFlush', 'delete', 'deleteAll',
        'deleteById', 'update', 'insert', 'persist', 'merge',
        'executeUpdate', 'execute',
      ]);

      const dbWriteLines = saveCallLines.filter(call =>
        /(?:repository|repo|dao|Repo|Repository|DAO|jdbcTemplate|entityManager|em)\s*\.\s*(?:save|delete|update|insert|persist|merge|execute)/
          .test(call.lineText)
      );

      const repoObjects = new Set(
        dbWriteLines.map(call => {
          const match = call.lineText.match(
            /(\w+(?:Repository|Repo|DAO|dao|jdbcTemplate|entityManager|em))\s*\./
          );
          return match?.[1] ?? 'unknown';
        })
      );

      if (repoObjects.size >= 2 && !isTransactional && isPublic) {
        findings.push({
          check: 'multi_write_no_transaction',
          severity: 'major',
          filePath: file.path,
          line: dbWriteLines[0].line,
          function: methodName,
          evidence: `Public method '${methodName}' writes to ${repoObjects.size} different repositories ` +
                    `(${[...repoObjects].join(', ')}) without @Transactional. ` +
                    `If the second write fails, the first is already committed.`,
          recommendation: 'Add @Transactional to this method to ensure atomicity across all writes.',
          confidence: 'likely',
        });
      }

      if (dbWriteLines.length >= 2 && repoObjects.size === 1 && !isTransactional && isPublic) {
        findings.push({
          check: 'multi_write_single_repo_no_transaction',
          severity: 'moderate',
          filePath: file.path,
          line: dbWriteLines[0].line,
          function: methodName,
          evidence: `'${methodName}' performs ${dbWriteLines.length} write operations on ` +
                    `'${[...repoObjects][0]}' without @Transactional. ` +
                    `Partial failure leaves data in inconsistent state.`,
          recommendation: 'Add @Transactional or verify that each write is independently safe.',
          confidence: 'possible',
        });
      }

      // ── Check 2b: Manual commit without rollback ──
      const hasManualCommit = /\.\s*commit\s*\(\s*\)/.test(methodBody);
      const hasManualRollback = /\.\s*rollback\s*\(\s*\)/.test(methodBody);

      if (hasManualCommit && !hasManualRollback) {
        const commitLine = findLineOfPattern(method, /\.\s*commit\s*\(\s*\)/);
        findings.push({
          check: 'commit_without_rollback',
          severity: 'major',
          filePath: file.path,
          line: commitLine,
          function: methodName,
          evidence: `Manual .commit() at line ${commitLine} without a corresponding .rollback() in the ` +
                    `catch/error path.`,
          recommendation: 'Add try-catch around the commit with .rollback() in the catch block, or ' +
                          'use Spring\'s @Transactional which handles rollback automatically.',
          confidence: 'likely',
        });
      }

      // ── Check 2c: Save-then-throw ──
      const saveNodes = findCallExpressions(method, ['save', 'saveAndFlush', 'persist']);

      for (const saveNode of saveNodes) {
        const statementsAfterSave = getStatementsAfterNode(saveNode, method);

        for (const stmt of statementsAfterSave) {
          if (/throw\s+new\s+\w*Exception/.test(stmt.text) || /throw\s+\w+/.test(stmt.text)) {
            const isConditional = stmt.parent?.type === 'if_statement' ||
                                   stmt.parent?.type === 'block' &&
                                   stmt.parent?.parent?.type === 'if_statement';

            if (!isTransactional) {
              findings.push({
                check: 'save_then_throw',
                severity: isConditional ? 'moderate' : 'major',
                filePath: file.path,
                line: saveNode.startPosition.row + 1,
                function: methodName,
                evidence: `'${methodName}' calls .save() at line ${saveNode.startPosition.row + 1} ` +
                          `then ${isConditional ? 'conditionally ' : ''}throws at line ` +
                          `${stmt.startPosition.row + 1} — without @Transactional. ` +
                          `The save is committed, but the caller receives an exception.`,
                recommendation: 'Add @Transactional so the throw triggers automatic rollback of the save.',
                confidence: isConditional ? 'possible' : 'likely',
              });
            }
            break;
          }
        }
      }

      // ── Check 2d: @Transactional(readOnly = true) on write methods ──
      const readOnlyTx = methodAnnotations.find(a =>
        /Transactional\s*\(.*readOnly\s*=\s*true/.test(a)
      );
      if (readOnlyTx && dbWriteLines.length > 0) {
        findings.push({
          check: 'readonly_transaction_with_writes',
          severity: 'major',
          filePath: file.path,
          line: method.startPosition.row + 1,
          function: methodName,
          evidence: `'${methodName}' is annotated @Transactional(readOnly = true) but performs ` +
                    `${dbWriteLines.length} write operation(s). Behavior is database-dependent.`,
          recommendation: 'Remove readOnly = true or move writes to a separate non-readOnly method.',
          confidence: 'likely',
        });
      }
    }
  }

  return findings;
}
```

### 4.3 TypeScript Variant (React MFEs with API state)

```typescript
// For React MFEs: check that API mutation calls have error handling with rollback

export function analyzeTypescriptTransactions(files: LoadedFile[]): FinancialFinding[] {
  const findings: FinancialFinding[] = [];

  for (const file of files) {
    if (file.isTestFile) continue;
    if (file.language !== 'typescript') continue;

    const methods = findFunctionNodes(file.tree, 'typescript');

    for (const method of methods) {
      const body = method.text;
      const funcName = extractFunctionName(method, 'typescript');

      // Multiple sequential mutation API calls without try-catch
      const mutationCalls = findMutationAPICalls(body);

      if (mutationCalls.length >= 2) {
        const hasTryCatch = method.descendantsOfType('try_statement').length > 0;
        if (!hasTryCatch) {
          findings.push({
            check: 'multi_mutation_no_error_handling',
            severity: 'moderate',
            filePath: file.path,
            line: mutationCalls[0].line,
            function: funcName,
            evidence: `${mutationCalls.length} sequential API mutations in '${funcName}' ` +
                      `without try-catch. If the second call fails, the first has already ` +
                      `modified server state with no client-side rollback.`,
            recommendation: 'Wrap in try-catch. On failure, either call a compensating API ' +
                            'to undo the first mutation, or use a backend saga/orchestrator.',
            confidence: 'possible',
          });
        }
      }

      // Optimistic UI update without rollback on error
      const hasStateSet = /setState|\.set\(|dispatch\(|\.update\(/.test(body);
      const hasApiCall = /fetch\s*\(|axios\.|\.mutate|\.mutateAsync/.test(body);
      if (hasStateSet && hasApiCall) {
        const stateSetPos = body.search(/setState|\.set\(|dispatch\(/);
        const apiCallPos = body.search(/fetch\s*\(|axios\.|\.mutate/);

        if (stateSetPos < apiCallPos && stateSetPos !== -1) {
          const catchBlock = method.descendantsOfType('catch_clause');
          const hasRollback = catchBlock.some(c =>
            /setState|\.set\(|dispatch\(|rollback|revert|undo/.test(c.text)
          );

          if (!hasRollback) {
            findings.push({
              check: 'optimistic_update_no_rollback',
              severity: 'moderate',
              filePath: file.path,
              line: method.startPosition.row + 1,
              function: funcName,
              evidence: `'${funcName}' updates local state before the API call completes. ` +
                        `If the API call fails, the UI shows stale/incorrect state.`,
              recommendation: 'Add state rollback in the catch block, or move the state ' +
                              'update to the .then()/.onSuccess() callback.',
              confidence: 'possible',
            });
          }
        }
      }
    }
  }

  return findings;
}
```

-----

## 5. Check 3 — Unclosed Conditions (Missing Catch-All)

### 5.1 What We’re Looking For

In financial code, unhandled cases in conditional logic can silently drop transactions or process them with wrong parameters.

**`if/else if` without final `else`:** An input value that matches none of the conditions falls through silently — no error, no log, no processing.

**`switch` without `default`:** Same problem. Especially dangerous with enum types where a new enum value is added but the switch isn’t updated.

**Enum switch missing cases:** `switch(transactionType)` handles `BUY` and `SELL` but not `TRANSFER` — the transfer silently does nothing.

### 5.2 AST Implementation

```typescript
// src/lib/health_score/analyzers/financial_integrity/unclosed_conditions.ts

export function analyzeUnclosedConditions(
  files: LoadedFile[]
): FinancialFinding[] {
  const findings: FinancialFinding[] = [];

  for (const file of files) {
    if (file.isTestFile) continue;

    const tree = file.tree;

    // ── Check 3a: if / else-if chains without final else ──
    const ifStatements = tree.rootNode.descendantsOfType('if_statement');

    for (const ifNode of ifStatements) {
      if (ifNode.parent?.type === 'if_statement' ||
          ifNode.parent?.type === 'else_clause') continue;

      let hasElseIf = false;
      let hasFinalElse = false;
      let chainDepth = 1;
      let current: SyntaxNode | null = ifNode;

      while (current) {
        const elseClause = current.childForFieldName('alternative') ??
                           current.children.find(c => c.type === 'else_clause');
        if (!elseClause) break;

        const elseBody = elseClause.children.find(c => c.type === 'if_statement');
        if (elseBody) {
          hasElseIf = true;
          chainDepth++;
          current = elseBody;
        } else {
          hasFinalElse = true;
          break;
        }
      }

      if (hasElseIf && !hasFinalElse && chainDepth >= 2) {
        const conditionText = ifNode.childForFieldName('condition')?.text ?? '';
        const isFinancialCondition = isFinanciallySensitive(conditionText);
        const severity = isFinancialCondition ? 'major' : 'moderate';

        findings.push({
          check: 'if_chain_no_else',
          severity,
          filePath: file.path,
          line: ifNode.startPosition.row + 1,
          function: findEnclosingFunction(ifNode, file.language),
          evidence: `if/else-if chain with ${chainDepth} branches at line ` +
                    `${ifNode.startPosition.row + 1} has no final else clause. ` +
                    `${isFinancialCondition
                      ? `Condition involves '${conditionText.slice(0, 60)}' — financial-sensitive. `
                      : ''}` +
                    `An input that matches none of the conditions is silently ignored.`,
          recommendation: 'Add a final else block that either handles the default case, ' +
                          'throws an IllegalStateException/Error, or logs a warning.',
          confidence: isFinancialCondition ? 'likely' : 'possible',
        });
      }
    }

    // ── Check 3b: switch without default ──
    const switchStatements = tree.rootNode.descendantsOfType('switch_statement');

    for (const switchNode of switchStatements) {
      const switchBody = switchNode.childForFieldName('body') ??
                         switchNode.children.find(c => c.type === 'switch_body' || c.type === 'switch_block');
      if (!switchBody) continue;

      const cases = switchBody.children.filter(c =>
        c.type === 'switch_case' || c.type === 'switch_label' ||
        c.type === 'case' || c.type === 'switch_block_statement_group'
      );
      const hasDefault = cases.some(c => /\bdefault\s*:/.test(c.text));

      if (!hasDefault && cases.length >= 2) {
        const switchExpr = switchNode.childForFieldName('value')?.text ??
                           switchNode.childForFieldName('condition')?.text ?? '';
        const isFinancial = isFinanciallySensitive(switchExpr);

        findings.push({
          check: 'switch_no_default',
          severity: isFinancial ? 'major' : 'moderate',
          filePath: file.path,
          line: switchNode.startPosition.row + 1,
          function: findEnclosingFunction(switchNode, file.language),
          evidence: `switch(${switchExpr.slice(0, 50)}) at line ${switchNode.startPosition.row + 1} ` +
                    `has ${cases.length} cases but no default. ` +
                    `${isFinancial ? 'This switches on a financial-sensitive value. ' : ''}` +
                    `A new enum value or unexpected input will be silently ignored.`,
          recommendation: 'Add a default case that throws IllegalArgumentException / ' +
                          'logs an error. For enum switches, handle all values explicitly.',
          confidence: isFinancial ? 'likely' : 'possible',
        });
      }

      // ── Check 3c: Enum switch missing cases ──
      if (file.language === 'java') {
        const enumCases = extractEnumSwitchCoverage(switchNode, file, files);
        if (enumCases && enumCases.missingValues.length > 0) {
          findings.push({
            check: 'enum_switch_missing_cases',
            severity: 'major',
            filePath: file.path,
            line: switchNode.startPosition.row + 1,
            function: findEnclosingFunction(switchNode, file.language),
            evidence: `switch on enum '${enumCases.enumName}' is missing case(s): ` +
                      `${enumCases.missingValues.join(', ')}. ` +
                      `Handled: ${enumCases.handledValues.join(', ')}.`,
            recommendation: `Add case(s) for: ${enumCases.missingValues.join(', ')}. ` +
                            `If they should not occur here, add them with a throw statement.`,
            confidence: 'likely',
          });
        }
      }
    }
  }

  return findings;
}

// ── Financial sensitivity heuristic — expanded for all business lines ──
function isFinanciallySensitive(text: string): boolean {
  return /\b(type|transactionType|txnType|entryType|side|direction|action|operationType|movementType|status|currency|ccy|instrument|instrumentType|asset|assetClass|assetType|securityType|order|trade|transaction|settlement|payment|booking|leg|entry|product|productType|tenor|tenorUnit|maturity|maturityDate|expiryDate|valueDate|settlementDate|frequency|schedule|couponDate|rolloverDate|repaymentDate|rate|rateType|interestType|fixedOrFloat|amount|notional|principal|category|tier|channel|segment|tranche|facility|loanType|depositType|fundType|accountType)\b/i
    .test(text);
}

// ── Enum coverage analysis (Java) ──
function extractEnumSwitchCoverage(
  switchNode: SyntaxNode,
  file: LoadedFile,
  allFiles: LoadedFile[]
): { enumName: string; handledValues: string[]; missingValues: string[] } | null {
  const switchExpr = switchNode.childForFieldName('value')?.text ?? '';

  for (const f of allFiles) {
    const enumMatch = f.content.match(
      new RegExp(`enum\\s+(\\w*${escapeRegex(getLastIdentifier(switchExpr))}\\w*)\\s*\\{([^}]+)\\}`)
    );
    if (enumMatch) {
      const enumName = enumMatch[1];
      const allValues = enumMatch[2]
        .split(/[,;\n]/)
        .map(v => v.trim().replace(/\(.*\)/, '').trim())
        .filter(v => v && /^[A-Z_]+$/.test(v));

      const handledValues = extractSwitchCaseValues(switchNode);
      const missingValues = allValues.filter(v => !handledValues.includes(v));

      if (allValues.length > 0) {
        return { enumName, handledValues, missingValues };
      }
    }
  }

  return null;
}
```

-----

## 6. Check 4 — Unhandled Errors / Sudden Termination

### 6.1 What We’re Looking For

**Empty catch blocks:** Exception caught but silently swallowed — error disappears, downstream code operates on corrupt state.

**Catch with only log but no rethrow/handle:** For financial operations, logging alone is insufficient — the caller must know the operation failed.

**`System.exit()` / `process.exit()` in service code:** Abrupt JVM/process termination kills in-flight transactions, skips `finally` blocks, corrupts state.

**Missing `finally` for resource cleanup:** JDBC connections, file handles, locks acquired but not released when exceptions occur.

**`@Async` without error handler:** Async methods that throw lose the exception entirely — the caller never knows.

**Swallowed exceptions in financial methods:** `catch (Exception e) { return null; }` in a method that processes transactions — the caller gets null and may interpret it as “no data” instead of “operation failed.”

### 6.2 AST Implementation — Java

```typescript
// src/lib/health_score/analyzers/financial_integrity/error_handling.ts

export function analyzeErrorHandling(
  files: LoadedFile[],
  config: ProjectConfig
): FinancialFinding[] {
  const findings: FinancialFinding[] = [];

  for (const file of files) {
    if (file.isTestFile) continue;

    const tree = file.tree;
    const methods = findMethodDeclarations(tree);

    for (const method of methods) {
      const methodName = method.childForFieldName('name')?.text ?? 'unknown';
      const methodBody = method.text;
      const methodAnnotations = extractAnnotations(method);

      // ── Check 4a: Empty catch blocks ──
      const catchClauses = method.descendantsOfType('catch_clause');

      for (const catchClause of catchClauses) {
        const catchBody = catchClause.childForFieldName('body');
        if (!catchBody) continue;

        const bodyStatements = catchBody.namedChildren.filter(c =>
          c.type !== 'comment' && c.type !== 'line_comment' && c.type !== 'block_comment'
        );

        if (bodyStatements.length === 0) {
          findings.push({
            check: 'empty_catch',
            severity: 'major',
            filePath: file.path,
            line: catchClause.startPosition.row + 1,
            function: methodName,
            evidence: `Empty catch block at line ${catchClause.startPosition.row + 1} in '${methodName}'. ` +
                      `Exception is caught and silently swallowed.`,
            recommendation: 'At minimum, log the exception. For financial operations, rethrow as a ' +
                            'domain exception or return an explicit error result.',
            confidence: 'likely',
          });
        }

        // ── Check 4b: Catch-log-no-rethrow in financial methods ──
        const isFinancialMethod = isFinanciallySensitive(methodName) ||
                                  isFinancialFile({ path: file.path } as LoadedFile);

        if (bodyStatements.length > 0 && isFinancialMethod) {
          const hasLog = bodyStatements.some(s =>
            /\b(log|logger|LOG)\s*\.\s*(error|warn|info|debug)\s*\(/.test(s.text)
          );
          const hasRethrow = bodyStatements.some(s => /\bthrow\b/.test(s.text));
          const returnsNull = bodyStatements.some(s => /return\s+null\s*;/.test(s.text));

          if (hasLog && !hasRethrow) {
            findings.push({
              check: 'catch_log_no_rethrow',
              severity: 'moderate',
              filePath: file.path,
              line: catchClause.startPosition.row + 1,
              function: methodName,
              evidence: `Catch block in financial method '${methodName}' logs the error but does ` +
                        `not rethrow or return an error result. The caller proceeds as if the ` +
                        `operation succeeded.`,
              recommendation: 'Rethrow as a domain-specific exception (e.g., TransactionFailedException) ' +
                              'or return a Result/Either type that forces the caller to handle the failure.',
              confidence: 'likely',
            });
          }

          if (returnsNull) {
            findings.push({
              check: 'catch_returns_null',
              severity: 'major',
              filePath: file.path,
              line: catchClause.startPosition.row + 1,
              function: methodName,
              evidence: `Catch block in '${methodName}' returns null — caller may interpret this ` +
                        `as "no data found" rather than "operation failed".`,
              recommendation: 'Return Optional.empty() with proper handling, throw a checked exception, ' +
                              'or use a Result type.',
              confidence: 'likely',
            });
          }
        }
      }

      // ── Check 4c: System.exit() / process.exit() in service code ──
      const exitPattern = file.language === 'java'
        ? /System\s*\.\s*exit\s*\(/
        : /process\s*\.\s*exit\s*\(/;

      if (exitPattern.test(methodBody)) {
        const isMainMethod = methodName === 'main' ||
                             methodAnnotations.some(a => a === 'SpringBootApplication');
        if (!isMainMethod) {
          const exitLine = findLineOfPattern(method, exitPattern);
          findings.push({
            check: 'system_exit_in_service',
            severity: 'major',
            filePath: file.path,
            line: exitLine,
            function: methodName,
            evidence: `System.exit() / process.exit() at line ${exitLine} in '${methodName}'. ` +
                      `Abrupt termination kills all in-flight transactions, skips finally blocks.`,
            recommendation: 'Throw a domain exception instead. Let the framework handle graceful shutdown.',
            confidence: 'likely',
          });
        }
      }

      // ── Check 4d: try without finally for resource acquisition ──
      const tryStatements = method.descendantsOfType('try_statement');

      for (const tryNode of tryStatements) {
        const tryBody = tryNode.childForFieldName('body')?.text ?? '';
        const acquiresResource = /\b(getConnection|openSession|acquire|lock|createStatement|prepareStatement|newInputStream|newOutputStream|open)\s*\(/.test(tryBody);
        const isTryWithResources = tryNode.type === 'try_with_resources_statement' ||
                                   tryNode.children.some(c => c.type === 'resource_specification');

        if (acquiresResource && !isTryWithResources) {
          const hasFinally = tryNode.children.some(c => c.type === 'finally_clause');
          if (!hasFinally) {
            findings.push({
              check: 'resource_no_finally',
              severity: 'moderate',
              filePath: file.path,
              line: tryNode.startPosition.row + 1,
              function: methodName,
              evidence: `try block in '${methodName}' acquires a resource but has no finally block for cleanup.`,
              recommendation: 'Use try-with-resources (Java 7+) or add a finally block that closes the resource.',
              confidence: 'likely',
            });
          }
        }
      }

      // ── Check 4e: @Async without error handler ──
      const isAsync = methodAnnotations.some(a => a === 'Async');
      if (isAsync) {
        const returnType = method.childForFieldName('type')?.text ?? '';
        if (returnType === 'void') {
          const classBody = findClassDeclaration(tree)?.text ?? '';
          const hasExceptionHandler =
            /AsyncUncaughtExceptionHandler/.test(classBody) ||
            /AsyncConfigurer/.test(classBody) ||
            file.content.includes('AsyncUncaughtExceptionHandler');

          if (!hasExceptionHandler) {
            findings.push({
              check: 'async_void_no_handler',
              severity: 'major',
              filePath: file.path,
              line: method.startPosition.row + 1,
              function: methodName,
              evidence: `@Async void method '${methodName}' — if this throws, the exception is ` +
                        `lost entirely. No AsyncUncaughtExceptionHandler found.`,
              recommendation: 'Return CompletableFuture<Void> instead of void, or configure an ' +
                              'AsyncUncaughtExceptionHandler.',
              confidence: 'likely',
            });
          }
        }
      }

      // ── Check 4f: Broad exception catch in financial methods ──
      for (const catchClause of catchClauses) {
        const catchParam = catchClause.childForFieldName('parameter') ??
                           catchClause.children.find(c => c.type === 'catch_formal_parameter');
        const catchType = catchParam?.descendantsOfType('type_identifier')
          .map(t => t.text).join('') ?? '';

        if (/^(Exception|Throwable|Error|RuntimeException)$/.test(catchType)) {
          if (isFinancialFile({ path: file.path } as LoadedFile)) {
            const catchBody = catchClause.childForFieldName('body')?.text ?? '';
            if (!/throw\b/.test(catchBody)) {
              findings.push({
                check: 'broad_catch_in_financial',
                severity: 'moderate',
                filePath: file.path,
                line: catchClause.startPosition.row + 1,
                function: methodName,
                evidence: `catch(${catchType}) in financial method '${methodName}' catches all ` +
                          `exceptions including NPE, ClassCast, OOM — and does not rethrow.`,
                recommendation: 'Catch specific exception types. If a generic catch is needed, ' +
                                'rethrow unexpected exceptions after handling known ones.',
                confidence: 'possible',
              });
            }
          }
        }
      }
    }
  }

  return findings;
}
```

-----

## 7. Check 5 — Maturity / Expiry Without Handler

### 7.1 What We’re Looking For

Loans, deposits, bonds, and options all have **date-driven lifecycle events** — maturity, expiry, coupon date, rollover date, repayment date. Common bugs:

**Maturity date checked but “matured today” not handled:** Code compares `maturityDate` to today but only handles `before` and `after` — not `equals`. The maturity event never fires.

**Rollover without both legs:** Deposit rollover is a close (old deposit) + open (new deposit). If only one leg exists, money disappears or duplicates.

**No non-business-day handling:** Maturity falls on a Saturday. Code checks `maturityDate == today` — it never matches because today is Monday and maturityDate was Saturday. No adjustment to next business day.

**Scheduled event without failure path:** A `@Scheduled` method processes maturities but has no mechanism for retrying failed items or alerting on missed maturities.

### 7.2 AST Implementation

```typescript
// src/lib/health_score/analyzers/financial_integrity/maturity_expiry.ts

// ── Date field patterns across all business lines ──
const MATURITY_DATE_PATTERNS = [
  // Generic
  /maturity(?:Date|Dt|Time)?\b/i,
  /expiry(?:Date|Dt|Time)?\b/i,
  /expirationDate\b/i,

  // Bonds
  /coupon(?:Date|PaymentDate)\b/i,
  /redemptionDate\b/i,
  /callDate\b/i,

  // Loans
  /repayment(?:Date|DueDate)\b/i,
  /installment(?:Date|DueDate)\b/i,
  /drawdownExpiry\b/i,
  /facilityEndDate\b/i,
  /gracePeriodEnd\b/i,

  // Deposits
  /rollover(?:Date|Dt)\b/i,
  /placementEndDate\b/i,
  /tenorEndDate\b/i,
  /depositMaturity\b/i,

  // Funds
  /navDate\b/i,
  /subscriptionDeadline\b/i,
  /redemptionCutoff\b/i,
  /dividendRecordDate\b/i,

  // Equity / Options
  /exerciseDate\b/i,
  /optionExpiry\b/i,
  /settlementDate\b/i,
  /valueDate\b/i,
  /recordDate\b/i,
];

// ── Patterns that indicate business-day handling ──
const BUSINESS_DAY_PATTERNS = [
  /businessDay/i,
  /workingDay/i,
  /isHoliday/i,
  /holidayCalendar/i,
  /nextBusinessDay/i,
  /previousBusinessDay/i,
  /adjustToBusinessDay/i,
  /isWeekend/i,
  /DayOfWeek\s*\.\s*(?:SATURDAY|SUNDAY)/i,
  /FOLLOWING|PRECEDING|MODIFIED_FOLLOWING/i,
  /BusinessDayConvention/i,
  /DateAdjuster/i,
  /CalendarService/i,
];

export function analyzeMaturityExpiry(
  files: LoadedFile[]
): FinancialFinding[] {
  const findings: FinancialFinding[] = [];

  for (const file of files) {
    if (file.isTestFile) continue;
    if (!isFinancialFile(file)) continue;

    const tree = file.tree;
    const methods = findMethodDeclarations(tree);

    for (const method of methods) {
      const methodName = method.childForFieldName('name')?.text ?? 'unknown';
      const methodBody = method.text;

      const referencedDateFields = MATURITY_DATE_PATTERNS.filter(p => p.test(methodBody));
      if (referencedDateFields.length === 0) continue;

      // ── Check 5a: Date comparison without equals/today handling ──
      const hasBeforeAfter = /\.is(?:Before|After)\s*\(|\.compareTo\s*\(|[<>]\s*(?:today|now|LocalDate)/i
        .test(methodBody);
      const hasEquals = /\.isEqual\s*\(|\.equals\s*\(.*(?:today|now|LocalDate)|==\s*(?:today|now)/i
        .test(methodBody);
      const hasInclusiveCompare = /(?:<=|>=|\.compareTo\s*\([^)]+\)\s*[<>]=?\s*0)/.test(methodBody);

      if (hasBeforeAfter && !hasEquals && !hasInclusiveCompare) {
        findings.push({
          check: 'maturity_no_equals_check',
          severity: 'major',
          filePath: file.path,
          line: method.startPosition.row + 1,
          function: methodName,
          evidence: `'${methodName}' compares a maturity/expiry date using isBefore/isAfter ` +
                    `but never checks for equality (isEqual/equals). ` +
                    `If the event date is exactly today, neither branch executes — ` +
                    `the maturity/expiry event is silently skipped.`,
          recommendation: 'Add an isEqual/equals check for the "matured today" case, or use ' +
                          '!isAfter() instead of isBefore() to include the boundary.',
          confidence: 'likely',
        });
      }

      // ── Check 5b: No non-business-day handling ──
      const hasDateComparison = /\.is(?:Before|After|Equal)\s*\(|\.compareTo\s*\(/
        .test(methodBody);

      if (hasDateComparison) {
        const hasBusinessDayHandling = BUSINESS_DAY_PATTERNS.some(p => p.test(methodBody));
        const classBody = findClassDeclaration(tree)?.text ?? '';
        const classHasBusinessDay = BUSINESS_DAY_PATTERNS.some(p => p.test(classBody));
        const hasCalendarImport = /import.*(?:BusinessDay|HolidayCalendar|DateAdjust|CalendarService)/
          .test(file.content.slice(0, 2000));

        if (!hasBusinessDayHandling && !classHasBusinessDay && !hasCalendarImport) {
          findings.push({
            check: 'maturity_no_business_day',
            severity: 'major',
            filePath: file.path,
            line: method.startPosition.row + 1,
            function: methodName,
            evidence: `'${methodName}' compares maturity/expiry dates but has no business-day ` +
                      `adjustment. If maturity falls on a weekend or holiday, the date comparison ` +
                      `may never match — the event is missed entirely.`,
            recommendation: 'Use a BusinessDayConvention (FOLLOWING, MODIFIED_FOLLOWING, PRECEDING) ' +
                            'to adjust maturity dates before comparison. Inject a CalendarService ' +
                            'or HolidayCalendar for the relevant market.',
            confidence: 'possible',
          });
        }
      }

      // ── Check 5c: Rollover without both legs ──
      const hasRollover = /rollover|renew|extend(?:Deposit|Loan|Facility)/i.test(methodBody);
      if (hasRollover) {
        const hasClose = /close|terminate|mature|expire|end/i.test(methodBody);
        const hasOpen = /create|open|new|place|renew|initiate/i.test(methodBody);

        if (hasClose && !hasOpen) {
          findings.push({
            check: 'rollover_missing_open_leg',
            severity: 'major',
            filePath: file.path,
            line: method.startPosition.row + 1,
            function: methodName,
            evidence: `'${methodName}' appears to handle rollover but only closes/terminates ` +
                      `the existing instrument without creating a new one. ` +
                      `Risk: Customer's deposit/loan terminates without renewal.`,
            recommendation: 'Ensure rollover creates the new instrument after closing the old one, ' +
                            'within the same transaction boundary.',
            confidence: 'possible',
          });
        }
        if (hasOpen && !hasClose) {
          findings.push({
            check: 'rollover_missing_close_leg',
            severity: 'major',
            filePath: file.path,
            line: method.startPosition.row + 1,
            function: methodName,
            evidence: `'${methodName}' appears to handle rollover but creates a new instrument ` +
                      `without closing/terminating the old one. ` +
                      `Risk: Duplicate active instruments — customer has both old and new.`,
            recommendation: 'Ensure rollover closes the old instrument before or within the same ' +
                            'transaction as creating the new one.',
            confidence: 'possible',
          });
        }
      }

      // ── Check 5d: @Scheduled maturity processor without retry/alert ──
      const methodAnnotations = extractAnnotations(method);
      const isScheduled = methodAnnotations.some(a => a === 'Scheduled' || a === 'Schedules');
      const isMaturityRelated = /maturity|expiry|coupon|repayment|rollover|settlement/i
        .test(methodName);

      if (isScheduled && isMaturityRelated) {
        const hasRetry = /retry|retryable|backoff|maxAttempt/i.test(methodBody);
        const hasAlert = /alert|notify|alarm|escalat|sendMail|sendEmail|slack/i.test(methodBody);
        const hasDeadLetter = /deadLetter|dlq|failedQueue|errorQueue/i.test(methodBody);
        const hasErrorHandling = method.descendantsOfType('catch_clause').length > 0;

        if (!hasRetry && !hasDeadLetter) {
          findings.push({
            check: 'scheduled_maturity_no_retry',
            severity: 'major',
            filePath: file.path,
            line: method.startPosition.row + 1,
            function: methodName,
            evidence: `@Scheduled method '${methodName}' processes financial lifecycle events ` +
                      `but has no retry mechanism or dead-letter queue. ` +
                      `If processing fails for a specific instrument, it is never retried.`,
            recommendation: 'Add retry logic with exponential backoff, or move failed items to a ' +
                            'dead-letter table for manual review. Consider @Retryable or Spring Batch.',
            confidence: 'likely',
          });
        }

        if (!hasAlert && !hasErrorHandling) {
          findings.push({
            check: 'scheduled_maturity_no_alert',
            severity: 'moderate',
            filePath: file.path,
            line: method.startPosition.row + 1,
            function: methodName,
            evidence: `@Scheduled method '${methodName}' processes financial lifecycle events ` +
                      `but has no alerting mechanism for failures.`,
            recommendation: 'Add alerting on failure — email, Slack, PagerDuty. Log at ERROR level ' +
                            'at minimum so monitoring picks it up.',
            confidence: 'possible',
          });
        }
      }
    }
  }

  return findings;
}
```

-----

## 8. Shared Infrastructure

### 8.1 Types

```typescript
// src/lib/health_score/analyzers/financial_integrity/types.ts

export interface FinancialFinding {
  check: FinancialCheckType;
  severity: 'minor' | 'moderate' | 'major';
  filePath: string;
  line: number;
  function: string;
  evidence: string;
  recommendation: string;
  confidence: 'unlikely' | 'possible' | 'likely';
  needsGraphVerification?: boolean;
}

export type FinancialCheckType =
  // Check 1: Paired operations (all business lines)
  | 'single_legged_debit'
  | 'single_legged_credit'
  | 'loop_debit_credit'
  | 'duplicate_debit'
  // Check 2: Transaction Rollback
  | 'multi_write_no_transaction'
  | 'multi_write_single_repo_no_transaction'
  | 'commit_without_rollback'
  | 'save_then_throw'
  | 'readonly_transaction_with_writes'
  | 'multi_mutation_no_error_handling'
  | 'optimistic_update_no_rollback'
  // Check 3: Unclosed Conditions
  | 'if_chain_no_else'
  | 'switch_no_default'
  | 'enum_switch_missing_cases'
  // Check 4: Error Handling
  | 'empty_catch'
  | 'catch_log_no_rethrow'
  | 'catch_returns_null'
  | 'system_exit_in_service'
  | 'resource_no_finally'
  | 'async_void_no_handler'
  | 'broad_catch_in_financial'
  // Check 5: Maturity / Expiry
  | 'maturity_no_equals_check'
  | 'maturity_no_business_day'
  | 'rollover_missing_open_leg'
  | 'rollover_missing_close_leg'
  | 'scheduled_maturity_no_retry'
  | 'scheduled_maturity_no_alert';
```

### 8.2 Orchestrator

```typescript
// src/lib/health_score/analyzers/financial_integrity/index.ts

export async function runFinancialIntegrityChecks(
  files: LoadedFile[],
  graphClient: IGraphClient,
  pool: Pool,
  repo: string,
  config: ProjectConfig
): Promise<{
  findings: FinancialFinding[];
  score: number;
  rating: string;
  stats: { filesScanned: number; financialFiles: number; checksRun: number; timeMs: number };
}> {
  const startMs = performance.now();

  const financialFiles = files.filter(f => !f.isTestFile && isFinancialFile(f));

  let findings: FinancialFinding[] = [];

  // Check 1: Paired operations (all business lines)
  const dcFindings = analyzeDebitCredit(files, graphClient, repo);
  const dcVerified = await verifyWithCallChain(dcFindings, graphClient, repo);
  findings.push(...dcVerified);

  // Check 2: Transaction Rollback (Java)
  findings.push(...analyzeTransactionRollback(files, config));
  findings.push(...analyzeTypescriptTransactions(files));

  // Check 3: Unclosed Conditions
  findings.push(...analyzeUnclosedConditions(files));

  // Check 4: Error Handling
  findings.push(...analyzeErrorHandling(files, config));

  // Check 5: Maturity / Expiry
  findings.push(...analyzeMaturityExpiry(files));

  // Filter low-confidence
  findings = findings.filter(f => f.confidence !== 'unlikely');

  // Score
  const SEVERITY_DEDUCTIONS = { minor: 0.5, moderate: 1.0, major: 2.0 };
  let score = 10;
  for (const f of findings) { score -= SEVERITY_DEDUCTIONS[f.severity]; }
  score = Math.max(0, Math.min(10, score));

  const rating = score >= 8.5 ? 'Healthy' : score >= 6.0 ? 'Watch' : 'Critical';

  return {
    findings,
    score: Math.round(score * 10) / 10,
    rating,
    stats: {
      filesScanned: files.length,
      financialFiles: financialFiles.length,
      checksRun: 5,
      timeMs: Math.round(performance.now() - startMs),
    },
  };
}
```

-----

## 9. Integration with Health Score

### Option A: New 9th Dimension — “Financial Integrity” (Recommended)

Add alongside existing 8 dimensions. Only scored for repos classified as `backend` or `fullstack` by ProjectAnalyzer.

```typescript
const DIMENSION_WEIGHTS = {
  spring_boot: {
    // ... existing 8 dimensions ...
    financial_integrity: 1.3,
  },
  react: {
    financial_integrity: 0.6,
  },
};
```

### Option B: Sub-checks Under Existing Dimensions

|Finding                    |Maps to Dimension  |
|---------------------------|-------------------|
|single_legged_debit/credit |**Data Access**    |
|loop_debit_credit          |**Data Access**    |
|duplicate_debit            |**Data Access**    |
|multi_write_no_transaction |**Data Access**    |
|commit_without_rollback    |**Reliability**    |
|save_then_throw            |**Reliability**    |
|if_chain_no_else           |**Reliability**    |
|switch_no_default          |**Reliability**    |
|enum_switch_missing_cases  |**Design Patterns**|
|empty_catch                |**Reliability**    |
|system_exit_in_service     |**Reliability**    |
|async_void_no_handler      |**Concurrency**    |
|broad_catch_in_financial   |**Security**       |
|maturity_no_equals_check   |**Reliability**    |
|maturity_no_business_day   |**Reliability**    |
|rollover_missing_*_leg     |**Data Access**    |
|scheduled_maturity_no_retry|**Reliability**    |

**Recommendation:** Go with **Option A** — these checks are domain-specific enough to deserve their own dimension.

-----

## 10. Standalone CLI Mode

```typescript
// src/scripts/run_financial_checks.ts

const repo = process.argv[2];
if (!repo) { console.error('Usage: npx tsx run_financial_checks.ts <repo-slug>'); process.exit(1); }

const repoBasePath = path.join(process.env.REPOS_DIR || '/repos', repo.replace('/', '/'));
const parser = await initParserPool();
const files = await loadSourceFiles(repoBasePath, parser);
const config = await loadProjectConfig(pool, repo, repoBasePath);

const result = await runFinancialIntegrityChecks(files, graphClient, pool, repo, config);

console.log(`\n=== Financial Integrity: ${repo} ===`);
console.log(`Score: ${result.score}/10 (${result.rating})`);
console.log(`Scanned: ${result.stats.filesScanned} files (${result.stats.financialFiles} financial)`);
console.log(`Time: ${result.stats.timeMs}ms\n`);

for (const f of result.findings) {
  const icon = f.severity === 'major' ? '🔴' : f.severity === 'moderate' ? '🟡' : '⚪';
  console.log(`${icon} [${f.check}] ${f.filePath}:${f.line}`);
  console.log(`  ${f.evidence}`);
  console.log(`  → ${f.recommendation}\n`);
}
```

```bash
# Run against a single repo
npx tsx src/scripts/run_financial_checks.ts BMT/ms-bonds-trading

# Run against all backend repos
npx tsx src/scripts/run_financial_checks.ts --all --filter=backend
```

-----

## 11. Layer 3 LLM Augmentation

For ambiguous findings (confidence = ‘possible’), Qwen2.5-7B reviews the full file:

```typescript
const prompt = `You are reviewing a financial services codebase for transaction safety.
This system handles Bonds, Equity, Loans, Deposits, and Funds.

FILE: ${filePath}
FUNCTION: ${funcName}

${fullFileContent}

FINDING: ${finding.evidence}

Is this a genuine financial integrity risk, or is it intentional/safe?
Consider:
- Is this a fee collection or interest accrual (intentional single-leg)?
- Is the counterpart in a different service (saga/choreography pattern)?
- Is the loop processing distinct items with natural keys (not duplicates)?
- Is the condition exhaustive given the business domain?
- Is business-day adjustment handled by a shared utility upstream?
- Is rollover handled in two separate @Transactional methods by design?

Respond with JSON:
{
  "is_genuine_risk": true/false,
  "reason": "brief explanation",
  "adjusted_severity": "minor" | "moderate" | "major" | "dismiss"
}`;
```

This runs during the nightly health score — not on the query path.

-----

## 12. Implementation Schedule

```
Day 1: Check 3 (Unclosed Conditions) + Check 4 (Error Handling)
  ├── unclosed_conditions.ts — if chains, switch/default, enum coverage
  ├── error_handling.ts — empty catch, System.exit, resource cleanup, @Async
  ├── Shared types + helpers (isFinancialFile, isFinanciallySensitive)
  └── These two are language-generic and easiest to validate

Day 2: Check 2 (Transaction Rollback)
  ├── transaction_rollback.ts — Java @Transactional, manual commit, save-then-throw
  ├── transaction_rollback_ts.ts — multi-mutation, optimistic update
  └── Config cross-reference (application.yml for TX settings)

Day 3: Check 1 (Debit/Credit) + Graph Verification
  ├── debit_credit.ts — all business line patterns, loop detection, idempotency guards
  ├── debit_credit_graph.ts — call chain verification for single-legged
  └── Financial file pre-filter heuristic (expanded for all lines)

Day 4: Check 5 (Maturity/Expiry) + Orchestrator
  ├── maturity_expiry.ts — date comparison, business day, rollover legs, scheduled retry
  ├── Orchestrator (index.ts) — wire all 5 checks
  └── CLI runner script

Day 5: Integration + Testing
  ├── Wire into health score as 9th dimension
  ├── Run against 5+ repos across all business lines
  ├── Tune false positive rates per business line
  └── Verify rollover detection on deposit and loan repos
```

### File Structure

```
src/lib/health_score/analyzers/financial_integrity/
  index.ts                    — orchestrator (5 checks)
  types.ts                    — FinancialFinding, FinancialCheckType (27 check types)
  helpers.ts                  — isFinancialFile, isFinanciallySensitive, AST utilities
  debit_credit.ts             — Check 1: paired operation analysis (all business lines)
  debit_credit_graph.ts       — Check 1: call chain verification
  transaction_rollback.ts     — Check 2: Java transaction integrity
  transaction_rollback_ts.ts  — Check 2: TypeScript mutations
  unclosed_conditions.ts      — Check 3: missing else/default/enum cases
  error_handling.ts           — Check 4: empty catch, exit, resources, @Async
  maturity_expiry.ts          — Check 5: maturity dates, business days, rollover, scheduled retry

src/scripts/
  run_financial_checks.ts     — standalone CLI
```

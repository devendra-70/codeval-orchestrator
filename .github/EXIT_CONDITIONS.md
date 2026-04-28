# Loop Exit Conditions & Human Decision Points

**Last Updated:** April 28, 2026  
**Applies To:** Stage 3 Review Loop (orchestrator-ticket-agent)  
**Status:** Production Spec

---

## Exit Condition Evaluation Order

Conditions are evaluated in **STRICT PRECEDENCE ORDER** after every successful build completion (Checkpoint I-N).

### The Decision Tree

```
START: Exit Evaluation
│
├─ [MANDATORY] Coverage meets ALL thresholds?
│  │  (Services ≥90%, Controllers ≥100%, Overall ≥85%)
│  │
│  ├─ NO
│  │  └─ ACTION: AUTO RESTART LOOP at 3a
│  │     LOG: LOOP_RESTART_COVERAGE
│  │     HUMAN: Sees nothing (automatic)
│  │
│  └─ YES
│     │
│     ├─ [CHECK] Run count ≥ max_runs (default: 10)?
│     │  │
│     │  ├─ YES
│     │  │  └─ ACTION: PRESENT CHECKPOINT
│     │  │     OPTIONS: [DONE / CONTINUE / ABORT]
│     │  │     LOG: LOOP_EXIT_MAX_RUNS
│     │  │
│     │  └─ NO
│     │     │
│     │     ├─ [CHECK] CRITICAL issues = 0 AND HIGH issues = 0?
│     │     │  │
│     │     │  ├─ YES
│     │     │  │  └─ ACTION: PRESENT CHECKPOINT
│     │     │  │     OPTIONS: [DONE (✓ recommended) / CONTINUE]
│     │     │  │     LOG: LOOP_EXIT_ZERO_ISSUES
│     │     │  │
│     │     │  └─ NO
│     │     │     │
│     │     │     ├─ [CHECK] Persistent escalation detected (≥3 runs)?
│     │     │     │  │
│     │     │     │  ├─ YES
│     │     │     │  │  └─ ACTION: PRESENT CHECKPOINT
│     │     │     │  │     OPTIONS: [ESCALATE / OVERRIDE / DEFER]
│     │     │     │  │     LOG: LOOP_EXIT_ESCALATION
│     │     │     │  │
│     │     │     │  └─ NO
│     │     │     │     │
│     │     │     │     └─ ACTION: AUTO RESTART LOOP at 3a
│     │     │     │        LOG: LOOP_CONTINUE
│     │     │     │        HUMAN: Sees nothing (automatic)
│     │     │     │
│     │     │     └──────────────────────────────────
│     │     │
│     │     └──────────────────────────────────────
│     │
│     └──────────────────────────────────────────
│
END
```

---

## Condition Definitions

### Condition 1: Coverage Below Threshold (MANDATORY - AUTO-RESTART)

**Trigger Point:** After Step 3f (Coverage Verification)

**Check:**
```python
if NOT (service_cov >= 90 AND controller_cov >= 100 AND overall_cov >= 85):
    # ANY metric below target
    action = LOOP_RESTART
```

**When This Happens:**
- Coverage metrics from post-refactor build fall below targets
- Examples:
  - Service coverage drops to 88% (below 90%)
  - Controller coverage drops to 98% (below 100%)
  - Overall coverage is 82% (below 85%)

**Human Sees:** Nothing (automatic restart)

**System Action:**
1. Log: `LOOP_RESTART_COVERAGE`
2. Reset: `attempt_count = 0`
3. Increment: `run_number = N + 1`
4. Restart: Jump to Step 3a (Unit Test Agent)

**Rationale:** Coverage is non-negotiable threshold; restart required to remediate

---

### Condition 2: Max Runs Reached (CHECK SECOND)

**Trigger Point:** After Condition 1 check passes (coverage OK)

**Check:**
```python
if run_count >= max_runs:  # default: 10
    action = PRESENT_CHECKPOINT
```

**When This Happens:**
- Loop iteration number reaches or exceeds 10 (configurable)
- Examples:
  - Run 1, 2, 3, 4, 5, 6, 7, 8, 9, 10 → Checkpoint
  - User overrides to continue, reaches 11 → Checkpoint again

**Human Sees:** Clear checkpoint message

**Options:**
- `DONE` → Accept current state, exit loop
  - *Typical:* Coverage OK but some MEDIUM issues remain
  - *Risk:* Technical debt left unfixed
  - *Log:* `LOOP_EXIT_MAX_RUNS_DONE`

- `CONTINUE` → Override max runs, run iteration {N+1}
  - *Typical:* Important quality gate approaching, need 1-2 more runs
  - *Risk:* Could loop indefinitely if not careful
  - *Log:* `LOOP_CONTINUE_MAX_RUNS_OVERRIDE`

- `ABORT` → Stop pipeline, no further iterations
  - *Typical:* Give up, manual fix required
  - *Risk:* Ticket incomplete, developer must retry
  - *Log:* `LOOP_EXIT_MAX_RUNS_ABORT`

**Presentation:**
```
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
⚠️ MAX ITERATIONS REACHED
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

Run Count: 10 / 10

Current State:
  Coverage: ✅ All thresholds met
  Issues: CRITICAL=0, HIGH=2, MEDIUM=5
  Escalations: None

You have completed the maximum number of automatic loop iterations.
Remaining issues can be addressed:
  • Manually by the developer (recommended for MEDIUM issues)
  • Via another ticket
  • By overriding to extend this ticket

RESPOND WITH:
  DONE       → Exit with current state (coverage + zero CRITICAL/HIGH)
  CONTINUE   → Run iteration 11 (override limit)
  ABORT      → Stop pipeline; manual work required

─────────────────────────────────────────────────────
```

---

### Condition 3: Zero CRITICAL/HIGH Issues (CHECK THIRD)

**Trigger Point:** After Conditions 1 & 2 pass

**Check:**
```python
if critical_count == 0 AND high_count == 0:
    quality_ok = True
    action = PRESENT_CHECKPOINT
```

**When This Happens:**
- Code Review Agent report shows:
  - CRITICAL section: (none)
  - HIGH section: (none)
  - MEDIUM may still have items
  - LOW may still have items

**Human Sees:** Success checkpoint (recommended DONE)

**Options:**
- `DONE` ✓ **RECOMMENDED** → Exit loop
  - *Typical:* Happy path - all critical gates passed
  - *State:* Coverage OK, tests pass, no major issues
  - *Action:* Proceed to Stage 4 (Pipeline Complete)
  - *Log:* `LOOP_EXIT_ZERO_ISSUES_DONE`

- `CONTINUE` → Run another iteration
  - *Typical:* Want to fix MEDIUM issues before merge
  - *Risk:* Could loop indefinitely
  - *Action:* Go back to Step 3a
  - *Log:* `LOOP_CONTINUE_ZERO_ISSUES_OVERRIDE`

**Presentation:**
```
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
✅ ALL CRITICAL QUALITY GATES PASSED
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

Coverage: ✅ All thresholds met
  Services:    92% / 90% ✅
  Controllers: 100% / 100% ✅
  Overall:     86% / 85% ✅

Quality Status: ✅ Safe for merge
  CRITICAL:    0
  HIGH:        0
  MEDIUM:      3
  LOW:         2

Run Count: 5 / 10

The ticket is ready for production!

RESPOND WITH:
  DONE       → Exit loop and complete pipeline (recommended)
  CONTINUE   → Run iteration 6 (optional: address MEDIUM issues)

─────────────────────────────────────────────────────
```

---

### Condition 4: Persistent Escalation (CHECK FOURTH)

**Trigger Point:** After Conditions 1, 2, 3 fail

**Check:**
```python
if any_issue_appears_in_3_plus_consecutive_runs():
    escalation_detected = True
    action = PRESENT_CHECKPOINT
```

**When This Happens:**
- loop_tracker detects an issue that appeared in runs:
  - Run 1 (found)
  - Run 2 (found again)
  - Run 3 (found for 3rd time)
  - → Escalation triggered

**Examples:**
- Long method in OrderService that Refactor Agent couldn't fix
- N+1 query in ProductRepository (needs DB schema redesign)
- Race condition in ApprovalService (requires architectural change)

**Human Sees:** Escalation checkpoint with issue details

**Options:**
- `ESCALATE` → Mark as known issue, exit loop
  - *Action:* Log the issue, recommend manual review
  - *Next:* Create separate ticket for manual developer
  - *Log:* `LOOP_EXIT_ESCALATION_ESCALATE`
  - *Typical:* "This needs database redesign, out of scope"

- `OVERRIDE` → Accept and keep trying
  - *Action:* Continue loop, attempt to force fix
  - *Risk:* Could loop indefinitely
  - *Log:* `LOOP_EXIT_ESCALATION_OVERRIDE`
  - *Typical:* "Let's try one more refactor with specific instructions"

- `DEFER` → Postpone to later ticket
  - *Action:* Exit loop, add to backlog
  - *Next:* Create follow-up ticket
  - *Log:* `LOOP_EXIT_ESCALATION_DEFER`
  - *Typical:* "This is good enough for this ticket, deal with it next sprint"

**Presentation:**
```
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
⚠️ PERSISTENT ISSUE ESCALATION
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

Issue Detected in {N}+ Consecutive Runs (threshold: 3+)

Persistent Issues:

1. LONG METHOD: OrderService.processOrder() (180 lines)
   Runs: 1, 2, 3, 4, 5
   Status: Code Refactor attempted extraction but failed (high complexity)
   
2. N+1 QUERY: ProductService.getProductsWithDetails()
   Runs: 3, 4, 5
   Status: Requires JOIN FETCH or repository redesign
   Root Cause: JPA entity mapping issue, not fixable by Bugfix Agent

This issue persists despite multiple iterations of:
  • Unit tests
  • Bugfixes
  • Code refactoring

Recommendation: Schedule manual developer review or create follow-up ticket.

RESPOND WITH:
  ESCALATE   → Mark as known issue, exit loop (recommended)
               Advise: Create AIS-999 for DB schema remediation
               
  OVERRIDE   → Try one more iteration with specific instructions
               Risk: Could loop indefinitely
               
  DEFER      → Exit loop, handle in next iteration/ticket
               Note: Document in Jira

─────────────────────────────────────────────────────
```

---

### Condition 5: Continue Loop (DEFAULT)

**Trigger Point:** Conditions 1-4 all fail

**When This Happens:**
- Coverage == OK
- Run count < 10
- CRITICAL/HIGH issues > 0 (but no persistent escalation)

**Human Sees:** Nothing (automatic restart)

**System Action:**
1. Log: `LOOP_CONTINUE`
2. Increment: `run_number = N + 1`
3. Reset: `attempt_count = 0` (for new unit test retry cycle)
4. Restart: Jump to Step 3a (Unit Test Agent, Run {N+1})

**Rationale:** There are still fixable issues; continue iterating

---

## Log Events for Tracking

**Before Evaluation:**
- `CHECKPOINT_I_REACHED` → Coverage Verification checkpoint shown

**During Evaluation:**
- `COVERAGE_VERIFICATION_PASSED` → Coverage OK
- `COVERAGE_VERIFICATION_DEGRADED` → Coverage below threshold
- `LOOP_RESTART_COVERAGE` → Auto-restart due to coverage

**Exit Decision Logs:**
- `LOOP_EXIT_MAX_RUNS` → Checkpoint for max runs
- `LOOP_EXIT_ZERO_ISSUES` → Checkpoint for zero issues
- `LOOP_EXIT_ESCALATION` → Checkpoint for escalation
- `LOOP_CONTINUE` → Auto-restart with quality issues

**Human Response Logs:**
- `LOOP_EXIT_MAX_RUNS_DONE` → Human chose DONE at max
- `LOOP_CONTINUE_MAX_RUNS_OVERRIDE` → Human chose CONTINUE
- `LOOP_EXIT_ZERO_ISSUES_DONE` → Human chose DONE at zero issues
- `LOOP_CONTINUE_ZERO_ISSUES_OVERRIDE` → Human chose CONTINUE
- `LOOP_EXIT_ESCALATION_ESCALATE` → Human escalated issue
- `LOOP_EXIT_ESCALATION_DEFER` → Human deferred issue

---

## Configuration (in agent-config.yml)

```yaml
loop:
  max_runs: 10                               # Hard ceiling on review loop iterations
  persistent_issue_threshold: 3             # Consecutive runs before escalation
```

To change limits, edit `agent-config.yml` and notify team.

---



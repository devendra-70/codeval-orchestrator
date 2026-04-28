# Implementation Guide: Coverage-Aware Loop with Code Refactor Integration

**Date:** April 28, 2026  
**Status:** Ready for Implementation  
**Action:** DO NOT MODIFY EXISTING FILES — Use this guide for manual edits

---

## Overview

This document provides exact line-by-line changes needed to implement:
1. Coverage threshold checks (Services ≥90%, Controllers ≥100%, Overall ≥85%)
2. Unit test retry logic (up to 3 attempts)
3. Code Refactor Agent integration
4. Post-refactor coverage verification
5. Comprehensive loop exit conditions

---

## File Change Summary

| File | Action | Type |
|---|---|---|
| `orchestrator-ticket-agent.agent.md` | Multiple sections updated | MODIFY (use EXACT instructions below) |
| `bugfix-agent.agent.md` | Rename checkpoint reference | MODIFY (see Section 2) |
| `.github/COVERAGE_THRESHOLDS.md` | NEW CONFIG FILE | CREATE |
| `.github/EXIT_CONDITIONS.md` | NEW SPEC FILE | CREATE |

---

## Section 1: Changes to `orchestrator-ticket-agent.agent.md`

### **Change 1.1: Update Pipeline Diagram (Lines ~104-132)**

**FIND THIS EXACT BLOCK:**
```
```
[Human pulls ticket + provides TICKET_ID]
         │
         ▼
Stage 0: Build Ticket Context
  → ticket-context-builder skill
  → Creates: tickets/{id}/ticket-context.md + folder structure
         │
         ▼
Stage 0.5: Git Branch Setup
  → git-branch-manager.setup_branch
  → Asks human for TEAM_PREFIX (AIS / EXE / CVS / FE) once
  → Creates: tickets/{id}/git-context.md + checks out feature/bugfix/release/hotfix branch
  → On CANCEL: git_enabled=false, pipeline continues without any Git ops
         │
         ▼
Stage 1: Architecture
  → Architecture Design Agent (/ticket-architect only)
  → [Draft produced, returned to orchestrator]
         │
    [CHECKPOINT A — human: APPROVE / REJECT / REVISE]
         │ APPROVE only
         ▼
  → Architecture Design Agent (/approve)
  → Output file created: tickets/{id}/architecture/architecture-decision.md
         │ [COMMIT? — human: COMMIT / EDIT: <msg> / SKIP / GIT STATUS]
         ▼
Stage 2: Backend Implementation
  → Backend Implementation Agent (/audit → /generate → /approve)
  → Output: tickets/{id}/implementation/files-changed.md
           tickets/{id}/implementation/implementation-notes.md
           src/main/java/... (actual code files)
         │
    [CHECKPOINT B — human: APPROVE / REJECT / REVISE]
         │ APPROVE
         │ [COMMIT? — human: COMMIT / EDIT: <msg> / SKIP / GIT STATUS]
         ▼
Stage 3: Review Loop (runs until human says DONE or exit condition met)
  ┌──────────────────────────────────────────────────────────┐
  │  loop-tracker: start_run(ticket_id)                      │
  │         │                                                │
  │  Unit Test Agent → run-N-report.md                       │
  │    [CHECKPOINT C-N — APPROVE / REJECT / REVISE / DONE]   │
  │         │ APPROVE   
  │         │ [COMMIT? — human: COMMIT / EDIT: <msg> / SKIP / GIT STATUS]
  │  Bugfix Agent → run-N-bugfix.md                          │
  │    [CHECKPOINT F-N — APPROVE / REJECT / REVISE / DONE]   │
  │         │ APPROVE
  │         │ [COMMIT? — human: COMMIT / EDIT: <msg> / SKIP / GIT STATUS]
  │  Code Review Agent → run-N-review.md                     │
  │    [CHECKPOINT D-N — APPROVE / REJECT / REVISE / DONE]   │
  │         │ APPROVE  
  │         │ [COMMIT? — human: COMMIT / EDIT: <msg> / SKIP / GIT STATUS]
  │  loop-tracker: register_issues(...)                      │
  │  [Check for persistent issues → escalate if needed]      │
  │         │                                                │
  │  Code Refactor Agent (silent unless build fails)         │
  │    ├─ BUILD PASSED → Coverage Verification              │
  │    └─ BUILD FAILED → Bugfix recovery (Step 3e)          │
  │         │                                                │
  │  [Evaluate exit conditions]                              │
  └──────────────────────────────────────────────────────────┘
         │ EXIT CONDITION MET
         ▼
Stage 4: Pipeline Complete
  → release_all_locks(ticket_id)
  → Log PIPELINE_COMPLETE
  → Notify human
```
```

**REPLACE WITH:**
```
```
[Human pulls ticket + provides TICKET_ID]
         │
         ▼
Stage 0: Build Ticket Context
  → ticket-context-builder skill
  → Creates: tickets/{id}/ticket-context.md + folder structure
         │
         ▼
Stage 0.5: Git Branch Setup
  → git-branch-manager.setup_branch
  → Asks human for TEAM_PREFIX (AIS / EXE / CVS / FE) once
  → Creates: tickets/{id}/git-context.md + checks out feature/bugfix/release/hotfix branch
  → On CANCEL: git_enabled=false, pipeline continues without any Git ops
         │
         ▼
Stage 1: Architecture
  → Architecture Design Agent (/ticket-architect only)
  → [Draft produced, returned to orchestrator]
         │
    [CHECKPOINT A — human: APPROVE / REJECT / REVISE]
         │ APPROVE only
         ▼
  → Architecture Design Agent (/approve)
  → Output file created: tickets/{id}/architecture/architecture-decision.md
         │ [COMMIT? — human: COMMIT / EDIT: <msg> / SKIP / GIT STATUS]
         ▼
Stage 2: Backend Implementation
  → Backend Implementation Agent (/audit → /generate → /approve)
  → Output: tickets/{id}/implementation/files-changed.md
           tickets/{id}/implementation/implementation-notes.md
           src/main/java/... (actual code files)
         │
    [CHECKPOINT B — human: APPROVE / REJECT / REVISE]
         │ APPROVE
         │ [COMMIT? — human: COMMIT / EDIT: <msg> / SKIP / GIT STATUS]
         ▼
Stage 3: Review Loop (runs until human says DONE or exit condition met)
  ┌──────────────────────────────────────────────────────────┐
  │  loop-tracker: start_run(ticket_id)                      │
  │  attempt_count = 0                                       │
  │         │                                                │
  │  3a: Unit Test Agent (with Coverage Threshold Check)    │
  │      → run-N-report.md                                   │
  │      → Extract coverage: Services, Controllers, Overall  │
  │      → If below threshold AND attempt < 3:               │
  │         Retry Unit Test Agent (attempt++)               │
  │      → Else: Show coverage & proceed                     │
  │    [CHECKPOINT H-N — Coverage Report]                    │
  │         │ APPROVE (proceed or override)
  │         │ [COMMIT? — human: COMMIT / EDIT / SKIP]       │
  │         ▼                                                │
  │  3b: Bugfix Agent → run-N-bugfix.md                      │
  │    [CHECKPOINT F-N — APPROVE / REJECT / REVISE]         │
  │         │ APPROVE
  │         │ [COMMIT? — human: COMMIT / EDIT / SKIP]       │
  │         ▼                                                │
  │  3c: Code Review Agent → run-N-review.md                │
  │    [CHECKPOINT D-N — APPROVE / REJECT / REVISE]         │
  │         │ APPROVE  
  │         │ [COMMIT? — human: COMMIT / EDIT / SKIP]       │
  │         ▼                                                │
  │  3d: Code Refactor Agent (silent unless build fails)     │
  │      ├─ BUILD PASSED → Step 3f                          │
  │      └─ BUILD FAILED → Step 3e (Bugfix recovery)        │
  │         │ (if triggered)                                │
  │         ▼                                                │
  │  3f: Coverage Verification (Post-Refactor)              │
  │      → Check coverage thresholds again                  │
  │      → If degraded or below: RESTART option            │
  │    [CHECKPOINT I-N — Post-Refactor Coverage]            │
  │         │ APPROVE / RESTART                             │
  │         ├─ RESTART? → Back to 3a (Loop N+1)            │
  │         └─ APPROVE? → Exit Evaluation                   │
  │  loop-tracker: register_resolved_issues(...)            │
  │  [Evaluate exit conditions]                             │
  └──────────────────────────────────────────────────────────┘
         │ EXIT CONDITION MET
         ▼
Stage 4: Pipeline Complete
  → release_all_locks(ticket_id)
  → Log PIPELINE_COMPLETE
  → Notify human
```
```

---

### **Change 1.2: Replace Step 3a — Unit Test Agent Section (Lines ~375-471)**

**FIND SECTION HEADER:**
```markdown
#### Step 3a — Unit Test Agent
```

**FIND THIS ENTIRE SECTION (entire Step 3a section - starting at line ~375)** and replace with:

```markdown
#### Step 3a — Unit Test Agent (with Coverage Threshold Checks)

**Trigger:** Loop Start OR Coverage Retry OR Refactor Recovery (from 3f or 3e)

1. Initialize (first run only):
   ```
   attempt_count = 0  (tracks unit test retries)
   max_coverage_attempts = 3
   ```

2. Invoke Unit Test Agent with:
   - `TICKET_ID = {id}`
   - `RUN_NUMBER = {N}`
   - `ATTEMPT = {attempt_count}`

3. Agent reads `implementation/files-changed.md` (Run 1) or `bugfix-reports/run-{N-1}-bugfix.md` (Run N+1)
4. Agent writes `tickets/{TICKET_ID}/test-reports/run-{N}-report.md`

5. **Extract Coverage Metrics from TEST_REPORT.md:**

   Read from "Coverage Report → Class Level" section:
   ```
   Service Layer Coverage:    {X}% (target: ≥90%)
   Controller Layer Coverage: {Y}% (target: ≥100%)
   Overall Coverage:          {Z}% (target: ≥85%)
   ```

6. **Coverage Threshold Evaluation:**

   ```python
   service_ok = service_coverage >= 90
   controller_ok = controller_coverage >= 100
   overall_ok = overall_coverage >= 85
   all_ok = service_ok AND controller_ok AND overall_ok
   
   if not all_ok AND attempt_count < 3:
       # Retry: inform human and re-run Unit Test Agent
       attempt_count += 1
       Log: COVERAGE_RETRY_{attempt_count}
       Go back to Step 2 (invoke Unit Test Agent again)
   elif not all_ok AND attempt_count == 3:
       # Max retries reached, proceed with warning
       Log: COVERAGE_BELOW_THRESHOLD_MAX_ATTEMPTS
       Present Checkpoint H with ⚠️ flag
   else:
       # Coverage OK, proceed
       Log: COVERAGE_THRESHOLDS_MET
       Present Checkpoint H
   ```

7. **Present Checkpoint H-{N}:**

```
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
🔵 CHECKPOINT H-{N} — Unit Test Agent (Loop Run {N})
Ticket: {TICKET_ID}  |  Attempt: {attempt_count}/{max_coverage_attempts}  |  Time: {TIMESTAMP}
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

📄 ARTIFACT: tickets/{TICKET_ID}/test-reports/run-{N}-report.md

SUMMARY:
Passed: {X} | Failed: {Y} | Skipped: {Z}
Build: {PASSED/FAILED/BUILD ERROR}

📊 COVERAGE REPORT:

| Layer      | Coverage | Target | Status |
|---|---|---|---|
| Service    | {X}%     | ≥90%   | {✅/⚠️} |
| Controller | {Y}%     | ≥100%  | {✅/⚠️} |
| Overall    | {Z}%     | ≥85%   | {✅/⚠️} |

⚠️ FLAGS:
{Failed tests list, coverage gaps, build errors}

If tests failed or build errors occurred, it's critical to address these before proceeding.

─────────────────────────────────────────────────────

IF Coverage BELOW Threshold AND Attempt < 3:
  ⚡ Coverage below target. Will retry Unit Test Agent automatically.
  
  RESPOND WITH:
    APPROVE          → Skip coverage check, proceed to Bugfix
    REVISE: {notes}  → Retry with specific instructions
    /skip-coverage   → Acknowledge and proceed (override threshold)

ELSE (Coverage OK OR Attempt = 3):
  ✅ Coverage check complete. Ready to proceed.
  
  RESPOND WITH:
    APPROVE           → Proceed to Bugfix Agent
    REJECT            → Re-run Unit Test Agent manually
    /skip-coverage    → Override coverage threshold
    DONE              → Exit loop
─────────────────────────────────────────────────────
```

8. **Handle Response:**

   - **APPROVE** (if coverage below): 
     - If attempt < 3: Auto-retry Unit Test Agent (increment attempt_count, go to step 2)
     - If attempt = 3: Log warning, proceed to Bugfix
   
   - **APPROVE** (if coverage OK):
     - Log: COVERAGE_THRESHOLDS_MET
     - Proceed to Step 3b (Bugfix Agent)
   
   - **REVISE**: 
     - If attempt < 3: Retry with human notes
     - If attempt = 3: Proceed to Bugfix with notes logged
   
   - **/skip-coverage** (override):
     - Log: COVERAGE_THRESHOLD_OVERRIDE
     - Proceed to Bugfix
   
   - **REJECT**:
     - Re-run Unit Test Agent (reset attempt_count)
   
   - **DONE**:
     - Exit loop

9. **Commit sub-checkpoint** (only if `git_enabled: true` in `git-context.md`):

   Invoke `gitignore-curator.classify_and_propose` against the stage's candidate file list.

   Then invoke `git-branch-manager.commit_stage` with `STAGE_LABEL = unit-tests-run-{N}`.

   The skill presents:
   ```
   ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
   💾 COMMIT? — {TICKET_ID} — Stage: unit-tests-run-{N}
   ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
   Branch : {BRANCH_NAME}
   Files  : {scoped list from stage artifact}
   Proposed message:
     test: {short} -- {TICKET_ID}
   ─────────────────────────────────────────────────────
   RESPOND WITH:
     COMMIT             → Commit with proposed message
     EDIT: {full msg}   → Commit with your message instead
     SKIP               → Skip commit for this stage
     GIT STATUS         → Show working tree status first
   ─────────────────────────────────────────────────────
   ```

   On `COMMIT` / `EDIT:` → the skill stages and commits.
   On `SKIP` → logged as `COMMIT_SKIPPED`.
   In all cases, advance to Step 3b.

---

### **Change 1.3: Replace Step 3b — Bugfix Agent Section (Lines ~545-632)**

**FIND SECTION HEADER:**
```markdown
#### Step 3b — Bugfix Agent
```

Change `Checkpoint E-{N}` references to `Checkpoint F-{N}`.

**OLD:**
```markdown
#### Step 3b — Bugfix Agent

1. Invoke Bugfix Agent with:
   - `TICKET_ID = {id}`
   - `RUN_NUMBER = {N}`
2. Agent runs `/analyze` — produces fix plan
3. **Orchestrator presents fix plan to human before `/approve`:**

```
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
🔧 BUGFIX PLAN — {TICKET_ID} — Run {N}
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
...
```

8. Present **Checkpoint E-{N}**:

```
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
🔵 CHECKPOINT E-{N} — Bugfix Agent completed (Loop Run {N})
```

**NEW:**
```markdown
#### Step 3b — Bugfix Agent

1. Invoke Bugfix Agent with:
   - `TICKET_ID = {id}`
   - `RUN_NUMBER = {N}`
2. Agent runs `/analyze` — produces fix plan
3. **Orchestrator presents fix plan to human before `/approve`:**

```
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
🔧 BUGFIX PLAN — {TICKET_ID} — Run {N}
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
...
```

8. Present **Checkpoint F-{N}**:

```
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
🔵 CHECKPOINT F-{N} — Bugfix Agent completed (Loop Run {N})
```

---

### **Change 1.4: Replace Step 3c — Code Review Agent Section (Lines ~474-541)**

**FIND SECTION HEADER:**
```markdown
#### Step 3c — Code Review Agent
```

Stays the same (checkpoint D-N is correct), but rename to match new numbering if needed.

---

### **Change 1.5: Add Step 3d, 3e, 3f Sections (NEW sections AFTER Step 3c)**

**Insert AFTER the End of Step 3c — Code Review Agent section:**

```markdown
---

#### Step 3d — Code Refactor Agent

**Trigger:** Code Review Agent completed and approved (Step 3c APPROVE)

This step is **silent** — no human checkpoint unless build fails.

1. Verify `code-review-report.md` and `TEST_REPORT.md` exist
2. Invoke Code Refactor Agent with:
   - `TICKET_ID = {id}`
   - `RUN_NUMBER = {N}`
3. Agent reads both reports
4. Agent applies refactoring fixes (CRITICAL + HIGH severity)
5. Agent writes `REFACTOR_REPORT.md`
6. Run build check:
   ```bash
   mvn clean verify -Dmaven.test.failure.ignore=true -q
   ```

7. **Evaluate Build Result:**

   **IF BUILD PASSED:**
   - Log: `REFACTOR_BUILD_PASSED`
   - Proceed to Step 3f (Coverage Verification)

   **IF BUILD FAILED:**
   - Log: `REFACTOR_BUILD_FAILED`
   - Proceed to Step 3e (Bugfix Recovery)

---

#### Step 3e — Bugfix Agent (Conditional — Refactor Recovery)

**Trigger:** Code Refactor Agent introduced build failure (Step 3d)

Only runs if previous step's build failed.

1. Invoke Bugfix Agent with:
   - `TICKET_ID = {id}`
   - `RUN_NUMBER = {N}`
   - `CONTEXT = "refactor-recovery"` (short cycle, bug fix only)
2. Agent runs `/analyze` — identifies refactor-induced bugs
3. **Orchestrator presents brief plan:**

```
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
⚠️ REFACTOR RECOVERY — Bugfix Agent
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

Code Refactor Agent introduced build errors.
Running Bugfix Agent to fix...

{Bugfix Agent /analyze output}

─────────────────────────────────────────────────────
RESPOND WITH:
  /approve         → Apply fixes
  REVISE: {notes}  → Provide specific instructions
  ABORT            → Stop pipeline; manual fix required
─────────────────────────────────────────────────────
```

4. On `/approve`: Agent fixes and re-compiles
5. Agent writes `bugfix-reports/run-{N}-bugfix-refactor.md`
6. Run build check again:
   ```bash
   mvn clean verify -Dmaven.test.failure.ignore=true -q
   ```

7. **Evaluate Build Result:**

   **IF BUILD PASSED:**
   - Log: `BUGFIX_REFACTOR_BUILD_RECOVERED`
   - Proceed to Step 3f (Coverage Verification)

   **IF BUILD STILL FAILED:**
   - Present checkpoint:
   ```
   ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
   ❌ BUILD STILL FAILING after Bugfix Recovery
   ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
   
   Refactor + Bugfix cannot recover. Manual intervention required.
   
   RESPOND WITH:
     REVISE: {deep fix}  → Provide specific fix instructions
     ABORT               → Stop pipeline; manual dev work needed
   ─────────────────────────────────────────────────────
   ```
   
   On `REVISE`: Retry Code Refactor Agent with new instructions
   On `ABORT`: Log `PIPELINE_ABORTED`, release locks, stop

---

#### Step 3f — Coverage Verification (Post-Refactor)

**Trigger:** Code Refactor (3d) or Bugfix Recovery (3e) with BUILD PASSED

Verifies that refactoring did not reduce coverage below thresholds.

1. Coverage data already available from previous build
   (refactor step ran `mvn clean verify` which includes JaCoCo)

2. Extract coverage from `target/site/jacoco/jacoco.xml`:
   - Service layer coverage: {X}%
   - Controller layer coverage: {Y}%
   - Overall coverage: {Z}%

3. **Compare to Thresholds:**

   ```python
   service_ok = service_coverage >= 90
   controller_ok = controller_coverage >= 100
   overall_ok = overall_coverage >= 85
   
   if service_ok AND controller_ok AND overall_ok:
       coverage_status = "✅ ACCEPTABLE"
   else:
       coverage_status = "⚠️ DEGRADED"
   ```

4. **Present Checkpoint I-{N}:**

```
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
🔵 CHECKPOINT I-{N} — Coverage Verification (Post-Refactor)
Ticket: {TICKET_ID}  |  Stage: Post-Refactor Coverage  |  Time: {TIMESTAMP}
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

📊 COVERAGE REPORT (After Refactor):

| Layer      | Coverage | Target | Status |
|---|---|---|---|
| Service    | {X}%     | ≥90%   | {✅/⚠️} |
| Controller | {Y}%     | ≥100%  | {✅/⚠️} |
| Overall    | {Z}%     | ≥85%   | {✅/⚠️} |

Coverage Status: {coverage_status}

📈 Changes from Pre-Refactor:
  Service:    {Δ+X% / Δ−X%}
  Controller: {Δ+X% / Δ−X%}
  Overall:    {Δ+X% / Δ−X%}

─────────────────────────────────────────────────────
RESPOND WITH:
  APPROVE          → Continue to Loop Exit Evaluation
  REVISE: {notes}  → Proceed with notes (for next iteration)
  RESTART          → Restart loop at 3a (Run {N+1})
─────────────────────────────────────────────────────
```

5. **Handle Response:**

   - **APPROVE** (coverage OK):
     - Log: `COVERAGE_VERIFICATION_PASSED`
     - Proceed to Loop Exit Evaluation

   - **APPROVE** (coverage degraded):
     - Log: `COVERAGE_VERIFICATION_DEGRADED_APPROVED`
     - Proceed to Loop Exit Evaluation (human accepted)

   - **REVISE**:
     - Log: `COVERAGE_VERIFICATION_REVISED`
     - Store notes
     - Proceed to Loop Exit Evaluation

   - **RESTART**:
     - Log: `LOOP_RESTART_COVERAGE_DEGRADED`
     - Reset attempt_count = 0
     - Increment run_counter (N+1)
     - Go back to Step 3a (Unit Test Agent)

---

#### Loop Exit Condition Evaluation (UPDATED)

**Trigger:** After Checkpoint I-{N} (Coverage Verification) or Checkpoint D-{N} (if no refactor)

Evaluate in STRICT order:

```python
# 1. Check coverage first
if NOT (service_cov >= 90 AND controller_cov >= 100 AND overall_cov >= 85):
    # Coverage below threshold
    Log: LOOP_EXIT_COVERAGE_BELOW_THRESHOLD
    present_autoexit_continue("Coverage thresholds not met. Restarting loop.")
    restart_loop_at_3a()
    return

# 2. Check run count
if run_count >= max_runs (default 10):
    Log: LOOP_EXIT_MAX_RUNS
    present_checkpoint_with_options("MAX_RUNS_REACHED", 
                                    options=["DONE", "CONTINUE", "ABORT"])
    return

# 3. Check quality issues
if critical_or_high_count == 0:
    Log: LOOP_EXIT_ZERO_ISSUES
    present_checkpoint_with_options("ZERO_ISSUES_COVERAGE_OK",
                                    options=["DONE (recommended)", "CONTINUE"])
    return

# 4. Check escalations
if persistent_escalation_detected:
    Log: LOOP_EXIT_ESCALATION
    present_checkpoint_with_options("ESCALATION_DETECTED",
                                    options=["ESCALATE", "OVERRIDE", "DEFER"])
    return

# 5. Continue loop
Log: LOOP_CONTINUE
present_autoexit_continue("Quality issues remain; restarting loop.")
restart_loop_at_3a()
```

**Auto-Exit Recommendation Display:**

```
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
⚡ LOOP EXIT EVALUATION — Run {N}
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

Run Count: {N} / {max_runs}

📊 COVERAGE STATUS:
  Service:    {X}% / 90%  {✅/⚠️}
  Controller: {Y}% / 100% {✅/⚠️}
  Overall:    {Z}% / 85%  {✅/⚠️}

🔍 QUALITY STATUS:
  CRITICAL issues: {C}
  HIGH issues: {H}
  MEDIUM issues: {M}

⚠️ ESCALATIONS: {count} (if any)

─────────────────────────────────────────────────────

EVALUATION RESULT:

{IF coverage below:}
  → Coverage thresholds not met
  → Auto-restarting loop at Unit Test Agent

{IF coverage OK + zero issues:}
  → ✅ All gates passed!
  → Recommendation: EXIT LOOP
  → RESPOND WITH: DONE (recommended) / CONTINUE

{IF coverage OK + issues remain:}
  → Quality issues still present
  → Auto-restarting loop at Unit Test Agent

{IF run count = max:}
  → Maximum iterations reached
  → RESPOND WITH: DONE / CONTINUE / ABORT

{IF escalation:}
  → Persistent issues detected (≥3 runs)
  → RESPOND WITH: ESCALATE / OVERRIDE / DEFER

─────────────────────────────────────────────────────
```

---

### **Change 1.6: Update Agent Invocation Reference Table (Lines ~780)**

**FIND THIS:**
```markdown
| Stage | Agent | Command sequence |
|---|---|---|
| 1 | Architecture Design Agent | `/ticket-architect` → `/approve` |
| 2 | Backend Implementation Agent | `/audit` → `/generate` → `/approve` |
| 3a | Unit Test Agent | Invoked directly (no command needed — auto-detects mode) |
| 3b | Code Review Agent | Invoked directly (scoped to files-changed.md automatically) |
| 3c | Bugfix Agent | `/analyze` → (human reviews plan) → `/approve` |
| 0.5 | git-branch-manager (skill) | `setup_branch` |
| post-each-stage | gitignore-curator (skill) → git-branch-manager (skill) | `classify_and_propose` → `commit_stage` |
| on demand | git-branch-manager (skill) | `push_branch` / `report_status` |
```

**REPLACE WITH:**
```markdown
| Stage | Agent | Command sequence |
|---|---|---|
| 1 | Architecture Design Agent | `/ticket-architect` → `/approve` |
| 2 | Backend Implementation Agent | `/audit` → `/generate` → `/approve` |
| 3a | Unit Test Agent | Invoked directly + coverage threshold check (retry up to 3x) |
| 3b | Bugfix Agent | `/analyze` → (human reviews plan) → `/approve` |
| 3c | Code Review Agent | Invoked directly (scoped to files-changed.md automatically) |
| 3d | Code Refactor Agent | Invoked directly (silent; outputs REFACTOR_REPORT.md only) |
| 3e | Bugfix Agent (conditional) | `/analyze` → (brief plan) → `/approve` (only if refactor build fails) |
| 3f | Coverage Verification | Check post-refactor coverage, present checkpoint |
| Loop | Exit Evaluation | Check: coverage + quality + escalations + run count |
| 0.5 | git-branch-manager (skill) | `setup_branch` |
| post-each-stage | gitignore-curator (skill) → git-branch-manager (skill) | `classify_and_propose` → `commit_stage` |
| on demand | git-branch-manager (skill) | `push_branch` / `report_status` |
```

---

### **Change 1.7: Update Master Log Events (Lines ~792)**

**ADD these lines to the Master Log Events block:**

```
COVERAGE_BELOW_THRESHOLD        → Coverage metrics below target thresholds
COVERAGE_RETRY_{N}              → Unit test retry attempt N/3
COVERAGE_THRESHOLDS_MET         → All coverage thresholds achieved
COVERAGE_THRESHOLD_OVERRIDE     → Human overrode coverage check
COVERAGE_THRESHOLD_DEGRADED     → Refactor reduced coverage
COVERAGE_VERIFICATION_PASSED    → Post-refactor coverage check passed
COVERAGE_VERIFICATION_DEGRADED  → Post-refactor coverage check failed
CHECKPOINT_H_REACHED            → Unit Test coverage checkpoint
CHECKPOINT_I_REACHED            → Post-Refactor coverage checkpoint
REFACTOR_BUILD_PASSED           → Code Refactor Agent build succeeded
REFACTOR_BUILD_FAILED           → Code Refactor Agent build failed
BUGFIX_REFACTOR_TRIGGERED       → Bugfix Agent launched for refactor recovery
BUGFIX_REFACTOR_BUILD_RECOVERED → Bugfix recovery restored build
LOOP_RESTART_COVERAGE           → Restarting loop due to coverage
LOOP_RESTART_QUALITY            → Restarting loop due to quality
LOOP_CONTINUE                   → Loop incrementing to next iteration
LOOP_EXIT_COVERAGE_BELOW        → Exit condition: coverage below threshold
LOOP_EXIT_MAX_RUNS              → Exit condition: max iterations reached
LOOP_EXIT_ZERO_ISSUES           → Exit condition: no critical/high issues
LOOP_EXIT_ESCALATION            → Exit condition: escalation detected
```

---

## Section 2: Changes to `bugfix-agent.agent.md`

### **Change 2.1: Add Inherited Behaviors Section (after line ~50)**

The bugfix-agent.agent.md already has this, but verify it exists:

```markdown
## Inherited Behaviours

- **`context-reader.behaviour.md`** — Read ticket context, implementation files, and TEST_REPORT.md before fixing
- **`config-permission.behaviour.md`** — Gate all config file changes through orchestrator approval
- **`log-writer.behaviour.md`** — Log bugfix plan and execution to master-agent-log
```

No changes needed if already present.

---

## Section 3: NEW FILE — `.github/COVERAGE_THRESHOLDS.md`

**Create NEW file:**

```markdown
# Coverage Thresholds Configuration

**Last Updated:** April 28, 2026  
**Applies To:** All tickets in Stage 3 (Review Loop)

---

## Coverage Requirements

The Unit Test Agent (Step 3a) will enforce these thresholds before allowing pipeline progression.

### Service Layer (Business Logic)

| Target | Threshold | Severity |
|---|---|---|
| Line Coverage | ≥90% | MANDATORY |
| Branch Coverage | ≥80% | RECOMMENDED |

**Scope:** `src/main/java/com/project/service/**`  
**Rationale:** Core logic must be well-tested

---

### Controller Layer (REST APIs)

| Target | Threshold | Severity |
|---|---|---|
| Line Coverage | ≥100% | MANDATORY |
| Branch Coverage | ≥95% | RECOMMENDED |

**Scope:** `src/main/java/com/project/controller/**`  
**Rationale:** Public APIs must have complete coverage

---

### Repository Layer (Data Access)

| Target | Threshold | Severity |
|---|---|---|
| Line Coverage | ≥85% | RECOMMENDED |
| Branch Coverage | ≥75% | OPTIONAL |

**Scope:** `src/main/java/com/project/repository/**`  
**Rationale:** JPA queries need validation

---

### Overall Project

| Target | Threshold | Severity |
|---|---|---|
| Overall Line Coverage | ≥85% | MANDATORY |
| Overall Branch Coverage | ≥75% | RECOMMENDED |

---

## Retry Logic

**If coverage below thresholds:**
1. Unit Test Agent retries automatically (up to 3 times)
2. Human sees coverage gap at Checkpoint H-N
3. Human can:
   - **APPROVE** → Override threshold and proceed
   - **REVISE** → Provide specific instructions for retry
   - **/skip-coverage** → Acknowledge and skip check
   - **REJECT** → Retry from scratch (reset attempt counter)

**After 3 attempts:**
- If still below: Continue with ⚠️ warning
- Log: `COVERAGE_BELOW_THRESHOLD_MAX_ATTEMPTS`
- Human must explicitly acknowledge via APPROVE or /skip-coverage

---

## Post-Refactor Verification

**Step 3f** checks coverage again after Code Refactor Agent completes.

If coverage is:
- ✅ **Still meeting thresholds**: Proceed to exit evaluation
- ⚠️ **Degraded but acceptable**: Human can proceed (APPROVE)
- ❌ **Below thresholds**: Human can RESTART loop at Unit Test

---

## Exceptions & Overrides

**When human can override:**
- `/skip-coverage` command at Checkpoint H-N
- APPROVE with ⚠️ flag (acknowledges risk)
- RESTART to re-run tests (resets attempt counter)

**What triggers override requirement:**
- Coverage degradation > 5% after refactor
- Coverage remained below threshold after 3 attempts
- Test failures preventing coverage measurement

---

## Tools & Extraction

**Coverage data source:** `target/site/jacoco/jacoco.xml`  
**Tool:** JaCoCo Maven plugin (configured in pom.xml)  
**Report:** TEST_REPORT.md "Coverage Report" section

---
```

---

## Section 4: NEW FILE — `.github/EXIT_CONDITIONS.md`

**Create NEW file:**

```markdown
# Loop Exit Conditions & Flowchart

**Last Updated:** April 28, 2026  
**Pipeline Stage:** Stage 3 (Review Loop)

---

## Exit Condition Evaluation Order

Conditions are evaluated in **STRICT PRECEDENCE ORDER** after every successful build.

```python
# Evaluation sequence:

if NOT coverage_meets_all_thresholds():
    # CONDITION 1: Coverage below threshold
    action = "AUTO_RESTART_LOOP"
    goto Step_3a_Unit_Test()

elif run_count >= max_runs:
    # CONDITION 2: Max runs reached
    action = "PRESENT_CHECKPOINT"
    human_options = ["DONE", "CONTINUE", "ABORT"]

elif critical_or_high_count == 0:
    # CONDITION 3: Zero critical/high issues
    action = "PRESENT_CHECKPOINT"
    human_options = ["DONE (recommended)", "CONTINUE"]

elif persistent_escalation_detected():
    # CONDITION 4: Persistent issues (≥3 consecutive runs)
    action = "PRESENT_CHECKPOINT"
    human_options = ["ESCALATE", "OVERRIDE", "DEFER"]

else:
    # DEFAULT: Continue loop
    action = "AUTO_RESTART_LOOP"
    goto Step_3a_Unit_Test()
```

---

## Exit Condition Definitions

### CONDITION 1: Coverage Below Threshold

**Trigger:** ANY coverage metric below target  
**Evaluation Point:** After Step 3f (Coverage Verification)  
**Human Sees:** Nothing (automatic restart)  
**Log Event:** `LOOP_RESTART_COVERAGE`  
**Action:** Auto-restart at Step 3a  
**Rationale:** Coverage is hardcoded requirement; no override at exit point

---

### CONDITION 2: Max Runs Reached

**Trigger:** `run_count >= 10` (default, configurable)  
**Evaluation Point:** After Step 3f  
**Human Sees:** Checkpoint with options  
**Log Events:** `LOOP_EXIT_MAX_RUNS`  
**Action:** Present choice to human  
**Options:**
- `DONE` → Exit loop (accept current state)
- `CONTINUE` → Override and run iteration {N+1}
- `ABORT` → Stop pipeline entirely

---

### CONDITION 3: Zero Critical/High Issues

**Trigger:** Code Review report shows CRITICAL=0, HIGH=0  
**Evaluation Point:** After Step 3f  
**Human Sees:** Checkpoint (recommended DONE)  
**Log Events:** `LOOP_EXIT_ZERO_ISSUES`  
**Action:** Recommend exit  
**Options:**
- `DONE` (✓ default recommended) → Exit loop
- `CONTINUE` → Run another iteration (optional)

**Precedence:** Only evaluated if coverage meets thresholds

---

### CONDITION 4: Persistent Escalation

**Trigger:** Issue appears in ≥3 consecutive runs  
**Evaluation Point:** After Step 3f (via loop_tracker)  
**Human Sees:** Checkpoint with escalation notice  
**Log Events:** `LOOP_EXIT_ESCALATION`  
**Action:** Force human decision  
**Options:**
- `ESCALATE` → Mark as known issue, exit loop
- `OVERRIDE` → Accept and continue forcing fixes
- `DEFER` → Postpone to later ticket

---

## Decision Tree

```
┌─ START: Exit Evaluation
│
├─ Coverage OK?
│  ├─ NO → AUTO_RESTART_LOOP (3a)
│  │
│  └─ YES
│     │
│     ├─ Run count ≥ 10?
│     │  ├─ YES → PRESENT { DONE / CONTINUE / ABORT }
│     │  │
│     │  └─ NO
│     │     │
│     │     ├─ CRITICAL=0 AND HIGH=0?
│     │     │  ├─ YES → PRESENT { DONE (✓) / CONTINUE }
│     │     │  │
│     │     │  └─ NO
│     │     │     │
│     │     │     ├─ Persistent Escalation?
│     │     │     │  ├─ YES → PRESENT { ESCALATE / OVERRIDE / DEFER }
│     │     │     │  │
│     │     │     │  └─ NO
│     │     │     │     │
│     │     │     │     └─ AUTO_RESTART_LOOP (3a)
└─────────────────────────────────
```

---

## Checkpoint Presentations

### CHECKPOINT: Max Runs Reached

```
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
⚠️ MAX ITERATIONS REACHED
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

Run Count: {N} / {max} iterations

Ticket: {TICKET_ID}

Current State:
  Coverage: {status}
  Issues: CRITICAL={C}, HIGH={H}
  Escalations: {count}

You have reached the maximum number of loop iterations.
Continue manually if additional fixes are needed.

RESPOND WITH:
  DONE       → Accept current state, exit loop
  CONTINUE   → Override limit, run iteration {N+1}
  ABORT      → Stop pipeline entirely

────────────────────────────────────────────────────
```

---

### CHECKPOINT: Zero Issues

```
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
✅ ALL QUALITY GATES PASSED
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

Coverage: ✅ All thresholds met
Quality:  ✅ Zero CRITICAL/HIGH issues
Tests:    ✅ All passing
Build:    ✅ Success

Run Count: {N}

Ready for pipeline completion!

RESPOND WITH:
  DONE       → Accept and complete pipeline (recommended)
  CONTINUE   → Run additional iteration (optional)

────────────────────────────────────────────────────
```

---

### CHECKPOINT: Persistent Escalation

```
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
⚠️ PERSISTENT ISSUE ESCALATION
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

Consecutive Runs: {N}+ (threshold: 3)

Persisting Issue(s):
  {issue_1}: {N} consecutive runs
  {issue_2}: {N} consecutive runs

This issue has appeared in {N}+ consecutive loop iterations
and may require manual developer intervention or architectural review.

RESPOND WITH:
  ESCALATE   → Mark as known issue, exit loop (for manual review)
  OVERRIDE   → Continue forcing automatic fixes (extend to {N+1})
  DEFER      → Postpone issue, exit loop (for later ticket)

────────────────────────────────────────────────────
```

---

## Configuration

| Parameter | Default | Configurable | Location |
|---|---|---|---|
| `max_runs` | 10 | YES | agent-config.yml |
| `service_coverage_target` | 90% | YES | COVERAGE_THRESHOLDS.md |
| `controller_coverage_target` | 100% | YES | COVERAGE_THRESHOLDS.md |
| `overall_coverage_target` | 85% | YES | COVERAGE_THRESHOLDS.md |
| `persistent_issue_threshold` | 3 | YES | agent-config.yml |

---
```

---

## Summary of All Changes

### Orchestrator Agent Changes
1. ✅ Update Pipeline Diagram (add 3a coverage logic, 3d/3e/3f)
2. ✅ Replace Step 3a (add coverage threshold logic + retry)
3. ✅ Replace Step 3b (checkpoint F-N only)
4. ✅ Keep Step 3c (stays D-N)
5. ✅ Add Step 3d (Code Refactor Agent)
6. ✅ Add Step 3e (Bugfix recovery)
7. ✅ Add Step 3f (Coverage verification)
8. ✅ Add Loop Exit Evaluation (updated)
9. ✅ Update Agent Invocation Reference Table
10. ✅ Add Master Log Events

### Bugfix Agent Changes
- ✅ No changes needed (already correct)

### New Files
- ✅ `.github/COVERAGE_THRESHOLDS.md` (NEW)
- ✅ `.github/EXIT_CONDITIONS.md` (NEW)

---

**All needed changes are documented above. Use these as your copy-paste guide.**



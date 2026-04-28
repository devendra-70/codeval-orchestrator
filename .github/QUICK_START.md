# QUICK START: What Files I Created & What YOU Need to Do

**Date:** April 28, 2026  
**Status:** Ready for Implementation  
**Action Required:** Manual edits to existing files (see below)

---

## Files Created ✅ (DO NOT MODIFY - These are READ ONLY)

| File | Purpose | Location |
|---|---|---|
| `IMPLEMENTATION_GUIDE.md` | Complete line-by-line changes needed | `.github/IMPLEMENTATION_GUIDE.md` |
| `COVERAGE_THRESHOLDS.md` | Coverage target configuration | `.github/COVERAGE_THRESHOLDS.md` |
| `EXIT_CONDITIONS.md` | Exit condition decision tree | `.github/EXIT_CONDITIONS.md` |
| `bugfix-agent.agent.md` | (Already exists - no changes) | `.github/agents/bugfix-agent.agent.md` |

---

## Files YOU Must Manually Edit ❌→✅

### **File 1: orchestrator-ticket-agent.agent.md**

**Location:** `.github/agents/orchestrator-ticket-agent.agent.md`

**Changes Required:** 7 major changes (see IMPLEMENTATION_GUIDE.md for exact details)

1. **Change 1.1** — Update Pipeline Diagram (lines ~104-132)
   - Add coverage retry logic to Unit Test step
   - Add Code Refactor (3d), Bugfix Recovery (3e), Coverage Verification (3f)

2. **Change 1.2** — Replace Step 3a — Unit Test Agent Section (lines ~375-471)
   - Add `attempt_count` tracking
   - Add coverage threshold extraction
   - Add retry logic (up to 3 times)
   - Replace checkpoint from C-N to H-N with new coverage display

3. **Change 1.3** — Replace Step 3b — Bugfix Agent Section (lines ~545-632)
   - Change checkpoint from E-N to F-N
   - Add STAGE_LABEL = bugfix-run-{N} for commits

4. **Change 1.4** — Step 3c — Code Review Agent Section
   - Rename checkpoint to maintain D-N (already correct, verify only)

5. **Change 1.5** — Add NEW sections 3d, 3e, 3f (INSERT AFTER 3c)
   - Step 3d: Code Refactor Agent (silent unless build fails)
   - Step 3e: Bugfix Agent (conditional, refactor recovery)
   - Step 3f: Coverage Verification (post-refactor, new checkpoint I-N)

6. **Change 1.6** — Update Agent Invocation Reference Table (lines ~780)
   - Update all row entries to show new order and coverage logic

7. **Change 1.7** — Add Master Log Events (lines ~792)
   - Add ~27 new log event types for coverage, refactor, exit conditions

**How to Make These Changes:**
1. Open `.github/agents/orchestrator-ticket-agent.agent.md`
2. Follow the exact line ranges and text replacements in `IMPLEMENTATION_GUIDE.md` Section 1
3. Copy-paste the old text first, then the new text
4. Verify numbering and checkpoints are correct

---

## Reference Documents (For Understanding) 📖

### IMPLEMENTATION_GUIDE.md
- Contains ALL exact changes needed
- Organized by: File → Section → Change #
- Includes old text → new text replacements
- Best for: Systematic editing

### COVERAGE_THRESHOLDS.md
- Defines coverage targets for each layer
- Explains retry logic when below threshold
- When to override vs. when to enforce
- Best for: Understanding coverage requirements

### EXIT_CONDITIONS.md
- Complete decision tree for loop exit
- Definitions of all 5 conditions
- Sample checkpoint presentations
- Configuration parameters
- Best for: Understanding when loop stops

---

## Implementation Checklist

### Step 1: Read Documentation
- [ ] Read `IMPLEMENTATION_GUIDE.md` Section 1 (changes needed)
- [ ] Read `COVERAGE_THRESHOLDS.md` (understand targets)
- [ ] Read `EXIT_CONDITIONS.md` (understand exit logic)

### Step 2: Backup Original File
- [ ] Copy `orchestrator-ticket-agent.agent.md` → `orchestrator-ticket-agent.agent.md.backup`
- [ ] Keep backup for comparison

### Step 3: Make Changes to orchestrator-ticket-agent.agent.md
- [ ] Change 1.1: Update Pipeline Diagram
- [ ] Change 1.2: Replace Step 3a (Unit Test with coverage)
- [ ] Change 1.3: Update Step 3b (checkpoint F-N)
- [ ] Change 1.4: Verify Step 3c (checkpoint D-N)
- [ ] Change 1.5: Add new sections 3d, 3e, 3f
- [ ] Change 1.6: Update Agent Invocation Table
- [ ] Change 1.7: Add Master Log Events

### Step 4: Validate Changes
- [ ] Run linter/spell-check on updated file
- [ ] Verify all checkpoint names are correct:
  - H-N (Unit Test with coverage)
  - F-N (Bugfix)
  - D-N (Code Review)
  - I-N (Coverage Verification post-refactor)
- [ ] Count checkpoints: Should be 4 per loop iteration (H, F, D, I)
- [ ] Verify agent names in sections match Agent Invocation Table

### Step 5: Test Implementation
- [ ] Run orchestrator-ticket-agent against test ticket
- [ ] Verify coverage thresholds are checked at Checkpoint H-N
- [ ] Verify retry logic works (< 90% = retry up to 3x)
- [ ] Verify all exit conditions work
- [ ] Verify loop restarts correctly after refactor if coverage degrades

---

## New Pipeline Flow (After Changes)

```
Stage 3 Loop Run N:
  ┌─────────────────────────────────────────────┐
  │ 3a: Unit Test Agent                         │
  │     • Check coverage thresholds              │
  │     • If below: Retry (max 3x)              │
  │     • Checkpoint H-N (human approval)       │
  └─ APPROVE ─────────┐                         │
                      │                         │
  ┌─────────────────────────────────────────┐ │
  │ 3b: Bugfix Agent                         │ │
  │     • Fix test failures                   │ │
  │     • Checkpoint F-N                      │ │
  └─ APPROVE ─────────┐                      │ │
                      │                      │ │
  ┌─────────────────────────────────────────┐ │ │
  │ 3c: Code Review Agent                   │ │ │
  │     • Quality review                     │ │ │
  │     • Checkpoint D-N                     │ │ │
  └─ APPROVE ─────────┐                     │ │ │
                      │                     │ │ │
  ┌─────────────────────────────────────────┐ │ │ │
  │ 3d: Code Refactor Agent                 │ │ │ │
  │     • Apply CRITICAL+HIGH fixes         │ │ │ │
  │     • If build fails: 3e                 │ │ │ │
  │     • If pass: proceed to 3f             │ │ │ │
  └─────────────────────────────────────────┘ │ │ │
                      │                       │ │ │
  ┌─────────────────────────────────────────┐ │ │ │
  │ 3f: Coverage Verification (Post-Refactor)│ │ │ │
  │     • Check if coverage degraded         │ │ │ │
  │     • If degraded: RESTART option        │ │ │ │
  │     • Checkpoint I-N                     │ │ │ │
  └─ APPROVE ─────────┐                     │ │ │ │
                      │                     │ │ │ │
  ┌─────────────────────────────────────────┐ │ │ │ │
  │ EXIT EVALUATION                          │ │ │ │ │
  │  1. Coverage OK? (else → RESTART 3a)    │ │ │ │ │
  │  2. Max runs? (else → wait human choice) │ │ │ │ │
  │  3. Zero issues? (else → RESTART 3a)    │ │ │ │ │
  │  4. Escalation? (else → RESTART 3a)     │ │ │ │ │
  │  5. Otherwise → RESTART 3a               │ │ │ │ │
  └─────────────────────────────────────────┘ │ │ │ │
                      │                       │ │ │ │
                      └───[RESTART N+1]───────┘ │ │ │
                                                │ │ │
                      ┌────────────────────────┘ │ │
                      │                         │ │
   (if build failed)  │      [3e runs only if build failed from 3d]
                      │
  ┌─────────────────────────────────────────┐
  │ 3e: Bugfix Agent (Refactor Recovery)    │
  │     • Fix refactor-induced bugs          │
  │     • If still fails: HALT               │
  │     • If passes: continue to 3f          │
  └─────────────────────────────────────────┘
```

---

## Checkpoints Summary

After implementation, your pipeline will have 4 checkpoints per loop iteration:

| Checkpoint | Name | Where | Decision |
|---|---|---|---|
| **H-N** | Unit Test (with coverage) | After 3a | APPROVE / REVISE / REJECT / /skip-coverage |
| **F-N** | Bugfix | After 3b | APPROVE / REVISE / SKIP |
| **D-N** | Code Review | After 3c | APPROVE / REVISE |
| **I-N** | Coverage Verification (post-refactor) | After 3f | APPROVE / REVISE / RESTART |

---

## Quick Reference: Coverage Thresholds

| Layer | Target | Mandatory |
|---|---|---|
| Services | ≥90% | YES |
| Controllers | ≥100% | YES |
| Overall | ≥85% | YES |

If below threshold: Auto-retry Unit Test (max 3 times)

---

## Questions?

**For detailed implementation steps:** See `IMPLEMENTATION_GUIDE.md`  
**For coverage rules:** See `COVERAGE_THRESHOLDS.md`  
**For exit logic:** See `EXIT_CONDITIONS.md`

---



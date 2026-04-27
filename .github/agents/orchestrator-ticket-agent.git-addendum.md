# Orchestrator Ticket Agent — Git Addendum

> **Purpose:** This file is a *paste-in guide*. The existing
> `orchestrator-ticket-agent.agent.md` is not modified by anyone except a human
> who deliberately copies the snippets below into the right anchors. Each
> snippet is self-contained and additive — nothing here removes or rewrites
> existing pipeline behaviour.
>
> **Companion files (already created):**
> - `.github/skills/git-branch-manager.skill.md`
> - `.github/skills/gitignore-curator.skill.md`
> - `.github/behaviours/git-operations.behaviour.md`
> - `.github/document-templates/git-branch-naming.md`
> - `.gitignore`

---

## How to apply this addendum

For each snippet, find the **anchor** (an existing heading or list item in
`orchestrator-ticket-agent.agent.md`) and insert the snippet **immediately
after** that anchor. Do not delete or reorder existing content.

The snippets are listed in the order they appear in the orchestrator file.

---

## Snippet 1 — Add Behaviour

**Anchor:** the existing list under `## Inherited Behaviours (ALL MANDATORY)`
**Insert after:** the `config-permission.behaviour.md` line.

```markdown
- **`git-operations.behaviour.md`** — Govern all Git/GitLab side-effects; only the orchestrator + git-branch-manager skill may run Git
```

---

## Snippet 2 — Add Skills

**Anchor:** the existing list under `## Skills Used`
**Insert after:** the `loop-tracker.skill.md` line.

```markdown
- **`git-branch-manager.skill.md`** — Create the ticket branch, commit per stage on human prompt, push on explicit `PUSH`
- **`gitignore-curator.skill.md`** — Auto-detect non-deliverable files at commit time and propose `.gitignore` additions (gated by config-permission)
```

---

## Snippet 3 — Update Pipeline Stages Overview

**Anchor:** the ASCII flow diagram under `## Pipeline Stages Overview`
**Insert after:** the `Stage 0` block, before the `Stage 1` block.

```
         │
         ▼
Stage 0.5: Git Branch Setup
  → git-branch-manager.setup_branch
  → Asks human for TEAM_PREFIX (AIS / EXE / CVS / FE) once
  → Creates: tickets/{id}/git-context.md + checks out feature/bugfix/release/hotfix branch
  → On CANCEL: git_enabled=false, pipeline continues without any Git ops
         │
         ▼
```

Also append, **after** every stage's APPROVE arrow (Stages 1, 2, 3a, 3b, 3c)
in the same diagram, the following inline note (one-line each):

```
    [COMMIT? — human: COMMIT / EDIT: <msg> / SKIP / GIT STATUS]
```

---

## Snippet 4 — New Stage 0.5 Section

**Anchor:** the heading `### Stage 0 — Build Ticket Context` and its body.
**Insert after:** the line `7. Proceed to Stage 1 automatically (no checkpoint here)`
(rewriting it slightly so we proceed to **Stage 0.5** instead — see Snippet 4a).

### Snippet 4a — minor edit to Stage 0 closing line

Replace:
```
7. Proceed to Stage 1 automatically (no checkpoint here)
```
with:
```
7. Proceed to Stage 0.5 (Git Branch Setup) automatically
```

### Snippet 4b — new Stage 0.5 block

Insert this **whole block** between the existing `### Stage 0` section
and the existing `### Stage 1` section:

```markdown
---

### Stage 0.5 — Git Branch Setup

**Trigger:** Stage 0 complete; `tickets/{TICKET_ID}/ticket-context.md` exists.

1. Verify `git-operations.behaviour.md` is loaded.
2. Invoke `git-branch-manager.setup_branch` with `TICKET_ID`.
3. The skill prompts the human for team prefix (and optional branch type override):
   - Valid responses: `PREFIX: AIS|EXE|CVS|FE` (with optional `TYPE: feature|bugfix|release|hotfix`) or `CANCEL`.
4. On `CANCEL`: skill writes `git_enabled: false` to `git-context.md`. The
   pipeline proceeds, but every later commit/push checkpoint is skipped.
5. On success: skill writes `tickets/{TICKET_ID}/git-context.md`, checks out
   the branch locally, and logs `BRANCH_CREATED` (or `BRANCH_REUSED`).
6. Display result to human:
   ```
   ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
   🌿 BRANCH READY — {TICKET_ID}
   Branch : {BRANCH_NAME}
   Base   : {BASE_BRANCH}
   Status : checked out (local only — push later via PUSH)
   ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
   ```
7. Proceed to Stage 1 automatically (no checkpoint here).
```

---

## Snippet 5 — Post-Stage Commit Checkpoints

After each stage's existing APPROVE flow, insert a commit sub-checkpoint.
The format below is reused for every stage; only `STAGE_LABEL` and the
default `<type>` change.

### Snippet 5a — Reusable Commit Sub-Checkpoint Block

```markdown
8. **Commit sub-checkpoint** (only if `git_enabled: true` in `git-context.md`):

   Invoke `gitignore-curator.classify_and_propose` against the stage's
   candidate file list. If it surfaces flagged files, run that sub-checkpoint
   first and apply the human's IGNORE / STAGE / SKIP decisions before
   continuing.

   Then invoke `git-branch-manager.commit_stage` with `STAGE_LABEL = {label}`.
   The skill presents:
   ```
   ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
   💾 COMMIT? — {TICKET_ID} — Stage: {STAGE_LABEL}
   ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
   Branch : {BRANCH_NAME}
   Files  : {scoped list from stage artifact}
   Proposed message:
     {default-type}: {short} -- {TICKET_ID}
   ─────────────────────────────────────────────────────
   RESPOND WITH:
     COMMIT             → Commit with proposed message
     EDIT: {full msg}   → Commit with your message instead
     SKIP               → Skip commit for this stage
     GIT STATUS         → Show working tree status first
   ─────────────────────────────────────────────────────
   ```

   On `COMMIT` / `EDIT:` → the skill stages the scoped files and commits.
   On `SKIP` → no commit; logged as `COMMIT_SKIPPED`.
   In all cases, advance to the next pipeline step.
```

### Snippet 5b — Where to insert it

Insert the block above as a new numbered step in each of the following
existing sections, with `STAGE_LABEL` set as shown:

| Existing section | `STAGE_LABEL` | Default `<type>` |
|---|---|---|
| `### Stage 1 — Architecture` (after the APPROVE step) | `architecture` | `docs` |
| `### Stage 2 — Backend Implementation` (after APPROVE) | `implementation` | `feat` (Story/Task) or `fix` (Bug) |
| `#### Step 3a — Unit Test Agent` (after APPROVE) | `tests-run-{N}` | `test` |
| `#### Step 3b — Code Review Agent` (after APPROVE) | `review-run-{N}` | `docs` |
| `#### Step 3c — Bugfix Agent` (after Checkpoint E APPROVE) | `bugfix-run-{N}` | `fix` |

> Note: under the new `.gitignore`, the architecture and review stages will
> usually have nothing to commit (their artifacts are ignored). The skill
> will auto-`SKIP` those, which is the intended behaviour — they remain in
> the spec so the flow is symmetric and the human can override via `EDIT:`.

---

## Snippet 6 — `PUSH` Command

**Anchor:** the section `## Config Permission Interception` (any global
"command" location works; the existing file does not have one yet).
**Insert before:** `## Config Permission Interception`.

```markdown
---

## Push Command (Human-Initiated)

The human may type `PUSH` at any point after at least one commit exists on the
ticket branch. The orchestrator forwards to `git-branch-manager.push_branch`,
which runs the standard pre-flight checks (branch is not protected, no
divergence, no rebase/merge in progress) and pushes the branch with
`git push -u origin <BRANCH_NAME>`.

`PUSH` is never automatic. `--force` is never used. If the remote has
diverged, the orchestrator halts and surfaces the conflict to the human.

Log events: `PUSH_CREATED` on success, `GIT_PREFLIGHT_FAILED` on refusal.
```

---

## Snippet 7 — Stage 4 "NEXT STEPS" Update

**Anchor:** the closing summary block under `### Stage 4 — Pipeline Complete`.
**Replace** the existing `NEXT STEPS:` lines with:

```
NEXT STEPS:
  → Type PUSH to push {BRANCH_NAME} to origin (if not already pushed)
  → Open MR/PR manually for human developer review
  → Address any escalated issues manually
```

(The original lines were `Push implementation to feature branch / Create MR/PR
manually / Address escalated issues`. The new wording bakes in the explicit
`PUSH` command.)

---

## Snippet 8 — Master Log Events

**Anchor:** the code block under `## Master Log Events (Orchestrator Owns)`.
**Append** these lines inside that code block:

```
BRANCH_CREATED        → Stage 0.5 created a new branch
BRANCH_REUSED         → Stage 0.5 reused an existing local branch
BRANCH_REUSED_REMOTE  → Stage 0.5 checked out a tracking branch from origin
COMMIT_CREATED        → A commit_stage produced a new commit
COMMIT_SKIPPED        → Human responded SKIP at a commit checkpoint
PUSH_CREATED          → push_branch pushed to origin
GIT_PREFLIGHT_FAILED  → A Git pre-flight check failed
GIT_BOUNDARY_VIOLATION→ A non-authorised agent attempted Git
GIT_DISABLED          → git_enabled=false recorded for the ticket
GITIGNORE_SUGGESTED   → gitignore-curator surfaced flagged files
GITIGNORE_APPLIED     → Patterns appended to .gitignore (after ALLOW)
GITIGNORE_DENIED      → Human DENYed proposed .gitignore additions
GITIGNORE_AUTO_APPLIED→ Pattern matched a learned IGNORE — auto-withheld
GITIGNORE_LEARNED     → Pattern remembered via GITIGNORE LEARN
GITIGNORE_FORGOTTEN   → Pattern removed via GITIGNORE FORGET
```

---

## Snippet 9 — Non-Goals Update

**Anchor:** the `## Non-Goals` list at the very bottom of the orchestrator file.
**Append** these bullets:

```markdown
- Do NOT execute Git or GitLab commands directly — always go through `git-branch-manager.skill.md`
- Do NOT auto-commit, auto-push, or modify `.gitignore` without explicit human input
- Do NOT push to `main`, `master`, `develop`, or `release/*`
- Do NOT use `--force` / `--force-with-lease` under any circumstance
```

---

## Snippet 10 — Agent Invocation Reference Update

**Anchor:** the table under `## Agent Invocation Reference`.
**Append** these rows:

```markdown
| 0.5 | git-branch-manager (skill) | `setup_branch` |
| post-each-stage | gitignore-curator (skill) → git-branch-manager (skill) | `classify_and_propose` → `commit_stage` |
| on demand | git-branch-manager (skill) | `push_branch` / `report_status` |
```

---

## Optional follow-ups (not part of this addendum, listed for awareness)

1. Add a `git-operations-log.md` template under `.github/document-templates/`
   so the orchestrator has a dedicated git audit trail (mirrors the master log).
2. Migrate scratch state to a single `.codeval/` umbrella folder
   (we discussed this; deferred per your instruction).
3. Add an MR/PR opener skill on top of `git-branch-manager`.
```

---

## Verification checklist (run after pasting)

- [ ] `Inherited Behaviours` lists `git-operations.behaviour.md`.
- [ ] `Skills Used` lists `git-branch-manager.skill.md` and `gitignore-curator.skill.md`.
- [ ] Pipeline overview shows Stage 0.5 between Stage 0 and Stage 1.
- [ ] Each of Stages 1, 2, 3a, 3b, 3c has a commit sub-checkpoint after APPROVE.
- [ ] A `PUSH` command section exists above `Config Permission Interception`.
- [ ] Stage 4 `NEXT STEPS` mentions typing `PUSH`.
- [ ] Master log events include the new Git + Gitignore events.
- [ ] Non-Goals include the four new Git restrictions.
- [ ] Agent Invocation Reference includes the three new rows.


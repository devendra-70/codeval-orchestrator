# Behaviour: File Lock

## Purpose
Prevent concurrent file modifications by multiple agents operating on the same ticket.
Lock files are plain text files stored in `tickets/{TICKET_ID}/.locks/`.

---

## Lock Areas

Each agent owns one or more lock areas. An agent MUST only acquire locks for its own areas.

| Lock File                  | Owner Agent               | Protected Paths                              |
|----------------------------|---------------------------|----------------------------------------------|
| `architecture.lock`        | Architecture Agent        | `tickets/{id}/architecture/**`               |
| `implementation.lock`      | Backend Implementation Agent | `tickets/{id}/implementation/**` + `src/main/java/**` |
| `tests.lock`               | Unit Test Agent           | `tickets/{id}/test-reports/**` + `src/test/java/**` |
| `review.lock`              | Code Review Agent         | `tickets/{id}/review-reports/**` (READ-ONLY agent — lock is advisory) |
| `bugfix.lock`              | Bugfix Agent              | `tickets/{id}/bugfix-reports/**` + `src/main/java/**` (patching) |
| `orchestrator.lock`        | Orchestrator Agent        | `tickets/{id}/*.md` + `.github/logs/**`      |

---

## Acquire Lock Protocol

Before writing ANY file, an agent MUST:

1. Check if `tickets/{TICKET_ID}/.locks/{area}.lock` exists
2. If the lock file EXISTS:
   - Read the lock file to identify the owner agent and timestamp
   - HALT immediately
   - Report to the orchestrator:
     ```
     LOCK CONFLICT: {area}.lock is held by {owner-agent} since {timestamp}.
     Cannot proceed. Waiting for orchestrator to resolve.
     ```
   - Do NOT proceed under any circumstance
3. If the lock file DOES NOT EXIST:
   - Create it immediately with the following content:
     ```
     agent: {AGENT_NAME}
     ticket: {TICKET_ID}
     acquired: {ISO_8601_TIMESTAMP}
     purpose: {brief description of what the agent is about to write}
     ```
   - Proceed with the write operation

---

## Release Lock Protocol

After completing a write operation, the agent MUST:

1. Verify the lock file content matches its own agent name (safety check)
2. Delete the lock file
3. Log the release to `.github/logs/master-agent-log.md`

An agent MUST NEVER delete a lock file it did not create.

---

## Lock Timeout Rule

If a lock file is older than **30 minutes** (compare `acquired` timestamp to current time):
- The lock is considered stale
- The orchestrator (and ONLY the orchestrator) may delete the stale lock
- Individual agents must NOT self-resolve stale locks — they must report and wait

---

## Cross-Area Rule

An agent MUST NOT write to files outside its designated lock areas, even if no lock exists for that area.

| Agent                  | MAY write to                                   | MUST NOT write to                          |
|------------------------|------------------------------------------------|--------------------------------------------|
| Architecture Agent     | `tickets/{id}/architecture/**`                 | `src/**`, `tickets/{id}/implementation/**` |
| Backend Agent          | `tickets/{id}/implementation/**`, `src/main/java/**` | `src/test/java/**`                   |
| Unit Test Agent        | `tickets/{id}/test-reports/**`, `src/test/java/**` | `src/main/java/**`                     |
| Code Review Agent      | `tickets/{id}/review-reports/**`               | ALL source files (read-only agent)         |
| Bugfix Agent           | `tickets/{id}/bugfix-reports/**`, `src/main/java/**` (patch only) | `src/test/java/**`          |
| Orchestrator           | `tickets/{id}/*.md`, `.github/logs/**`, `.locks/**` | All source files                      |

---

## Violation Handling

If an agent detects it is about to write outside its designated area:
- STOP immediately
- Do NOT write the file
- Report the violation to the orchestrator with full context:
  ```
  BOUNDARY VIOLATION ATTEMPT:
  Agent: {AGENT_NAME}
  Attempted path: {file path}
  Reason flagged: Outside designated area
  Action taken: Aborted. Awaiting orchestrator instruction.
  ```
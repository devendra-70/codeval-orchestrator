# QA Backlog Generation Prompt

## How to Use This Prompt

1. Attach this file using `#file: QA-prompt.md`
2. Attach your requirements file using `#file: <your-SRS-file>` (e.g. `#file: SRS.txt`)
3. Send the following command:

```
RUN BACKLOG CATEGORY=QA PROJECT=<project-name>
```

**Example:**
```
#file: QA-prompt.md
#file: SRS.txt

RUN BACKLOG CATEGORY=QA PROJECT=CodEval
```

> The agent will automatically use the attached requirements file as the INPUT source.
> You do NOT need to specify an INPUT path manually.

---

## Active Instructions

You are running as the **orchestrator-agent** defined in:
`agents/orchestrator-agent/orchestrator-agent.md`

Your downstream agents are:
- `agents/requirements-to-backlog-agent/requirements-to-backlog-agent.md`
- `agents/backlog-materializer-agent/backlog-materializer-agent.md`

---

## Input Resolution Rule

The requirements document is provided as an **attached file** in this conversation via `#file:`.

- Do NOT look for a hardcoded file path
- Use the content of the attached requirements file directly as the INPUT source
- The `inputFilePath` field in JSON output must reflect the attached file's name (e.g. `projects/<project-name>/requirements/SRS.txt`)

---

## Runtime Directive

**CATEGORY_FILTER = QA**

Apply the following rules for this run:

- Generate **only QA user stories** (`storyCategory = "QA"`)
- Set `actor = "Admin"` or `actor = "User"` for all stories (never `"System"`)
- Exclude all DEV stories (`storyCategory = "DEV"`)
- Focus on: admin actions, user-visible flows, UI behavior, review steps, verification scenarios, publish/discard actions
- All Acceptance Criteria and Business Rules must be relevant to QA (user/admin-facing) behavior only

---

## Orchestration Steps

Execute the full pipeline in this order:

### Step 1 — Validate Command
Confirm that `CATEGORY` and `PROJECT` are present.
Confirm the requirements file is attached in the conversation.

### Step 2 — Invoke Requirements Agent
Read the requirements content from the attached file.
Apply `CATEGORY_FILTER = QA`.
Generate the Jira-ready JSON bundle containing only QA stories.

Save bundle to:
`projects/<project-name>/artifacts/QA/epics/epics-bundle-qa.json`

### Step 3 — Invoke Backlog Materializer Agent
Read the generated JSON bundle.
Materialize:
- Individual Epic JSON files → `projects/<project-name>/artifacts/QA/epics/<epic-name>/<epic-id>.json`
- Individual User Story JSON files → `projects/<project-name>/artifacts/QA/epics/<epic-name>/user-stories/<story-id>.json`
- Backlog review document → `projects/<project-name>/artifacts/QA/documents/backlog-review-qa.md`

### Step 4 — Return Final Summary
Return a structured JSON summary:

```json
{
  "projectName": "<project-name>",
  "category": "QA",
  "inputFile": "<attached-filename>",
  "bundlePath": "projects/<project-name>/artifacts/QA/epics/epics-bundle-qa.json",
  "epicCount": 0,
  "userStoryCount": 0,
  "readableDocumentPath": "projects/<project-name>/artifacts/QA/documents/backlog-review-qa.md",
  "status": "Success"
}
```

---

## Output Quality Checklist

Before completing, verify:

- [ ] Only QA stories are present (`storyCategory = "QA"` on every story)
- [ ] All actors are `"Admin"` or `"User"` — never `"System"`
- [ ] Every story follows: `"As an Admin/User, I want <goal>, so that <benefit>."`
- [ ] Every story has at least 2 Acceptance Criteria
- [ ] Every story has at least 1 Business Rule
- [ ] No duplicate epics or stories
- [ ] All IDs are globally unique
- [ ] All `status` fields are `"Draft"`
- [ ] JSON is syntactically valid
- [ ] Save paths follow `artifacts/QA/` structure

---

## Constraints

- Do NOT generate DEV stories
- Do NOT generate test cases or BDD scenarios
- Do NOT generate code or scripts
- Do NOT invent requirements not present in the attached document
- JSON bundle is the single source of truth — the readable document is derived from it

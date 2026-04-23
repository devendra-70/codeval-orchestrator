# Epics and User Stories Agent Workflow

This workflow is a short chain of agents that turns a requirements document into Epic and User Story artifacts.

## High-Level Flow

1. A run starts from either `DEV_prompt.md` or `QA-prompt.md`.
2. That prompt activates the `orchestrator-agent`.
3. The orchestrator validates the command, project name, and attached requirements file.
4. The orchestrator sends the requirements into `requirements-to-backlog-agent`.
5. The requirements agent reads the document, identifies major capabilities, creates Epics, and creates User Stories under each Epic.
6. The requirements agent outputs a backlog JSON bundle as the source of truth.
7. The orchestrator then passes that bundle to `backlog-materializer-agent`.
8. The materializer creates the final artifact structure: epic files, user story files, and a readable backlog review document.
9. The flow ends with a short summary showing what was generated.

## How The Agents Behave

### Prompt Layer

- `DEV_prompt.md` runs the flow for DEV stories only.
- `QA-prompt.md` runs the flow for QA stories only.
- The prompt decides the category filter and passes the attached requirements file into the chain.

### Orchestrator Agent

- Acts as the controller of the workflow.
- Does not create backlog content itself.
- Decides which downstream agent to run and in what order.
- Ensures outputs are saved in the correct project artifact folders.

### Requirements-to-Backlog Agent

- Reads the requirements document.
- Groups related functionality into Epics.
- Creates User Stories for each Epic.
- Adds Acceptance Criteria and Business Rules.
- Produces the main JSON backlog bundle.

### Backlog Materializer Agent

- Uses the JSON bundle as the only source of truth.
- Does not reinterpret or rewrite requirements.
- Splits the bundle into individual Epic and User Story files.
- Generates the readable backlog review markdown document.

## DEV vs QA Routing

- `DEV` focuses on system and backend behavior.
- `QA` focuses on admin, user-facing, and verification behavior.
- The chain stays the same in both cases.
- The main difference is the category filter applied before User Stories are finalized.

## Simple Chain View

`Prompt -> Orchestrator Agent -> Requirements-to-Backlog Agent -> Backlog Materializer Agent -> Epic and User Story Artifacts`

## Final Output

At the end of the workflow, the project gets:

- an epic bundle JSON
- individual Epic JSON files
- individual User Story JSON files
- a readable backlog review document

So in short: one agent controls the flow, one agent creates the backlog structure, and one agent materializes that structure into project files.
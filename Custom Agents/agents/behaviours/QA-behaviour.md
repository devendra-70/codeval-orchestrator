# QA Behavior Prompt

## Objective

Bias the backlog generation toward a **QA / testing perspective**.

The agent must interpret the requirement document with a focus on **verification, validation, failure handling, and user-visible behavior**, instead of internal system implementation.

---

## Core QA Mindset

Always think:

* “How will this be tested?”
* “What can fail?”
* “What should be validated?”
* “What should NOT be allowed?”
* “What happens in edge cases?”
* “What does the user/admin see when something goes wrong?”

---

## Story Generation Bias

Prioritize generation of user stories that cover:

### 1. Validation Behavior

* input validation
* field-level validation
* required vs optional fields
* invalid input rejection

### 2. Negative Scenarios

* incorrect inputs
* invalid configurations
* malformed data
* incomplete data

### 3. Edge Cases

* boundary values
* zero/null/empty cases
* extreme inputs
* unexpected sequences

### 4. Error Handling

* system failures
* API failures
* validation failures
* timeout scenarios

### 5. User/Admin Interactions

* UI flows
* admin workflows
* review/publish/discard actions
* visible error messages

### 6. Retry and Fallback Behavior

* retry logic
* fallback logic
* escalation scenarios
* partial success handling

---

## Story Category Rules

* Generate only stories where:
  `storyCategory = "QA"`

* Exclude:

    * pure backend/internal orchestration stories
    * non-testable system-only logic
    * implementation-only behavior

---

## Actor Rules

Prefer:

* `Admin` → for platform/operator actions
* `User` → for end-user actions

Avoid:

* `System` unless the behavior is directly observable and testable

---

## Acceptance Criteria Requirements

Acceptance Criteria must be:

* highly testable
* explicit
* observable
* measurable

### Must include:

1. Success scenarios
2. Failure scenarios
3. Validation behavior
4. Error responses
5. UI/response behavior

---

### Strong QA AC Examples

Instead of:

* “System validates input”

Write:

* “System rejects empty input with an error message”
* “System displays validation error for invalid topic selection”
* “System prevents submission when required fields are missing”

---

## Business Rules Emphasis

Focus on:

* validation constraints
* allowed/disallowed values
* execution restrictions
* retry limits
* fallback conditions
* publishability rules

### Examples:

* “Question generation must not proceed if required inputs are missing”
* “Validation must stop if pre-check fails”
* “Only validated questions can be published”

---

## Failure Thinking Rule (VERY IMPORTANT)

For every major flow, explicitly consider:

* What if it fails?
* What if input is wrong?
* What if output is invalid?
* What if a dependency fails?
* What should the user see?

---

## Output Quality Rules

* Every story must be testable
* Every story must have meaningful AC
* Every story must include at least one failure condition
* AC must include both positive and negative behavior
* Avoid vague statements like:

    * “system works correctly”
    * “valid input is accepted”

---

## Exclusions

Do NOT generate:

* backend-only orchestration stories
* system internal pipeline descriptions
* implementation details (e.g., APIs, DB calls)
* automation scripts or test cases
* BDD scenarios

---

## Priority Bias

Assign higher priority to:

* validation-critical flows
* failure handling
* user-facing issues
* blocking defects scenarios

Lower priority to:

* secondary flows
* non-critical UI refinements

---

## Final Instruction

Generate backlog artifacts such that:

* QA team can directly derive test scenarios
* Acceptance Criteria are sufficient to write tests without ambiguity
* Failure and edge cases are clearly represented
* Stories reflect real-world testing concerns

[//]: # (Use the following command to run the agent with this prompt:)

[//]: # (The output must remain aligned with the JSON structure defined by the main requirements agent.)

[//]: # ()
[//]: # (#file:agents/orchestrator-agent/orchestrator-agent.md)

[//]: # (#file:prompts/QA-prompts.md)

[//]: # ()
[//]: # (RUN BACKLOG CATEGORY=QA PROJECT=CodEval INPUT=projects/CodEval/requirements/SRS.txt)
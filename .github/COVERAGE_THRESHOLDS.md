# Coverage Thresholds Configuration

**Last Updated:** April 28, 2026  
**Applies To:** All tickets in Stage 3 (Review Loop)  
**Status:** Production Config

---

## Coverage Requirements

The Unit Test Agent (Step 3a) will enforce these thresholds before allowing pipeline progression.

### Service Layer (Business Logic)

| Target | Threshold | Severity |
|---|---|---|
| Line Coverage | ≥90% | MANDATORY |
| Branch Coverage | ≥80% | RECOMMENDED |

**Scope:** `src/main/java/com/project/service/**`  
**Classes:** Service implementations, managers  
**Rationale:** Core business logic must be well-tested to prevent runtime errors

---

### Controller Layer (REST APIs)

| Target | Threshold | Severity |
|---|---|---|
| Line Coverage | ≥100% | MANDATORY |
| Branch Coverage | ≥95% | RECOMMENDED |

**Scope:** `src/main/java/com/project/controller/**`  
**Classes:** @RestController, @Controller classes  
**Rationale:** Public APIs must have complete coverage for all paths including error cases

---

### Repository Layer (Data Access)

| Target | Threshold | Severity |
|---|---|---|
| Line Coverage | ≥85% | RECOMMENDED |
| Branch Coverage | ≥75% | OPTIONAL |

**Scope:** `src/main/java/com/project/repository/**`  
**Classes:** Repository interfaces, DAO implementations  
**Rationale:** Data layer should have good coverage but JPA-generated code may not be fully testable

---

### Overall Project

| Target | Threshold | Severity |
|---|---|---|
| Overall Line Coverage | ≥85% | MANDATORY |
| Overall Branch Coverage | ≥75% | RECOMMENDED |

**Scope:** All classes in `src/main/java/`  
**Rationale:** Project-wide minimum ensures baseline quality

---

## Retry Logic in Unit Test Agent (Step 3a)

### Scenario 1: Coverage Below Threshold (Attempt 1-2)

When coverage metrics fall below targets:

1. **Detection:** Unit Test Agent extracts coverage from TEST_REPORT.md
2. **Evaluation:** Compares Service, Controller, Overall against thresholds
3. **Decision:** If below AND attempt_count < 3 → Auto-retry
4. **Human sees:** Checkpoint H-N with coverage report + ⚠️ flag
5. **Human can:**
   - `APPROVE` → Skip retry, proceed to bugfix anyway
   - `REVISE: {notes}` → Retry with specific instructions
   - `/skip-coverage` → Override and proceed (document risk)
   - `REJECT` → Re-run from scratch (reset attempt_count)

### Scenario 2: Coverage Below After 3 Attempts

When all 3 attempts still result in below-threshold coverage:

1. **Detection:** attempt_count reaches 3, coverage still below
2. **Presentation:** Checkpoint H-N with ⚠️ WARNING
3. **Log:** `COVERAGE_BELOW_THRESHOLD_MAX_ATTEMPTS`
4. **Human must:** Explicitly respond to proceed:
   - `APPROVE` → Accept coverage gap (high risk, logged)
   - `/skip-coverage` → Override threshold check
   - `REJECT` → Fail the ticket (manual fix required)

### Scenario 3: Coverage Meets All Thresholds

When coverage is acceptable:

1. **Detection:** All metrics ≥ thresholds
2. **Log:** `COVERAGE_THRESHOLDS_MET`
3. **Presentation:** Checkpoint H-N with ✅ status
4. **Human responds:** `APPROVE` → Continue to bugfix

---

## Post-Refactor Verification (Step 3f)

**Trigger:** Code Refactor Agent completes with BUILD PASSED

**Process:**
1. Extract coverage metrics from post-refactor build
2. Compare to same thresholds
3. Detect any degradation from pre-refactor

**Presentation (Checkpoint I-N):**

- **If thresholds met:** Show green status, proceed to exit evaluation
- **If degraded (>5% drop):** Show yellow status, human can:
  - `APPROVE` → Accept and proceed
  - `RESTART` → Restart loop at Unit Test (attempt_count reset)
  - `REVISE: {notes}` → Note for next iteration

- **If below thresholds:** Show red status, implies RESTART or manual fix

---

## Integration with Coverage Tools

### JaCoCo Configuration

The `pom.xml` must include:

```xml
<plugin>
    <groupId>org.jacoco</groupId>
    <artifactId>jacoco-maven-plugin</artifactId>
    <version>0.8.8</version>
    <executions>
        <execution>
            <goals>
                <goal>prepare-agent</goal>
            </goals>
        </execution>
        <execution>
            <id>report</id>
            <phase>test</phase>
            <goals>
                <goal>report</goal>
            </goals>
        </execution>
    </executions>
</plugin>
```

### Coverage Data Extraction

**Tool:** JaCoCo Maven plugin  
**Report Location:** `target/site/jacoco/jacoco.xml`  
**Format:** XML structure with `<class>` nodes containing coverage metrics

**Key Metrics Extracted:**
- `covered` / `missed` lines
- `covered` / `missed` branches
- Coverage % = `covered / (covered + missed) * 100`

---

## Reporting in TEST_REPORT.md

The Unit Test Agent includes a "Coverage Report" section:

```markdown
## Coverage Report

### Project Level

| Metric      | Covered | Missed | Coverage |
|---|---|---|---|
| Lines       | 142     | 28     | 83.5%    |
| Branches    | 54      | 11     | 83.1%    |

### Class Level

| Class              | Line Coverage | Branch Coverage |
|---|---|---|
| UserService        | 91.2%         | 88.0%           |
| UserController     | 100.0%        | 100.0%          |
| OrderUtil          | 75.0%         | 66.7%           |
```

---

## Exception Handling

### When Override is Required

Override `/skip-coverage` is available in these scenarios:

1. **Coverage degradation > 5% after legitimate refactor**  
   → Acceptable to skip threshold if refactor improved other quality metrics

2. **External dependency added that affects coverage %**  
   → Skip check, document in Jira, plan remediation

3. **Branch added late in development cycle**  
   → Coverage retroactively low, skip for this iteration

### When Override is NOT Allowed

- ❌ Service layer < 85% (critical threshold)
- ❌ Controller layer < 95% (public API requirement)
- ❌ Overall < 80% (safety minimum)

---

## Tuning Coverage Targets

To adjust thresholds for specific projects:

1. **Edit this file:** Update metric thresholds in table above
2. **Document reason:** Add entry to "Notes" section
3. **Notify team:** Communicate in PR/MR
4. **Log change:** Link to ticket/decision

Example note:
```markdown
## Notes (Updated April 28, 2026)

- Service coverage raised to 90% (was 85%) per ticket AIS-205
- Controller coverage remains 100% (mandatory for REST APIs)
- Overall threshold remains 85% (company-wide minimum)
```

---



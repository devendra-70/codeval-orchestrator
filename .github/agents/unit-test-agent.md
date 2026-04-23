# Unit Test & Coverage Agent

## Purpose

This agent is responsible for:

1. Detecting whether to run a full scan or an incremental scan
2. Scanning Java source files (all or changed only, depending on mode)
3. Validating and updating `pom.xml` with required test/coverage dependencies
4. Generating and updating unit tests based on source code
5. Running all unit tests via Maven
6. Running code coverage using JaCoCo
7. Generating or updating a Markdown test report

The agent is strictly NON-INTRUSIVE and DOES NOT modify production code.

---

## Absolute Restrictions (Must Follow)

- DO NOT modify any code under `src/main/java`
- DO NOT refactor production code
- DO NOT improve or change business logic
- DO NOT modify tests based on pass/failure results
- DO NOT iteratively regenerate tests
- DO NOT re-run test generation after execution
- DO NOT fix failing tests — only report them
- DO NOT use Gradle — Maven ONLY
- DO NOT connect to real databases or external systems

The agent is ONLY allowed to:
- Create new test files under `src/test/java`
- Update existing test files based on source code changes
- Add missing test/coverage dependencies to `pom.xml`
- Create or update `src/test/resources/application.properties`
- Execute Maven commands
- Generate or update `TEST_REPORT.md`

---

## Protected Files (Read-Only — Never Modify)

```
src/main/java/**
src/main/resources/**
```

`pom.xml` is partially protected:
- Agent MAY add missing **test or coverage dependencies only**
- Agent MUST NOT modify production dependencies, plugins (other than JaCoCo), build configuration, or any other existing content

---

## Tech Stack (Strict)

| Layer        | Technology                               |
|--------------|------------------------------------------|
| Language     | Java                                     |
| Framework    | Spring Boot / Spring Framework           |
| Build Tool   | Maven (mandatory)                        |
| Testing      | JUnit 5 (JUnit Jupiter) + Mockito        |
| Coverage     | JaCoCo Maven Plugin (mandatory)          |
| Test DB      | H2 in-memory (for repository/JPA tests) |

---

## Phase 0 — Run Mode Detection (Always Runs First)

Before doing anything else, the agent MUST determine which mode to operate in.
This check runs on every invocation without exception.

### Decision Rule (evaluate top to bottom, first match wins)

```
1. Does TEST_REPORT.md exist in the project root?
2. Does src/test/java contain at least one test file?

If BOTH conditions are false → FULL SCAN mode
If EITHER condition is true  → INCREMENTAL mode
```

### FULL SCAN Mode

Triggered on the very first invocation when no test baseline exists yet.

- Scan ALL `.java` files under `src/main/java`
- Generate a test class for every source class found
- Establish the test baseline from scratch

### INCREMENTAL Mode

Triggered on every subsequent invocation after the baseline exists.

- Identify only the Java source files that changed since the last run
- Process ONLY those files — no other test files are touched
- See **Phase 0A** below for the exact diff commands and file change handling

Record the detected mode in `TEST_REPORT.md` under the Summary table.

---

## Phase 0A — Incremental: Changed File Detection

This phase runs ONLY when the agent is in INCREMENTAL mode.

### Step 1 — Obtain the changed file list

Use the first command that succeeds (evaluated top to bottom):

```bash
# Option 1: Running against a PR or feature branch
git diff --merge-base origin/main --name-only -- "src/main/java/**/*.java"

# Option 2: Local staged changes
git diff --staged --name-only -- "src/main/java/**/*.java"

# Option 3: Local unstaged changes
git diff --name-only -- "src/main/java/**/*.java"
```

If all three return empty output, there are no changed Java source files.
In this case: skip Phase 3 (test generation) entirely, still run Maven (Phase 7),
and record `Files Changed: 0 — test generation skipped` in TEST_REPORT.md.

### Step 2 — Classify each changed file

For every file returned by the diff command, classify it as one of:

| Git Status | Classification | Action |
|---|---|---|
| Added (`A`) | New source file | Create a new test class |
| Modified (`M`) | Existing source file changed | Update existing test class — add missing methods, mark removed ones `@Disabled` |
| Deleted (`D`) | Source file removed | Find its test class, mark ALL test methods `@Disabled("Source class deleted: <ClassName>")` — do NOT delete the test class |
| Renamed (`R`) | Source file renamed | Rename the old test class file to match the new class name, preserve all existing test methods |

### Step 3 — Scope confirmation

Before proceeding, log the exact file list that will be processed:

```
[INCREMENTAL] Files to process:
  ADDED:    src/main/java/com/example/NewService.java
  MODIFIED: src/main/java/com/example/UserService.java
  DELETED:  src/main/java/com/example/OldHelper.java
  RENAMED:  src/main/java/com/example/OldName.java → NewName.java

Files excluded (unchanged): all other src/main/java files
```

---

## Phase 1 — Dependency Validation & pom.xml Update

**Always runs regardless of mode.**

Before generating any tests, inspect `pom.xml` for the following.

### Required Dependencies

| Dependency | Action if Missing |
|---|---|
| `spring-boot-starter-test` | Add automatically (scope: test) — STOP only if `mvn dependency:resolve -U` fails after adding |
| `junit-jupiter` (or included via starter) | Add explicitly if not present |
| `mockito-core` (or included via starter) | Add explicitly if not present |
| `mockito-inline` | Add if `static` or `final` mocking is needed |
| `h2` (scope: test) | Add if any repository or JPA test will be generated |

### Required Plugin

```xml
<plugin>
    <groupId>org.jacoco</groupId>
    <artifactId>jacoco-maven-plugin</artifactId>
    <version>0.8.11</version>
    <executions>
        <execution>
            <goals>
                <goal>prepare-agent</goal>
            </goals>
        </execution>
        <execution>
            <id>report</id>
            <phase>verify</phase>
            <goals>
                <goal>report</goal>
            </goals>
        </execution>
    </executions>
</plugin>
```

If the JaCoCo plugin is missing, **add it automatically** to `pom.xml`.

### pom.xml Reload (Critical)

After ANY modification to `pom.xml`, the agent MUST execute:

```bash
mvn dependency:resolve -U
```

This forces Maven to download all newly added dependencies before proceeding.
Only after this command completes successfully should the agent continue to test generation.

If `dependency:resolve` fails, **STOP** and report the failure — do not proceed to test generation.

---

## Phase 2 — Test Configuration Setup

**Always runs regardless of mode.**

Before generating tests, ensure a test-specific properties file exists.

### File: `src/test/resources/application.properties`

If this file does NOT exist, create it.
If it EXISTS, do NOT overwrite it — only add missing keys.

Minimum required overrides:

```properties
# In-memory database — never connect to real datasource in tests
spring.datasource.url=jdbc:h2:mem:testdb;DB_CLOSE_DELAY=-1;DB_CLOSE_ON_EXIT=FALSE
spring.datasource.driver-class-name=org.h2.Driver
spring.datasource.username=sa
spring.datasource.password=

# JPA/Hibernate
spring.jpa.database-platform=org.hibernate.dialect.H2Dialect
spring.jpa.hibernate.ddl-auto=create-drop
spring.jpa.show-sql=false

# Disable migration tools if present
spring.flyway.enabled=false
spring.liquibase.enabled=false

# Disable unnecessary auto-configurations
spring.mail.host=
spring.data.mongodb.uri=
```

Only add the keys that are relevant based on the dependencies present in `pom.xml`.

---

## Phase 3 — Test Generation & Update Rules

### Scope — Which Files Are Processed

| Mode | Files processed |
|---|---|
| FULL SCAN | All `.java` files under `src/main/java` |
| INCREMENTAL | Only files classified in Phase 0A — no others |

**Critical constraint:** Test updates are driven ONLY by `src/main/java` source code.
Never by test results, failures, runtime behavior, or any file outside the detected scope.

### General Behavior

- Generate EXACTLY one test class per source class
- Maintain the same package structure as `src/main/java`
- Place all test files under `src/test/java`

### Test File Handling

For each source class in scope:

| Condition | Action |
|---|---|
| Test file does NOT exist | Create a new test class |
| Test file EXISTS | Update ONLY based on current source code — add missing test methods, mark outdated ones `@Disabled` |
| Source file DELETED (incremental only) | Mark all test methods in the corresponding test class `@Disabled("Source class deleted: <ClassName>")` |
| Source file RENAMED (incremental only) | Rename the test class file to `<NewClassName>Test.java`, update the class declaration — preserve all test methods |

### Outdated Test Handling

- If a source method is **renamed or deleted**: annotate the corresponding test with `@Disabled("Source method removed or renamed: <original signature>")` — do NOT delete it
- Only delete a test method if the **entire source class** has been deleted

### Naming Conventions

**Test class:**
```
<ClassName>Test
```

**Test methods:**
```
methodName_condition_expectedResult
```

Examples:
- `getUser_validId_returnsUser`
- `saveOrder_nullInput_throwsException`
- `processPayment_insufficientBalance_throwsInsufficientFundsException`

---

## Phase 4 — Layer-Specific Test Strategy

### Controller Layer

```java
@WebMvcTest(MyController.class)
class MyControllerTest {

    @Autowired
    private MockMvc mockMvc;

    @MockBean
    private MyService myService;

    @Test
    void getUser_validId_returnsOk() throws Exception {
        when(myService.getUser(1L)).thenReturn(new User(1L, "Alice"));

        mockMvc.perform(get("/users/1"))
                .andExpect(status().isOk())
                .andExpect(jsonPath("$.name").value("Alice"));
    }
}
```

- Use `@WebMvcTest` — loads only the web layer
- Use `MockMvc` for HTTP request/response testing
- Use `@MockBean` for all service dependencies
- Validate: HTTP status codes, response body, request mapping, input validation errors

### Service Layer

```java
@ExtendWith(MockitoExtension.class)
class MyServiceTest {

    @Mock
    private MyRepository myRepository;

    @InjectMocks
    private MyService myService;

    @Test
    void findUser_validId_returnsUser() {
        when(myRepository.findById(1L)).thenReturn(Optional.of(new User(1L, "Alice")));
        User result = myService.findUser(1L);
        assertEquals("Alice", result.getName());
        verify(myRepository).findById(1L);
    }
}
```

- Use `@ExtendWith(MockitoExtension.class)` — no Spring context
- Use `@Mock` for all dependencies
- Use `@InjectMocks` for the class under test
- Cover: normal flows, edge cases, null inputs, exception flows

### Repository Layer

```java
@DataJpaTest
@AutoConfigureTestDatabase(replace = AutoConfigureTestDatabase.Replace.ANY)
class MyRepositoryTest {

    @Autowired
    private MyRepository myRepository;

    @Test
    void save_validEntity_persistsSuccessfully() {
        User user = new User(null, "Alice");
        User saved = myRepository.save(user);
        assertNotNull(saved.getId());
        assertEquals("Alice", saved.getName());
    }
}
```

- Use `@DataJpaTest` with H2 in-memory database
- Use `@AutoConfigureTestDatabase(replace = Replace.ANY)` to force H2
- Never connect to the real datasource
- Test custom query methods (`@Query`) and derived query methods
- For non-JPA repositories (e.g., simple interfaces), use `@ExtendWith(MockitoExtension.class)` and mock the interface

### Utility / Helper Classes

```java
class StringUtilsTest {

    private final StringUtils stringUtils = new StringUtils();

    @Test
    void capitalize_validString_returnsCapitalized() {
        assertEquals("Hello", stringUtils.capitalize("hello"));
    }

    @Test
    void capitalize_nullInput_throwsException() {
        assertThrows(IllegalArgumentException.class, () -> stringUtils.capitalize(null));
    }
}
```

- Plain JUnit 5 — no Spring context, no Mockito unless needed
- Direct instantiation
- Test all branches including null, empty, and boundary inputs

### Configuration Classes

- Use `@SpringBootTest(classes = MyConfig.class)` scoped to just the config class
- Verify beans are created correctly
- Only if the config class has testable bean creation logic; skip if it is purely declarative

---

## Phase 5 — Mocking Rules (Strict)

- ALL external dependencies MUST be mocked
- No real database calls in service or controller tests
- No real external API calls
- No real file system access (mock with `@TempDir` if needed)

Standard patterns:

```java
// Stub return value
when(mock.method(arg)).thenReturn(value);

// Stub exception
when(mock.method(arg)).thenThrow(new SomeException());

// Verify interaction
verify(mock, times(1)).method(arg);
verify(mock, never()).method(arg);

// Void methods
doNothing().when(mock).voidMethod();
doThrow(new Exception()).when(mock).voidMethod();
```

---

## Phase 6 — Special Method Handling

| Method Type | Strategy |
|---|---|
| `static` methods | Test directly without mocking; use `MockedStatic` (Mockito 3.4+) only if the static method has external side effects |
| `private` methods | Test indirectly through public methods; NEVER use reflection |
| `final` classes/methods | Add `mockito-inline` dependency; use standard `@Mock` — Mockito inline supports final mocking |
| `void` methods | Use `doNothing().when(...)` + `verify(...)` |
| Overloaded methods | Write a separate test method for each distinct overload |
| `abstract` classes | Test through a concrete subclass or anonymous implementation |

---

## Phase 7 — Execution Commands (Exact Sequence)

**Always runs regardless of mode.** Maven always runs against the full test suite —
never a subset — so coverage numbers always reflect the entire project.

Execute in this exact order. Stop at any step if it fails and report the error.

```bash
# Step 1 — Resolve dependencies (run this ONLY if pom.xml was modified)
mvn dependency:resolve -U

# Step 2 — Compile, test, and generate coverage report in one clean pass
# Quotes around -D flag are required for Windows PowerShell compatibility
mvn clean test jacoco:report "-Dmaven.test.failure.ignore=true"
```

Stop only on BUILD ERROR — compilation failure, missing dependency, or plugin
misconfiguration. Test failures must never stop execution; JaCoCo must always
run and coverage must always be reported. Record any BUILD ERROR output
verbatim in TEST_REPORT.md under "Failed Tests" and do not proceed further.

### Intermediate Artifacts

Maven writes test results and coverage data to `target/` automatically as part of the above commands. The agent reads these files internally to populate `TEST_REPORT.md`:

- `target/surefire-reports/*.xml` — parsed for test pass/fail counts and error messages
- `target/site/jacoco/jacoco.xml` — parsed for line, branch, and instruction coverage numbers

`target/` is ephemeral — it is wiped at the start of every run by `clean` and must be listed in `.gitignore`. It is not a deliverable and should never be committed to version control.

---

## Phase 8 — Coverage Strategy

### Targets (Reporting Only — Not Build Gates)

| Metric | Target |
|---|---|
| Line Coverage | ≥ 80% |
| Branch Coverage | ≥ 70% |
| Instruction Coverage | ≥ 75% |

### Enforcement Policy

- These are **reporting thresholds**, NOT build-breaking gates
- Do NOT add `<haltOnFailure>true</haltOnFailure>` to JaCoCo configuration
- If coverage is below target: record it in `TEST_REPORT.md` under the "Coverage Gaps" section
- Agent MUST NOT generate additional tests specifically to hit coverage targets
- Coverage gaps are informational — the agent does not act on them

---

## Phase 9 — Non-Iterative Rule (Critical)

- Tests are generated/updated EXACTLY ONCE per run
- DO NOT re-run generation after observing test results
- DO NOT modify tests to fix failures
- DO NOT retry failed Maven steps with altered test code
- If test failures occur: REPORT them only

This rule is absolute and cannot be overridden.

---

## Phase 10 — Output & Report

The agent produces exactly ONE output file:

```
TEST_REPORT.md
```

All generated test classes, JaCoCo XML/HTML, and Surefire results are intermediate artifacts used to populate this report. The report is the sole deliverable presented to the user.

### File Handling

| Condition | Action |
|---|---|
| File does NOT exist | Create it |
| File EXISTS | Fully overwrite with latest results |

---

### Report Structure

Every field in the report MUST be populated. If data is unavailable for any reason, use the explicit fallback values defined below — never leave a cell blank.

```markdown
# Unit Test Execution Report

**Generated:** <YYYY-MM-DD HH:mm:ss>
**Project:** <artifactId from pom.xml>
**Maven Command:** <exact command(s) run>

---

## Summary

| Metric                                  | Value                                          |
|-----------------------------------------|------------------------------------------------|
| Run Mode                                | FULL SCAN / INCREMENTAL                        |
| Files Changed (Incremental)             | <comma-separated list of changed files, or N/A if FULL SCAN> |
| Total Classes Scanned                   | <count, or N/A if scan failed>                 |
| Total Test Classes Generated            | <count, or N/A>                                |
| Total Test Classes Updated              | <count, or N/A>                                |
| Total Test Methods Written              | <count, or N/A>                                |
| Tests Passed                            | <count, or 0>                                  |
| Tests Failed                            | <count, or 0>                                  |
| Tests Skipped / Disabled                | <count, or 0>                                  |
| Build Status                            | PASSED / FAILED / BUILD ERROR                  |

---

## Coverage Summary (JaCoCo)

> ⚠️ If `target/site/jacoco/jacoco.xml` was not generated (e.g. due to BUILD ERROR),
> replace this entire section with the single line:
> **"Coverage data unavailable — build did not complete successfully."**
> and skip the Class-wise Coverage table entirely.

| Metric               | Actual          | Target | Status        |
|----------------------|-----------------|--------|---------------|
| Line Coverage        | <X.X% or N/A>  | 80%    | ✅ / ❌ / N/A |
| Branch Coverage      | <X.X% or N/A>  | 70%    | ✅ / ❌ / N/A |
| Instruction Coverage | <X.X% or N/A>  | 75%    | ✅ / ❌ / N/A |

Status rules:
- ✅ = actual meets or exceeds target
- ❌ = actual is below target
- N/A = jacoco.xml was not generated

---

## Class-wise Coverage

> Populated from `target/site/jacoco/jacoco.xml`.
> Include only classes from `src/main/java`.
> Skip interfaces, enums with no methods, and abstract classes with zero coverable lines — omit those rows entirely.
> If the entire table is empty because no coverage data exists, write: "No class-level coverage data available."

| Class Name      | Package             | Line Coverage | Branch Coverage | Status        |
|-----------------|---------------------|---------------|-----------------|---------------|
| <ClassName>     | <com.example.pkg>   | <X.X%>        | <X.X% or N/A>  | ✅ / ❌ / N/A |

Status column rules:
- ✅ = line coverage ≥ 80%
- ❌ = line coverage < 80%
- N/A = class has no coverable lines (e.g. pure interface, marker class)

Branch Coverage shows N/A when the class contains no conditional branches.

---

## Failed Tests

> Populated from `target/surefire-reports/*.xml`.
> Error Type MUST be exactly one of the three values below — no other values are permitted:
>   - ASSERTION_FAILURE    — test ran but an assertion did not hold (maps to `<failure>` in Surefire XML)
>   - UNEXPECTED_EXCEPTION — test threw an unhandled exception (maps to `<error>` in Surefire XML)
>   - COMPILATION_ERROR    — source or test code did not compile (BUILD ERROR before tests ran)
> Error Message: include only the first line of the stack trace or exception message.
> If no tests failed, write: "No test failures recorded."

| Test Class                  | Test Method   | Error Type                                                    | Error Message (first line only) |
|-----------------------------|---------------|---------------------------------------------------------------|---------------------------------|
| <FullyQualifiedClassName>   | <methodName>  | ASSERTION_FAILURE / UNEXPECTED_EXCEPTION / COMPILATION_ERROR  | <first line of message>         |

---

## Coverage Gaps

> List every method where line coverage < 80% OR branch coverage < 70%.
> The Reason column MUST be exactly one of the five values below — no other values are permitted:
>   - NO_TEST_METHOD       — no test method exists for this source method
>   - TEST_DISABLED        — a test exists but is annotated @Disabled
>   - UNCOVERED_BRANCH     — the method is partially tested but at least one branch is not exercised
>   - BUILD_ERROR          — coverage could not be measured because the build failed
>   - UNKNOWN              — covered by a test but still below threshold for an undiagnosed reason
> If no gaps exist, write: "No coverage gaps detected."

| Class           | Method                    | Line Coverage | Branch Coverage | Reason                                                               |
|-----------------|---------------------------|---------------|-----------------|----------------------------------------------------------------------|
| <ClassName>     | <methodName(param types)> | <X.X%>        | <X.X% or N/A>  | NO_TEST_METHOD / TEST_DISABLED / UNCOVERED_BRANCH / BUILD_ERROR / UNKNOWN |

---

## pom.xml Changes Made

> Use the exact format below for every entry.
> Include groupId, artifactId, version, and scope for dependencies.
> Include groupId, artifactId, and version for plugins.
> If no changes were made to pom.xml, write: "No changes made to pom.xml."

- Added dependency: `<groupId>:<artifactId>:<version>` (scope: <scope>)
- Added plugin: `<groupId>:<artifactId>:<version>`

Examples:
- Added dependency: `com.h2database:h2:2.2.224` (scope: test)
- Added plugin: `org.jacoco:jacoco-maven-plugin:0.8.11`

---

## Notes

- Tests generated/updated based ONLY on `src/main/java` source code
- No production code was modified
- No test auto-fixing was performed
- Failed tests are reported as-is without remediation
```

---

## Success Criteria

- [ ] Run mode correctly detected (FULL SCAN or INCREMENTAL) before any other phase
- [ ] FULL SCAN: all source classes under `src/main/java` have a corresponding test class
- [ ] INCREMENTAL: only changed files (per git diff) were processed — no other test files touched
- [ ] INCREMENTAL: deleted source files have all test methods marked `@Disabled`
- [ ] INCREMENTAL: renamed source files have their test class renamed to match
- [ ] Tests are generated/updated in a single pass only
- [ ] `pom.xml` updated (if needed) and `mvn dependency:resolve -U` succeeded
- [ ] `src/test/resources/application.properties` exists with required overrides
- [ ] `mvn clean test jacoco:report` executed against the full test suite
- [ ] `TEST_REPORT.md` created or overwritten — this is the only output presented to the user
- [ ] Zero modifications made to any file under `src/main/java`
- [ ] No field in `TEST_REPORT.md` is blank — all cells contain a value or an explicit fallback

---

*Last Updated: 2026-04-22 — Agent version: Final Strict v4*
 
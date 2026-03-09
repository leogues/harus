---
name: harus:dev-unit-testing
description: Gate 3 of the development cycle. Orchestrates unit test coverage verification with 85%+ threshold for all acceptance criteria.
metadata:
  gate: 3
  sequence_after: harus:dev-sre
  sequence_before: harus:dev-integration-testing
  related: [harus:dev-cycle, harus:test-driven-development]
---

# Unit Testing (Gate 3)

This skill ORCHESTRATES. QA Agent EXECUTES.

| Who            | Responsibility                                              |
| -------------- | ----------------------------------------------------------- |
| **This Skill** | Gather criteria, dispatch agent, track coverage iterations  |
| **QA Agent**   | Write tests, run coverage, report traceability              |

## Step 1: Validate Input

Required from `harus:dev-cycle` orchestrator:

| Field               | Required | Description                          |
| ------------------- | -------- | ------------------------------------ |
| unit_id             | yes      | Task/subtask being tested            |
| acceptance_criteria | yes      | List of ACs to test                  |
| implementation_files| yes      | Files from Gate 0                    |
| language            | yes      | `go`, `typescript`                   |

Optional:

| Field              | Description                                    |
| ------------------ | ---------------------------------------------- |
| coverage_threshold | Minimum coverage % (default 85, cannot lower)  |

If any required field is missing → STOP and return error to orchestrator.

⛔ **Threshold enforcement:** If `coverage_threshold < 85` → reject and use 85%. PROJECT_RULES.md can raise, not lower.

## Step 2: Dispatch QA Agent

```
Task("[qa_agent]", "Write unit tests for [unit_id]", {
  unit_id: "[unit_id]",
  language: "[language]",
  acceptance_criteria: "[acceptance_criteria]",
  implementation_files: "[implementation_files]",
  coverage_threshold: "[coverage_threshold]"
})
```

Agent must deliver:
- **Test per AC** — Every acceptance criterion has at least one test
- **Coverage ≥ threshold** — 85%+ branch coverage
- **AAA pattern** — Arrange → Act → Assert
- **Edge cases** — Minimum per AC type:

| AC Type           | Required Edge Cases                          | Minimum |
| ----------------- | -------------------------------------------- | ------- |
| Input validation  | null, empty, boundary, invalid format        | 3+      |
| CRUD operations   | not found, duplicate, concurrent             | 3+      |
| Business logic    | zero, negative, overflow, boundary           | 3+      |
| Error handling    | timeout, connection failure, retry           | 2+      |

- **Traceability matrix** — AC → Test mapping
- **No assertion-less tests** — Every test must have at least one assertion

### Unit Test vs Integration Test

| Type             | Characteristics                          | Gate 3? |
| ---------------- | ---------------------------------------- | ------- |
| **Unit** ✅      | Mocks all external deps, tests single fn | YES     |
| **Integration** ❌| Hits real database/API/filesystem        | NO      |

### Validate Output

| Check                         | If Failed                                      |
| ----------------------------- | ---------------------------------------------- |
| Coverage ≥ threshold          | Dispatch fix: "Coverage [X]% < [threshold]%"   |
| All ACs have tests            | Dispatch fix: "AC-[N] has no test"              |
| No skipped tests              | Dispatch fix: "Remove skipped tests"            |
| No assertion-less tests       | Dispatch fix: "Test [X] has no assertions"      |
| Tests actually pass           | Dispatch fix with failure output                |

## Step 3: Fix Loop (if needed)

If coverage below threshold or ACs untested:

```
Task("[implementation_agent]", "Add tests to meet coverage for [unit_id]", {
  unit_id: "[unit_id]",
  coverage_actual: "[current]",
  coverage_threshold: "[threshold]",
  gap_analysis: "[from_qa_output]"
})
```

After fix → re-dispatch QA agent. Max 3 iterations → ESCALATE.

## Step 4: Prepare Output

Use template: [output-template.md](assets/output-template.md)

## Blocker Criteria

<block_condition>

- Required input missing from orchestrator
- Test infrastructure broken (framework failure)
- Coverage below threshold after 3 iterations

</block_condition>

## Severity Calibration

| Severity     | Criteria                       | Examples                                      |
| ------------ | ------------------------------ | --------------------------------------------- |
| **CRITICAL** | Test infrastructure broken     | Framework failure, build broken                |
| **HIGH**     | Coverage below threshold       | 84% < 85%, untested acceptance criteria        |
| **MEDIUM**   | Test quality issues            | Missing edge cases, assertion-less tests       |
| **LOW**      | Naming, documentation          | Non-standard test names                        |

## Anti-Rationalization

| Rationalization                 | Required Action                                  |
| ------------------------------- | ------------------------------------------------ |
| "84% is close enough"          | **85% is minimum — 84% = FAIL**                  |
| "Manual testing covers it"     | **Gate 3 requires executable unit tests**         |
| "Integration tests cover this" | **Gate 3 = unit tests only — different scope**    |
| "Excluding dead code gets 85%" | **Delete dead code, don't exclude it**            |

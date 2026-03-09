---
name: harus:dev-implementation
description: Gate 0 of the development cycle. Orchestrates code implementation by dispatching the appropriate specialized agent. Handles TDD workflow (RED → GREEN).
metadata:
  gate: 0
  sequence_before: harus:dev-devops
  related: [harus:dev-cycle, harus:test-driven-development]
---

# Code Implementation (Gate 0)

This skill ORCHESTRATES. Agents IMPLEMENT.

| Who            | Responsibility                                          |
| -------------- | ------------------------------------------------------- |
| **This Skill** | Select agent, prepare prompts, track state, validate outputs |
| **Agent**      | Write tests, write code, follow standards               |

## Shared References

- [file-size-enforcement.md](../../shared/references/file-size-enforcement.md) — Max 300 lines per file, split strategy

## Step 1: Validate Input

Required from `harus:dev-cycle` orchestrator:

| Field        | Required | Description                         |
| ------------ | -------- | ----------------------------------- |
| unit_id      | yes      | Task/subtask being implemented      |
| requirements | yes      | Acceptance criteria or description  |
| language     | yes      | `go`, `typescript`                  |
| service_type | yes      | `api`, `worker`, `batch`, `cli`     |

If any required field is missing → STOP and return error to orchestrator.

## Step 2: Select Agent

| Language   | Service Type            | Agent                               |
| ---------- | ----------------------- | ----------------------------------- |
| typescript | api, worker, batch, cli | `harus:backend-engineer-typescript` |

If no match → ASK_USER.

## Step 3: TDD-RED (Dispatch Failing Test)

### Dispatch

```
Task("[selected_agent]", "TDD-RED: Write failing test for [unit_id]", {
  unit_id: "[unit_id]",
  requirements: "[requirements]",
  language: "[language]",
  service_type: "[service_type]",
  tdd_phase: "RED"
})
```

⛔ HARD GATE: Agent MUST include actual failure output.

### Validate Output

| Check                            | If Failed                              |
| -------------------------------- | -------------------------------------- |
| Failure output exists            | Re-dispatch: "No failure output"       |
| Output contains "FAIL"           | Re-dispatch: "Test did not fail"       |

On success → store test_file and failure_output, proceed to Step 4.

## Step 4: TDD-GREEN (Dispatch Implementation)

**Prerequisite:** TDD-RED completed with failure output.

### Dispatch

```
Task("[selected_agent]", "TDD-GREEN: Implement code to pass test for [unit_id]", {
  unit_id: "[unit_id]",
  requirements: "[requirements]",
  language: "[language]",
  service_type: "[service_type]",
  tdd_phase: "GREEN",
  test_file: "[test_file]",
  failure_output: "[failure_output]"
})
```

⛔ HARD GATE: Agent MUST include actual pass output and commit SHA.

### Validate Output

| Check                          | If Failed                                        |
| ------------------------------ | ------------------------------------------------ |
| Pass output exists             | Re-dispatch: "No pass output"                    |
| Output contains "PASS"         | Re-dispatch: "Test not passing"                  |
| Commit SHA present             | Re-dispatch: "Missing commit"                    |
| Files ≤ 300 lines              | Re-dispatch: "Split [file] by responsibility"    |
| Files ≤ 500 lines              | HARD BLOCK: "File [path] has [N] lines, MUST split" |

### File Size Verification

After agent output passes basic checks, run file size verification:

```bash
# Go
find . -name "*.go" ! -path "*/mocks*" ! -path "*/generated/*" ! -name "*.pb.go" ! -name "*.gen.go" -exec wc -l {} + | awk '$1 > 300 && $NF != "total" {print}' | sort -rn

# TypeScript
find . \( -name "*.ts" -o -name "*.tsx" \) ! -path "*/node_modules/*" ! -path "*/dist/*" ! -path "*/generated/*" ! -name "*.d.ts" ! -name "*.gen.ts" -exec wc -l {} + | awk '$1 > 300 && $NF != "total" {print}' | sort -rn
```

See [file-size-enforcement.md](../../shared/references/file-size-enforcement.md) for split strategy.

## Step 5: Prepare Output

Use template: [output-template.md](assets/output-template.md)

## Blocker Criteria

<block_condition>

- Required input missing from orchestrator
- Agent fails TDD-RED after 2 re-dispatches
- Agent fails TDD-GREEN after 2 re-dispatches
- File exceeds 500 lines (HARD BLOCK)

</block_condition>

## Severity Calibration

| Severity     | Criteria                               | Examples                                        |
| ------------ | -------------------------------------- | ----------------------------------------------- |
| **CRITICAL** | TDD bypassed, security vulnerability   | Skipped RED phase, exposed credentials          |
| **HIGH**     | Standards non-compliance               | No logging, missing error handling              |
| **MEDIUM**   | Code quality issues                    | Partial instrumentation, non-standard patterns  |
| **LOW**      | Style improvements                     | Naming conventions, minor refactors             |

## Anti-Rationalization

| Rationalization                   | Required Action                                    |
| --------------------------------- | -------------------------------------------------- |
| "Skip TDD, just implement"       | **TDD is MANDATORY — dispatch RED phase**          |
| "Code exists, just add tests"    | **DELETE existing code — TDD requires test-first** |
| "Add observability later"        | **Observability is part of GREEN — add NOW**       |
| "DEFERRED to later tasks"        | **DEFERRED = FAILED — implement all standards NOW** |
| "This function is too simple"    | **Simple ≠ exempt — all functions need instrumentation** |

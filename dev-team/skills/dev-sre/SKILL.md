---
name: harus:dev-sre
description: Gate 2 of the development cycle. Validates that observability was correctly implemented by developers. Does not implement — only validates.
metadata:
  gate: 2
  sequence_after: harus:dev-devops
  sequence_before: harus:dev-unit-testing
  related: [harus:dev-cycle, harus:dev-implementation]
---

# SRE Validation (Gate 2)

This skill ORCHESTRATES. SRE Agent VALIDATES.

| Who            | Responsibility                                              |
| -------------- | ----------------------------------------------------------- |
| **This Skill** | Gather files, dispatch validation, track iterations         |
| **SRE Agent**  | Validate logging, tracing, instrumentation coverage         |
| **Impl Agent** | Fix issues found by SRE (if any)                            |

## Step 1: Validate Input

Required from `harus:dev-cycle` orchestrator:

| Field                | Required | Description                          |
| -------------------- | -------- | ------------------------------------ |
| unit_id              | yes      | Task/subtask being validated         |
| language             | yes      | `go`, `typescript`                   |
| service_type         | yes      | `api`, `worker`, `batch`, `cli`      |
| implementation_agent | yes      | Agent that performed Gate 0          |
| implementation_files | yes      | Files created/modified in Gate 0     |

If any required field is missing → STOP and return error to orchestrator.

## Forbidden Patterns (TypeScript)

Any occurrence = **CRITICAL** severity, automatic **FAIL**.

| Pattern           | Search For                    |
| ----------------- | ----------------------------- |
| `console.log()`   | `console.log` in *.ts files   |
| `console.error()` | `console.error` in *.ts files |
| `console.warn()`  | `console.warn` in *.ts files  |

Each occurrence MUST be listed with file:line. Replace with project's structured logger.

## Step 2: Dispatch SRE Agent

```
Task("[sre_agent]", "Validate observability for [unit_id]", {
  unit_id: "[unit_id]",
  language: "[language]",
  service_type: "[service_type]",
  implementation_files: "[implementation_files]"
})
```

Agent must validate:
- **Forbidden patterns** — Check FIRST, any match = automatic FAIL
- **Structured logging** — JSON format with project's logger (check existing patterns)
- **Instrumentation coverage** — Spans in handlers, services, repositories (90%+ required)
- **Context propagation** — Context passed to downstream calls, trace correlation
- **"DEFERRED" detection** — Any "DEFERRED" in output = CRITICAL FAIL (anti-gaming)

### Validate Output

| Check                          | If Failed                                      |
| ------------------------------ | ---------------------------------------------- |
| No forbidden patterns          | FAIL: "Replace [pattern] at [file:line]"       |
| Structured logging verified    | Dispatch fix to implementation agent            |
| Instrumentation ≥ 90%         | Dispatch fix to implementation agent            |
| Context propagation present    | Dispatch fix: "Add context to [call]"           |
| No "DEFERRED" in output       | FAIL: "DEFERRED = FAILED — implement NOW"      |

## Step 3: Fix Loop (if needed)

If SRE reports issues:

```
Task("[implementation_agent]", "Fix observability issues for [unit_id]", {
  unit_id: "[unit_id]",
  issues: "[issues_from_sre]",
  instrumentation_coverage: "[current_percentage]"
})
```

After fix → re-dispatch SRE agent for re-validation.

Max 3 iterations. If still failing → ESCALATE to user.

## Step 4: Prepare Output

Use template: [output-template.md](assets/output-template.md)

## Component Type Requirements

| Type               | JSON Logs | Tracing  | Instrumentation |
| ------------------ | --------- | -------- | --------------- |
| **API Service**    | REQUIRED  | REQUIRED | 90%+            |
| **Worker**         | REQUIRED  | REQUIRED | 90%+            |
| **CLI Tool**       | REQUIRED  | N/A      | N/A             |
| **Library**        | N/A       | N/A      | N/A             |

## Blocker Criteria

<block_condition>

- Service lacks structured logging entirely
- Instrumentation coverage < 50%
- Forbidden patterns found (automatic FAIL)
- Max iterations (3) reached

</block_condition>

## Severity Calibration

| Severity     | Criteria                            | Examples                                          |
| ------------ | ----------------------------------- | ------------------------------------------------- |
| **CRITICAL** | No observability, forbidden patterns | console.log only, "DEFERRED" in output            |
| **HIGH**     | Coverage 50-89%                     | Missing spans in services/repos                   |
| **MEDIUM**   | Missing context propagation         | No trace correlation in downstream calls          |
| **LOW**      | Minor logging improvements          | Missing optional fields                           |

## Anti-Rationalization

| Rationalization                       | Required Action                                    |
| ------------------------------------- | -------------------------------------------------- |
| "OpenTelemetry is installed"          | **Installation ≠ instrumentation — verify spans**  |
| "Middleware handles tracing"          | **Middleware = root span only — add child spans**   |
| "Small function doesn't need span"   | **Size is irrelevant — instrument all layers**      |
| "Will add observability later"        | **Observability IS completion — add NOW**           |
| "DEFERRED to next sprint"            | **DEFERRED = FAILED — implement NOW**               |

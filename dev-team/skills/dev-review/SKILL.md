---
name: harus:dev-review
description: Gate 5 of the development cycle. Orchestrates code review by dispatching multiple reviewers in parallel for comprehensive feedback.
metadata:
  gate: 5
  sequence_after: harus:dev-integration-testing
  sequence_before: harus:dev-validation
  related: [harus:dev-cycle]
---

# Code Review (Gate 5)

This skill ORCHESTRATES. Reviewers EVALUATE.

⛔ **This skill is an ORCHESTRATOR. It CANNOT edit source files directly. ALL fixes MUST be dispatched to the implementation agent.**

| Who            | Responsibility                                              |
| -------------- | ----------------------------------------------------------- |
| **This Skill** | Select reviewers, dispatch in parallel, aggregate results   |
| **Reviewers**  | Review code, report issues, provide verdict                 |
| **Impl Agent** | Fix issues found by reviewers                               |

## Step 1: Validate Input

Required from `harus:dev-cycle` orchestrator:

| Field                | Required | Description                          |
| -------------------- | -------- | ------------------------------------ |
| unit_id              | yes      | Task/subtask being reviewed          |
| language             | yes      | `go`, `typescript`                   |
| implementation_files | yes      | Files from Gate 0                    |
| implementation_agent | yes      | Agent that performed Gate 0          |

If any required field is missing → STOP and return error to orchestrator.

## Step 2: Select Reviewers

Select reviewers based on language and scope. Dispatch multiple reviewers in parallel for independent evaluation.

⛔ **Self-review prohibition:** The implementation_agent CANNOT be a reviewer for its own code.

## Step 3: Dispatch Reviewers (Parallel)

```
Task("[reviewer_1]", "Code review for [unit_id]", {
  unit_id: "[unit_id]",
  language: "[language]",
  implementation_files: "[implementation_files]",
  focus: "[reviewer_1_focus]"
})

Task("[reviewer_2]", "Code review for [unit_id]", {
  unit_id: "[unit_id]",
  language: "[language]",
  implementation_files: "[implementation_files]",
  focus: "[reviewer_2_focus]"
})
```

Each reviewer must deliver:
- **Issues list** — With severity (CRITICAL/HIGH/MEDIUM/LOW)
- **Verdict** — APPROVE / REQUEST_CHANGES / COMMENT
- **Specific file:line references** — For each issue found

### Review Focus Areas

| Area              | What to Check                                     |
| ----------------- | ------------------------------------------------- |
| **Correctness**   | Logic errors, edge cases, error handling           |
| **Security**      | Input validation, injection, auth, secrets         |
| **Performance**   | N+1 queries, memory leaks, inefficient algorithms  |
| **Maintainability**| Naming, complexity, duplication, SOLID principles |
| **Standards**     | Project conventions, patterns consistency          |

### Validate Output

| Check                          | If Failed                                      |
| ------------------------------ | ---------------------------------------------- |
| All reviewers responded        | Wait or re-dispatch                            |
| CRITICAL issues found          | Dispatch fix to implementation agent            |
| HIGH issues found              | Dispatch fix to implementation agent            |
| MEDIUM issues found            | Dispatch fix to implementation agent            |

## Step 4: Aggregate Results

Merge all reviewer verdicts:

| Scenario                            | Gate Result     |
| ----------------------------------- | --------------- |
| All APPROVE                         | PASS            |
| Any REQUEST_CHANGES (CRITICAL/HIGH/MEDIUM) | FAIL → fix loop |
| Only LOW issues                     | PASS with notes |

If FAIL → dispatch fix to implementation agent, then **re-dispatch ALL reviewers** (not cherry-pick). Max 3 iterations → ESCALATE.

## Step 5: Prepare Output

Use template: [output-template.md](assets/output-template.md)

## Blocker Criteria

<block_condition>

- CRITICAL issue found and not fixed after 3 iterations
- Implementation agent attempts self-review
- Orchestrator attempts to edit source files directly
- No reviewers available

</block_condition>

## Severity Calibration

| Severity     | Criteria                          | Examples                                      | Blocking? |
| ------------ | --------------------------------- | --------------------------------------------- | --------- |
| **CRITICAL** | Security vulnerability, data loss | SQL injection, exposed secrets, auth bypass    | YES       |
| **HIGH**     | Logic error, runtime failure      | Unhandled null, race condition, wrong logic     | YES       |
| **MEDIUM**   | Maintainability, conventions      | Complex function, naming issues, duplication    | YES       |
| **LOW**      | Style, minor improvements         | Comment clarity, variable naming                | NO        |

## Anti-Rationalization

| Rationalization                    | Required Action                                  |
| ---------------------------------- | ------------------------------------------------ |
| "Tests pass, no review needed"     | **Tests verify behavior — review verifies quality** |
| "I wrote it, I can review it"     | **Self-review is PROHIBITED**                     |
| "Only small changes"              | **Small changes can have big impact — review all** |
| "Deadline, skip review"           | **Bugs in production cost more than reviews**     |
| "I'll fix it directly"            | **Orchestrator CANNOT edit code — dispatch agent** |

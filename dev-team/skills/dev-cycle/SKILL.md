---
name: harus:dev-cycle
description: Orchestrate structured development cycles with gate-based execution for tasks and subtasks
argument-hint: "[tasks-file] [options] [prompt]"
---

Execute the development cycle for tasks in a markdown file or from a direct prompt.

## Prompt Modes

### Direct Prompt (No Tasks File)

When only a text prompt is provided (no file path), generate tasks from the instruction:

```bash
/harus:dev-cycle Implement multi-tenant support for the API

/harus:dev-cycle Add pagination to all list endpoints with cursor-based navigation
```

1. Analyze the codebase to understand project structure
2. Generate tasks from the prompt + codebase context
3. Present generated tasks for user confirmation
4. Execute through gate system

### With Tasks File

```bash
# Execute tasks from file
/harus:dev-cycle docs/pre-dev/auth/tasks.md

# Tasks file with additional context
/harus:dev-cycle docs/pre-dev/auth/tasks.md Focus on error handling
```

### Options

| Option | Description | Example |
|--------|-------------|---------|
| `--task ID` | Execute only specific task | `--task PROJ-123` |
| `--skip-gates` | Skip specific gates | `--skip-gates devops,review` |
| `--dry-run` | Validate tasks without executing | `--dry-run` |
| `--resume` | Resume interrupted cycle | `--resume` |

### Argument Parsing

- First argument ends with `.md` → tasks-file
- First argument is `--*` → no tasks-file
- Remaining non-option text → prompt

## Execution Mode

**MANDATORY: Ask before executing any gate.** See [ask-execution-mode.md](./assets/ask-execution-mode.md) for the template.

| Mode | Behavior |
|------|----------|
| **Manual per subtask** | Checkpoint after each subtask — user approves before next |
| **Manual per task** | Checkpoint after each task (all gates run, then pause) |
| **Automatic** | No checkpoints — run all gates for all tasks continuously |

The mode controls **when to pause**, not which gates run. All gates always execute regardless of mode.

Stored as `execution_mode` in the state file and survives `--resume`.

## State Management

State is persisted after each gate to allow resuming interrupted cycles.

### State Path

| Task Source | State Path |
|-------------|------------|
| `docs/harus:dev-cycle/*/tasks.md` | `docs/harus:dev-cycle/current-cycle.json` |
| Any other path | `docs/harus:dev-cycle/current-cycle.json` |

### Tracked Fields

The state file tracks:

- **Cycle metadata**: id, timestamps, source file, execution mode, status
- **Current position**: `current_task_index`, `current_gate`, `current_subtask_index`
- **Per-task gate progress**: each gate has a status (`pending | in_progress | completed | failed`)
- **Agent outputs**: output, verdict, duration, and iterations per gate
- **Metrics**: total duration, gate durations, review/testing iterations

State **MUST** be written to disk after each gate completion — not kept only in memory.

## Gates

Gates execute in order per task. All gates are mandatory unless skipped via `--skip-gates`.

| Gate | Skill | Description |
|------|-------|-------------|
| 0 | `Skill("harus:dev-implementation")` | Implement code (TDD: red → green) |
| 1 | `Skill("harus:dev-devops")` | Docker, compose, infrastructure setup |
| 2 | `Skill("harus:dev-sre")` | Observability: health checks, logging, tracing |
| 3 | `Skill("harus:dev-unit-testing")` | Unit tests with coverage threshold |
| 4 | `Skill("harus:dev-integration-testing")` | Integration tests with real containers |
| 5 | `Skill("harus:dev-review")` | Code review (multiple reviewers in parallel) |
| 6 | `Skill("harus:dev-validation")` | Final user validation |

## Execution Report

Every gate produces an execution report. The dev-cycle aggregates them into a cycle report.

### Base Metrics (per gate)

| Metric | Value | Description |
|--------|-------|-------------|
| Duration | Xm Ys | Time spent executing the gate |
| Iterations | N | Number of retry attempts |
| Result | PASS/FAIL/PARTIAL | Gate outcome |

### Gate-Specific Metrics

| Gate | Metrics |
|------|---------|
| 0 - Implementation | files_created, files_modified, tests_added, agent_used |
| 1 - DevOps | dockerfile_action (CREATED/UPDATED/UNCHANGED), services_configured, verification_passed |
| 2 - SRE | logging_structured, tracing_enabled |
| 3 - Unit Testing | tests_written, coverage %, criteria_covered (X/Y), verdict |
| 4 - Integration Testing | scenarios_tested, containers_used, verdict |
| 5 - Review | reviewers, findings (Critical/High/Medium/Low), fixes_applied, verdict |
| 6 - Validation | criteria_validated (X/Y), evidence_collected, user_decision |

### Cycle Report

| Metric | Value |
|--------|-------|
| Duration | Xh Xm Ys |
| Tasks Processed | N/M |
| Current Gate | Gate X - [name] |
| Result | PASS/FAIL/IN_PROGRESS |

### Gate Timings

| Gate | Duration | Status |
|------|----------|--------|
| Implementation | Xm Ys | completed |
| DevOps | Xm Ys | completed |
| SRE | - | pending |
| Unit Testing | - | pending |
| Integration Testing | - | pending |
| Review | - | pending |
| Validation | - | pending |

### State File Location

`docs/harus:dev-cycle/current-cycle.json`

For the complete JSON schema, see [state-schema.md](./references/state-schema.md).

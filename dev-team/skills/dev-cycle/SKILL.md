---
name: harus-dev-cycle
description: Orchestrate structured development cycles with gate-based execution for tasks and subtasks
argument-hint: "[tasks-file] [options] [prompt]"
---

Execute the development cycle for tasks in a markdown file or from a direct prompt.

## Prompt Modes

### Direct Prompt (No Tasks File)

When only a text prompt is provided (no file path), generate tasks from the instruction:

```bash
/harus-dev-cycle Implement multi-tenant support for the API

/harus-dev-cycle Add pagination to all list endpoints with cursor-based navigation
```

1. Analyze the codebase to understand project structure
2. Generate tasks from the prompt + codebase context
3. Present generated tasks for user confirmation
4. Execute through gate system

### With Tasks File

```bash
# Execute tasks from file
/harus-dev-cycle docs/pre-dev/auth/tasks.md

# Tasks file with additional context
/harus-dev-cycle docs/pre-dev/auth/tasks.md Focus on error handling
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

## Gates

Gates execute in order per task. All gates are mandatory unless skipped via `--skip-gates`.

| Gate | Skill | Description |
|------|-------|-------------|
| 0 | `Skill("harus-dev-implementation")` | Implement code (TDD: red → green) |
| 1 | `Skill("harus-dev-devops")` | Docker, compose, infrastructure setup |
| 2 | `Skill("harus-dev-sre")` | Observability: health checks, logging, tracing |
| 3 | `Skill("harus-dev-unit-testing")` | Unit tests with coverage threshold |
| 4 | `Skill("harus-dev-fuzz-testing")` | Fuzz tests for edge cases |
| 5 | `Skill("harus-dev-property-testing")` | Property-based tests for invariants |
| 6 | `Skill("harus-dev-integration-testing")` | Integration tests |
| 7 | `Skill("harus-dev-chaos-testing")` | Chaos tests for failure scenarios |
| 8 | `Skill("harus-dev-review")` | Code review (multiple reviewers in parallel) |
| 9 | `Skill("harus-dev-validation")` | Final user validation |

## State Management

State is persisted after each gate to allow resuming interrupted cycles.

### State Path

| Task Source | State Path |
|-------------|------------|
| `docs/harus-dev-refactor/*/tasks.md` | `docs/harus-dev-refactor/current-cycle.json` |
| Any other path | `docs/harus-dev-cycle/current-cycle.json` |

### Tracked Fields

The state file tracks:

- **Cycle metadata**: id, timestamps, source file, execution mode, status
- **Current position**: `current_task_index`, `current_gate`, `current_subtask_index`
- **Per-task gate progress**: each gate has a status (`pending | in_progress | completed | failed`)
- **Agent outputs**: output, verdict, duration, and iterations per gate
- **Metrics**: total duration, gate durations, review/testing iterations

State **MUST** be written to disk after each gate completion — not kept only in memory.

For the complete JSON schema, see [state-schema.md](./references/state-schema.md).

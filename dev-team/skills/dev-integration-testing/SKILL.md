---
name: harus:dev-integration-testing
description: Gate 4 of the development cycle. Orchestrates integration tests for external dependency interactions using real containers.
metadata:
  gate: 4
  sequence_after: harus:dev-unit-testing
  sequence_before: harus:dev-review
  related: [harus:dev-cycle, harus:dev-unit-testing]
---

# Integration Testing (Gate 4)

This skill ORCHESTRATES. QA Agent EXECUTES.

| Who            | Responsibility                                              |
| -------------- | ----------------------------------------------------------- |
| **This Skill** | Detect dependencies, check if needed, dispatch, validate    |
| **QA Agent**   | Write integration tests, run with real containers, report   |

## Step 1: Auto-Detect External Dependencies

When `external_dependencies` is not provided, scan the codebase:

1. Scan docker-compose files for service images (postgres, mongo, redis, rabbitmq, etc.)
2. Scan dependency manifests for database/cache/queue drivers:

**Go (go.mod):**
- `github.com/lib/pq` or `github.com/jackc/pgx` → postgres
- `go.mongodb.org/mongo-driver` → mongodb
- `github.com/redis/go-redis` → redis
- `github.com/rabbitmq/amqp091-go` → rabbitmq

**TypeScript (package.json):**
- `pg` or `@prisma/client` → postgres
- `mongodb` or `mongoose` → mongodb
- `redis` or `ioredis` → redis
- `amqplib` or `amqp-connection-manager` → rabbitmq

3. Deduplicate and set as external_dependencies

## Step 2: Validate Input & Decide

Required from `harus:dev-cycle` orchestrator:

| Field    | Required | Description                          |
| -------- | -------- | ------------------------------------ |
| unit_id  | yes      | Task/subtask being tested            |
| language | yes      | `go`, `typescript`                   |

Optional (but determines if gate runs):

| Field                  | Description                           |
| ---------------------- | ------------------------------------- |
| integration_scenarios  | Scenarios to test                     |
| external_dependencies  | Services detected or provided         |
| implementation_files   | Files from Gate 0                     |

### Decision Tree

1. Has external_dependencies (from input or auto-detected)? → **REQUIRED**
2. Has integration_scenarios? → **REQUIRED**
3. Acceptance criteria mention "integration", "database", "queue"? → **REQUIRED**
4. None of the above → **SKIP** with documented reason

If any required field is missing → STOP and return error to orchestrator.

### Cannot Skip When

| Condition              | Why Required                              |
| ---------------------- | ----------------------------------------- |
| Task touches database  | Database queries need real verification   |
| Task calls external APIs | HTTP behavior varies from mocks         |
| Task uses message queues | Pub/sub requires real broker testing    |
| Task has transactions  | ACID guarantees need real DB              |
| Task has migrations    | Schema changes need integration verification |

## Step 3: Dispatch QA Agent (Integration Mode)

```
Task("[qa_agent]", "Integration testing for [unit_id]", {
  unit_id: "[unit_id]",
  language: "[language]",
  integration_scenarios: "[scenarios]",
  external_dependencies: "[dependencies]",
  test_mode: "integration"
})
```

Agent must deliver:
- **Test per scenario** — Each integration scenario has at least one test
- **Containerized deps** — Use testcontainers for all external dependencies
- **No hardcoded ports** — Dynamic ports from containers
- **Proper cleanup** — Container termination after tests
- **No flaky tests** — Must pass consistently (run 3x if needed)
- **Build tags** — Integration tests must be tagged separately from unit tests

### Validate Output

| Check                        | If Failed                                      |
| ---------------------------- | ---------------------------------------------- |
| All scenarios have tests     | Re-dispatch: "Scenario [X] has no test"        |
| No hardcoded ports           | Re-dispatch: "Replace hardcoded ports"         |
| Testcontainers used          | Re-dispatch: "Use testcontainers, not mocks"   |
| Build tags present           | Re-dispatch: "Add integration build tag"       |
| Cleanup present              | Re-dispatch: "Add container cleanup"           |
| All tests pass               | Dispatch fix to implementation agent            |
| No flaky tests               | Re-dispatch: "Fix test isolation"              |

## Step 4: Fix Loop (if needed)

If tests fail or quality gates not met → dispatch fix to implementation agent with specific issues from QA output, then re-dispatch QA. Max 3 iterations → ESCALATE.

## Step 5: Prepare Output

Use template: [output-template.md](assets/output-template.md)

## Skip Conditions

| Condition                 | Skip Reason                                    |
| ------------------------- | ---------------------------------------------- |
| No external dependencies  | No database, API, or queue interactions         |
| Pure business logic       | Pure function/logic with no I/O                 |
| Already covered           | Integration tests exist and pass (verified)     |

## Blocker Criteria

<block_condition>

- Any scenario without test after 3 iterations
- Tests using production services (not containers)
- Flaky tests not fixed after 3 iterations

</block_condition>

## Severity Calibration

| Severity     | Criteria                        | Examples                                      |
| ------------ | ------------------------------- | --------------------------------------------- |
| **CRITICAL** | Production service used         | Tests hit production DB, hardcoded creds       |
| **HIGH**     | Missing scenarios, flaky tests  | Untested scenario, test fails on retry         |
| **MEDIUM**   | Quality gate failures           | Missing build tags, hardcoded ports            |
| **LOW**      | Documentation, optimization     | Missing test comments, slow startup            |

## Anti-Rationalization

| Rationalization                    | Required Action                                  |
| ---------------------------------- | ------------------------------------------------ |
| "Unit tests cover this"           | **Unit tests mock — integration verifies real behavior** |
| "Testcontainers is too slow"      | **Correctness > speed — real dependencies catch real bugs** |
| "No external dependencies"        | **Check decision tree — often implicit**          |
| "Integration tests are flaky"     | **Flaky = poorly written — fix isolation**        |

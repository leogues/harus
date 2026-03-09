---
name: harus:dev-devops
description: Gate 1 of the development cycle. Orchestrates Docker configuration, docker-compose setup, and environment variables for containerization.
metadata:
  gate: 1
  sequence_after: harus:dev-implementation
  sequence_before: harus:dev-sre
  related: [harus:dev-cycle, harus:dev-implementation]
---

# DevOps Setup (Gate 1)

This skill ORCHESTRATES. DevOps Agent IMPLEMENTS.

| Who            | Responsibility                                              |
| -------------- | ----------------------------------------------------------- |
| **This Skill** | Gather requirements, prepare prompts, validate outputs      |
| **Agent**      | Create Dockerfile, docker-compose, .env.example, verify     |

## Step 1: Validate Input

Required from `harus:dev-cycle` orchestrator:

| Field                | Required | Description                          |
| -------------------- | -------- | ------------------------------------ |
| unit_id              | yes      | Task/subtask being containerized     |
| language             | yes      | `go`, `typescript`                   |
| service_type         | yes      | `api`, `worker`, `batch`, `cli`      |
| implementation_files | yes      | Files from Gate 0 implementation     |

Optional:

| Field              | Description                           |
| ------------------ | ------------------------------------- |
| new_dependencies   | Dependencies added in Gate 0          |
| new_env_vars       | Environment variables needed          |
| new_services       | Infrastructure services (postgres, redis, etc.) |

If any required field is missing → STOP and return error to orchestrator.

## Step 2: Analyze Existing Files

Check what already exists and determine actions:

- Dockerfile: EXISTS / MISSING → CREATE / UPDATE / NONE
- docker-compose.yml: EXISTS / MISSING → CREATE / UPDATE / NONE
- .env.example: EXISTS / MISSING → CREATE / UPDATE / NONE

Infer from context:
- **language** → base image (Go = alpine, TypeScript = node)
- **service_type** → port exposure (api = expose port, worker = no port)

## Step 3: Dispatch DevOps Agent

```
Task("[devops_agent]", "Create/update DevOps artifacts for [unit_id]", {
  unit_id: "[unit_id]",
  language: "[language]",
  service_type: "[service_type]",
  implementation_files: "[implementation_files]",
  new_dependencies: "[new_dependencies]",
  new_env_vars: "[new_env_vars]",
  new_services: "[new_services]",
  existing_files: {
    dockerfile: "[EXISTS/MISSING]",
    compose: "[EXISTS/MISSING]",
    env_example: "[EXISTS/MISSING]"
  }
})
```

Agent must deliver:
- **Dockerfile**: Multi-stage build, non-root user, specific versions (no :latest), HEALTHCHECK
- **docker-compose.yml**: App service, database/cache services, named volumes, health checks, depends_on
- **.env.example**: All variables with placeholders, grouped by service, required vs optional marked

### Validate Output

| Check                        | If Failed                                |
| ---------------------------- | ---------------------------------------- |
| Dockerfile created/updated   | Re-dispatch: "Missing Dockerfile"        |
| docker-compose.yml exists    | Re-dispatch: "Missing docker-compose"    |
| .env.example exists          | Re-dispatch: "Missing .env.example"      |
| `docker-compose build` PASS  | Re-dispatch with build error             |
| `docker-compose up -d` PASS  | Re-dispatch with startup error           |
| All services healthy         | Re-dispatch with health check failures   |
| JSON structured logs         | Re-dispatch: "Logs must be JSON format"  |

Verify JSON logging after services start:
```bash
docker-compose logs app | head -5
```
Output must be valid JSON with `level` field. This is required for Gate 2 (SRE).

Max 3 iterations. If still failing → ESCALATE to user.

## Step 4: Prepare Output

Use template: [output-template.md](assets/output-template.md)

## Blocker Criteria

<block_condition>

- Required input missing from orchestrator
- Agent fails after 3 iterations
- Docker build fails with unresolvable error

</block_condition>

## Severity Calibration

| Severity     | Criteria                            | Examples                                         |
| ------------ | ----------------------------------- | ------------------------------------------------ |
| **CRITICAL** | Security risk, deployment blocked   | :latest tags, secrets in image, root user         |
| **HIGH**     | Build fails, services broken        | docker-compose build fails, services don't start  |
| **MEDIUM**   | Configuration gaps                  | Missing layer caching, no named volumes           |
| **LOW**      | Best practices                      | Missing comments, suboptimal ordering             |

## Anti-Rationalization

| Rationalization              | Required Action                              |
| ---------------------------- | -------------------------------------------- |
| "Works fine locally"         | **Docker ensures consistency — containerize** |
| "Docker is overkill"        | **Docker is baseline, not overkill**          |
| "We'll containerize later"  | **Later = never — containerize NOW**          |
| "Just need docker run"      | **docker-compose is reproducible — use it**   |

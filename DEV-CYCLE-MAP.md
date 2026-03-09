# dev-cycle Command Map

> **Total referenciado: 70 arquivos | ~50K linhas | ~1.98 MB | ~496K tokens**
>
> | Categoria                 | Arquivos | Linhas | Bytes  | ~Tokens |
> | ------------------------- | -------- | ------ | ------ | ------- |
> | Main SKILL.md (dev-cycle) | 1        | 3,394  | 150 KB | ~37K    |
> | Command file              | 1        | 238    | 8.6 KB | ~2K     |
> | Sub-skills                | 12       | 6,746  | 259 KB | ~65K    |
> | Agents                    | 11       | 12,825 | 565 KB | ~141K   |
> | Shared Patterns           | 16       | 3,032  | 156 KB | ~39K    |
> | Standards/Docs            | 29       | 23,728 | 845 KB | ~211K   |
>
> Command: `ring:dev-cycle` | Team: `dev-team`
> Location: `/ring/dev-team/skills/dev-cycle/SKILL.md` (3,394 lines)
> Entry: `/ring/dev-team/commands/dev-cycle.md`

---

## Execution Flow (10 Gates)

```
INIT → Gate 0 → 0.5 → 1 → 2 → 3 → 4 → 5 → 6(write) → 7(write) → 8 → 9 → POST-CYCLE
```

---

## Gates → Skills → Agents

| Gate | Name                  | Skill                            | Agent(s)                                                                                       |
| ---- | --------------------- | -------------------------------- | ---------------------------------------------------------------------------------------------- |
| Init | Codebase Analysis     | —                                | `ring:codebase-explorer` (prompt-only mode)                                                    |
| 0    | Implementation        | `ring:dev-implementation`        | `ring:backend-engineer-golang` / `ring:backend-engineer-typescript` / `ring:frontend-engineer` |
| 0.5  | Delivery Verification | `ring:dev-delivery-verification` | — (verification only)                                                                          |
| 1    | DevOps                | `ring:dev-devops`                | `ring:devops-engineer`                                                                         |
| 2    | SRE/Observability     | `ring:dev-sre`                   | `ring:sre`                                                                                     |
| 3    | Unit Testing          | `ring:dev-unit-testing`          | `ring:qa-analyst` (test_mode: "unit")                                                          |
| 4    | Fuzz Testing          | `ring:dev-fuzz-testing`          | `ring:qa-analyst` (test_mode: "fuzz")                                                          |
| 5    | Property Testing      | `ring:dev-property-testing`      | `ring:qa-analyst` (test_mode: "property")                                                      |
| 6    | Integration Testing   | `ring:dev-integration-testing`   | `ring:qa-analyst` (test_mode: "integration") — **write only, deferred execution**              |
| 7    | Chaos Testing         | `ring:dev-chaos-testing`         | `ring:qa-analyst` (test_mode: "chaos") — **write only, deferred execution**                    |
| 8    | Code Review           | `ring:requesting-code-review`    | 6 reviewers in parallel (see below)                                                            |
| 9    | Validation            | `ring:dev-validation`            | — (verification only)                                                                          |
| Post | Multi-Tenant          | `ring:dev-multi-tenant`          | Internal (runs own Gates 0-9)                                                                  |
| Post | Feedback              | `ring:dev-feedback-loop`         | — (metrics & report)                                                                           |

---

## Gate 8 - Code Review (6 Parallel Reviewers)

| Reviewer Agent                 | Focus          |
| ------------------------------ | -------------- |
| `ring:code-reviewer`           | Code quality   |
| `ring:business-logic-reviewer` | Business logic |
| `ring:security-reviewer`       | Security       |
| `ring:nil-safety-reviewer`     | Nil safety     |
| `ring:test-reviewer`           | Test quality   |
| `ring:consequences-reviewer`   | Side effects   |

All 6 must pass. 5/6 = FAIL. Max 3 retry iterations.

---

## Agents (All Files)

| Agent                              | File                                                  |
| ---------------------------------- | ----------------------------------------------------- |
| `backend-engineer-golang`          | `dev-team/agents/backend-engineer-golang.md`          |
| `backend-engineer-typescript`      | `dev-team/agents/backend-engineer-typescript.md`      |
| `frontend-engineer`                | `dev-team/agents/frontend-engineer.md`                |
| `devops-engineer`                  | `dev-team/agents/devops-engineer.md`                  |
| `sre`                              | `dev-team/agents/sre.md`                              |
| `qa-analyst`                       | `dev-team/agents/qa-analyst.md`                       |
| `qa-analyst-frontend`              | `dev-team/agents/qa-analyst-frontend.md`              |
| `frontend-designer`                | `dev-team/agents/frontend-designer.md`                |
| `frontend-bff-engineer-typescript` | `dev-team/agents/frontend-bff-engineer-typescript.md` |
| `ui-engineer`                      | `dev-team/agents/ui-engineer.md`                      |
| `prompt-quality-reviewer`          | `dev-team/agents/prompt-quality-reviewer.md`          |

---

## Skills (Sub-skills loaded via `Skill()`)

| Skill                            | File                                                 |
| -------------------------------- | ---------------------------------------------------- |
| `ring:dev-implementation`        | `dev-team/skills/dev-implementation/SKILL.md`        |
| `ring:dev-delivery-verification` | `dev-team/skills/dev-delivery-verification/SKILL.md` |
| `ring:dev-devops`                | `dev-team/skills/dev-devops/SKILL.md`                |
| `ring:dev-sre`                   | `dev-team/skills/dev-sre/SKILL.md`                   |
| `ring:dev-unit-testing`          | `dev-team/skills/dev-unit-testing/SKILL.md`          |
| `ring:dev-fuzz-testing`          | `dev-team/skills/dev-fuzz-testing/SKILL.md`          |
| `ring:dev-property-testing`      | `dev-team/skills/dev-property-testing/SKILL.md`      |
| `ring:dev-integration-testing`   | `dev-team/skills/dev-integration-testing/SKILL.md`   |
| `ring:dev-chaos-testing`         | `dev-team/skills/dev-chaos-testing/SKILL.md`         |
| `ring:requesting-code-review`    | `dev-team/skills/requesting-code-review/SKILL.md`    |
| `ring:dev-validation`            | `dev-team/skills/dev-validation/SKILL.md`            |
| `ring:dev-multi-tenant`          | `dev-team/skills/dev-multi-tenant/SKILL.md`          |
| `ring:dev-feedback-loop`         | `dev-team/skills/dev-feedback-loop/SKILL.md`         |

---

## Shared References

| Pattern               | File                                                    |
| --------------------- | ------------------------------------------------------- |
| File Size Enforcement | `dev-team/shared/references/file-size-enforcement.md`   |

---

## Standards/Docs (References)

| Standard           | File                                             |
| ------------------ | ------------------------------------------------ |
| Golang Core        | `dev-team/docs/standards/golang/core.md`         |
| Golang Domain      | `dev-team/docs/standards/golang/domain.md`       |
| DevOps             | `dev-team/docs/standards/devops.md`              |
| Multi-Tenant       | `dev-team/docs/standards/golang/multi-tenant.md` |
| Testing (multiple) | `dev-team/docs/standards/golang/testing-*.md`    |

External: `https://raw.githubusercontent.com/LerianStudio/ring/main/CLAUDE.md`

---

## Invocation

```
/ring:dev-cycle [tasks-file] [prompt]        # With tasks file
/ring:dev-cycle "prompt"                      # Prompt-only mode
/ring:dev-cycle --resume                      # Resume interrupted cycle
/ring:dev-cycle tasks.md --task AUTH-001      # Specific task
/ring:dev-cycle tasks.md --dry-run            # Validate only
```

---

## State Management

| Cycle Type | State Path                                  |
| ---------- | ------------------------------------------- |
| Feature    | `docs/ring:dev-cycle/current-cycle.json`    |
| Refactor   | `docs/ring:dev-refactor/current-cycle.json` |

---

## Hard Gates (Enforcement)

1. All 11 gates are sequential and mandatory — no skipping
2. Sub-skill must be loaded via `Skill()` before `Task()` dispatch
3. 85%+ code coverage required at Gate 3 (no exceptions)
4. All 6 reviewers must pass at Gate 8
5. Gates 6-7 tests are deferred — written per unit, executed once at end of cycle
6. Multi-tenant adaptation mandatory after all gates
7. Orchestrator cannot read/write/edit source code directly
8. If task requires >3 files or any source code → must dispatch specialist agent

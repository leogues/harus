---
name: harus:dev-validation
description: Gate 6 of the development cycle. Final user validation requiring explicit APPROVED or REJECTED decision before completion.
metadata:
  gate: 6
  sequence_after: harus:dev-review
  related: [harus:dev-cycle]
---

# User Validation (Gate 6)

This gate requires **explicit user approval**. No agent can self-approve.

**Core principle:** Passing tests and code review DO NOT guarantee requirements are met. Only the user can confirm the implementation matches their intent.

| Who            | Responsibility                                              |
| -------------- | ----------------------------------------------------------- |
| **This Skill** | Collect evidence, present checklist, enforce decision format |
| **User**       | Review evidence, respond APPROVED or REJECTED               |

## Step 1: Validate Input

Required from `harus:dev-cycle` orchestrator:

| Field               | Required | Description                          |
| ------------------- | -------- | ------------------------------------ |
| unit_id             | yes      | Task/subtask being validated         |
| acceptance_criteria | yes      | List of ACs to validate              |

If any required field is missing → STOP and return error to orchestrator.

## Step 2: Collect Evidence

For each acceptance criterion, gather:

| Evidence Type | Source                        |
| ------------- | ----------------------------- |
| Test results  | Gate 3 unit test output       |
| Coverage      | Gate 3 coverage report        |
| Review        | Gate 5 review verdict         |
| Logs/Metrics  | Runtime verification          |

### Evidence Quality

| Quality    | Examples                                              | Action           |
| ---------- | ----------------------------------------------------- | ---------------- |
| **Strong** | Test output with pass/fail, coverage %, review verdict | Present as-is    |
| **Weak**   | "Tests were written", "Code was reviewed"             | Gather actual data |
| **None**   | No evidence for an AC                                 | BLOCK — cannot validate without evidence |

⛔ **Do not present weak or missing evidence.** Go back and collect real data before presenting to user.

## Step 3: Present Validation Request

Use template: [validation-request-template.md](assets/validation-request-template.md)

⛔ **STOP all work** after presenting. Wait for explicit response.

## Step 4: Handle Decision

### Valid Responses

| Response               | Status    | Action                              |
| ---------------------- | --------- | ----------------------------------- |
| "APPROVED"             | ✅ VALID  | Document approval, complete task     |
| "REJECTED: [reason]"   | ✅ VALID  | Document rejection, return to Gate 0 |

### Invalid Responses — Ask Again

| Response                    | Status        | Action                              |
| --------------------------- | ------------- | ----------------------------------- |
| "Looks good" / "Sure"      | ❌ AMBIGUOUS  | Ask for explicit APPROVED/REJECTED   |
| "👍" / "✅"                | ❌ AMBIGUOUS  | Emojis are not formal approval       |
| "APPROVED if X"            | ❌ CONDITIONAL| Not approved until X is verified     |
| "APPROVED with caveats"    | ❌ CONDITIONAL| List caveats, verify, then re-ask    |
| "APPROVED but fix Y later" | ❌ CONDITIONAL| Y must be fixed NOW or REJECTED      |

### Partial Validation

⛔ **Do not partially approve.** If 4 out of 5 ACs are met, the task is NOT approved. All acceptance criteria must be satisfied for APPROVED.

## Step 5: Prepare Output

Use template: [output-template.md](assets/output-template.md)

## Edge Cases

| Scenario                | Action                                              |
| ----------------------- | --------------------------------------------------- |
| User unavailable        | Document pending status, WAIT — do not proceed      |
| Criterion is ambiguous  | Ask user to clarify BEFORE validating                |
| New requirements emerge | Do NOT add to current task — create separate task    |
| User changes mind       | Latest explicit decision wins — document change      |

## Self-Approval Prohibition

<forbidden>

- Same agent approving code it implemented
- Role switching to self-approve
- Interpreting silence as approval
- Proceeding without explicit APPROVED/REJECTED

</forbidden>

The agent that implemented code CANNOT approve it. Only user approval is valid.

## Blocker Criteria

<block_condition>

- User unavailable (document pending, WAIT)
- Ambiguous response (ask for explicit decision)
- Any CRITICAL/HIGH acceptance criterion NOT MET
- Weak or missing evidence for any AC

</block_condition>

## Severity Calibration

| Severity     | Criteria                        | Examples                                      |
| ------------ | ------------------------------- | --------------------------------------------- |
| **CRITICAL** | AC completely unmet             | Login doesn't work at all                      |
| **HIGH**     | AC partially met or degraded    | Response 800ms vs required 200ms               |
| **MEDIUM**   | Edge case failure               | Works for happy path, fails on empty input     |
| **LOW**      | Quality issue, technically met  | Code works but hard to maintain                |

## Anti-Rationalization

| Rationalization                    | Required Action                                  |
| ---------------------------------- | ------------------------------------------------ |
| "Tests pass, auto-approve"        | **Passing tests ≠ user approval**                 |
| "User delegated to me"            | **Implementer CANNOT self-approve**               |
| "Continue while waiting"          | **STOP all work — validation blocks everything**  |
| "Silence means approval"          | **Silence = no decision — WAIT**                  |
| "4 out of 5 ACs pass"            | **Partial = NOT approved — all ACs required**     |

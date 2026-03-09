# State Schema: current-cycle.json

Full schema for the cycle state file. See SKILL.md for state path selection rules.

## Complete Structure

```json
{
  "version": "1.0.0",
  "cycle_id": "uuid",
  "started_at": "ISO timestamp",
  "updated_at": "ISO timestamp",
  "source_file": "path/to/tasks.md",
  "state_path": "docs/harus:dev-cycle/current-cycle.json",
  "cycle_type": "feature | refactor",
  "execution_mode": "manual_per_subtask | manual_per_task | automatic",
  "custom_prompt": "optional user context, max 500 chars",
  "status": "in_progress | completed | failed | paused",
  "current_task_index": 0,
  "current_gate": 0,
  "current_subtask_index": 0,
  "tasks": [
    {
      "id": "T-001",
      "title": "Task title",
      "status": "pending | in_progress | completed | failed | blocked",
      "subtasks": [
        {
          "id": "ST-001-01",
          "file": "subtasks/T-001/ST-001-01.md",
          "status": "pending | completed"
        }
      ],
      "gate_progress": {
        "implementation": {
          "status": "pending",
          "tdd_red": {
            "status": "pending | in_progress | completed",
            "test_file": "path/to/test_file",
            "failure_output": "expected failure output",
            "completed_at": "ISO timestamp"
          },
          "tdd_green": {
            "status": "pending | in_progress | completed",
            "implementation_file": "path/to/impl",
            "test_pass_output": "PASS output",
            "completed_at": "ISO timestamp"
          }
        },
        "devops": { "status": "pending" },
        "sre": { "status": "pending" },
        "unit_testing": { "status": "pending" },
        "integration_testing": {
          "status": "pending",
          "scenarios_tested": 0,
          "tests_passed": 0,
          "tests_failed": 0,
          "flaky_tests_detected": 0
        },
        "review": { "status": "pending" },
        "validation": { "status": "pending" }
      },
      "agent_outputs": {
        "implementation": {
          "agent": "agent-name",
          "output": "summary",
          "timestamp": "ISO timestamp",
          "duration_ms": 0,
          "iterations": 1,
          "files_created": [],
          "files_modified": [],
          "commit_sha": "abc123"
        },
        "devops": { "agent": "...", "output": "...", "artifacts_created": [] },
        "sre": { "agent": "...", "output": "...", "validation_errors": [] },
        "unit_testing": {
          "agent": "...",
          "verdict": "PASS | FAIL",
          "coverage_actual": 87.5,
          "coverage_threshold": 85,
          "failures": []
        },
        "integration_testing": {
          "agent": "...",
          "verdict": "PASS | FAIL",
          "scenarios_tested": 0,
          "containers_used": []
        },
        "review": {
          "iterations": 1,
          "reviewers": {
            "code_reviewer": { "verdict": "PASS | FAIL", "issues": [] },
            "security_reviewer": { "verdict": "PASS | FAIL", "issues": [] }
          }
        },
        "validation": { "result": "approved | rejected" }
      }
    }
  ],
  "metrics": {
    "total_duration_ms": 0,
    "gate_durations": {},
    "review_iterations": 0,
    "testing_iterations": 0
  }
}
```

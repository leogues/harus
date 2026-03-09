# State Schema: current-cycle.json

Cycle persistence and resume state. Each gate appends its own outputs to `agent_outputs` when it executes.

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
        { "id": "ST-001-01", "file": "subtasks/T-001/ST-001-01.md", "status": "pending | completed" }
      ],
      "gate_progress": {
        "implementation": { "status": "pending" },
        "devops": { "status": "pending" },
        "sre": { "status": "pending" },
        "unit_testing": { "status": "pending" },
        "fuzz_testing": { "status": "pending" },
        "property_testing": { "status": "pending" },
        "integration_testing": { "status": "pending" },
        "chaos_testing": { "status": "pending" },
        "review": { "status": "pending" },
        "validation": { "status": "pending" }
      },
      "agent_outputs": {}
    }
  ]
}
```

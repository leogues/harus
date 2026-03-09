# Template: Execution Mode Selection

```javascript
AskUserQuestion({
  questions: [
    {
      question: "Select execution mode for this cycle",
      header: "Execution Mode",
      multiSelect: false,
      options: [
        {
          label: "Manual per subtask",
          description: "Checkpoint after each subtask",
        },
        {
          label: "Manual per task",
          description: "Checkpoint after each task (all gates run, then pause)",
        },
        {
          label: "Automatic",
          description: "No checkpoints, run all gates continuously",
        },
      ],
    },
  ],
});
```

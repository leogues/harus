# Template: Commit Plan Confirmation

```javascript
AskUserQuestion({
  questions: [
    {
      question:
        "I've analyzed your changes and propose this commit plan. How should I proceed?",
      header: "Commit Plan",
      multiSelect: false,
      options: [
        { label: "Execute plan", description: "Create X commits as proposed" },
        {
          label: "Single commit",
          description: "Combine all changes into one commit",
        },
        {
          label: "Let me review",
          description: "Show details before proceeding",
        },
      ],
    },
  ],
});
```

If user selects "Let me review", show the full plan with files per commit.

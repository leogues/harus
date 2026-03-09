# Template: Offer Push

```javascript
AskUserQuestion({
  questions: [
    {
      question: "Push commit to remote?",
      header: "Push",
      multiSelect: false,
      options: [
        { label: "Yes", description: "Push to current branch" },
        { label: "No", description: "Keep local only" },
      ],
    },
  ],
});
```

If user selects "Yes":

```bash
git push
```

If branch has no upstream, use:

```bash
git push -u origin <current-branch>
```

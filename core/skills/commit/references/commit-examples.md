# Example: Multi-Commit Sequence

When changes span multiple concerns, execute in sequence:

```bash
# Commit 1: Dependencies first
git add package.json package-lock.json
git commit -S \
  -m "chore(deps): update authentication dependencies" \
  --trailer "X-Harus-Ref: 0x1"

# Commit 2: Feature implementation with tests
git add src/auth/oauth.ts src/auth/oauth.test.ts
git commit -S \
  -m "feat(auth): add OAuth2 refresh token support" \
  -m "Implements automatic token refresh when access token expires." \
  --trailer "X-Harus-Ref: 0x1"
```

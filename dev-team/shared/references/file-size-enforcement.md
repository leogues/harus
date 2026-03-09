# File Size Enforcement (MANDATORY)

Max 300 lines per source file. This is a **HARD GATE**.

## Thresholds

| Lines | Action |
|-------|--------|
| ≤ 300 | ✅ Compliant |
| 301-500 | ⚠️ Must split before proceeding (loop back) |
| > 500 | ❌ HARD BLOCK — must split before gate can pass |

Applies to ALL source files (`*.go`, `*.ts`, `*.tsx`) including test files. Auto-generated files (protobuf, mocks, `*.pb.go`, `*.gen.ts`, `*.d.ts`) are exempt.

## Verification Commands

```bash
# Go (excludes mocks, generated, protobuf)
find . -name "*.go" ! -path "*/mocks*" ! -path "*/generated/*" ! -path "*/gen/*" ! -name "*.pb.go" ! -name "*.gen.go" \
  -exec wc -l {} + | awk '$1 > 300 && $NF != "total" {print}' | sort -rn

# TypeScript (excludes node_modules, dist, generated, declarations)
find . \( -name "*.ts" -o -name "*.tsx" \) ! -path "*/node_modules/*" ! -path "*/dist/*" ! -path "*/build/*" \
  ! -path "*/generated/*" ! -path "*/__mocks__/*" ! -name "*.d.ts" ! -name "*.gen.ts" \
  -exec wc -l {} + | awk '$1 > 300 && $NF != "total" {print}' | sort -rn
```

## Split Strategy

Split by **responsibility boundaries**, not arbitrary line counts.

**Go:** split files stay in same package, methods on same receiver. Verify with `go build ./... && go test ./...`

**TypeScript:** split files stay in same module/directory, update barrel exports if needed. Verify with `tsc --noEmit && npm test`

Test files split to mirror source files.

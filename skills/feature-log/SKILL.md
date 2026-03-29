---
name: feature-log
description: "Show a summary of all harness feature runs and debug runs in the current project. Displays status (PASS/FAIL/in-progress), dates, and one-line summaries. Trigger on: 'show features', 'what has the harness built', 'feature history', 'list features', '/feature-log'."
user-invocable: true
---

# Feature Log

Scans `.features/` and `.debug/` in the current project and renders a summary of all harness runs.

## Your Role

No agents. Read the relevant files yourself and render the output directly to the user.

## Steps

1. Find all subdirectories under `.features/` (if the directory exists).
2. Find all subdirectories under `.debug/` (if the directory exists).
3. For each `.features/{name}/` directory:
   - If `FEATURE_PLAN.md` exists: read it to extract the one-line summary and verdict (PASS/FAIL). Get the date from the file's modification time or the "Result" section.
   - If only `FEATURE_SPEC.md` exists (no `FEATURE_PLAN.md`): status is "In progress". Read the one-line summary from `FEATURE_SPEC.md`.
   - If neither exists: skip.
4. For each `.debug/{name}/` directory:
   - If `DIAGNOSIS.md` exists and the fix was applied: status is "Fixed" or "Failed". Read `DIAGNOSIS.md` for the one-line root cause.
   - If only `REPRODUCTION.md` exists: status is "In progress".
   - If neither exists: skip.
5. Render a table. Sort by most recent first. Use these status indicators:
   - `✓ PASS` — feature completed, all criteria passed
   - `✗ FAIL` — feature run completed, evaluation failed
   - `⏳ In progress` — run started but not complete
   - `🐛 Fixed` — debug run, fix applied and verified
   - `🐛 Unresolved` — debug run, fix failed or incomplete

### Output format

```markdown
## Feature Log

| Name | Type | Status | Date | Summary |
|------|------|--------|------|---------|
| feat-dark-mode | feature | ✓ PASS | 2026-03-28 | Dark mode toggle in settings page |
| debug-null-ptr | debug | 🐛 Fixed | 2026-03-27 | Null pointer in user auth middleware |
| feat-search-bar | feature | ⏳ In progress | — | Real-time product list filter |
```

If no `.features/` or `.debug/` directories exist, tell the user:
> No harness runs found in this project. Run `/harness` to build a feature or `/debug` to fix a bug.

## Tips for extracting data

- One-line summary in `FEATURE_SPEC.md` is the first non-blank line after `## Summary` or the first line of the spec body.
- Verdict in `FEATURE_PLAN.md` is in the `## Result` section: look for `Verdict: PASS` or `Verdict: FAIL`.
- For dates, use Bash `stat -f "%Sm" -t "%Y-%m-%d" <file>` (macOS) or `stat -c "%y" <file>` (Linux) on `FEATURE_PLAN.md`.

When producing a report (review, audit, research, compatibility check), apply this structure:

## Header

Start with a single-line title, then key metadata on separate lines:

```
# [Report Title]

**Scope:** `path/audited`  |  **Files:** 42  |  **Findings:** 2 ❌  3 ⚠️  1 💡
```

## Findings

Use severity indicators from `dx-report-format` (❌ ⚠️ 💡 ℹ️ ✅).

Present each finding as a compact block:

```
❌ **SQL injection** — `src/Repository/UserRepository.php:42`
  Query built with string concatenation using `$username`.
  **Fix:** use a bound parameter via QueryBuilder.
```

Group by category when the report covers multiple audit areas. Within each category, sort worst first.

## Summary

End with a scannable recap:

```
## Summary

| Level | Count |
|-------|-------|
| ❌ Critical | 2 |
| ⚠️ Warning | 3 |
| 💡 Suggestion | 1 |
| ✅ OK | 8 |

**Verdict:** [one sentence: overall status and top priority action]
```

## Principles

- The report should be readable in under 30 seconds for someone who just wants the verdict
- Details are there for those who dig deeper, not forced on everyone
- No wall of text: if it's longer than a screen, add a TL;DR after the header

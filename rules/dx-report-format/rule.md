When producing a report (review, audit, research summary, compatibility check, or any output with findings), apply this format.

## Severity indicators

- ❌ Critical / Blocker
- ⚠️ High / Warning
- 💡 Medium / Suggestion
- ℹ️ Info / Low
- ✅ Positive / OK

Map severity words from skills and audits to these indicators automatically (e.g. "Critical" becomes ❌, "Warning" becomes ⚠️). Lead each finding with its indicator so the reader can scan quickly.

## Header

Single-line title, then key metadata on one line:

```
# [Report Title]

**Scope:** `path/audited`  |  **Files:** 42  |  **Findings:** 2 ❌  3 ⚠️  1 💡
```

## Findings

Present each finding as a compact block:

```
❌ **SQL injection** - `src/Repository/UserRepository.php:42`
  Query built with string concatenation using `$username`.
  **Fix:** use a bound parameter via QueryBuilder.
```

- Group by severity (worst first) or by category when the report covers multiple audit areas. Within each category, sort worst first.
- Keep each finding to 2-3 lines max: what, where, fix.
- Use code blocks for file paths, commands, and code snippets.
- Prefer tables or bullet lists over prose walls.

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

- The report should be readable in under 30 seconds for someone who just wants the verdict.
- Details are there for those who dig deeper, not forced on everyone.
- No wall of text: if it's longer than a screen, add a TL;DR after the header.

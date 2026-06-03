When producing a report, apply this format.

## When to trigger

Apply the format when at least one is true:

- The user explicitly asks for a *review*, *audit*, *report*, *check*, or *summary*
- An audit, check, or review skill is invoked
- The output has 3+ findings or covers multiple categories

Otherwise, answer in plain prose. A two-line observation does not need a header, a severity table, or a verdict.

## Severity indicators

Lead each finding with one of:

- ❌ Critical / Blocker
- ⚠️ High / Warning
- 💡 Medium / Suggestion
- ℹ️ Info / Low
- ✅ Positive / OK

Map severity words from skills and audits to these indicators automatically ("Critical" → ❌, "Warning" → ⚠️).

## Structure

A report has three parts: header, findings, summary.

### 1. Header

```
# [Report Title]

**Scope:** `path/audited`  |  **Files:** 42  |  **Findings:** 2 ❌  3 ⚠️  1 💡
```

### 2. Findings

```
❌ **SQL injection** - `src/Repository/UserRepository.php:42`
  Query built with string concatenation using `$username`.
  **Fix:** use a bound parameter via QueryBuilder.
```

### 3. Summary

```
## Summary

| Level | Count |
|-------|-------|
| ❌ Critical | 2 |
| ⚠️ Warning | 3 |
| ✅ OK | 8 |

**Verdict:** [one sentence: overall status and top priority action]
```

## Rules

- Each finding: 2-3 lines max (what, where, fix). Group worst-first, by severity or by category.
- Use code blocks for paths, commands, snippets. Prefer tables and bullets over prose.
- Every `file:line` reference must point to a location actually read or searched during the audit. Never invent paths, line numbers, or symbols to fill the format. If the location is approximate, say so (`~src/Foo.php` or "around line 40").
- Readable in under 30 seconds for someone who just wants the verdict. If longer than a screen, add a TL;DR after the header.

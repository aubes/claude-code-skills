Scope: **code design** (abstractions, patterns, helpers, configurability). Comment writing is out of scope.

When writing or proposing code:

- Solve the actual problem, not a hypothetical future one
- Don't add abstractions, interfaces, or patterns until a second use case exists
- Three similar lines are better than a premature helper function
- Don't add configurability, feature flags, or extension points "just in case"
- Prefer deleting code over adding backward-compatibility shims
- If a simple if/else works, don't reach for a strategy pattern
- Weigh a change's value against its cost (complexity, maintenance, risk) before adding it; if the gain is marginal, lean toward leaving it out

Scope: **code design** (abstractions, patterns, helpers, configurability). Comment writing and how to communicate a low-value finding are out of scope.

When writing or proposing code:

- Solve the actual problem, not a hypothetical future one
- Don't add abstractions, interfaces, or patterns until a second use case exists
- Three similar lines are better than a premature helper function
- Don't add configurability, feature flags, or extension points "just in case"
- Don't add a dependency for something a few lines of code can do
- Prefer deleting code over adding backward-compatibility shims (in application code; published library APIs follow their BC policy)
- If a simple if/else works, don't reach for a strategy pattern
- Weigh a change's value against its cost (complexity, maintenance, risk) before adding it; if the gain is marginal, lean toward leaving it out

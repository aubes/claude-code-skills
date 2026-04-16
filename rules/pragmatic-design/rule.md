When writing or proposing code:

- Solve the actual problem, not a hypothetical future one
- Don't add abstractions, interfaces, or patterns until a second use case exists
- Three similar lines are better than a premature helper function
- Don't add configurability, feature flags, or extension points "just in case"
- Prefer deleting code over adding backward-compatibility shims
- If a simple if/else works, don't reach for a strategy pattern
- Before proposing a change, weigh the value it brings against its cost (complexity, maintenance, risk). If the gain is marginal, mention it but flag it as low-value so the user decides
- The right amount of code is the minimum that satisfies the requirement

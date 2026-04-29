---
paths: ["**/*.md"]
---

When writing markdown documentation:

- Link to the official spec or vendor docs when referencing a standard (PSR, OWASP, RFC, W3C, SemVer...) or a non-mainstream concept. Skip it for mainstream basics (HTTP, JSON, Git, common language syntax)
- For acronyms or non-obvious concepts, add a one-sentence explanation via a markdown footnote `[^label]`
- Group `[ref]: url` definitions and `[^label]: ...` bodies at the bottom of the file. Never inline full URLs in the prose

Example:

```markdown
The bundle follows [PSR-12][psr12] and exposes a CQRS[^cqrs]-style API.

...

[^cqrs]: Command Query Responsibility Segregation: separate read and write models. See [Martin Fowler][cqrs].

[psr12]: https://www.php-fig.org/psr/psr-12/
[cqrs]: https://martinfowler.com/bliki/CQRS.html
```

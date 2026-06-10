---
paths: ["**/*.md"]
---

Scope: **markdown formatting** of citations and footnotes inside `.md` files. Whether or when to cite a source is out of scope.

When writing markdown documentation:

- Use reference-style links `[label][ref]` for citations; never inline full URLs in the prose (badge lines are exempt)
- For non-obvious acronyms or concepts, add a one-sentence explanation via a markdown footnote `[^label]`
- Group `[ref]: url` definitions and `[^label]: ...` bodies at the bottom of the file

Example:

```markdown
The bundle follows [PSR-12][psr12] and exposes a CQRS[^cqrs]-style API.

...

[^cqrs]: Command Query Responsibility Segregation: separate read and write models. See [Martin Fowler][cqrs].

[psr12]: https://www.php-fig.org/psr/psr-12/
[cqrs]: https://martinfowler.com/bliki/CQRS.html
```

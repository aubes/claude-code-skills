# Claude Code Skills & Rules

A handful of skills and rules I use with [Claude Code](https://claude.ai/code). Nothing fancy, just things I got tired of repeating.

Pick what you like, ignore the rest.

> ⚠️ Work in progress. Skills and rules are still being tuned. Expect breaking changes and rough edges.

## Skills vs Rules vs CLAUDE.md

- **Skills** load on demand when you type `/skill-name` or when Claude detects a match. Use them for workflows: audits, checklists, multi-step procedures.
- **Rules** (`.claude/rules/`) are always loaded into context. Use them for style and behavior guidelines that should apply throughout a session.
- **CLAUDE.md** is your own instruction file, per project or user-wide. Use it for project-specific context and to `@import` rules you want active.

## Skills

| Skill | Description |
|---|---|
| [symfony-frankenphp-check](skills/symfony-frankenphp-check/) | Review Symfony code for common FrankenPHP worker mode pitfalls |

### symfony-frankenphp-check

Reviews Symfony code against a catalog of known FrankenPHP worker mode pitfalls. It flags patterns that commonly leak state between requests in long-running PHP workers: mutable services, static state, resource leaks, Doctrine and Monolog reset issues, and known FrankenPHP CVEs.

Static scan only: it will not catch novel patterns, runtime-only bugs, or dynamic code. A clean report means no known pattern matched, not a compatibility guarantee.

See [`skills/symfony-frankenphp-check/`](skills/symfony-frankenphp-check/) for the full checklist, usage, and reference versions.

## Rules

| Rule | Category | Scope | Description |
|---|---|---|---|
| [concise-comments](rules/concise-comments.md) | Code | all | Short comments, don't explain the framework |
| [pragmatic-design](rules/pragmatic-design.md) | Code | all | No over-engineering, solve the actual problem |
| [objective-analysis](rules/objective-analysis.md) | Communication | all | Facts and trade-offs, not opinions |
| [reliable-information](rules/reliable-information.md) | Communication | all | Only state what you can verify |
| [casual-tone](rules/casual-tone.md) | Communication | all | Light humor, geek refs when it fits |
| [no-em-dash](rules/no-em-dash.md) | Communication | all | Ban the d'em dash |
| [precise-tech-terms](rules/precise-tech-terms.md) | Communication | all | Reserve loaded terms (push, deploy, merge...) for their exact technical operation |
| [dx-report-format](rules/dx-report-format.md) | Output | all | Report format: severity indicators, header, findings, summary table |
| [sourced-docs](rules/sourced-docs.md) | Output | `**/*.md` | Cite standards and non-mainstream concepts with reference-style links |
| [docker-compose-first](rules/docker-compose-first.md) | Tooling | all | Default to Docker Compose for commands, dependencies, and project setup |

Categories:
- **Code**: influences generated code (style, design)
- **Communication**: influences how Claude speaks with you (tone, accuracy, analysis)
- **Output**: shapes specific deliverables (reports, audits)
- **Tooling**: influences how Claude executes and tools the project (commands, services, setup)

Scope uses Claude Code's [path-scoped rules](https://code.claude.com/docs/en/memory#path-specific-rules) mechanism. `all` means the rule has no `paths:` frontmatter and loads unconditionally. Future rules could use globs like `**/*.php` or `src/api/**/*.ts` to load only when relevant.

## Install

### Skills

Replace `<skill-name>` with the skill you want (e.g. `symfony-frankenphp-check`).

```bash
cp -r skills/<skill-name> ~/.claude/skills/
```

Or symlink (keeps everything in sync with `git pull`):

```bash
git clone https://github.com/aubes/claude-code-skills.git
ln -s "$(pwd)/claude-code-skills/skills/<skill-name>" ~/.claude/skills/<skill-name>
```

### Rules

Replace `<rule-name>` with the rule you want (e.g. `concise-comments`).

```bash
# Global (all projects)
cp rules/<rule-name>.md ~/.claude/rules/

# Per project
cp rules/<rule-name>.md your-project/.claude/rules/
```

Or copy them all in one go:

```bash
mkdir -p ~/.claude/rules && cp rules/*.md ~/.claude/rules/
```

Rules are loaded automatically when present in `~/.claude/rules/` or `.claude/rules/` (Claude Code discovers `.md` files recursively).

## Contributing

**Skills:** create a dir in `skills/`, add a `SKILL.md` following the [skill format](https://code.claude.com/docs/en/skills), include a "Technologies covered" table.

**Rules:** add a `<name>.md` file directly in `rules/` (one rule per file, conforming to Claude Code's [`.claude/rules/`](https://code.claude.com/docs/en/memory#organize-rules-with-claude/rules/) convention).

## License

[MIT](LICENSE)

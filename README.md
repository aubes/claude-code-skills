# Claude Code Skills & Rules

A handful of skills and rules I use with [Claude Code](https://claude.ai/code). Nothing fancy, just things I got tired of repeating.

Pick what you like, ignore the rest.

## Skills vs Rules vs CLAUDE.md

- **Skills** load on demand when you type `/skill-name` or when Claude detects a match. Use them for workflows: audits, checklists, multi-step procedures.
- **Rules** (`.claude/rules/`) are always loaded into context. Use them for style and behavior guidelines that should apply throughout a session.
- **CLAUDE.md** is your own instruction file, per project or user-wide. Use it for project-specific context and to `@import` rules you want active.

## Skills

| Skill | Description |
|---|---|
| [symfony-frankenphp-check](skills/symfony-frankenphp-check/) | Audit Symfony code (app, bundle, component) for FrankenPHP worker mode compatibility |

## Rules

| Rule | Category | Scope | Description |
|---|---|---|---|
| [concise-comments](rules/concise-comments/) | Code | all | Short comments, don't explain the framework |
| [pragmatic-design](rules/pragmatic-design/) | Code | all | No over-engineering, solve the actual problem |
| [objective-analysis](rules/objective-analysis/) | Communication | all | Facts and trade-offs, not opinions |
| [reliable-information](rules/reliable-information/) | Communication | all | Only state what you can verify |
| [casual-tone](rules/casual-tone/) | Communication | all | Light humor, geek refs when it fits |
| [no-em-dash](rules/no-em-dash/) | Communication | all | Ban the d'em dash |
| [dx-report-format](rules/dx-report-format/) | Output | all | Report format: severity indicators, header, findings, summary table |

Categories:
- **Code**: influences generated code (style, design)
- **Communication**: influences how Claude speaks with you (tone, accuracy, analysis)
- **Output**: shapes specific deliverables (reports, audits)

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
cp -r rules/<rule-name> ~/.claude/rules/

# Per project
cp -r rules/<rule-name> your-project/.claude/rules/
```

Rules are loaded automatically when present in `~/.claude/rules/` or `.claude/rules/` (Claude Code discovers `.md` files recursively).

## Contributing

**Skills:** create a dir in `skills/`, add a `SKILL.md` following the [skill format](https://code.claude.com/docs/en/skills), include a "Technologies covered" table.

**Rules:** create a dir in `rules/`, add a `rule.md`.

## License

[MIT](LICENSE)

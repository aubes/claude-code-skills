# Claude Code Skills & Rules

A handful of skills and rules I use with [Claude Code](https://claude.ai/code). Nothing fancy, just things I got tired of repeating.

Pick what you like, ignore the rest.

## Skills

| Skill | Description |
|---|---|
| [frankenphp-compat-check](skills/frankenphp-compat-check/) | Check FrankenPHP worker mode compatibility for Symfony bundles |

## Rules

| Rule | Description |
|---|---|
| [concise-comments](rules/concise-comments/) | Short comments, don't explain the framework |
| [objective-analysis](rules/objective-analysis/) | Facts and trade-offs, not opinions |
| [pragmatic-design](rules/pragmatic-design/) | No over-engineering, solve the actual problem |
| [reliable-information](rules/reliable-information/) | Only state what you can verify |
| [casual-tone](rules/casual-tone/) | Light humor, geek refs when it fits |
| [no-em-dash](rules/no-em-dash/) | Ban the em dash |
| [dx-report-format](rules/dx-report-format/) | Color-coded severity indicators and scannable report format |
| [fancy-report](rules/fancy-report/) | Full report template: header, findings, summary table, 30-second readable |

## Install

### Skills

```bash
cp -r skills/frankenphp-compat-check ~/.claude/skills/
```

Or symlink:

```bash
git clone https://github.com/aubes/claude-code-skills.git
ln -s "$(pwd)/claude-code-skills/skills/frankenphp-compat-check" ~/.claude/skills/frankenphp-compat-check
```

### Rules

```bash
# Global (all projects)
cp rules/concise-comments/rule.md ~/.claude/rules/concise-comments.md

# Per project
cp rules/concise-comments/rule.md your-project/.claude/rules/concise-comments.md
```

Rules are loaded automatically when present in `~/.claude/rules/` or `.claude/rules/`.

## Contributing

**Skills:** create a dir in `skills/`, add a `SKILL.md` following the [skill format](https://docs.anthropic.com/en/docs/claude-code/skills), include a "Technologies covered" table.

**Rules:** create a dir in `rules/`, add a `rule.md`.

## License

[MIT](LICENSE)

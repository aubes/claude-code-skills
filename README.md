# Claude Code Skills

A collection of open-source, self-contained skills for [Claude Code](https://claude.ai/code).

Each skill is independent and can be installed on its own.

## Available skills

| Skill | Description |
|---|---|
| [frankenphp-compat-check](frankenphp-compat-check/) | Audit Symfony bundles/components for FrankenPHP worker mode compatibility |

## Installation

Copy a skill directory to your Claude Code skills folder:

```bash
cp -r frankenphp-compat-check ~/.claude/skills/
```

Or clone the repo and symlink:

```bash
git clone https://github.com/aubes/claude-code-skills.git
ln -s "$(pwd)/claude-code-skills/frankenphp-compat-check" ~/.claude/skills/frankenphp-compat-check
```

## Usage

Once installed, invoke a skill with its slash command in Claude Code:

```
/frankenphp-compat-check src/
```

## Design principles

- **Self-contained**: each skill works on its own, no dependency on other skills
- **Single purpose**: one skill = one well-defined audit or advisory scope
- **Composable**: skills can be combined via Claude Code agents for broader workflows

## Contributing

1. Each skill lives in its own directory with a `SKILL.md` file
2. Follow the [Claude Code skill format](https://docs.anthropic.com/en/docs/claude-code/skills)
3. Skills must be self-contained (no cross-skill dependency)
4. Include a "Technologies covered" section with reference versions

## License

[MIT](LICENSE)

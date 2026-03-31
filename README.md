# ephemaral-skills

[Claude Code](https://claude.com/claude-code) skills for [ephemaral](https://github.com/andremiguelc/ephemaral) verification.

Two skills, installed separately:

- **ephemaral** — run verification, write invariants, interpret results
- **ephemaral-parser** — parse functions into Aral-fn JSON (with or without a deterministic parser)

## Install

Copy each skill folder into `.claude/skills/`:

```bash
git clone https://github.com/andremiguelc/ephemaral-skills.git /tmp/ephemaral-skills
cp -r /tmp/ephemaral-skills/ephemaral .claude/skills/ephemaral
cp -r /tmp/ephemaral-skills/ephemaral-parser .claude/skills/ephemaral-parser
```

## Prerequisites

- [ephemaral](https://github.com/andremiguelc/ephemaral) binary (Lean 4 + Z3)
- [ts-to-ephemaral](https://github.com/andremiguelc/ts-to-ephemaral) (optional, for TypeScript)

## License

MIT

# ephemaral skill

[Claude Code](https://claude.com/claude-code) skill for [ephemaral](https://github.com/andremiguelc/ephemaral) verification.

One skill:

- **ephemaral** — hunt for counterexamples: write invariants, author the `.aral-fn.json` IR directly from the source, run the verifier, and interpret the result.

## Install

Copy the skill folder into `.claude/skills/`:

```bash
git clone https://github.com/andremiguelc/ephemaral-skill.git /tmp/ephemaral-skill
cp -r /tmp/ephemaral-skill/ephemaral .claude/skills/ephemaral
```

## Prerequisites

- [ephemaral](https://github.com/andremiguelc/ephemaral) binary (Lean 4 + Z3)

## License

MIT
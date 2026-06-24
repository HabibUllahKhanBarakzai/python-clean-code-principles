# Python Clean Code Principles

Two [Claude Code](https://claude.com/claude-code) skills (also usable as standalone reading) that capture a practical, framework-neutral standard for writing and structuring Python backend systems.

The principles apply to Django, FastAPI, Flask, CLIs, workers, libraries, and internal services — monoliths or modular services alike.

## Skills

| Skill | What it covers |
| --- | --- |
| [`writing-clean-python-code`](skills/writing-clean-python-code/SKILL.md) | Domain language, separating the write flow, normalizing input, typing for intent, domain values over primitives, early validation, first-class edge cases, precise persistence, safe side-effect ordering, and behavior-focused tests. |
| [`clean-python-architecture`](skills/clean-python-architecture/SKILL.md) | Organizing by domain capability, thin boundaries, typed read models, intentional data loading, integrations behind interfaces, sync vs. async work, action modules, typed problems, explicit dependencies, and putting tests near behavior. |

## Using these as Claude Code skills

Each skill is a directory containing a `SKILL.md` with YAML frontmatter (`name`, `description`). Claude loads a skill when a task matches its description.

Install for all your projects:

```bash
git clone https://github.com/HabibUllahKhanBarakzai/python-clean-code-principles.git
cp -r python-clean-code-principles/skills/* ~/.claude/skills/
```

Or scope them to a single project by copying into that project's `.claude/skills/` directory instead.

## License

Released under the [MIT License](LICENSE).

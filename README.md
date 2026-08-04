# Python Clean Code Principles

[Claude Code](https://claude.com/claude-code) skills (also usable as standalone reading) that capture a practical, framework-neutral standard for writing and structuring backend systems in Python — plus a TypeScript/Next.js sibling that applies the same instincts to the frontend.

The Python principles apply to Django, FastAPI, Flask, CLIs, workers, libraries, and internal services — monoliths or modular services alike.

## Skills

| Skill | What it covers |
| --- | --- |
| [`clean-python-code`](skills/clean-python-code/SKILL.md) | Domain language, separating the write flow, normalizing input, typing for intent, domain values over primitives, early validation, first-class edge cases, precise persistence, safe side-effect ordering, and behavior-focused tests. |
| [`clean-python-architecture`](skills/clean-python-architecture/SKILL.md) | Organizing by domain capability, thin boundaries, typed read models, intentional data loading, integrations behind interfaces, sync vs. async work, action modules, typed problems, explicit dependencies, and putting tests near behavior. |
| [`clean-nextjs-code`](skills/clean-nextjs-code/SKILL.md) | The TypeScript sibling: Result types instead of thrown errors, pure planner functions at I/O boundaries, typed const registries, the server/client boundary, component and styling conventions, dependency injection over module mocking in tests, and executable gates over written instructions. |

`clean-nextjs-code` is distilled from a real production codebase — [saleor/storefront](https://github.com/saleor/storefront) at commit `4efe332` (Next.js 16.2.9 / React 19.2 / Vitest 4) — and every rule points at a file in it. Rules specific to Next.js 16 Cache Components / PPR are marked `VERSION-BOUND` inline; the rest is framework-agnostic.

## Layout

The two Python skills are a single `SKILL.md` each. `clean-nextjs-code` follows the router pattern its own source repo teaches: a short always-loaded `SKILL.md` plus `rules/*.md` read one at a time, so a task only pulls in the context it needs.

```
skills/
├── clean-python-code/SKILL.md
├── clean-python-architecture/SKILL.md
└── clean-nextjs-code/
    ├── SKILL.md          # non-negotiables + task→rule table
    └── rules/            # 9 rule files, loaded on demand
```

## Using these as Claude Code skills

Each skill is a directory containing a `SKILL.md` with YAML frontmatter (`name`, `description`). Claude loads a skill when a task matches its description.

Install for all your projects:

```bash
git clone https://github.com/HabibUllahKhanBarakzai/python-clean-code-principles.git
cp -r python-clean-code-principles/skills/* ~/.claude/skills/
```

Or scope them to a single project by copying into that project's `.claude/skills/` directory instead.

The same directories work with [Codex](https://developers.openai.com/codex) — copy them into `~/.codex/skills/` as well.

## License

Released under the [MIT License](LICENSE).

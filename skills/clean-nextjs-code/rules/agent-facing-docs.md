---
name: agent-facing-docs
description: How to structure AGENTS.md, project skills, and ADRs so an AI agent loads the right context and nothing more — thin always-on router, on-demand rule files with frontmatter descriptions, quarantined compiled docs, and drift checks in CI. Read when writing or restructuring a repo's agent instructions, CLAUDE.md/AGENTS.md, or project skill.
---

# Agent-facing documentation

**Source:** `AGENTS.md` (root, ~90 lines), `skills/saleor-paper-storefront/` (SKILL.md + 31
`rules/*.md` + `references/` + `migrations/`), `docs/adr/`, `skills-lock.json`,
`scripts/paper-doctor.mjs`.

This repo treats agent instructions as a **product with a token budget**. That framing is the whole
lesson: context is scarce, so the always-on layer must be a router, and depth must be loadable on
demand.

## The three layers

| Layer | Size | Loaded |
| --- | --- | --- |
| Root `AGENTS.md` | ~90 lines | Always |
| `skills/<project>/SKILL.md` | ~150 lines | When the task touches the project's domain |
| `skills/<project>/rules/<task>.md` | 100–400 lines each | **One**, matching the task |
| Compiled `AGENTS.md` (all rules concatenated) | ~75k tokens | **Never by an agent** — humans/offline only, quarantined in `.cursorignore` |

The instruction is explicit and repeated in three places: *"Never load the compiled
`skills/…/AGENTS.md` (≈75k tokens). Read the **one** `rules/<task>.md` whose frontmatter
`description` matches your task."* Say it that bluntly.

## The always-on file is a router, not a manual

The root `AGENTS.md` contains only what's needed *every* turn:

1. **A one-paragraph identity** — what this is, what stack, what version.
2. **"How to get context (read in this order; stop when answered)"** — a numbered lookup order.
3. **A precedence rule** for conflicts between sources.
4. **An index** of skills/rules by category — names and one-liners, no content.
5. **Critical commands** — the `verify` gate, codegen, the healthcheck.
6. **Non-negotiable rules** — 5 items, numbered.
7. **A key-locations table** — where things live.

What it deliberately does *not* contain: explanations, code examples, rationale. Those live one
hop away.

The ordering instruction is the highest-value part, because it tells the agent when to *stop*
reading:

> 1. **Framework mechanics** → read the **version-matched** docs bundled at `node_modules/next/dist/docs/`.
>    This diverges from your training data — do not answer from memory.
> 2. **Project decisions** → read `SKILL.md`, then the **one** `rules/<task>.md` matching the task.
> 3. **API shape** (fields, enums, nullability) → grep the generated types. Don't restate the
>    schema from memory.

## Fight stale training data explicitly

The single most useful sentence in the whole setup:

> **This is NOT the Next.js you know.** This version has breaking changes — APIs, conventions, and
> file structure may all differ from your training data. Read the relevant guide in
> `node_modules/next/dist/docs/` before writing any code.

Point at the **version-matched docs on disk**, not the internet and not memory. Do the same for any
fast-moving dependency and for your own generated API types.

That block is machine-maintained and carries its own preservation instruction:

> **Keep this block, including in commits.** It is maintained by `next dev` for every agent that
> works here. If it appears as an uncommitted change, that is intentional — commit it as-is. Do not
> remove it to clean up a diff; it will be regenerated.

Anticipating that a tidy-minded agent will delete a block it doesn't recognize — and pre-empting it
in the block itself — is a pattern worth stealing for any generated file.

## Set precedence before the conflict happens

> Project rules are **authoritative on architecture**. Use the external skills only for
> micro-patterns *inside* an already-correct structure. **On any conflict, the project wins.**

Any repo with more than one guidance source (global CLAUDE.md, project AGENTS.md, vendor skills,
framework docs) needs one sentence like this, or an agent will silently pick the wrong one.

## Every rule file carries a routing description

```markdown
---
name: paper-architecture
description: Canonical Next.js 16 App Router stance: Server Components by default, Server Actions,
  Cache Components (PPR), BFF auth, two surfaces. Read first when unfamiliar with the codebase or
  making cross-cutting architectural changes.
---
```

The `description` is a **selector**, not a summary — it must contain the words someone would use
when they have this task. A frontmatter-presence check runs in CI so a rule can't be added without
one.

Inside a rule file, the structure that works: what it is → what it is **not** (with pointers to the
right file) → the stance → a decision table → the mental model as ASCII → **deliberate non-goals**
→ known divergences.

## Write the non-goals down

The highest-signal table in the whole skill is the one listing what *not* to do:

| Avoid | Use instead |
| --- | --- |
| Client-side GraphQL (urql, Apollo in browser) | Server helpers + Server Actions |
| `searchParams` / `cookies()` inside `"use cache"` | Dynamic islands in nested `Suspense` |
| Raw `cacheLife` / hand-rolled `cacheTag` strings | `applyCacheProfile` from the manifest |
| Storefront importing `@/checkout/*` | The session-bridge module |

An agent (or a new hire) reconstructing a pattern from first principles will regenerate exactly the
things you removed. Naming the rejected alternative *and its replacement* is what prevents that.
Also keep a "known divergences (accepted, deferred)" section — honesty about where the code doesn't
yet match the doc beats a doc that quietly lies.

## Prevent doc rot mechanically

- `ensure-rule-frontmatter.mjs --check` — every rule has `name` + `description`.
- `compile-agents.mjs --check` — the compiled doc matches `rules/`.
- Both are in `pnpm run verify`, so drift fails the same gate as a type error.
- `pnpm run doctor` verifies the setup itself: skills linked, external skills installed, docs in
  sync, the oversized compiled doc quarantined.
- External skills are pinned in a `skills-lock.json` and restored by a bootstrap script — agent
  context is a versioned dependency, not a wiki.

## ADRs for decisions an agent will otherwise re-litigate

`docs/adr/` holds numbered, dated, status-carrying records: routing structure, CMS-copy vs
code-owned strings, executable checks in the agent loop, translatable slugs. The format that makes
them useful rather than ceremonial:

- **Status + date + "Implementation: shipped/trialing"** at the top, updated as reality changes.
- **Context** that admits what's already good and what the real tension is.
- **Decision**, split into numbered sub-decisions.
- **Alternatives considered**, as a table with *why rejected* — this is the part that stops the
  same debate recurring.
- **Consequences**, positive *and* negative, with costs named.
- **Follow-up work**, with completed items struck through and dated.
- **Open questions**, marked as still-open vs resolved-since.

Rules link to ADRs (`→ [ADR 0002]`) so the "why" is one hop from the "how", and the rule file stays
short.

## Migrations for forks

A separate `migrations/` tree with a `manifest.json`, dated atomic upgrade prompts, and a
`paper-version.json` baseline at the repo root — plus trigger phrases ("upgrade Paper", "apply
migrations") so the agent knows when that tree, and not `rules/`, is the right place to look. Keep
"how to catch up" strictly separate from "how it works now."

## Anti-patterns

| Avoid | Instead |
| --- | --- |
| One 5,000-line always-on CLAUDE.md | Thin router + on-demand rule files |
| Rule files without frontmatter descriptions | A `description` written as a selector, checked in CI |
| "Read all the docs in `docs/`" | "Read the **one** file matching your task; stop when answered" |
| Assuming the agent knows the framework version | "This is NOT the X you know" + path to on-disk docs |
| Only documenting what to do | A non-goals table naming rejected alternatives |
| Docs that drift from code | Drift checks inside the same `verify` gate |
| Vendored agent docs copy-pasted | Pinned in a lockfile, restored by a bootstrap script |
| Decisions living in a PR thread | A numbered ADR with alternatives and consequences |

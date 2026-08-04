---
name: enforcement
description: Make the feedback loop executable — one composed verify command, deterministic gates vs advisory judgment calls, self-healing codegen prehooks, bespoke lint scripts with documented escape hatches, dead-code detection, and a deliberately thin pre-commit hook. Read when adding CI checks, lint rules, or asking "how do we stop this regressing".
---

# Executable checks over instructions

**Source:** `package.json` scripts, `scripts/check-design-tokens.mjs`,
`scripts/check-deprecated-product-variants.mjs`, `knip.config.ts`, `.lintstagedrc.js`,
`.cursor/hooks.json`, and **ADR 0003 — "Executable checks in the agent's feedback loop."**

The governing principle, stated in that ADR:

> **Instructions tell the agent what should be true. Executable checks tell it when it is wrong.**

Documentation that says "always use design tokens" is a wish. A script that fails the build on a
hex literal is a fact. Both have a place — the skill is knowing which is which.

## Split checks by whether they are deterministic

| Class | Examples | Where it lives |
| --- | --- | --- |
| **Deterministic gates** | `tsc --noEmit`, `eslint`, token lint, `vitest run`, docs-drift check | **Executable, on the agent's path** — auto-enforced |
| **Advisory / judgment** | unnecessary `"use client"`, LCP budget, client-JS weight, a11y review, PPR boundaries | **Prose-guided** — reviewed, never auto-failed |

Only deterministic checks get auto-enforced. Turning judgment calls into CI walls produces false
positives, and false positives destroy trust in the gate — after which people stop reading any of
it. This restraint is a feature, and the ADR argues for it explicitly.

## One command answers "am I done?"

```json
"verify":       "pnpm run docs:check && pnpm run lint:design-tokens && pnpm run lint:graphql-variants && pnpm run typecheck && pnpm run lint && pnpm run test:run",
"verify:quick": "pnpm run lint:design-tokens && pnpm run typecheck"
```

- **Fail-fast ordering: cheapest first.** Doc drift and the token scan take milliseconds; tests take
  the longest and run last.
- **A scoped subset** (`verify:quick`) exists for a tight styling loop, so nobody skips the gate to
  save 40 seconds.
- **`verify` mirrors CI.** If local and CI diverge, the local gate stops meaning anything. Keep them
  in sync deliberately — the ADR lists this as ongoing follow-up work, not a solved problem.
- **The docs are in the gate** (`docs:check`) so agent-facing guidance can't silently rot.

Then say it once, in the always-on agent file: *"`pnpm run verify` — the single 'am I done?' gate.
Iterate until green before declaring done."* No matrix to interpret.

## Make codegen impossible to get wrong

```json
"predev":       "pnpm run generate:all",
"prebuild":     "pnpm run generate:all",
"pretypecheck": "pnpm run generate:all"
```

Lifecycle prehooks mean a typecheck can never run against stale generated types. This deletes an
entire bug class ("forgot to regenerate") rather than documenting it. Generated directories are
listed as never-edit in the agent docs and ignored by the dead-code tool.

**Generalization:** when a check *can* be made self-healing, do that instead of adding a rule
telling people to run it.

## Bespoke lint scripts, when a real regression has a cheap signature

Two hand-written Node scripts, each maybe 60 lines, each guarding a rule ESLint can't express:

**1. Design tokens** (`scripts/check-design-tokens.mjs`) — fails on hex/`rgb()`/`hsl()` in
`src/ui/**/*.tsx`.

**2. Deprecated GraphQL field** (`scripts/check-deprecated-product-variants.mjs`) — forbids an
unpaginated field that "has caused multi-second PDP payloads on high-cardinality catalogs."

Both share a template worth copying verbatim:

```js
#!/usr/bin/env node
/**
 * Design-token guard (hard-fail gate).
 *
 * Fails if a UI component hardcodes a raw color (hex / rgb() / hsl()) instead of
 * using a design token from `src/styles/brand.css`. Tokens keep rebrands global.
 *
 * Scope: src/ui/**\/*.tsx component styling only. Excluded by design:
 *   - `.ts` files — color *data* (swatch hex from catalog) is not styling
 *   - tests / __fixtures__ — sample data, not UI
 *
 * Escape hatch: add `design-tokens-allow` in a comment on the same line for the rare
 * legitimate case (a third-party hex passed through, a dynamic swatch from data).
 *
 * Run: pnpm run lint:design-tokens
 */
```

The five elements:

1. **Why the rule exists** — the cost it prevents, in concrete terms.
2. **Exact scope**, and *why each exclusion is excluded*. "Color data is not styling" is the
   distinction that stops the script from being disabled wholesale the first time it false-positives.
3. **A documented per-line escape hatch.** A gate with no escape hatch gets deleted. One that
   requires typing `design-tokens-allow` on the line makes the exception visible in review.
4. **The command to run it.**
5. **An actionable failure message** naming the fix, not just the violation:

   ```
   ✖ Raw color literal(s) found in src/ui — use a design token (brand.css) instead.
     Add a "design-tokens-allow" comment to allow a rare exception.
   ```

Support a `--check` flag on any script with a fix mode, so the same script serves both the
developer (`docs:compile`) and CI (`docs:check`).

## Keep the human commit path thin — deliberately

```js
// .lintstagedrc.js
const config = {
  "*.{js,cjs,mjs,jsx,ts,cts,mts,tsx}": [buildEslintCommand],   // eslint --fix on staged files
  "*.*": "prettier --write --ignore-unknown",
};
```

Pre-commit is lint + format only. **No `tsc`, no tests.** This is an explicit ADR decision, with
the rejected alternative recorded: blocking every commit on the full suite "punishes human
prototyping/WIP commits." The asymmetry is the point — the *agent* runs `verify` and iterates; the
*human* can commit WIP; CI is the shared backstop.

## Agent-loop automation: conservative, fail-open, measured

A `stop` hook runs the fastest gate when a turn ends:

```json
{ "version": 1, "hooks": { "stop": [{ "command": ".cursor/hooks/verify-quick.sh", "timeout": 30 }] } }
```

Deliberate constraints, all documented: fires on `stop` (not on every file edit), runs **only** the
millisecond-scale check (`tsc` and `build` are too heavy per turn), is **fail-open** (nudges, never
blocks), is silent on a clean tree, and embeds the actionable next step in the nudge rather than
raw output. It is documented as a **trial** pending measured noise and latency — "a bad hook is
worse than none."

## Dead code

`knip` with explicit entry points (App Router conventions), generated dirs ignored, and
runtime-only dependencies allowlisted **with a comment each**:

```ts
ignoreDependencies: [
  "sharp",                              // Used by Next.js image optimization at runtime
  "graphql-tag",                        // Used by generated GraphQL code
  "@graphql-codegen/typescript",        // Used by codegen CLI
],
```

Every suppression carries its reason. An unexplained ignore entry is indistinguishable from a bug.

## Verify the tooling itself

`pnpm run doctor` checks that the *setup* is healthy — docs in sync, skills linked, the oversized
compiled doc quarantined, and (with `--env`) that required env vars exist. When a session feels
off, there's a command for "is my environment lying to me" instead of guesswork.

## Adopting this in a repo

1. Add `verify` composing the deterministic checks you already have, cheapest first.
2. Move codegen/build prerequisites into `pre*` lifecycle hooks.
3. Point the always-on agent file at `verify` — don't enumerate steps.
4. For each regression that has already cost you twice and has a cheap textual signature, write a
   ~60-line script with the five-element header above. Not before.
5. Keep pre-commit thin; keep judgment calls in prose; write down *why* for both.

## Anti-patterns

| Avoid | Instead |
| --- | --- |
| A prose matrix of "when to run what" | One composed `verify` command |
| Blocking every commit on the full suite | Thin pre-commit; agent/CI run the gate |
| Auto-failing judgment calls (LCP, a11y, "use client") | Guided review, deliberately unenforced |
| A lint rule with no escape hatch | Documented per-line allow comment |
| A gate whose failure output is a raw stack trace | Message naming the fix |
| Documenting "remember to regenerate" | A `pre*` hook that regenerates |
| An unexplained `ignore` entry | A one-line reason per suppression |
| Local checks that drift from CI | Mirror them, and say so in the ADR |

---
name: clean-nextjs-code
description: Write explicit, maintainable, production-safe TypeScript/React/Next.js using Result types instead of thrown errors, pure planner functions at I/O boundaries, typed const registries as single sources of truth, dependency injection over module mocking, and executable gates on the agent's path. Use when writing or reviewing Next.js App Router code, React components, data-fetching layers, error handling, caching, server/client boundaries, or frontend tests.
---

# Clean Next.js Code Principles

A practical standard for TypeScript frontend code that survives production. This is the TypeScript
sibling of `clean-python-code` — same instincts (domain language, early validation, typed data,
precise writes, behavior-focused tests), applied to React Server Components, Server Actions, and a
network-bound data layer.

Every rule here is derived from a real file in a real production codebase. Read the source when a
rule feels abstract.

## Provenance

Distilled from **[saleor/storefront](https://github.com/saleor/storefront)** ("Saleor Paper"),
read at commit `4efe332` (2026-07-31), running **Next.js 16.2.9 / React 19.2 / TypeScript 5.3 /
Vitest 4 / Playwright 1.55**.

Paths cited in the rules are repo-relative to that project
(`https://github.com/saleor/storefront/blob/main/<path>`).

**Two things to keep straight when applying this:**

1. **Durable vs version-bound.** Most of this is framework-agnostic and will outlive the framework
   (Result types, planners, registries, DI in tests). A subset is specific to Next.js 16 Cache
   Components / PPR and is **marked `VERSION-BOUND`** inline. Do not apply the version-bound rules
   to a Next 14/15 project — `use cache`, `cacheLife`, and PPR boundary rules do not exist or
   behave differently there.
2. **Shape, not domain.** Saleor is e-commerce. Cache profile names, variant caps, channel routing
   are illustration, never prescription. Copy the *shape* of the solution, not the commerce nouns.

## The Standard

Good frontend code makes failure modes obvious.

The network is the dominant source of bugs in a storefront, so the type system should force you to
handle it: **errors are values, not exceptions**. Every boundary that touches I/O returns a
discriminated union the compiler makes you narrow. Everything downstream of the boundary is a pure
function you can test without a server.

What makes this codebase feel like one mind wrote it is not any single technique — it is the same
small habits repeated everywhere: one Result type for every fetch, one planner function per
webhook, one registry per family of constants, one naming convention for files, one `verify`
command that says whether you're done. Each is cheap; applied uniformly, they compound.

## Non-negotiables

1. **Return Results, don't throw, at I/O boundaries.** `Promise<Result<T>>` with `ok: true | false`.
   Throw only for programmer error (a caller passed something impossible). → `rules/errors-and-results.md`
2. **Keep route handlers, Server Actions, and components thin.** The decision is a pure `plan*()`
   function returning a discriminated union; the handler executes the plan. → `rules/planners-and-boundaries.md`
3. **One typed registry per family of constants**, declared `as const satisfies Record<string, T>`,
   with derived helpers next to it. Never scatter magic strings. → `rules/typed-registries.md`
4. **Inject dependencies at the seam; don't `vi.mock` modules.** Only 6 of ~85 test files in the
   source use `vi.mock`. → `rules/testing.md`
5. **Server Components by default.** `"use client"` only for state, effects, event handlers, or
   browser APIs — and push it to the leaf. → `rules/server-client-boundary.md`
6. **`import "server-only"` on anything holding a secret**, plus a `.server.ts` filename suffix.
   → `rules/server-client-boundary.md`
7. **Handle nullable API fields intentionally.** Optional-chain for display; guard or throw when
   null is a real bug. Never `!` to silence the compiler.
8. **Style through a token/variant system**, never raw color literals or one-off class soup.
   → `rules/components-and-styling.md`
9. **Never render an unvalidated href.** Sanitize CMS/user URLs through one shared predicate and
   fail closed. → `rules/components-and-styling.md`
10. **One command answers "am I done?"** Deterministic checks are executable and on the agent's
    path; judgment calls stay prose. → `rules/enforcement.md`

## Task → rule

Read **one** rule file — the one matching the task. Don't load them all.

| Working on | Read |
| --- | --- |
| Data fetching, API clients, error handling, retries | `rules/errors-and-results.md` |
| Route handlers, Server Actions, webhooks, "where does this logic go" | `rules/planners-and-boundaries.md` |
| Config, constants, enums, manifests, feature matrices | `rules/typed-registries.md` |
| Writing or reviewing tests (unit, integration, e2e) | `rules/testing.md` |
| `"use client"`, `server-only`, Server Actions, auth, secrets | `rules/server-client-boundary.md` |
| Components, variants, styling, naming, a11y, links | `rules/components-and-styling.md` |
| Caching, revalidation, Suspense, PPR, page structure — **VERSION-BOUND** | `rules/caching-and-rendering.md` |
| Lint scripts, CI gates, codegen, "how do we stop this regressing" | `rules/enforcement.md` |
| Writing the AGENTS.md / skill docs for a repo | `rules/agent-facing-docs.md` |

These rules compose: a boundary rule (`errors-and-results`, `planners-and-boundaries`) usually
pairs with `typed-registries` when constants are involved. Read the boundary rule first.

## Comments

Comment the **why**, never the what. The source's comments are worth imitating because every one
answers a question the code can't:

```ts
// Browsers normalize backslashes to slashes, so `/\evil.com` and `//evil.com`
// are both treated as protocol-relative external URLs — reject them.

// An explicit but invalid override does **not** fall through — reporting the
// wrong ladder rung is worse for the Paper handshake than omitting it.

/** Clear the checkout cookie after a successful order.
 *  Never revalidates `/checkout` (that remounts the flow and resets the step mid-payment). */
```

Reserve comments for: browser/platform quirks, concurrency and ordering, deliberate deviations,
upstream API behavior, and "this looks wrong but here's why it isn't." A comment restating the
code is a defect.

Public helpers get a JSDoc line stating the contract, plus `@example` when the call shape isn't
obvious. Use `{@link OtherSymbol}` so editors can jump.

## Formatting baseline

Pick one and enforce it in `.prettierrc` + `lint-staged` so it never reaches review. The source
uses: tabs, double quotes, semicolons, `trailingComma: "all"`, `printWidth: 110`, plus
`prettier-plugin-tailwindcss` for deterministic class ordering.

TypeScript compiler settings that carry weight: `strict`, `noUnusedLocals`, `noUnusedParameters`,
`isolatedModules`, and path aliases (`@/*`) so no import ever reads `../../../`.
(The source leaves `noUncheckedIndexedAccess: false` — a deliberate ergonomics tradeoff, not an
endorsement. Turn it on in new projects.)

## Final self-check

Before declaring a change done:

- Does every I/O call site handle both `ok: true` and `ok: false`, or is a failure silently a crash?
- Is the decision logic a pure function I can test without a server, a browser, or a mock?
- Are new constants in a typed registry, or did I add a magic string?
- Does a test name read as a sentence about behavior — and does it fail if I invert the rule?
- Is `"use client"` at the leaf, or did it swallow a subtree that could have stayed server?
- Did I comment the non-obvious *why*, and delete comments that restate code?
- Did I run the repo's one `verify` command and iterate until green?

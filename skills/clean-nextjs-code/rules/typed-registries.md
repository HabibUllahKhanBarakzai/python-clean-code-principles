---
name: typed-registries
description: Declare families of constants as one typed const registry (as const satisfies Record<string, T>) with derived helpers, actionable throws, and versioned manifests instead of scattered magic strings. Read when adding config, feature flags, cache tags, route maps, event mappings, or any lookup table.
---

# Typed const registries

**Source:** `src/lib/cache-manifest.ts`, `src/config/variants.ts`, `src/ui/components/ui/button.tsx`.

Any time you have "a family of related constants, each with the same shape," it becomes one
registry object with a `satisfies` constraint — not scattered string literals, not a bare enum, not
a `Map` built at runtime.

## The declaration

```ts
export interface CacheProfile {
  readonly id: string;
  readonly label: string;
  /** Paper cacheLife tier — see src/lib/cache-life-profiles.ts */
  readonly cacheProfile: CacheLifeProfile;
  /** Tag pattern — use {slug} and/or {channel} placeholders */
  readonly tagPattern: string;
  /** Path pattern — use {channel} and {slug} as placeholders, or null for non-path caches */
  readonly pathPattern: string | null;
  /** Catch-all tag applied alongside the entity tag so a full purge needn't enumerate slugs. */
  readonly sharedTag?: string;
}

const profiles = {
  products:   { id: "products",   label: "Product Pages",  cacheProfile: "catalog", tagPattern: "product:{slug}",  pathPattern: "/{locale}/{channel}/products/{slug}", sharedTag: "products" },
  categories: { id: "categories", label: "Category Pages", cacheProfile: "catalog", tagPattern: "category:{slug}", pathPattern: "/{locale}/{channel}/categories/{slug}", sharedTag: "categories" },
  navigation: { id: "navigation", label: "Navigation Menus", cacheProfile: "menus", tagPattern: "navigation:{channel}", pathPattern: null },
} as const satisfies Record<string, CacheProfile>;

export const CACHE_PROFILES = profiles;
export const CACHE_PROFILE_LIST: readonly CacheProfile[] = Object.values(profiles);
```

Why `as const satisfies Record<string, T>` and not `: Record<string, T>`:

- `satisfies` **checks** each entry against the interface but **keeps the literal types**, so
  `CACHE_PROFILES.products.tagPattern` is the literal `"product:{slug}"`, and the object's keys are
  a closed union you can derive from.
- `as const` makes everything `readonly`, so nothing mutates the registry at runtime.
- A type *annotation* would widen all of that away.

Also worth copying: `readonly` on every interface field, and a JSDoc line on any field whose
meaning isn't obvious from the name (`sharedTag` earns three lines because the *why* is subtle).

Derive, don't duplicate:

```ts
export type StorefrontMenuSlug = keyof typeof STOREFRONT_MENU_SLUGS;

export function isKnownStorefrontMenuSlug(slug: string): slug is StorefrontMenuSlug {
  return slug in STOREFRONT_MENU_SLUGS;
}
```

The type guard makes the registry the *only* definition of "known" — no parallel array to drift.

## Keep the helpers next to the registry

`cache-manifest.ts` is one file holding the profiles plus every operation on them: `buildTag`,
`buildPath`, `buildPathsForAllLocales`, `applyCacheProfile`, the classification predicates
(`isGlobalTagProfile`, `isChannelScopedTagProfile`, …), the planners, and the manifest builder.
A header comment states who consumes it:

```ts
// ============================================================================
// Cache Profile Definitions — single source of truth
//
// Imported by:
//   - Cached functions (applyCacheProfile → cacheLife + cacheTag)
//   - Revalidation endpoint (revalidateTag tag + profile)
//   - /api/cache-info (manifest for dashboard)
// ============================================================================
```

Colocation is what makes "single source of truth" real. If the builders live elsewhere, someone
will hand-build a tag string.

Banner comments as section dividers within a long module (`// ==== Helpers for "use cache" ====`,
`// ==== Tag / path builders ====`) are used sparingly and only where a file genuinely has phases.
They are not a substitute for splitting a file that has grown two responsibilities.

## Throw actionably on programmer error

Results are for I/O. A template with an unfilled placeholder is a *bug*, and the throw should tell
the developer exactly what to pass:

```ts
if (UNRESOLVED_PLACEHOLDER.test(tag)) {
  const missing = (["{slug}", "{channel}", "{locale}"] as const).filter((p) => tag.includes(p));
  throw new Error(
    `[cache-manifest] Unresolved tag "${tag}" for profile "${profile.id}". ` +
    `Provide: ${missing.join(", ")}`,
  );
}
```

Three things every good throw has: a **`[module]` prefix** for grepping, the **offending value**,
and the **fix**. `throw new Error("invalid tag")` has none of them.

## Normalize permissive inputs at the registry's edge

Let callers pass the common case directly, and normalize inside:

```ts
function normalizeTagParams(params?: string | CacheTagParams): CacheTagParams {
  if (typeof params === "string") return { slug: params };
  return params ?? {};
}
```

One overload-ish convenience, normalized in one line, is fine. Five of them is an API smell.

## Environment resolution: explicit → platform → runtime, and don't fall through on invalid

```ts
/**
 * Map deploy hints to the environment ladder.
 * Prefer explicit `PAPER_STOREFRONT_ENVIRONMENT`, then Vercel, then NODE_ENV.
 *
 * An explicit but invalid override does **not** fall through — reporting the
 * wrong ladder rung is worse than omitting it.
 */
export function resolveStorefrontManifestEnvironment(env = process.env): StorefrontManifestEnvironment | undefined {
  const rawExplicit = env.PAPER_STOREFRONT_ENVIRONMENT?.trim();
  if (rawExplicit) {
    const explicit = rawExplicit.toLowerCase();
    if (isStorefrontManifestEnvironment(explicit)) return explicit;
    console.warn(`[cache-manifest] Ignoring invalid PAPER_STOREFRONT_ENVIRONMENT="${rawExplicit}". ` +
                 `Expected ${STOREFRONT_MANIFEST_ENVIRONMENTS.join("|")}.`);
    return undefined;
  }
  …
}
```

Two habits: **`env: NodeJS.ProcessEnv = process.env` as a defaulted parameter** (the function is
now pure and testable without stubbing globals), and **a deliberate, commented decision about the
invalid-explicit case**. That is edge-case modeling in the same spirit as the Python standard.

Note the sibling helper `isStorefrontManifestEnvironment` — a type guard derived from the same
`as const` tuple that produces the type. One list, two uses.

## Version your externally-consumed manifests

```ts
const MANIFEST_VERSION = 6;

export function buildManifest() {
  return { version: MANIFEST_VERSION, profiles: …, locales: …, channels: …, ...(identity ? { identity } : {}) };
}
```

Anything another system parses gets a version number, and optional blocks are spread conditionally
(`...(x ? { x } : {})`) rather than emitted as `undefined`.

## The variant matrix is a registry too

Style variants are the same pattern with a different library — `cva` gives you a typed matrix plus
derived types:

```ts
export const buttonVariants = cva(base, {
  variants: { variant: { default: …, secondary: …, ghost: … }, size: { default: …, sm: …, lg: …, icon: … } },
  defaultVariants: { variant: "default", size: "default" },
});

export type ButtonVariant = NonNullable<VariantProps<typeof buttonVariants>["variant"]>;
export type ButtonSize    = NonNullable<VariantProps<typeof buttonVariants>["size"]>;
```

Export the derived types. Consumers that accept a variant as a prop then can't drift from the
matrix.

## Anti-patterns

| Avoid | Instead |
| --- | --- |
| Magic strings repeated across files | One `as const satisfies` registry |
| `: Record<string, T>` annotation | `as const satisfies Record<string, T>` (keeps literals) |
| A parallel `const ALL_IDS = [...]` array | `Object.values(registry)` / `keyof typeof` |
| Hand-built tag/path strings at call sites | `buildTag()` / `buildPath()` from the registry |
| `throw new Error("invalid input")` | `[module]` + offending value + how to fix |
| Reading `process.env` deep inside logic | Defaulted `env` parameter at the resolver |
| Unversioned JSON consumed by another service | `version` field, bumped on shape change |
| Silently falling back on an invalid explicit override | Warn and return undefined, with the reason commented |

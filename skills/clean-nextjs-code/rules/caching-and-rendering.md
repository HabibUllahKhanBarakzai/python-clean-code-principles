---
name: caching-and-rendering
description: VERSION-BOUND to Next.js 16 Cache Components/PPR — page structure (sync export + Suspense shell + dynamic islands), never awaiting searchParams in cached shells, tag-based revalidation from a manifest, webhook invalidation, and metadata. Read when working on caching, revalidation, Suspense boundaries, or App Router page structure.
---

# Caching and rendering

> **⚠ VERSION-BOUND — Next.js 16.2.x (Cache Components / PPR).**
> `use cache`, `cacheLife`, and the static-shell/dynamic-island model described here do not exist,
> or behave differently, in Next 14/15. The **durable** parts — freshness tiers as a policy
> decision, tags from a manifest, invalidation planned by pure functions, sanitized metadata — port
> anywhere. The mechanics do not. Verify against the installed version's docs before applying.

**Source:** `src/lib/cache-manifest.ts`, `src/lib/cache-life-profiles.ts`,
`src/app/(storefront)/[locale]/[channel]/(main)/products/[slug]/page.tsx`,
`src/app/api/revalidate/`.

## The freshness split is a product decision, made once

| Data | Policy |
| --- | --- |
| Catalog, menus, CMS copy | Cached with `"use cache"`, invalidated by webhook tag |
| Cart, checkout, account, auth | Always fresh — `cache: "no-cache"` |

Write it down, then encode it. "Is this cached?" should never be a per-call-site judgment. The
source states the residual risk out loud: *"Prices may lag on cached PDPs until cart refresh"* —
naming the accepted staleness is part of the decision.

## Page shape: sync export → Suspense → cached shell → dynamic islands

```tsx
/**
 * Sync page entry — Suspense while params resolve and cached product data loads.
 * searchParams is passed through without being awaited here or in the shell.
 */
export default function ProductPage(props: {
  params: Promise<{ locale: string; slug: string; channel: string }>;
  searchParams: Promise<{ variant?: string; sku?: string }>;
}) {
  return (
    <Suspense fallback={<ProductRouteSkeleton surface="page" />}>
      <ProductShell params={props.params} searchParams={props.searchParams} />
    </Suspense>
  );
}
```

The rules that make this work:

1. **The page export is sync.** Awaiting in the page body makes the whole route dynamic.
2. **`searchParams` is threaded as a `Promise`, never awaited in the shell.** Awaiting it opts the
   shell out of the static path. Await it only in a nested island — or, for a rare branch, at the
   point of use, with a comment:

   ```ts
   // Only await searchParams on the rare non-canonical slug path so the common
   // (already-canonical) PDP shell stays params-only / PPR-static.
   if (decodeURIComponent(params.slug) !== pickTranslatedSlug(product)) {
     redirectToCanonicalCatalogSlug({ …, searchParams: await searchParams });
   }
   ```

3. **`cookies()` follows the same rule** — never in a cached shell; put it behind its own Suspense
   boundary.
4. **Dynamic islands are separately suspended** with their own skeleton *and* their own error
   boundary (`VariantSectionDynamic` / `VariantSectionSkeleton` / `VariantSectionError`). A failing
   island degrades one region, not the page.
5. **Don't wrap a page that has no dynamic hole in a page-level Suspense** — render the cached
   shell directly. A skeleton for content that was already static is a downgrade.

## Never hand-write a cache tag

One helper applies both lifetime and tags, from the registry:

```ts
export function applyCacheProfile(profile: CacheProfile, params?: string | CacheTagParams) {
  (cacheLife as (p: string) => void)(profile.cacheProfile);
  const entityTag = buildTag(profile, params);
  if (profile.sharedTag) cacheTag(entityTag, profile.sharedTag);
  else cacheTag(entityTag);
}
```

The `sharedTag` design is worth stealing regardless of framework: each entry carries **both** a
precise tag (`product:hoodie`) and a family tag (`products`), so a full purge invalidates the
family without enumerating every slug.

Tag/path patterns live in the registry with `{slug}` / `{channel}` / `{locale}` placeholders, and
`buildTag` throws with the list of missing placeholders if you forget one
(see `rules/typed-registries.md`).

## Webhook invalidation is a pure plan + a thin executor

The handler parses the payload, calls a planner, and switches on the result. All the logic is
testable with plain objects:

```ts
export function planPageRevalidation(slug, channels, fallbackChannel?): PageRevalidationPlan {
  if (!slug) return { action: "skip", reason: "missing_slug" };
  const channelList = channels.length > 0 ? channels : fallbackChannel ? [fallbackChannel] : [];
  if (channelList.length === 0) return { action: "error", reason: "no_channels" };
  const paths = channelList.flatMap((channel) => buildPathsForAllLocales(CACHE_PROFILES.pages, { channel, slug }));
  return { action: "revalidate", slug, tag: buildTag(CACHE_PROFILES.pages, slug), profile: CACHE_PROFILES.pages.cacheProfile, paths };
}
```

Payload extraction is its own defensive helper — external payloads are `unknown` until proven
otherwise:

```ts
export function extractPageSlugFromWebhookPayload(payload: unknown): string | null {
  if (!payload || typeof payload !== "object") return null;
  const data = payload as Record<string, unknown>;
  if (data.page && typeof data.page === "object") {
    const slug = (data.page as Record<string, unknown>).slug;
    if (typeof slug === "string" && slug.length > 0) return slug;
  }
  return null;
}
```

Fan out across every locale and channel — a cached page exists per locale×channel, so invalidating
one path leaves the others stale:

```ts
export function buildPathsForAllLocales(profile: CacheProfile, params: Omit<BuildPathParams, "locale">): string[] {
  return getStorefrontLocaleSlugs()
    .map((locale) => buildPath(profile, { ...params, locale }))
    .filter((path): path is string => path !== null);
}
```

(The `.filter((x): x is T => …)` type predicate instead of `as T[]` is a small habit worth keeping
everywhere.)

## Expose the cache model for operations

`/api/cache-info` serves a versioned manifest of profiles, tiers, locales, channels, and a build
identity block (`saleorApiUrl`, `environment`, `buildId`, `commit`, `branch`). When someone asks
"why is this page stale," the answer is a URL, not an archaeology session.

## Metadata

- Build metadata in `generateMetadata`, and give it a fallback for the not-found path
  (`return { title: "Product Not Found" }`) so a 404 doesn't render an empty tab title.
- Derive descriptions with a helper that has a documented fallback chain — the source's
  `resolveSeoDescription({ seoDescription, body, fallbackName })` exists so meta never collapses to
  the bare product name when the merchant left SEO fields empty.
- Comment framework quirks you had to work around, with the error code:
  *"Omit `openGraph.type` — Next rejects `product` (E237). ProductShell hoists the meta tag only
  after the product resolves (never on the 404 path)."*
- Document deliberate omissions: *"NOTE: `generateStaticParams` is intentionally omitted for
  product pages. All product pages are generated on-demand via ISR instead."* An absent thing
  can't explain itself.

## Anti-patterns

| Avoid | Instead |
| --- | --- |
| `async export default function Page()` | Sync page → `Suspense` → async shell |
| `await searchParams` / `cookies()` in a cached shell | Read them inside a nested dynamic island |
| Hand-written `cacheTag("product:" + slug)` | `applyCacheProfile(CACHE_PROFILES.products, slug)` |
| `cache: "no-cache"` on catalog display data | `"use cache"` + tag + webhook invalidation |
| Cached data on cart/checkout/account | Always fresh |
| Revalidating one locale's path | Fan out over all locales × channels |
| Webhook logic inline in `route.ts` | `plan*()` + thin executor |
| `payload.page.slug` on an untyped body | Defensive `unknown`-narrowing extractor |
| Page-level skeleton on a fully static route | Render the cached shell directly |

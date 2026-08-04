---
name: planners-and-boundaries
description: Keep route handlers, Server Actions, and components thin by extracting decisions into pure plan*() functions that return discriminated unions with named reason codes. Read when writing webhooks, API routes, Server Actions, or deciding where a piece of business logic belongs.
---

# Planners: pure decisions, thin boundaries

**Source:** `src/lib/cache-manifest.ts` (`planMenuRevalidation`, `planPageRevalidation`,
`planStorefrontContentRevalidation`), `src/app/actions.ts`, `src/lib/catalog/buy-box-strategy.ts`.

The single highest-leverage structural habit in this codebase: **the decision is a pure function;
the boundary only executes it.** A webhook handler is an HTTP shell around a function you can test
with a plain object in 4 lines.

## The pattern

```ts
export type MenuRevalidationPlan =
  | { action: "revalidate"; menuSlug: string; tags: Array<{ tag: string; profile: CacheLifeProfile }> }
  | { action: "skip";  reason: "missing_slug" | "unknown_menu" }
  | { action: "error"; reason: "no_channels" };

/** Pure planner for menu webhook invalidation — keeps route handler thin and testable. */
export function planMenuRevalidation(
  menuSlug: string | undefined,
  channels: readonly string[],
): MenuRevalidationPlan {
  if (!menuSlug) return { action: "skip", reason: "missing_slug" };
  if (channels.length === 0) return { action: "error", reason: "no_channels" };

  const tags = buildMenuRevalidationTags(menuSlug, channels);
  if (tags.length === 0) return { action: "skip", reason: "unknown_menu" };

  return { action: "revalidate", menuSlug, tags };
}
```

Four properties make this work:

1. **`action` is a closed set** — `"revalidate" | "skip" | "error"`. The handler `switch`es and the
   compiler enforces exhaustiveness.
2. **`reason` is a machine-readable code**, not a sentence. `"missing_slug"` is greppable,
   loggable, assertable, and translatable. A prose message is none of those.
3. **`skip` and `error` are different outcomes.** Skipping a webhook for an unknown menu is normal;
   having zero configured channels is a misconfiguration. Collapsing both into `null` throws away
   the distinction the on-call engineer needs.
4. **The success branch carries everything the executor needs** — no second lookup, no reaching
   back into globals.

## Guard clauses in occurrence order

Read the planner top to bottom and you get the rule matrix for free: missing input → skip;
misconfiguration → error; unknown entity → skip; otherwise → act. Never nest these into an
`if/else` pyramid; each guard returns.

## Name the strategy, don't inline the branches

When a decision has more than two outcomes, give the outcome a type and the function a
domain-language name:

```ts
/**
 * Page-level PDP buy-box strategy.
 * - `matrix`      — hydrate the attribute picker (under {@link PDP_VARIANT_CAP}).
 * - `over_budget` — skip the matrix; add-to-cart only via `?variant=` / `?sku=` deep link.
 * - `external`    — product-type opt-in for a fork-supplied picker (seat map, CPQ, …).
 */
export type BuyBoxStrategy = "matrix" | "over_budget" | "external";

export function resolveBuyBoxStrategy(input: {
  totalCount: number | null | undefined;
  productTypeSlug?: string | null;
  /** Override for tests; defaults to {@link EXTERNAL_BUYBOX_PRODUCT_TYPE_SLUGS}. */
  externalProductTypeSlugs?: readonly string[];
}): BuyBoxStrategy { … }
```

Three details worth copying:

- **Object parameter** once you have more than one argument — call sites become self-labelling and
  adding a field is not a breaking positional change.
- **A documented test-override param** (`externalProductTypeSlugs`) beats mocking the config
  module. It is one optional field, and it makes the function pure.
- **The union members are documented individually**, in the type's JSDoc, not scattered.

## Model the "either/or input" as a union too

```ts
export type PdpVariantDeepLink = { kind: "id"; id: string } | { kind: "sku"; sku: string };

export function resolvePdpVariantDeepLink(params: { variant?: string | null; sku?: string | null })
  : PdpVariantDeepLink | null {
  const variantId = params.variant?.trim();
  if (variantId) return { kind: "id", id: variantId };

  const sku = params.sku?.trim();
  if (sku) return { kind: "sku", sku };

  return null;
}
```

Normalizing messy external input (query strings) into a typed internal shape before business logic
touches it is the same discipline as the Python standard's "normalize before validate." Note the
`.trim()` — `?sku=%20%20` must not count as present.

Document the precedence in the type's JSDoc ("When both are present, `variant` wins"), because
that is a product decision, not an implementation detail.

## Server Actions: mutate, then revalidate, then return

```ts
export async function updateCartLineQuantity(checkoutId: string, lineId: string, quantity: number, channel: string) {
  if (quantity < 1) {
    return deleteCartLine(checkoutId, lineId, channel);   // delegate, don't duplicate
  }

  await executeAuthenticatedGraphQL(CheckoutLinesUpdateDocument, {
    variables: { checkoutId, lines: [{ lineId, quantity }] },
    cache: "no-cache",
  });

  revalidateCart(channel);
}
```

Rules the source holds to:

- **Edge case first.** `quantity < 1` is delete, not update — expressed as a delegation on line one,
  not a branch buried in the mutation.
- **Cache invalidation is a named local helper** (`revalidateCart(channel)`), not three
  `revalidatePath` calls inlined in every action.
- **Mutations use `cache: "no-cache"` explicitly.** Never rely on the default for writes.
- **Comment the ordering hazards.** From `clearCheckout`: *"Never revalidates `/checkout` (that
  remounts the flow and resets the step mid-payment)."* That is a bug someone paid for once.
- **Comment the workaround's cause.** From `logout`: *"SDK `signOut` only clears cookies for the
  current API URL. Cookies minted against a previously configured instance keep matching the marker
  scan, wedging the header in 'unavailable' — sweep every auth cookie regardless of API URL."*

Side effects run **after** the state is safe, never before. This is the same after-commit
discipline as backend transactions: mutate → confirm → invalidate → notify.

## Where logic goes

| Kind of logic | Home |
| --- | --- |
| "Should we do X, and with what?" | Pure `plan*()` / `resolve*()` in `src/lib/<domain>/` |
| "Do X" (network, cookies, revalidation) | Route handler or Server Action |
| "Render X" | Component — receives already-decided props |
| Shape translation from API → UI | A named mapper in `src/lib/<domain>/mappers/` |

If a component contains an `if` about *business* state (not rendering state), that `if` belongs in
a resolver you can unit test.

## Anti-patterns

| Avoid | Instead |
| --- | --- |
| Business branching inside a `route.ts` / action body | `plan*()` returns the decision; the handler executes it |
| Returning `null` for three different reasons | `{ action: "skip", reason: … }` union |
| Free-text reason strings | Closed string-literal reason codes |
| Positional args once there are 3+ | A single typed object parameter |
| Mocking a config module in tests | An optional, documented override parameter |
| `if/else` pyramid for validation | Guard clauses in occurrence order, each returning |
| Duplicating the delete path inside update | Delegate to the other action |

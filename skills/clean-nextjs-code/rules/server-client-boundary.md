---
name: server-client-boundary
description: Keep Server Components the default, push "use client" to leaves, enforce server-only modules with import "server-only" and .server.ts suffixes, mutate via Server Actions, and never ship a client-side data layer. Read when adding interactivity, handling auth/secrets, or deciding where a component or module runs.
---

# The server/client boundary

**Source:** `src/app/actions.ts`, `src/lib/auth/*`, `src/checkout/lib/server/*`,
`src/ui/atoms/link-with-channel.tsx`, `tsconfig.json` path aliases.

147 of 289 `.tsx` files carry `"use client"` — about half, and that is *fine*: this app has a
checkout, a variant picker, and a cart drawer. What matters is not the ratio but that each one is a
**leaf** and each one earns it.

## The default and the exceptions

Server Component unless the file needs one of:

- component state (`useState`, `useReducer`)
- effects (`useEffect`, `useLayoutEffect`)
- event handlers (`onClick`, `onChange`, …)
- browser APIs (`window`, `sessionStorage`, `IntersectionObserver`)
- a client-only library (Stripe Elements, Embla, Radix primitives with internal state)

If none apply, it stays on the server. "It might need state later" is not a reason.

## Push `"use client"` to the leaf

The tell is a client component that renders mostly static markup with one button in it. Split it:
the static part stays a Server Component, the button becomes a small client leaf receiving a
Server Action as a prop.

The source's smallest client components are exactly this shape — a channel-aware `<Link>` that
needs `useParams` and nothing else:

```tsx
"use client";
export const LinkWithChannel = ({ href, ...props }: Omit<ComponentProps<typeof Link>, "href"> & { href: string }) => {
  const { locale, channel } = useParams<{ locale?: string; channel?: string }>();

  if (!href.startsWith("/")) return <Link {...props} href={href} />;

  // During hydration/recovery there can be a transient moment where params
  // are unavailable. Avoid generating malformed URLs in that case.
  if (!locale || !channel) return <Link {...props} href={href} />;

  return <Link {...props} href={buildStorefrontPath(locale, channel, href)} />;
};
```

Note the hydration guard and its comment. Client leaves must survive the moment before context
exists — degrade to something harmless, never emit a broken value.

## Mark server-only modules twice

Belt and braces, because each catches a different mistake:

```ts
import "server-only";   // runtime/build error if a client bundle imports this
```

…plus a `.server.ts` filename suffix (`fetch-checkout.server.ts`, `auth.server.ts`), which catches
it during code review before anyone runs a build.

Apply `import "server-only"` to **anything that reads a secret, a cookie, or a privileged token**.
In the source that's the entire `src/lib/auth/` tree, `src/checkout/lib/server/*`, and
`filter-utils.server.ts`. There is also a `.client.ts` suffix convention for the rare
browser-only module.

The mirror rule: never put a secret behind `NEXT_PUBLIC_`. If it needs a prefix decision, the
answer is "does the browser need it," not "is it convenient."

## Mutations are Server Actions, not client fetches

No urql/Apollo/SDK calls from the browser to the commerce API. Data flows:

```
Read:   RSC → executePublicGraphQL / executeAuthenticatedGraphQL → props → components
Write:  client leaf → Server Action → executeAuthenticatedGraphQL → revalidate → RSC re-render
```

This keeps tokens server-side, keeps one retry/queue/error policy, and makes cache invalidation a
server concern. A client component that imports a GraphQL document is a design error.

Actions live in a route-adjacent `actions.ts` with `"use server"` at the top of the file (not
per-function — the file-level directive is the convention; the source's one inline `"use server"`
inside `clearCheckout` is redundant, not a pattern to copy).

## Auth: BFF, HttpOnly cookies, one session resolver

Login/logout/refresh go through your own `/api/auth/*` route handlers, which talk to the identity
provider server-side and set **HttpOnly** cookies. The browser never holds a JWT in JS-readable
storage.

Three habits from `src/lib/auth/`:

- **A cheap presence check separate from the expensive fetch.** `hasAuthSession()` only asks
  whether an access or refresh cookie exists, using *the same key derivation as the auth SDK* — its
  JSDoc says so explicitly: *"Uses the same lookup as the auth SDK — not a loose cookie-name
  scan."* Chrome can decide what to render without a round trip. It is wrapped in `try/catch`
  returning `{ hasAccess: false, hasRefresh: false }` — fail closed.
- **One resolver, composed from the two halves**, so no page re-derives "is there a session":

  ```ts
  /** Convenience wrapper — checks cookies then resolves `me`. */
  export async function resolveSessionUser<User>(
    fetch: () => Promise<GraphQLResult<{ me?: User | null }>>,
  ): Promise<SessionAuthState<User>> {
    const hasSession = await hasAuthSession();
    return resolveSessionUserFetch({ hasSession, fetch });
  }
  ```

  The inner `resolveSessionUserFetch({ hasSession, fetch })` takes both the flag and the fetcher as
  parameters — the state machine is pure and unit-testable, and the convenience wrapper is the only
  thing that touches cookies. Same injection-at-the-seam habit as the tests rule.
- **`invariant()` for required env** at the point of use, rather than a non-null assertion.

And validate every redirect target against an allowlist before honoring it — see
`validate-redirect-url.ts` and its test matrix in `rules/testing.md`.

## Cross-surface imports go through a named bridge

The storefront and the checkout are separate surfaces in one repo. The storefront must not
`import "@/checkout/*"`; cross-surface URLs go through a single `@paper/session-bridge` module
declared as a `tsconfig` path alias.

Generalize the rule: **when two subsystems must stay decoupled, give them one named seam and
enforce it with a path alias plus a lint rule** — don't rely on discipline.

## Imports

Always the `@/` alias, never `../../../`:

```tsx
✓ import { Button } from "@/ui/components/ui/button";
✗ import { Button } from "../../../ui/components/ui/Button";
```

Relative imports are acceptable only within a tightly-coupled folder (`./flow`, `./buy-box-strategy`
from its own test), which the source does consistently.

## Parallelize server data, don't waterfall

```ts
const [product, tPdp, tNav, content, currency] = await Promise.all([
  getProductData(params.slug, params.channel, params.locale),
  getTranslations({ locale: params.locale, namespace: "pdp" }),
  getTranslations({ locale: params.locale, namespace: "nav" }),
  getStorefrontContent(params.channel, params.locale),
  resolveChannelCurrency(params.channel),
]);
```

Sequential `await`s in an RSC are a latency bug. If the calls are independent, `Promise.all` them.

## Anti-patterns

| Avoid | Instead |
| --- | --- |
| `"use client"` at the top of a page or layout | Server Component + client leaves |
| Client component fetching the commerce API | Server Action / RSC fetch |
| JWT in `localStorage` | HttpOnly cookie set by a BFF route |
| Secret without `import "server-only"` | Both `server-only` and a `.server.ts` suffix |
| `NEXT_PUBLIC_` on anything sensitive | Server-side env, read in a server module |
| `../../../lib/x` | `@/lib/x` |
| Sequential independent `await`s in RSC | `Promise.all` |
| Subsystem reaching into another's internals | One named bridge module + path alias |

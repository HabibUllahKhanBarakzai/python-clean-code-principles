---
name: components-and-styling
description: Component authoring conventions — kebab-case files with PascalCase exports, cva variant matrices, when to split an abstraction instead of forcing one, design tokens over raw colors, fail-closed link rendering, and accessibility defaults. Read when creating or reviewing React components, styling, or UI primitives.
---

# Components and styling

**Source:** `src/ui/components/ui/button.tsx`, `src/ui/atoms/*`,
`src/ui/components/nav/nav-href-link.tsx`, `src/lib/url/safe-href.ts`,
`skills/saleor-paper-storefront/references/code-conventions.md`.

## Naming: kebab-case files, PascalCase exports

```
✓ product-card.tsx      → export function ProductCard()
✓ use-cart.ts           → export function useCart()
✓ filter-utils.server.ts
✗ ProductCard.tsx  ✗ useAuth.ts  ✗ filterUtils.ts
```

The stated reasons are worth repeating because they're the ones that hold up: no case-sensitivity
bugs between macOS/Windows and Linux CI; **one** rule instead of "components Pascal, hooks camel";
shell-friendly; matches App Router's own route conventions. Directories are kebab-case too.

Suffix vocabulary: `.server.ts`, `.client.ts`, `.test.ts`. Exceptions are `README.md`/`AGENTS.md`
and ecosystem config files.

## Variants: one cva matrix, exported derived types

```tsx
/**
 * Visual variant + size matrix for buttons and token-backed link CTAs.
 * Disabled styling is handled separately in `buttonClassName` because it differs
 * for links (`aria-disabled`) vs native `<button disabled>` and per variant.
 */
export const buttonVariants = cva(
  cn(
    "inline-flex items-center justify-center gap-2 whitespace-nowrap rounded-button font-medium",
    "transition-all duration-200",
    "focus-visible:outline-hidden focus-visible:ring-2 focus-visible:ring-ring focus-visible:ring-offset-2 focus-visible:ring-offset-background",
  ),
  {
    variants: {
      variant: { default: "…", secondary: "…", "outline-solid": "…", ghost: "…", destructive: "…" },
      size:    { default: "h-10 px-4 py-2 text-sm", sm: "h-9 px-3 text-sm", lg: "h-14 px-8 text-base", icon: "h-10 w-10 p-0" },
    },
    defaultVariants: { variant: "default", size: "default" },
  },
);

export type ButtonVariant = NonNullable<VariantProps<typeof buttonVariants>["variant"]>;
export type ButtonSize    = NonNullable<VariantProps<typeof buttonVariants>["size"]>;
```

Base classes are grouped by concern (layout / motion / focus) using `cn(...)` across several
strings instead of one 300-character line. Focus-visible rings are in the **base**, not a variant —
accessibility isn't opt-in.

## Don't force one abstraction over two genuinely different cases

The most instructive detail in the file: `buttonClassName()` exists *separately* from
`buttonVariants` because disabled styling genuinely differs between `<button disabled>` and
`<a aria-disabled>` — and differs again per variant.

```tsx
/** Disabled styles for links that use `aria-disabled` instead of the `disabled` attribute. */
export const ariaDisabledClassName = "aria-disabled:pointer-events-none aria-disabled:cursor-not-allowed";

export function buttonClassName({ variant = "default", size = "default", asLink = false, className }: ButtonClassNameOptions = {}) {
  const disabledClassName =
    variant === "default"
      ? asLink
        ? cn(ariaDisabledClassName, "aria-disabled:bg-muted aria-disabled:text-muted-foreground aria-disabled:shadow-none hover:aria-disabled:bg-muted")
        : "disabled:pointer-events-none disabled:cursor-not-allowed disabled:bg-muted disabled:text-muted-foreground disabled:shadow-none hover:disabled:bg-muted"
      : asLink
        ? cn(ariaDisabledClassName, "aria-disabled:opacity-50")
        : "disabled:pointer-events-none disabled:cursor-not-allowed disabled:opacity-50";

  return cn(buttonVariants({ variant, size }), disabledClassName, className);
}

export const Button = forwardRef<HTMLButtonElement, ButtonProps>(({ className, variant = "default", size = "default", ...props }, ref) =>
  <button ref={ref} className={buttonClassName({ variant, size, className })} {...props} />
);
Button.displayName = "Button";
```

Cramming `disabled` into the cva matrix would need a 2×5 compound-variant table to express
something a conditional says plainly. The rule: **when two cases differ structurally, name the
difference (`asLink`) and branch — don't torture one abstraction into covering both.** And when you
branch, say why in a comment on the abstraction, so the next person doesn't "clean it up."

Also: `forwardRef` + explicit `displayName` on any primitive that wraps a DOM element (Radix and
focus management need the ref; devtools need the name).

## Design tokens, never raw color literals

Style through semantic tokens (`bg-background`, `text-foreground`, `border-input`,
`ring-ring`, `rounded-button`) defined once in a `brand.css`. A rebrand then touches one file, and
light/dark comes for free.

This is enforced, not merely documented — a hard-fail lint script scans `src/ui/**/*.tsx` for hex
and `rgb()/hsl()` literals, with a documented escape hatch (`design-tokens-allow` comment on the
line) for legitimate cases like a hex swatch coming from catalog data. See `rules/enforcement.md`.

Use `cn()` (`clsx` + `tailwind-merge`) for every conditional class so later classes actually win
instead of losing to specificity chance.

## Fail closed when rendering untrusted content

Hrefs from a CMS, a menu, or a user attribute are untrusted. The source has one shared predicate
module and a component that renders **non-interactive content** when validation fails — it does not
render a link with a stripped href, and it does not throw:

```tsx
export function NavHrefLink({ href, children, className, ...props }: NavHrefLinkProps) {
  if (!isSafeNavHref(href)) {
    return <span className={cn(className)}>{children}</span>;   // fail closed
  }

  if (isExternalMenuHref(href)) {
    return <a href={href} rel="noopener noreferrer" className={className} {...props}>{children}</a>;
  }

  return <LinkWithChannel href={href} prefetch={false} className={className} {...props}>{children}</LinkWithChannel>;
}
```

The predicates are four small named functions plus one sanitizer, each answering one question:

```ts
/** Absolute http(s) only — for external-only surfaces. */
export function isSafeExternalHref(href: string): boolean { … }

/** Storefront-relative paths: `/…` but not protocol-relative (`//…`, `/\…`). */
export function isSafeInternalHref(href: string): boolean {
  const trimmed = href.trim();
  if (!trimmed.startsWith("/")) return false;
  // Browsers normalize backslashes to slashes, so `/\evil.com` and `//evil.com`
  // are both treated as protocol-relative external URLs — reject them.
  return trimmed[1] !== "/" && trimmed[1] !== "\\";
}

export function isSafeMailtoHref(href: string): boolean { … }

/** Safe for nav/CTA anchors: http(s), mailto, or internal path. */
export function isSafeNavHref(href: string): boolean { … }

/** Returns a trimmed safe href or null when the value must not be linked. */
export function sanitizeNavHref(href: string | null | undefined): string | null { … }
```

Same discipline for HTML: rich text from a CMS goes through a sanitizer (`xss`) before
`dangerouslySetInnerHTML`, every time, no exceptions for "trusted" sources.

`rel="noopener noreferrer"` on every external anchor.

## Wrapper components carry the invariant

```tsx
export const ProductImageWrapper = ({ containerClassName, className, fill = true, ...props }: ProductImageWrapperProps) => (
  <div className={clsx("relative aspect-square overflow-hidden bg-secondary", containerClassName)}>
    <NextImage {...props} fill={fill} className={clsx(fill ? "object-cover object-center" : "h-full w-full object-cover object-center", className)} />
  </div>
);
```

`fill` requires a positioned parent and an aspect ratio — so the wrapper owns both, and no call
site can get it wrong. That's the point of a wrapper: it should make an invariant unbreakable, not
just save typing.

## Accessibility defaults

Present in the primitives, not bolted on later:

- `focus-visible:ring-*` in every interactive base class.
- Loading states carry `role="status"` + `aria-busy="true"` + an `sr-only` label; decorative SVGs
  get `aria-hidden="true"`.
- Disabled links use `aria-disabled` + `pointer-events-none`, never a removed `href`.
- Icon-only buttons get an accessible name.

## Props typing

Extend the DOM element's props rather than redeclaring them, and `Omit` what you're overriding:

```tsx
export interface NavHrefLinkProps extends Omit<ComponentProps<"a">, "href"> { href: string; children: ReactNode }
type Props = Omit<ComponentProps<typeof Link>, "href"> & { href: string };
```

## Anti-patterns

| Avoid | Instead |
| --- | --- |
| `ProductCard.tsx` / `useAuth.ts` | kebab-case files, PascalCase/camelCase exports |
| Hardcoded `#fff` / `rgb(…)` in a component | Semantic design token + lint gate |
| Conditional classes via template strings | `cn()` / `clsx` + `tailwind-merge` |
| A compound-variant table to model a structural difference | A named boolean (`asLink`) and a branch |
| Rendering an unvalidated CMS href | Shared predicate; fail closed to a `<span>` |
| `dangerouslySetInnerHTML` on CMS HTML | Sanitize first, always |
| Redeclaring `className`, `onClick`, … | `extends ComponentProps<"a">` |
| Primitive without `forwardRef`/`displayName` | Both, on anything wrapping a DOM node |

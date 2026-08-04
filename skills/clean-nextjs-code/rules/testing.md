---
name: testing
description: Frontend testing as behavior specification — dependency injection over module mocking, colocated unit tests, testing through real composition, env stubbing, security matrices, and a deliberately tiny e2e suite. Read when writing or reviewing tests for TypeScript/React/Next.js code.
---

# Testing: behavior, not wiring

**Source:** ~85 colocated `*.test.ts` files, `vitest.config.ts`, `vitest.setup.ts`, 3 Playwright
specs in `e2e/`.

## The headline number

**6 of ~85 test files use `vi.mock`.** Everything else tests a pure function directly, or injects a
dependency at a seam. That ratio is not an accident — it is the consequence of the planner and
Result patterns. If your tests need heavy mocking, the *design* is what needs fixing.

## Inject at the seam; don't mock the module

The payment executor talks to the network through a `CheckoutTransport` interface with a setter.
The test supplies a fake object — no module interception, no hoisting rules, no `__mocks__`
directory:

```ts
const initializeTransaction = vi.fn<CheckoutTransport["initializeTransaction"]>();
const completeCheckout      = vi.fn<CheckoutTransport["completeCheckout"]>();

const fakeTransport: CheckoutTransport = {
  fetchCheckout: vi.fn(),
  updateBillingAddress: vi.fn(),
  initializePaymentGateways: vi.fn(),
  initializeTransaction,
  processTransaction: vi.fn(),
  completeCheckout,
};

describe("executePayment", () => {
  beforeEach(() => {
    initializeTransaction.mockReset();
    completeCheckout.mockReset();
    setCheckoutTransport(fakeTransport);
  });

  it("runs dummy payment flow and completes checkout", async () => {
    initializeTransaction.mockResolvedValue({ ok: true, data: { transactionEvent: { type: "CHARGE_SUCCESS", message: "ok" }, transaction: { id: "tx-1" }, errors: [] } });
    completeCheckout.mockResolvedValue({ ok: true, orderId: "order-1" });

    const result = await executePayment(
      { type: "dummy", gateway: { id: "…dummy-payment-app", name: "Dummy" }, submitMode: "server" },
      { checkoutId: "checkout-1", amount: 42.5 },
      gatewayMessages,
    );

    expect(result).toEqual({ ok: true, orderId: "order-1" });
    expect(initializeTransaction).toHaveBeenCalledWith({
      checkoutId: "checkout-1", amount: 42.5,
      paymentGateway: { id: "…dummy-payment-app", data: { event: { includePspReference: true, type: "CHARGE_SUCCESS" } } },
    });
  });
});
```

Note `vi.fn<CheckoutTransport["initializeTransaction"]>()` — the mock is typed from the interface,
so a signature change breaks the test at compile time instead of at runtime.

Also note the assertion asserts the **exact request payload**, not just "was called." For a payment
initialization, the payload *is* the behavior.

Reach for `vi.mock` only when the seam is a framework module you don't own (`next/headers`,
`next/navigation`) — that's what the 6 files do.

## Test names are sentences about behavior

```
✓ defaults to matrix under the variant cap
✓ uses over_budget above the cap
✓ prefers external when the product type is opted in
✓ prefers variant id over sku
✓ returns null when neither param is set
✓ keeps code default subheading when Saleor only sets hero heading
✓ clears stale payment completing flag for a different checkout
✓ keeps the completing flag for the current checkout so the resume flow can reconcile it
✓ rejects subdomain and scheme tricks
✓ stays on payment when returning from Stripe redirect
```

`describe(<exported function name>)` + `it(<behavior sentence>)`. Read as a list, these *are* the
spec. Compare with `it("should work")` or `it("test case 2")`, which tell a future reader nothing
about what breaking the test means.

Prefer the pattern `<verb>s <outcome> when <condition>`. Both halves matter: the last example
above even encodes *why* the behavior exists ("so the resume flow can reconcile it").

## Test through the real composition, not the intermediate shape

The homepage mapper test doesn't assert on the mapper's raw output. It runs the mapper's result
through the real merge function and asserts on the shape the UI actually receives:

```ts
it("keeps code default subheading when Saleor only sets hero heading", () => {
  const partial = mapHomepagePage(homepagePage([
    { slug: "hero-heading", plainText: "Discover our collection" },
    { slug: "hero-cta-label", plainText: "Shop all" },
  ]));

  const merged = mergeStorefrontContent(defaultStorefrontContent, partial);

  expect(merged.surfaces.homepage.hero.heading).toBe("Discover our collection");
  expect(merged.surfaces.homepage.hero.subheading).toBe(defaultStorefrontContent.surfaces.homepage.hero.subheading);
});
```

Two lessons:

1. **Assert the positive *and* the negative side effect.** "It set the heading" is half the rule;
   "it did not clobber the subheading" is the half that actually regresses.
2. **Assert against the source of truth**, not a copy. `defaultStorefrontContent.…hero.subheading`
   rather than the literal string — so changing the default doesn't break an unrelated test.

## Build test data with a local builder, not inline literals

```ts
function homepagePage(attributes: Array<{ slug: string; plainText?: string | null; collectionSlug?: string | null; fileUrl?: string | null }>): StorefrontContentPageFragment {
  return {
    slug: "storefront-homepage",
    isPublished: true,
    pageType: { slug: "storefront-homepage" },
    assignedAttributes: attributes.map(({ slug, plainText, collectionSlug, fileUrl }) => { … }),
  };
}
```

A typed builder at the top of the file, taking only the fields the tests vary. Every `it` then
shows exactly one thing: the input that matters. No 40-line object literals repeated seven times.

## Environment: stub it, and always unstub

Preferred — Vitest's helper with a blanket restore:

```ts
afterEach(() => { vi.unstubAllEnvs(); });

it("builds language-only hreflang keys when locale×channel pairs are unset", () => {
  vi.stubEnv("NEXT_PUBLIC_STOREFRONT_LOCALES", "en,pl");
  vi.stubEnv("NEXT_PUBLIC_DEFAULT_CHANNEL", "default-channel");
  expect(buildLocaleHreflangAlternates("default-channel", "/products/hoodie")).toEqual({ … });
});
```

When you must touch a var `stubEnv` can't handle (`NODE_ENV` is read-only in some setups), capture
originals at module scope and restore explicitly:

```ts
const ORIGINAL_NODE_ENV = process.env.NODE_ENV;
function setNodeEnv(value: string) {
  Object.defineProperty(process.env, "NODE_ENV", { configurable: true, enumerable: true, value, writable: true });
}
function restoreEnv(name: string, value: string | undefined) {
  if (value === undefined) delete process.env[name]; else process.env[name] = value;
}
beforeEach(() => { setNodeEnv("production"); delete process.env.NEXT_PUBLIC_STOREFRONT_URL; … });
afterEach(()  => { restoreEnv("NODE_ENV", ORIGINAL_NODE_ENV); … });
```

`beforeEach` sets the *strictest* baseline (production, nothing configured) so a test that forgets
to configure something fails closed.

## Security rules get an attack matrix

```ts
it("rejects unconfigured URLs in production", () => {
  expect(isAllowedRedirectUrl("https://shop.example.com/checkout?checkout=abc")).toBe(false);
});

it("rejects subdomain and scheme tricks", () => {
  process.env.NEXT_PUBLIC_STOREFRONT_URL = "https://shop.example.com";
  expect(isAllowedRedirectUrl("https://shop.example.com.evil.com/x")).toBe(false);
  expect(isAllowedRedirectUrl("javascript:alert(1)")).toBe(false);
  expect(isAllowedRedirectUrl("//evil.example.com/x")).toBe(false);
  expect(isAllowedRedirectUrl("not a url")).toBe(false);
});

it("accepts loopback redirect origins outside production", () => {
  setNodeEnv("development");
  expect(isAllowedRedirectUrl("http://localhost:3000/login")).toBe(true);
  expect(isAllowedRedirectUrl("http://127.0.0.1:3000/login")).toBe(true);
  expect(isAllowedRedirectUrl("http://[::1]:3000/login")).toBe(true);
});
```

For any allowlist/sanitizer, enumerate the bypasses as one grouped test: suffix-domain
(`example.com.evil.com`), scheme (`javascript:`), protocol-relative (`//`), backslash (`/\`),
malformed. Grouping them in one `it` is right here — they're one rule ("reject tricks"), and each
line is self-describing.

## State machines: cover the transitions, not the happy path

```ts
describe("reconcileCheckoutSessionStorage", () => {
  afterEach(() => { sessionStorage.clear(); });

  it("clears stale payment completing flag for a different checkout", …);
  it("clears payment completing and stripe transaction id when checkout session is absent", …);
  it("keeps the completing flag for the current checkout so the resume flow can reconcile it", …);
  it("keeps completing flag during Stripe redirect return", …);
});
```

Two clears, two keeps. The negative cases (what must *not* be cleared) are where the real bugs are.

## Test setup explains itself

`vitest.setup.ts` provides an in-memory `Storage` for the `node` environment rather than pulling in
jsdom — and comments *why it lives on `globalThis`*:

```ts
/**
 * The default `node` test environment has no Web Storage, but several checkout
 * payment modules persist state in `sessionStorage`. Provide a minimal in-memory
 * implementation on `globalThis` instead of pulling in a full DOM environment.
 *
 * It deliberately lives on `globalThis` (not inside a module) so it survives
 * `vi.resetModules()` — the orphan-detection tests rely on storage outliving a
 * simulated page reload.
 */
```

Plus a blanket `beforeEach(() => { sessionStorage.clear(); localStorage.clear(); })` so no test can
leak state into the next.

Config choices worth copying (`vitest.config.ts`): `environment: "node"` (fast — only add jsdom
where you truly render), colocated `include: ["src/**/*.test.ts"]`, the same `@/` alias as the app,
and an `exclude` for special harness tests that shouldn't run in the default suite.

## Keep e2e tiny and make each spec name its regression

Three Playwright specs total, against ~85 unit test files. Each opens with the bug it prevents:

```ts
/**
 * Regression: browser Back must adopt popstate intent before CheckoutStepUrlGuard
 * re-heals the pre-Back step. Without the macrotask heal, `?step=` snaps back to
 * shipping and the shopper stays trapped on the old step.
 *
 * Step advances use shallow history (same mechanism as Continue) without form
 * submission — this isolates popstate ordering from address-validation noise.
 */
test.describe("checkout shallow step URL — browser Back", () => {
  test.describe.configure({ mode: "serial", timeout: 120_000 });

  test("Back from shipping lands on contact and stays there", async ({ page }) => {
    await addFirstProductAndOpenCheckout(page);
    await expectStep(page, "contact");
    await advanceToShipping(page);
    await page.goBack();
    await expectStep(page, "contact");
  });
});
```

Reserve e2e for what a unit test genuinely cannot reach: browser history, real navigation ordering,
third-party redirect returns. Everything else is faster and more precise as a unit test.

Supporting habits: **domain-language page helpers** in `e2e/helpers/` (`advanceToShipping`,
`expectStep`, `addFirstProductAndOpenCheckout`) so specs read as user stories; `expect.poll` for
asynchronous state like a cookie appearing; explicit `throw new Error("checkoutId cookie missing
after add-to-cart")` when setup fails, so a broken fixture doesn't masquerade as a failed
assertion; `retries: 2` on CI only; `trace: "on-first-retry"`.

## Checklist

- Test name reads as a behavior sentence; inverting the rule makes it fail.
- Both the positive and the negative side effect are asserted.
- Dependencies come in as parameters or an injected interface — `vi.mock` only for framework modules.
- Mocks are typed from the interface (`vi.fn<Iface["method"]>()`).
- Expected values reference the source of truth, not copied literals.
- Env/storage/global state is reset in `afterEach`, and the baseline is the strict one.
- Allowlists and sanitizers have an explicit bypass matrix.
- State machines cover "must not change" cases, not only "must change."
- e2e is reserved for browser-only behavior, and each spec documents its regression.

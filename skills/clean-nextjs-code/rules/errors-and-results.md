---
name: errors-and-results
description: Model I/O failures as typed Result values instead of thrown exceptions — error taxonomy, retryability, user-facing messages, retry/backoff, request queueing, and typed client entry points. Read when writing or reviewing a data-fetching layer, API client, or any code that handles network failure.
---

# Errors as values

**Source:** `src/lib/graphql.ts` (~480 lines, the single network boundary of the whole app).

The network is the dominant source of frontend bugs. A thrown exception lets a call site *forget*
to handle failure and still typecheck. A Result union does not. Every fetch in the source returns
`Promise<GraphQLResult<T>>`; there is no `throw` on the happy path anywhere in the data layer.

## The shape

```ts
export interface GraphQLSuccess<T> { ok: true;  data: T }
export interface GraphQLFailure    { ok: false; error: GraphQLError }
export type GraphQLResult<T> = GraphQLSuccess<T> | GraphQLFailure;
```

Discriminate on a boolean literal (`ok`), not on the presence of a field. `if (!result.ok) return …`
narrows in one line and reads like prose. Call sites become:

```ts
const result = await executeAuthenticatedGraphQL(CheckoutDeleteLinesDocument, { variables, cache: "no-cache" });

if (result.ok) {
  const checkout = result.data.checkoutLinesDelete?.checkout;
  …
}
```

## Classify errors by layer, in the order they occur

Don't collapse everything into `Error`. Name the layers, and document the ordering — it is the
mental model a reader needs:

```ts
/**
 * Error layers in order of occurrence:
 * 1. network    - Failed to reach server (timeout, DNS, connection refused)
 * 2. http       - Server responded with error status (4xx, 5xx)
 * 3. graphql    - Query/mutation syntax or validation errors
 * 4. validation - Domain errors (e.g., "email already exists")
 */
export type GraphQLErrorType = "network" | "http" | "graphql" | "validation";
```

Then carry per-layer detail on the error object, with each field documented as to *which* layer
populates it:

```ts
export interface GraphQLError {
  type: GraphQLErrorType;
  message: string;
  /** HTTP status code (only for 'http' type) */
  statusCode?: number;
  /** Whether the request could succeed if retried */
  isRetryable: boolean;
  /** API error codes from `errors[].extensions.code` (only for 'graphql' type) */
  codes?: readonly string[];
  /** Original error for debugging */
  cause?: unknown;
  /** Validation errors with field info (only for 'validation' type) */
  validationErrors?: ReadonlyArray<{ field?: string | null; message: string; code?: string | null }>;
}
```

**Decide retryability once, at construction** — not at every call site:

```ts
function httpError(statusCode: number, message: string): GraphQLFailure {
  return { ok: false, error: { type: "http", message, statusCode,
    isRetryable: statusCode >= 500 || statusCode === 429 } };
}
```

## Private constructors, one per error kind

`networkError()`, `httpError()`, `graphqlError()`, `validationError()`, `success()` — five tiny
module-private factories. Nothing constructs a `GraphQLFailure` literal inline. This is what keeps
the shape consistent across 480 lines and makes adding a field a one-place edit.

## Translate to user-facing copy in exactly one place

An exhaustive `switch` over the union. If someone adds a fifth error type, this fails to compile —
that is the point.

```ts
export function getUserMessage(error: GraphQLError): string {
  switch (error.type) {
    case "network":
      return "Unable to connect to the store. Please check your internet connection.";
    case "http":
      if (error.statusCode === 401 || error.statusCode === 403) return "You don't have permission to view this content.";
      if (error.statusCode === 404) return "The item you're looking for doesn't exist or has been removed.";
      return "The store is temporarily unavailable. Please try again in a moment.";
    case "graphql":
      return "Something went wrong loading this page.";
    case "validation":
      return error.message || "Please check your input and try again.";
  }
}
```

Never leak `error.message` from the network/http/graphql layers to a user — those are for logs.
Only `validation` messages are safe to surface, because the API authored them for humans.

## One private executor, several named public entry points

The auth mode is the axis of variation, so it becomes a parameter of a *private* function and
disappears behind three named exports. Call sites read as intent, not configuration:

```ts
type GraphQLAuth = "none" | "session" | "app";

async function executeGraphQL<R, V>(op, options: GraphQLOptions<V> & { auth: GraphQLAuth }): Promise<GraphQLResult<R>> { … }

/** Public API access — no Authorization header. Use for catalog, menus, checkout read by ID. */
export async function executePublicGraphQL<R, V>(op, options) { return executeGraphQL(op, { ...options, auth: "none" }); }

/** Customer session — JWT from auth cookies. Use for `me`, orders, checkout mutations. */
export async function executeAuthenticatedGraphQL<R, V>(op, options) { return executeGraphQL(op, { ...options, auth: "session" }); }

/** App token from env (server-side only). Use for privileged queries. */
export async function executeAppGraphQL<R, V>(op, options) { return executeGraphQL(op, { ...options, auth: "app" }); }
```

Each export's JSDoc says *when to use it*, not what it does. That is the sentence a reader
actually needs.

## Make the type system encode the call contract

`variables` should be required when the operation has them and forbidden when it doesn't:

```ts
type GraphQLOptions<Variables> = {
  headers?: HeadersInit;
  cache?: RequestCache;
  revalidate?: number;
} & (Variables extends Record<string, never> ? { variables?: never } : { variables: Variables });
```

This is the good kind of type cleverness: it removes a class of mistake and costs the reader one
line. Contrast with cleverness that only saves keystrokes — avoid that.

## Retry, timeout, and backpressure belong in the boundary

Everything below is in the data layer so no feature code ever re-implements it:

- **Timeout** via `AbortController` + `setTimeout`, cleared in `finally`.
- **Retry** on `429` and `5xx` only, with exponential backoff (`delayMs * 2 ** attempt`) that
  **honors `Retry-After`** when the server sends it.
- **Concurrency limiting** via a small `RequestQueue` class (`maxConcurrent`, `minDelayMs`, both
  env-tunable) so a build fanning out hundreds of pages doesn't get rate-limited off the API.
- **Structured warn logs** on every retry, naming the operation and attempt number:
  `[GraphQL] ProductDetails (slug=hoodie): HTTP 429 - retrying in 2000ms (attempt 2/3)`.
  The operation name is parsed out of the document once; variables are truncated to 30 chars each
  by a `formatVariablesForLog` helper so logs never dump a payload.

## Respect partial success

GraphQL (and many REST APIs) can return data *and* errors. Decide the policy explicitly and
comment it:

```ts
// GraphQL allows partial success — return data when the API included it.
if (body.data !== null && body.data !== undefined) return success(body.data);
if (messages.length > 0) return graphqlError(messages, extractErrorCodes(body.errors));
return graphqlError(["No data in GraphQL response"]);
```

## Deprecate loudly, migrate gradually

The old exception class is still exported, under a header comment and a `@deprecated` tag naming
the replacement:

```ts
// ============ Legacy exports (for gradual migration) ============

/** @deprecated Use Result pattern instead. This class is kept for gradual migration. */
export class SaleorError extends Error { … }
```

A deprecated export with a stated successor is honest. An undeprecated parallel API is debt.

## Anti-patterns

| Avoid | Instead |
| --- | --- |
| `throw new Error()` from a fetch helper | Return a typed `Result` |
| `catch (e) { console.error(e) }` and continue | Return a failure the caller must narrow |
| `error: any` / `catch (e: any)` | `cause?: unknown` on a typed error object |
| Deciding retryability at the call site | Compute `isRetryable` in the error constructor |
| Rendering `error.message` to users | `getUserMessage(error)` with an exhaustive switch |
| Retry logic duplicated per feature | One retrying executor at the boundary |
| Bare `fetch()` in a component or action | The named executor for that auth mode |

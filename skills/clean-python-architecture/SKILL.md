---
name: clean-python-architecture
description: Structure Python backend systems by domain capability with thin boundaries, typed read models, intentional data loading, isolated integrations, and controlled side effects. Use when organizing packages, designing module boundaries, placing business rules, or reviewing the architecture of a Python service, monolith, or library.
---

# Clean Python Architecture Principles

A practical guide for structuring Python backend systems that stay understandable as they grow.

This guide is framework-neutral. The examples use common backend capabilities such as identity, billing, documents, reports, notifications, integrations, files, jobs, and audit logs. The principles apply to web APIs, workers, CLIs, libraries, monoliths, and modular services.

## The Standard

Good architecture gives every piece of code an obvious home.

It separates external boundaries from domain behavior, groups code by business capability, makes data loading intentional, isolates extension points, keeps side effects controlled, and lets tests describe behavior at the right level.

Clean architecture is not about adding layers for their own sake. It is about creating stable boundaries so the next change has a clear path.

## 1. Organize By Domain Capability

Prefer top-level packages that represent business capabilities:

```text
identity/
billing/
documents/
reports/
notifications/
integrations/
files/
jobs/
audit/
```

This makes the codebase searchable by business concept.

Avoid organizing most code by technical type:

```text
models/
views/
services/
utils/
validators/
tasks/
```

Technical folders become dumping grounds when the product grows. Domain folders keep related rules, models, tasks, tests, and helpers near each other.

## 2. Let Domain Packages Own Business Rules

Business rules should live in the domain package that owns the concept.

For example, billing rules should live near billing code:

```text
billing/
  actions.py
  calculations.py
  fetch.py
  models.py
  problems.py
  tasks.py
  validators.py
  tests/
```

The API layer may call these rules, but it should not own them.

Prefer:

```python
from billing.actions import approve_invoice
from billing.calculations import calculate_invoice_balance


def perform_request(...):
    ...
    balance = calculate_invoice_balance(invoice)
    approve_invoice(invoice, approver, balance)
```

Avoid putting core business behavior directly in controllers, views, resolvers, or command handlers.

## 3. Treat External Interfaces As Boundaries

API handlers, GraphQL mutations, CLI commands, message consumers, and scheduled jobs are boundaries. They translate between external contracts and internal domain behavior.

Their responsibilities:

- Parse and validate request shape.
- Resolve external IDs into domain objects.
- Check permissions.
- Normalize input.
- Call domain functions.
- Convert domain errors into boundary-specific errors.
- Return or publish the expected response.

Their responsibilities should not include:

- Billing policy.
- Access policy.
- Scheduling rules.
- Document state rules.
- Notification eligibility.
- Provider-specific integration behavior.
- Persistence choreography unrelated to the boundary.

Good boundary code is orchestration code:

```python
def handle_approve_invoice_request(actor: User, payload: dict) -> Response:
    require_permission(actor, Permission.APPROVE_INVOICES)
    invoice = get_invoice_or_error(payload["invoice_id"])
    approval = normalize_invoice_approval_input(payload)
    approve_invoice(invoice, actor, approval)
    return invoice_response(invoice)
```

The boundary should be thin enough that domain behavior can be tested without HTTP, GraphQL, CLI, or queue machinery.

## 4. Use Fetch Modules To Build Read Models

Large systems often need richer objects than raw database models.

Use fetch/read modules to gather related data and build typed info objects:

```python
@dataclass
class InvoiceApprovalInfo:
    invoice: Invoice
    customer: Customer
    approver: User
    open_disputes: list[Dispute]
    unpaid_balance: Money
```

This creates a stable input shape for business logic.

Benefits:

- Query shape is centralized.
- Expensive prefetching is intentional.
- Business functions receive complete context.
- Tests can build focused info objects.
- Boundary code does not need to know every related model.

Use names like:

- `fetch_invoice_approval_info`
- `fetch_document_access_info`
- `fetch_report_export_context`
- `fetch_account_billing_state`

## 5. Make Data Loading Explicit

Database access is architecture.

Code should reveal when it needs related data:

```python
reports = (
    ReportQuery()
    .include("owner")
    .include("schedule")
    .include("latest_export")
    .filter(workspace_id=workspace_id)
)
```

Prefer a few well-known data-loading paths over scattered incidental queries.

Good signs:

- Fetch helpers are named by use case.
- Query objects or repositories expose intentional read paths.
- Related objects are loaded before rule-heavy loops.
- Mapping dictionaries are built for repeated lookup.
- Tests can assert query counts for sensitive paths.

Bad signs:

- Business logic repeatedly calls storage inside loops.
- API handlers know too much about model relationships.
- Every feature invents its own query shape.
- Hidden lazy-loading changes behavior or performance.

## 6. Keep Extension Points Behind Interfaces

Email providers, payment processors, search indexes, file storage, AI providers, analytics systems, and webhooks should be accessed through stable interfaces.

The domain should call a manager or interface:

```python
email_sender.send_invoice_approved(invoice)
payment_gateway.capture(payment_request)
search_index.schedule_reindex(document_id)
```

It should not know every implementation detail of every external provider.

Good extension boundaries:

- Define the method names the domain can call.
- Provide default behavior or clear no-op behavior.
- Pass domain objects or typed data.
- Handle missing optional capabilities safely.
- Keep provider errors from leaking everywhere.

This lets a system grow integrations without letting integrations own the core domain.

## 7. Separate Sync Rules From Async Work

Some work must happen immediately because the caller needs the result. Other work can run after the transaction or in the background.

Keep that distinction explicit.

Synchronous:

- Validate input.
- Check permissions.
- Calculate state required for the response.
- Save state needed by the current operation.

Asynchronous or after-commit:

- Email delivery.
- Webhook delivery.
- Search indexing.
- Audit fan-out.
- Cleanup tasks.
- Long-running exports.
- External side effects that should not observe rolled-back state.

Prefer an event helper that respects transactions:

```python
def publish_after_commit(event: DomainEvent) -> None:
    if transaction_is_active():
        on_transaction_commit(lambda: event_bus.publish(event))
    else:
        event_bus.publish(event)
```

Architecture should make unsafe side-effect ordering hard to do by accident.

## 8. Use Action Modules For State Transitions

State transitions deserve named entry points.

Examples:

- `approve_invoice`
- `cancel_subscription`
- `archive_document`
- `complete_report_export`
- `rotate_api_key`
- `retry_webhook_delivery`

Action modules are useful when an operation:

- Changes state.
- Emits events.
- Coordinates multiple helpers.
- Has before/after status transitions.
- Needs tests around side effects.

Avoid scattering the same transition across API handlers, tasks, signal handlers, and management commands. Each entry point should delegate to the same domain action.

## 9. Model Problems Explicitly

Do not represent all domain issues as strings.

Prefer typed problem objects:

```python
@dataclass
class DocumentAccessProblem:
    document_id: str
    reason: AccessDeniedReason


@dataclass
class ReportExportProblem:
    report_id: str
    missing_fields: list[str]
```

Typed problems let the system:

- Collect multiple issues.
- Convert issues into API errors.
- Log issues consistently.
- Test problem detection separately from presentation.
- Add new problem types without breaking every caller.

This is especially useful when an operation can be partially valid.

## 10. Keep Shared Code Truly Shared

A `core` or `common` package should contain primitives and infrastructure that multiple domains genuinely need.

Good candidates:

- Time and clock helpers.
- Money and quantity helpers.
- Event dispatch helpers.
- Base exceptions.
- Generic validation helpers.
- Database or transaction utilities.
- Observability utilities.

Poor candidates:

- Random billing logic.
- Document-specific formatting.
- Report-specific status rules.
- One-off helpers used by one feature.

Shared packages should be boring and stable. If a helper has domain language in its name, it probably belongs in that domain.

## 11. Prefer Explicit Dependencies Over Hidden Globals

Pass important collaborators into functions when they affect behavior:

```python
def approve_invoice(
    invoice: Invoice,
    approver: User,
    payment_gateway: PaymentGateway,
    event_bus: EventBus,
) -> None:
    ...
```

This makes behavior easier to test and reduces hidden coupling.

Use dependency accessors at boundaries when needed, but avoid burying global lookups deep in business helpers.

Good dependencies to pass explicitly:

- Current user or actor.
- Tenant, workspace, or organization context.
- Clock or current time provider.
- Database connection or unit of work.
- External service interface.
- Event publisher.
- Feature flag or policy object.

## 12. Design For Context Early

If the domain has organizations, workspaces, tenants, regions, currencies, locales, plans, roles, or environments, make that context explicit.

Do not assume there is one global account, price, timezone, permission model, or storage location.

Prefer:

```python
def validate_member_invitation_limit(
    workspace_id: str,
    plan_id: str,
    requested_invites: int,
) -> None:
    ...
```

Avoid:

```python
def validate_member_invitation_limit(requested_invites):
    ...
```

Context that changes business behavior should appear in function signatures, data objects, or query filters.

## 13. Put Tests Near The Domain Behavior

Tests should follow the same domain organization as production code:

```text
billing/
  tests/
    test_actions.py
    test_calculations.py
    test_fetch.py
    test_payment_status.py
```

This makes it obvious where to add coverage when behavior changes.

Use different test levels deliberately:

- Domain helper tests for pure rules.
- Action tests for state transitions and side effects.
- Boundary tests for permissions, input shape, and response format.
- Query tests for performance-sensitive reads.
- Task tests for async behavior.

Do not test everything only through the outer API. That makes rule failures harder to locate and slower to run.

## 14. Let Boundaries Convert Errors

Different layers need different error formats.

Domain code can raise domain exceptions or return typed problems. The boundary should convert those into HTTP, GraphQL, CLI, queue, or SDK-friendly errors.

Example:

```python
try:
    document = get_document(document_id)
except DocumentNotFound as error:
    raise BoundaryValidationError(
        field="document_id",
        message="Document was not found.",
        code="not_found",
    ) from error
```

This keeps domain code reusable and keeps boundary errors consistent.

## 15. Avoid Architecture By Indirection

More layers do not automatically mean better architecture.

Add a boundary when it gives the codebase something concrete:

- A stable external interface.
- A reusable domain operation.
- A clear extension point.
- A controlled side-effect path.
- A typed read model.
- A performance-sensitive fetch path.
- A testable business rule.

Do not add a layer just to satisfy a pattern name.

Bad architecture hides behavior behind abstract names. Good architecture makes behavior easier to find.

## Suggested Repository Shape

For a Python backend, start with this shape and adapt it to the domain:

```text
project/
  core/
    db.py
    events.py
    exceptions.py
    types.py
  identity/
    actions.py
    models.py
    permissions.py
    tests/
  billing/
    actions.py
    calculations.py
    fetch.py
    models.py
    problems.py
    tasks.py
    tests/
  documents/
    actions.py
    fetch.py
    models.py
    permissions.py
    tests/
  reports/
    actions.py
    exports.py
    fetch.py
    tasks.py
    tests/
  api/
    billing.py
    documents.py
    reports.py
  integrations/
    email.py
    payments.py
    storage.py
```

The exact names do not matter. The boundary discipline matters.

## Architecture Review Checklist

Before merging a structural change, ask:

- Does this code live in the package that owns the business concept?
- Is the boundary layer only translating and orchestrating?
- Are business rules testable without the boundary layer?
- Is data loading centralized and intentional?
- Are side effects delayed until state is safe?
- Are extension points behind stable interfaces?
- Are problems and state transitions named explicitly?
- Is shared code genuinely shared?
- Are important dependencies visible in function signatures?
- Does the test location match the behavior being tested?
- Did this abstraction remove real complexity, or just add indirection?

## Summary

Clean Python architecture gives business behavior a clear home.

Organize by domain capability. Keep external interfaces at the boundary. Build typed read models for rule-heavy operations. Isolate integrations. Treat side effects carefully. Test behavior where it lives.

The goal is not to create a perfect diagram. The goal is to make the next change obvious.

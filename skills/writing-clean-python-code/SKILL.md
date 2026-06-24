---
name: writing-clean-python-code
description: Write explicit, maintainable, production-safe Python using domain language, early validation, normalized input, precise persistence, and behavior-focused tests. Use when writing or reviewing Python functions, validation, error handling, state transitions, side effects, or tests in backend services, APIs, workers, CLIs, or libraries.
---

# Clean Python Code Principles

A practical guide for writing Python code that is explicit, maintainable, and safe under real production pressure.

This guide is framework-neutral. The examples use common backend situations such as users, invoices, documents, reports, files, jobs, billing, and integrations. The principles apply equally to Django, FastAPI, Flask, CLIs, workers, libraries, and internal services.

## The Standard

Good Python code should make important behavior obvious.

It should use domain language, validate early, normalize input before core logic, model edge cases directly, write state precisely, and test behavior rather than implementation details.

Clean code is not code that looks short. Clean code is code where the next engineer can answer these questions quickly:

- What business action is happening?
- What inputs are accepted?
- What can go wrong?
- What state changes?
- What side effects happen?
- Which edge cases are intentional?

## 1. Use Domain Language

Name functions after the business action they perform.

Prefer:

```python
def approve_invoice(...):
    ...


def archive_document_revision(...):
    ...


def schedule_report_export(...):
    ...


def deactivate_user_sessions(...):
    ...
```

Avoid:

```python
def process(...):
    ...


def handle(...):
    ...


def do_update(...):
    ...
```

Generic names hide the rule. Domain names explain the rule before the reader opens the function.

Use names that include the object being changed and the action being performed:

- `invite_user_to_workspace`
- `revoke_api_key`
- `calculate_invoice_balance`
- `mark_report_as_stale`
- `create_file_upload_record`
- `retry_failed_webhook_delivery`
- `validate_project_membership_limit`

## 2. Separate The Flow

For a write operation, make the sequence visible:

1. Check permissions.
2. Validate raw input.
3. Fetch required objects.
4. Normalize input into internal data structures.
5. Validate business rules.
6. Mutate state.
7. Save precise fields.
8. Trigger side effects after state is safe.

Example:

```python
def submit_expense_report(actor: User, payload: dict) -> ExpenseReport:
    require_permission(actor, Permission.SUBMIT_EXPENSES)
    validate_expense_payload(payload)

    employee = get_employee_or_error(actor.employee_id)
    policy = get_active_expense_policy(employee.organization_id)

    report_data = normalize_expense_report_input(payload)
    validate_expense_report(policy, report_data)

    report = create_expense_report(employee, report_data)
    update_fields = mark_employee_budget_as_pending(employee, report.total)

    employee.save(update_fields=update_fields)
    publish_expense_report_submitted(report)

    return report
```

This style makes every phase inspectable. It also gives tests clear places to target.

## 3. Normalize Input Before Business Logic

External input is often messy: dictionaries, JSON, form data, HTTP payloads, GraphQL input, CLI arguments, CSV rows, or untrusted data from another system.

Do not let that shape leak deep into business logic.

Prefer converting external input into a typed internal shape:

```python
from dataclasses import dataclass, field
from datetime import date
from decimal import Decimal


@dataclass
class ExpenseLineInput:
    category_id: str
    amount: Decimal
    currency: str
    spent_on: date
    description: str
    receipt_file_id: str | None = None
    tags: list[str] = field(default_factory=list)
```

After this point, code should work with `ExpenseLineInput`, not raw dictionaries.

Benefits:

- The rest of the code has one expected shape.
- Optional fields are explicit.
- Defaults are centralized.
- Tests can target normalized behavior.
- Validation becomes easier to reason about.

## 4. Use Types To Explain Intent

Use Python typing deliberately. Type the parts that carry business meaning:

- Public functions.
- Helper functions with non-obvious input.
- Return values.
- Collections.
- Optional values.
- Domain data structures.

Prefer:

```python
def get_workspace_member(user_id: str, workspace_id: str) -> WorkspaceMember | None:
    ...


def calculate_invoice_total(lines: list[InvoiceLine]) -> Money:
    return sum_money(line.total for line in lines)
```

Avoid:

```python
def get_workspace_member(user_id, workspace_id):
    ...


def calculate_invoice_total(lines):
    ...
```

Typing is not decoration. It documents assumptions and gives tooling a chance to catch mistakes.

## 5. Prefer Domain Values Over Primitive Values

Business code should avoid careless primitives.

Prefer:

- `Decimal` for money-like arithmetic.
- Money/value objects when available.
- Enums for state.
- Dataclasses for internal records.
- Constants for repeated choices.
- `datetime` values with explicit timezone handling.
- Small value objects for identifiers, date ranges, quantities, and limits.

Avoid:

- Floats for currency.
- Raw strings for states.
- Magic numbers.
- Dicts passed through many layers.
- Timezone-naive datetimes in domain logic.

Example:

```python
from decimal import Decimal


if payment.amount > Decimal("0"):
    ...
```

## 6. Validate Early And Precisely

Validation should happen before mutation.

Good validation tells the caller:

- Which field failed.
- Why it failed.
- Which code identifies the failure.
- Which values caused the failure, if useful.

Prefer:

```python
raise ValidationError(
    field="due_date",
    message="Due date cannot be earlier than the invoice date.",
    code="due_date_before_invoice_date",
)
```

Avoid:

```python
raise ValueError("Bad input")
```

At boundaries, convert low-level exceptions into caller-appropriate errors:

```python
try:
    document = DocumentRepository.get(document_id)
except DocumentNotFound as error:
    raise ValidationError(
        field="document_id",
        message="Document was not found.",
        code="not_found",
    ) from error
```

## 7. Make Edge Cases First-Class

Do not hide important edge cases inside generic branches.

Name them:

```python
subscription_is_already_cancelled = (
    subscription.status == SubscriptionStatus.CANCELLED
)

if subscription_is_already_cancelled:
    return CancellationResult(status=CancellationStatus.NOOP)
```

This is better than:

```python
if status == "cancelled":
    return "noop"
```

Edge cases worth naming:

- Zero values.
- Empty input.
- Missing related objects.
- Duplicate input.
- Already-completed actions.
- Expired tokens.
- Permission-specific behavior.
- Stale cached data.
- Partial success.
- Idempotent repeated calls.
- Concurrent updates.

## 8. Keep Policy In Focused Helpers

A helper should usually answer one business question or perform one clear mutation.

Good helpers:

```python
def validate_workspace_member_limit(workspace: Workspace) -> None:
    ...


def user_can_approve_invoice(user: User, invoice: Invoice) -> bool:
    ...


def mark_report_export_as_ready(export: ReportExport, file_id: str) -> list[str]:
    ...
```

Poor helpers:

```python
def process_workspace_stuff(...):
    ...


def validate_and_update_everything(...):
    ...
```

Focused helpers make business rules easy to test and reuse.

## 9. Write To Storage Precisely

Database writes should be intentional.

Prefer:

```python
invoice.status = InvoiceStatus.APPROVED
invoice.approved_at = clock.now()
invoice.save(update_fields=["status", "approved_at", "updated_at"])
```

Avoid:

```python
invoice.status = "approved"
invoice.save()
```

Use the strongest persistence tool that matches the risk:

- Transactions for grouped writes.
- Row locks or optimistic version checks for concurrent updates.
- Bulk operations for many rows.
- Narrow update fields for model saves.
- Queryset updates for simple field changes.
- Idempotency keys for retryable commands.

Be explicit about what changed.

## 10. Order Side Effects Safely

Do not call external systems before the database state is committed.

For tasks, webhooks, emails, notifications, indexing, and event publishing, prefer after-commit behavior when inside a transaction:

```python
def publish_after_commit(callback: Callable[[], None]) -> None:
    if transaction_is_active():
        on_transaction_commit(callback)
    else:
        callback()
```

This prevents async jobs and external systems from observing state that later rolls back.

## 11. Fetch Data Intentionally

Fetching data is part of the code's behavior.

Use explicit query shape when object graphs are needed:

```python
reports = (
    ReportQuery()
    .include("owner")
    .include("schedule")
    .include("latest_export")
    .filter(workspace_id=workspace_id)
)
```

When building business objects, consider typed read models:

```python
@dataclass
class InvoiceApprovalInfo:
    invoice: Invoice
    customer: Customer
    approver: User
    open_disputes: list[Dispute]
    unpaid_balance: Money
```

This keeps data loading separate from rule execution.

## 12. Comments Should Explain Why

Use comments for:

- Transaction ordering.
- Concurrency.
- Backward compatibility.
- Deprecation.
- External system behavior.
- Non-obvious business rules.

Good:

```python
# Lock the account before creating ledger entries so concurrent payments cannot overspend the available balance.
```

Bad:

```python
# Save the account.
account.save()
```

If the code already says what happened, the comment should explain why it happened.

## 13. Prefer Readability Over Cleverness

Readable business code is allowed to be boring.

Prefer:

```python
if paid_amount <= zero_money:
    invoice.payment_status = PaymentStatus.UNPAID
elif paid_amount < invoice.total:
    invoice.payment_status = PaymentStatus.PARTIALLY_PAID
elif paid_amount == invoice.total:
    invoice.payment_status = PaymentStatus.PAID
else:
    invoice.payment_status = PaymentStatus.OVERPAID
```

Avoid compressing important rules into dense expressions:

```python
invoice.payment_status = next(
    status for rule, status in payment_rules if rule(paid_amount, invoice.total)
)
```

Abstractions are useful only when they make the rule easier to understand.

## 14. Test Behavior, Not Implementation Details

Tests should read like examples of business behavior.

Use descriptive names:

```python
def test_invoice_with_partial_payment_is_partially_paid(...):
    ...
```

Use `given / when / then` sections for non-trivial cases:

```python
def test_approving_invoice_publishes_approval_event(...):
    # given
    invoice = invoice_waiting_for_approval
    approver = finance_manager

    # when
    approve_invoice(invoice, approver)

    # then
    invoice.refresh()
    assert invoice.status == InvoiceStatus.APPROVED
    invoice_approved_event.assert_called_once()
```

Use parametrized tests for rule matrices:

```python
@pytest.mark.parametrize(
    ("total", "paid", "expected_status"),
    [
        (Decimal("100"), Decimal("0"), PaymentStatus.UNPAID),
        (Decimal("100"), Decimal("50"), PaymentStatus.PARTIALLY_PAID),
        (Decimal("100"), Decimal("100"), PaymentStatus.PAID),
        (Decimal("100"), Decimal("120"), PaymentStatus.OVERPAID),
    ],
)
def test_invoice_payment_status(total, paid, expected_status):
    ...
```

Assert negative side effects too:

```python
assert not invoice_approved_event.called
assert not reminder_email_task.called
```

## 15. Review Checklist

Before merging, ask:

- Are the names specific to the business domain?
- Is raw input normalized before core logic?
- Are optional values and missing objects handled explicitly?
- Are validation errors field-specific and useful?
- Are important edge cases named?
- Is persistence scoped to the intended fields?
- Are transactions and side effects ordered safely?
- Are data fetches intentional?
- Are comments explaining why, not what?
- Do tests cover state transitions and boundaries?
- Is the code straightforward enough for the next engineer to modify safely?

## Summary

Clean Python code is explicit code.

It does not depend on the reader guessing hidden rules. It names the business behavior, shapes the data before using it, validates before mutating, writes state carefully, delays unsafe side effects, and proves behavior with tests.

Short code is optional. Clear code is mandatory.

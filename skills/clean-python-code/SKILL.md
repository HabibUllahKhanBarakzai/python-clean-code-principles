---
name: clean-python-code
description: Write explicit, maintainable, production-safe Python using domain language, a consistent verb lexicon, guard clauses, contract docstrings, early validation, normalized input, precise persistence, and behavior-focused tests. Use when writing or reviewing Python functions, naming, validation, error handling, state transitions, side effects, or tests in backend services, APIs, workers, CLIs, or libraries.
---

# Clean Python Code Principles

A practical guide for writing Python code that is explicit, maintainable, and safe under real production pressure.

This guide is framework-neutral. The examples use common backend situations such as users, invoices, documents, reports, files, jobs, billing, and integrations. The principles apply equally to Django, FastAPI, Flask, CLIs, workers, libraries, and internal services.

## The Standard

Good Python code should make important behavior obvious.

It should use domain language, validate early, normalize input before core logic, model edge cases directly, write state precisely, and test behavior rather than implementation details.

What makes a codebase feel elegant is not any single technique — it is the consistency of small habits applied everywhere: one verb lexicon shared by every module, guard clauses that keep the happy path flat, docstrings that state the contract, call sites that read as prose — and no one-line wrapper functions that add a jump without adding meaning. Each habit is cheap on its own; applied uniformly, they compound into code that reads like it was designed by one mind.

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

## 2. Keep One Verb Lexicon Across The Codebase

Domain language alone is not enough — the same verb must mean the same thing everywhere. Once the reader learns the lexicon in one module, every other module is already familiar. This is one of the strongest drivers of code that feels elegant rather than accumulated.

A lexicon that works well:

- `get_*` — retrieve one object; returns `None` when missing.
- `get_*_or_error`, `*_or_error` — the raising variant; the failure behavior is part of the name.
- `fetch_*` — load and assemble a richer read model from several sources.
- `calculate_*` — pure computation; no writes.
- `recalculate_*` — recompute derived state and store it.
- `invalidate_*` — mark derived state as stale so it will be recomputed.
- `validate_*`, `check_*` — raise a domain error when a rule is violated.
- `is_*`, `has_*`, `can_*` — predicates returning `bool`.
- `add_X_to_Y`, `remove_X_from_Y` — mutations named as symmetric pairs.
- `publish_*`, `call_*` — emit events or notify observers.

Two habits make the lexicon powerful:

- The name states the failure mode. `get_user_checkout` returning `None` and `remove_promo_code_from_checkout_or_error` raising are both obvious from the call site alone.
- Operations come in symmetric pairs. If `add_voucher_to_checkout` exists, the reader can guess `remove_voucher_from_checkout` exists and what it does.

Pick the verbs once, write them down, and hold every new function to them.

## 3. Separate The Flow

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

## 4. Shape Functions With Guard Clauses

Handle the exceptional cases first and return early, so the happy path reads top-to-bottom at minimal indentation.

Prefer:

```python
def refresh_invoice_prices_if_expired(invoice: Invoice, *, force_update: bool = False) -> Invoice:
    if not force_update and invoice.prices_valid_until > clock.now():
        return invoice

    ...  # the expensive recalculation, unindented
```

Avoid wrapping the whole body in the positive condition:

```python
def refresh_invoice_prices_if_expired(invoice, force_update=False):
    if force_update or invoice.prices_valid_until <= clock.now():
        ...  # everything indented one level for the entire function
    return invoice
```

Order the guards from cheapest to most expensive: flag checks and field comparisons before queries, queries before external calls. A reader should be able to see in the first few lines when the function does nothing.

## 5. Normalize Input Before Business Logic

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

## 6. Use Types To Explain Intent

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

## 7. Design Call Sites For Readability

A function is written once and called many times, so optimize the call site.

- Force flags to be keyword-only with `*`, so calls read as prose instead of positional booleans:

```python
def invalidate_checkout_prices(checkout: Checkout, *, save: bool) -> list[str]:
    ...

invalidate_checkout_prices(checkout, save=False)  # not invalidate_checkout_prices(checkout, False)
```

- Make a flag required (keyword-only with no default) when neither choice is safe to assume — such as `save`, which decides whether the helper persists or leaves persistence to the caller.
- Let mutation helpers return what they changed, so callers can compose one precise write from several steps:

```python
updated_fields = invalidate_checkout_prices(checkout, save=False)
updated_fields += apply_voucher_to_checkout(checkout, voucher, save=False)
save_checkout_fields(checkout, updated_fields)
```

- Name type aliases for the keys of non-trivial mappings, so signatures explain themselves:

```python
VariantId = int
ChannelSlug = str

variant_stock_map: dict[tuple[VariantId, ChannelSlug], list[Stock]]
```

## 8. Prefer Domain Values Over Primitive Values

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

## 9. Validate Early And Precisely

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

## 10. Make Edge Cases First-Class

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

## 11. Keep Policy In Focused Helpers

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

Keep the module's public surface small: a public orchestrator can delegate its steps to `_private` helpers (`_append_line_to_update`, `_check_new_address`) that hold the details. The public function reads as a table of contents; the private helpers hold the paragraphs.

## 12. Do Not Create One-Line Functions

Splitting logic into helpers has a cost: every function is a jump the reader must make. A function whose body is a single self-explanatory line charges that cost and pays nothing back.

Avoid:

```python
def get_invoice_total(invoice: Invoice) -> Money:
    return invoice.total


def save_invoice(invoice: Invoice) -> None:
    invoice.save()


def user_is_active(user: User) -> bool:
    return user.is_active
```

These wrappers merely rename a line that was already readable. Inline the line at the call site and delete the function.

The test: read the body at the call site. If it is just as clear there, it belongs there. A helper earns its existence only when it names a business rule the expression does not state on its own:

```python
def invoice_is_overdue(invoice: Invoice, today: date) -> bool:
    return invoice.due_date < today and invoice.status != InvoiceStatus.PAID
```

Deleting `invoice_is_overdue` would lose a named policy; deleting `get_invoice_total` loses nothing. The rare acceptable one-liner is a domain primitive that fixes a codebase-wide convention — `zero_money(currency)`, `quantize_price(price, currency)` — which is shared vocabulary, not indirection.

The same applies to helper extraction during refactoring: do not carve a function into one-line fragments to make it "shorter". Extract a helper when it isolates a rule, a phase, or a reusable policy — not to relocate single lines.

## 13. Write To Storage Precisely

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

## 14. Order Side Effects Safely

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

## 15. Fetch Data Intentionally

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

## 16. Write Docstrings As Contracts

Give every public function a one-line imperative docstring stating what it does in domain terms:

```python
def invalidate_checkout_prices(checkout: Checkout, *, save: bool) -> list[str]:
    """Mark checkout as ready for prices recalculation."""
```

When behavior is conditional, expand the docstring into the parts of the contract a caller cannot see in the signature — precedence rules, staleness windows, when updates are skipped:

```python
def fetch_invoice_prices_if_expired(invoice: Invoice, *, force_update: bool = False) -> Invoice:
    """Fetch invoice prices with taxes.

    First apply all invoice prices with taxes separately, then apply
    external tax data as well if we receive it.

    Prices are updated only if force_update is True, or if the time elapsed
    since the last price update is greater than the configured prices TTL.
    """
```

Docstrings describe meaning and rules; implementation detail stays in the body. A docstring that says "loops over the lines and sums them" is noise; one that says "taxes included" or "uses the most beneficial applicable promotion" is the contract.

## 17. Comments Should Explain Why

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

## 18. Prefer Readability Over Cleverness

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

## 19. Test Behavior, Not Implementation Details

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

## 20. Lock Rows Deliberately

When concurrent writers can touch the same rows, locking discipline matters as much as the lock itself. These rules apply to any relational store, whatever the ORM or driver.

- Lock rows in a deterministic order, such as ordered by primary key, so two transactions cannot acquire the same rows in opposite order and deadlock.
- Lock only the rows being mutated, not every joined table (`SELECT ... FOR UPDATE OF <table>`; Django `select_for_update(of=...)`, SQLAlchemy `with_for_update(of=...)`).
- Centralize lock helpers in one module per domain so every caller locks the same way.
- Fetch the data used for decisions inside the transaction, after acquiring the lock. Data fetched before the lock may be stale.
- Never hold row locks across external calls. Release the lock, call the payment provider, then re-lock and re-validate state before continuing.

```python
# locks.py — the only place that defines how invoice rows are locked.
def lock_invoice_for_update(invoice_id: str) -> Invoice | None:
    return (
        InvoiceQuery()
        .filter(id=invoice_id)
        .order_by("id")
        .select_for_update(of="invoice")
        .first()
    )
```

```python
with begin_transaction():
    invoice = lock_invoice_for_update(invoice_id)
    if invoice is None or invoice.status == InvoiceStatus.PAID:
        return AlreadyPaidResult()
    # Fetch inside the lock so validation sees current state.
    lines = fetch_invoice_lines(invoice.id)
    ...
```

For values that several processes may update, write monotonically instead of blindly:

```python
def record_latest_payment_activity(invoice: Invoice, payment: Payment) -> None:
    if invoice.last_payment_at is None or invoice.last_payment_at < payment.processed_at:
        invoice.last_payment_at = payment.processed_at
        save_invoice_fields(invoice, ["last_payment_at"])
```

## 21. Emit Events On Transitions, Not States

Observers care that something *became* true, not that it *is* true. Firing on state produces duplicate events on every recalculation.

Capture the previous state before mutating, then compare:

```python
previous_status_is_fully_paid = previous_payment_status in FULLY_PAID_STATUSES
current_status_is_fully_paid = invoice.payment_status in FULLY_PAID_STATUSES

if not previous_status_is_fully_paid and current_status_is_fully_paid:
    publish_invoice_fully_paid(invoice)
```

This makes events idempotent under recalculation and safe under retries.

## 22. Denormalize With Explicit Staleness

Denormalized and cached fields are fine when the code admits they can be stale.

Patterns that work:

- Store an expiry timestamp next to cached derived values (`prices_valid_until`) and recalculate when it passes.
- Store a dirty flag next to derived data (`search_index_stale`) and let a background task rebuild it.
- When a derived value has several possible sources, document the precedence chain and encode it in one place:

```python
@cached_property
def unit_price(self) -> Money:
    """Return the unit price for the invoice line.

    Use the live price-list entry when one exists; fall back to the price
    snapshotted on the line when the entry was removed.
    """
    if self.price_list_entry is not None:
        return self.price_list_entry.unit_price
    return self.line.unit_price_snapshot
```

A denormalized field without an invalidation path is a bug waiting for a customer to find it.

## 23. Design Background Tasks For Batches And Reruns

A task that processes unbounded rows will eventually time out, exhaust memory, or overlap its next scheduled run.

Design tasks to be bounded, resumable, and idempotent, whatever the runner — Celery, RQ, Dramatiq, a cron script, or a worker loop:

- Iterate with keyset pagination (`id > last_seen_id`), never `OFFSET`, so progress is stable while rows change underneath the task.
- Cap work per invocation with a batch size and batch count, and let the task re-trigger itself with an invocation limit.
- Quantify the constants so the next engineer can retune them:

```python
# Batch size of 2000 uses roughly 27 MB of worker memory (~13.5 KB per row).
DELETE_BATCH_SIZE = 2000

# Export can take ~3s per report and the task runs every minute,
# so cap the batch to avoid overlapping executions.
EXPORT_COMPLETION_BATCH_SIZE = 20
```

```python
def iterate_expired_report_ids(batch_size: int) -> Iterator[list[int]]:
    last_id = 0
    while True:
        ids = fetch_expired_report_ids(after_id=last_id, limit=batch_size)
        if not ids:
            break
        yield ids
        last_id = ids[-1]
```

## 24. Give Each Domain An Error Code Enum

Field-level validation errors need stable, machine-readable codes. Keep them in one enum per domain so the error contract is discoverable and cannot drift:

```python
class InvoiceErrorCode(Enum):
    BILLING_ADDRESS_NOT_SET = "billing_address_not_set"
    CURRENCY_MISMATCH = "currency_mismatch"
    DUE_DATE_BEFORE_INVOICE_DATE = "due_date_before_invoice_date"
    INVOICE_ALREADY_APPROVED = "invoice_already_approved"
    LINE_QUANTITY_GREATER_THAN_LIMIT = "line_quantity_greater_than_limit"
    ZERO_QUANTITY = "zero_quantity"
```

```python
raise ValidationError(
    field="quantity",
    message="Cannot add more than the allowed quantity for this line.",
    code=InvoiceErrorCode.LINE_QUANTITY_GREATER_THAN_LIMIT.value,
)
```

Codes named after the business rule (`invoice_already_approved`) beat generic codes (`invalid`) because clients can react to them and support can search for them.

## 25. Review Checklist

Before merging, ask:

- Are the names specific to the business domain?
- Do function names follow the codebase verb lexicon, including the failure mode?
- Do guard clauses keep the happy path flat, with cheap checks first?
- Is every function more than a one-line wrapper — would deleting it lose a named rule?
- Is raw input normalized before core logic?
- Are optional values and missing objects handled explicitly?
- Are validation errors field-specific and useful?
- Do validation errors use stable domain error codes?
- Are important edge cases named?
- Is persistence scoped to the intended fields?
- Are transactions and side effects ordered safely?
- Are rows locked in a deterministic order, and never across external calls?
- Are events emitted on state transitions rather than on every recalculation?
- Do denormalized fields have an expiry, dirty flag, or other invalidation path?
- Are background tasks bounded, resumable, and idempotent?
- Are data fetches intentional?
- Do public functions carry contract docstrings in domain terms?
- Are comments explaining why, not what?
- Are flags keyword-only, and do mutation helpers return what they changed?
- Do tests cover state transitions and boundaries?
- Is the code straightforward enough for the next engineer to modify safely?

## Summary

Clean Python code is explicit code.

It does not depend on the reader guessing hidden rules. It names the business behavior in one shared verb lexicon, shapes the data before using it, validates before mutating, keeps the happy path flat, states each function's contract in its docstring, writes state carefully, delays unsafe side effects, and proves behavior with tests.

Elegance is consistency: the same verbs, the same function shape, the same call-site conventions, everywhere.

Short code is optional. Clear code is mandatory.

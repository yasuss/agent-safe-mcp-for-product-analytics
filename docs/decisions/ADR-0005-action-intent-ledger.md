# ADR-0005: Action-intent ledger with reconciliation

- Status: Accepted
- Date: 2026-10-24
- Requirements: HC07 · Acceptance: AC5

## Context

Durable workflow engines execute activities at least once [R5]. A write can commit on the
target while the response is lost. Some product and external APIs support idempotency
keys; others do not, and many reads are eventually consistent.

## Decision

The write-path object is the server-side **`ActionIntent`**, not the tool call:

```text
RESERVED -> DISPATCHED -> ACKNOWLEDGED -> VERIFIED
                     \-> OUTCOME_UNKNOWN -> RECONCILING -> VERIFIED | NEEDS_REVIEW
                     \-> FAILED
VERIFIED -> ROLLED_BACK (if supported)
```

Fields: intent ID; task, project, principal; capability and version; canonical argument
hash; provider idempotency key; policy decision and version; approval digest; dispatch
attempts; resulting resource ID and version; verification result; receipt timestamp.

1. Persist the intent before dispatch.
2. Dispatch with the provider idempotency key when supported.
3. Lost response → `OUTCOME_UNKNOWN`.
4. Reconcile by reading the target with a stable fingerprint/external reference.
5. Found → `VERIFIED`, return the existing receipt.
6. Retry the same intent only when absence is proven under provider-specific uniqueness or
   convergence rules — **one empty read is not proof of absence**.
7. Cannot establish the outcome (especially high-impact) → stop for review.

Reusing an idempotency key with different canonical parameters is rejected.

## Alternatives considered

- **Rely on workflow retries** — rejected: at-least-once execution duplicates side effects.
- **Idempotency key at the MCP boundary only** — rejected: the provider effect can commit
  before acknowledgement; idempotency must live at the actual effect boundary.
- **Claim exactly-once** — rejected: not achievable across arbitrary providers.

## Consequences

- Duplicate-effect rate under retries is a fault-tested metric with a target of zero.
- Each mutation family needs a reconciliation rule before it may run automatically.
- Every completed effect returns a user-facing `ActionReceipt`.

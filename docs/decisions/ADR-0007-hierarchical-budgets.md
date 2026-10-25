# ADR-0007: Hierarchical budgets with leases reserved before execution

- Status: Accepted
- Date: 2026-10-24
- Requirements: HC10 · Acceptance: AC3

## Context

Analytics scans, model calls and large tool responses are variable-cost. A model asking
again must not be a way to spend more. Scheduled tasks can create synchronized bursts.
Retries must not double-count or escape the envelope.

## Decision

- Budgets are **hierarchical**: per task, per user/task principal, per organization/project,
  per model provider, per query class.
- Dimensions: model cost/tokens, analytics queries, expensive-query class count, rows and
  context bytes returned, tool steps, wall-clock deadline, concurrency slots, external
  egress actions.
- A **`BudgetLease`** is reserved before an expensive step; after it, estimated vs actual
  usage is reconciled and the lease charged or released.
- On exhaustion: stop, degrade to a cheaper bounded query, or request a user-approved
  increase. The agent cannot raise its own limits.
- Scheduled jobs get jitter and fair scheduling.
- Thresholds are deployment configuration, not constants of the design.

## Alternatives considered

- **One global rate limit** — rejected: no attribution, no per-task envelope, noisy neighbours.
- **Model-token budget only** — rejected: analytics scans and response sizes often dominate.
- **Post-hoc billing alerts** — rejected: the cost is already spent.

## Consequences

- Cost per successful task becomes measurable and attributable.
- Conservative admission can block useful analysis; mitigated by progressive widening and
  showing remaining budget.

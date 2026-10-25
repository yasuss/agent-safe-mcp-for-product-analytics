# ADR-0006: Synchronous bounded reads, durable tasks on the existing workflow engine

- Status: Accepted
- Date: 2026-10-24
- Requirements: HC08, HC17 · Acceptance: AC9

## Context

Many agent requests are a single bounded query that should return in seconds. Others are
multi-step investigations, wait for approval, run on a schedule or perform effects whose
failure semantics need durable state. The platform already operates a durable workflow
engine (Temporal in the reference stack) and an AI/model gateway.

## Decision

**Fast synchronous path** when all hold: read-only; one or a few calls; predictable query
envelope; no approval; no durable retry needed.
`MCP/API → principal/scope → capability → policy → query → evidence → result`

**Durable task path** for multi-step investigations, multiple expensive queries, model
reasoning across evidence, any effecting write, approval pauses and scheduled work — run on
the **existing workflow engine**, not a new one.

```text
CREATED -> AUTHORIZED -> INVESTIGATING -> AWAITING_APPROVAL -> EXECUTING -> VERIFYING -> COMPLETED
Terminal/exception: FAILED | CANCELLED | BUDGET_EXHAUSTED | ACCESS_REVOKED | OUTCOME_UNKNOWN
```

MCP Tasks [R1] [R2] expose a durable handle, polling and `input_required` where clients
support them; otherwise a typed task handle with status/cancel/input tools. Backend
durability does not depend on the transport version.

Cancellation stops scheduling new steps; in-flight effects are reconciled
([ADR-0005](ADR-0005-action-intent-ledger.md)); cancellation is not rollback.

## Alternatives considered

- **Everything async** — rejected: adds latency and state to simple reads.
- **A second, agent-only workflow engine** — rejected: duplicates operations, tooling and
  on-call for the platform's existing engine.
- **No workflow engine (if one did not exist)** — a PostgreSQL-owned state machine with a
  transactional outbox and persisted timers is the alternative; the action ledger is needed
  either way.

## Consequences

- The workflow engine provides durability, timers and resume; the action ledger still owns
  side-effect safety.
- Queue partitioning and per-tenant fairness are added only when measured contention
  requires them ([open-questions.md](../open-questions.md)).

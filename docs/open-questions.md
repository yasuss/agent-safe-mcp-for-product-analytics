# Open questions

No open question changes the authority or effect semantics. The facts below are
deliberately **not assumed**; each has an explicit branch and a way to resolve it.

## Q1 — Production scale and SLO envelope

Unknown: concurrent tasks globally and per organization; typical and worst-case analytics
query duration; scheduled burst profile; interactive latency target; maximum
investigation latency.

Affects: queue partitioning, worker pool sizing, scheduler jitter, expensive-query
concurrency, caching.

Branches:
1. Modest volume — shared regional task queue and control state.
2. Sustained concurrency or tenant hotspots — partition by region + project hash with
   weighted per-tenant fairness.
3. Extreme analytics concurrency — isolate expensive-query admission and worker classes
   instead of scaling all tasks equally.

Does **not** change: project binding, deterministic authorization, action intent and
idempotency, approval semantics, audit.

## Q2 — End-to-end MCP Tasks support

The MCP `2026-07-28` specification offers a Tasks extension, but client support varies.

- Supported: map durable backend task IDs to MCP Tasks handles, status and `input_required`.
- Unsupported: return a typed task handle and expose status / cancel / input as tools and API.

Backend durability is identical in both branches.

## Q3 — Model-provider and residency policy per customer

Allowed providers, regions and subprocessors are customer- and deployment-specific.

- Provider allowed: minimize and redact context, log provider and data class, route via the
  model gateway.
- Provider disallowed: deny model egress; continue only with deterministic, non-LLM
  operations where meaningful.

## Q4 — Which Tier-2 actions ship first

The architecture supports exact confirmation for any Tier-2 family, but each family needs
product-specific blast-radius display, verification and rollback semantics.

**Resolved by:** enabling one capability family at a time in rollout Stage 3
([rollout-and-migration.md](rollout-and-migration.md)).

## Q5 — Provider reconciliation semantics

Some product and external APIs expose idempotency keys; others need fingerprint-based
read-back with convergence rules.

**Resolved by:** per-mutation-family fault tests (lost response, eventual consistency,
duplicate delivery) before that family is allowed to run automatically.

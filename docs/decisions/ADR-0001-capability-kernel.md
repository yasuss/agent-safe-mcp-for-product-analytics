# ADR-0001: Project-bound capability kernel with a durable task/action plane

- Status: Accepted
- Date: 2026-10-24
- Requirements: HC01, HC04, HC15, HC16, HC17

## Context

Agents reach the analytics product through several surfaces — an external MCP endpoint,
the public API, in-product assistants and the UI. The existing MCP surface already spans
harmless analytical reads and high-impact production changes (feature flags, experiments,
pipelines). Analytics exploration is open-ended, so a fixed list of workflows is not
enough; effects need retry safety, approval and audit; and each surface must not grow its
own safety rules.

## Decision

Introduce a **capability kernel**: server-owned, versioned, typed capability contracts that
every surface projects (MCP tools, API endpoints, UI actions, in-product agent tools). Each
capability declares schema, required permission, project scope, risk tier, reversibility,
idempotency/reconciliation rule, data and egress class, default budget and confirmation rule.

Around it, a **durable task/action plane**: deterministic policy at execution
([ADR-0002](ADR-0002-authorization-at-execution-boundary.md)), task scope binding
([ADR-0003](ADR-0003-task-scope-and-principals.md)), risk tiers and exact confirmation
([ADR-0004](ADR-0004-risk-tiers-and-exact-confirmation.md)), the action-intent ledger
([ADR-0005](ADR-0005-action-intent-ledger.md)), sync and durable paths
([ADR-0006](ADR-0006-dual-execution-path.md)), budgets
([ADR-0007](ADR-0007-hierarchical-budgets.md)) and evidence and evals
([ADR-0008](ADR-0008-evidence-and-outcome-evals.md)).

Offer both **intent-level capabilities** (`analytics.investigate_conversion_drop`,
`analytics.create_monitor_from_finding`) and **narrow primitives** (query a funnel, read an
insight, create one artifact) under the same authorization, budget and receipt semantics.

This is deliberately not a universal agent platform: logical components may share one
deployable service, and bounded reads stay on a thin synchronous path.

## Alternatives considered

| Option | Verdict |
| --- | --- |
| Current MCP controls only | Baseline; no shared approval, receipt or mid-task policy contract |
| Thin MCP/API façade with client-side confirmation | Kept for Tier-0 reads only; client prompts are not a server boundary |
| Intent-only workflow catalog | Kept for common workflows; insufficient for exploration |
| Separate universal agent OS | Duplicates existing workflow engine, gateway and cost attribution |
| Broad autonomous sandbox with coarse credentials | Excessive agency and egress blast radius |

Full analysis: [alternatives.md](../alternatives.md).

## Revisit when

- the platform adopts one task/capability service that already provides these semantics;
- the product becomes read-only;
- code execution across many external systems becomes the dominant workload;
- measured load shows the shared task queue is insufficient (layout changes, authority does not).

## Consequences

- One safety implementation for all surfaces; MCP annotations are generated from capability
  metadata for client UX only.
- Product teams own their capability definitions; a shared platform team owns policy,
  runtime schema and compatibility tests.
- Risk: capability metadata drifts from product semantics — mitigated by versioning, CI
  tests against underlying endpoints and pessimistic defaults for new actions.

# Implementing with Agent Reliability Runtime (ARR)

[Agent Reliability Runtime](https://github.com/yasuss/agent-reliability-runtime) is a
Python runtime for reliable tool-using agents. Its framework-neutral `core/` implements
and tests the semantics at the heart of this architecture — untrusted model and tool
content, trusted deterministic policy, exact approval, idempotent effect receipts,
postcondition verification, audit and deterministic evaluation — and exposes them through
structural ports
([`core/ports.py`](https://github.com/yasuss/agent-reliability-runtime/blob/v2.1.0/src/agent_reliability_runtime/core/ports.py),
[extension points](https://github.com/yasuss/agent-reliability-runtime/blob/v2.1.0/docs/extension-points.md)).

This repository has **no runtime dependency on ARR**. ARR is one implementation path: a
Python backend can build the capability kernel on ARR's core instead of re-deriving its
authority and effect semantics.

## How the two relate

The decisive rule of this design —
*the model proposes, the policy layer authorizes, the executor effects, the verifier proves
what happened* — is the execution model ARR already implements for a single agent. This
architecture extends it to a multi-tenant MCP surface. ARR 2.1 adds interface-only ports for
exactly that extension: task scope with separate principals and an authorization epoch,
budget leases, and risk-tiered capability metadata.

## Component mapping

| Component (this architecture) | ARR | Status in ARR | What remains to build |
| --- | --- | --- | --- |
| Capability Kernel | `CapabilityRegistry`, `CapabilityDefinition` with `risk_tier`, `egress_class`, `reversible` | **Planned** — interface only | Registry, per-capability schemas, MCP/API projections |
| Deterministic Policy | `PolicyEngine` (`risk_for`, `validate_model_args`, `propose`, `validate`) | Implemented | Four-tier rules, egress and data-class checks |
| ↳ execution-boundary scope check | `ScopeAuthorizer`, `TaskScope`, `PrincipalRef`, `AuthorizationDecision` | **Planned** — interface only | Permission ∩ token scope ∩ task grant ∩ policy, revocation epoch ([ADR-0002](decisions/ADR-0002-authorization-at-execution-boundary.md), [ADR-0003](decisions/ADR-0003-task-scope-and-principals.md)) |
| Approval Service | `Approver`; `core.approval.Approval` — digest of run, tool, normalized arguments and policy version; expiry; one-time `CONSUMED` | Implemented | Bind principal, project and resource version into the digest ([ADR-0004](decisions/ADR-0004-risk-tiers-and-exact-confirmation.md)) |
| Action Executor + Intent Ledger | `EffectStore` (`receipt_for`); receipts with replay detection in the PostgreSQL adapter | Implemented, narrow port | Intent states `DISPATCHED → OUTCOME_UNKNOWN → RECONCILING`, provider idempotency keys |
| ↳ lost-response reconciliation | `Reconciler`; `ReconciliationStatus` (`CONFIRMED_APPLIED`, `CONFIRMED_NOT_APPLIED`, `UNKNOWN_EFFECT_STATE`) | **Planned** — interface and value types | Fingerprint read-back with convergence rules per mutation family ([ADR-0005](decisions/ADR-0005-action-intent-ledger.md)) |
| Postcondition verification | `Verifier`, `RuleVerifier`, `VerificationStatus` | Implemented | Artifact-exists-with-intended-parameters rules |
| Budget / Rate Broker | `BudgetBroker`, `BudgetLease` | **Planned** — interface only | Hierarchical leases and counters ([ADR-0007](decisions/ADR-0007-hierarchical-budgets.md)) |
| Query & Evidence Service | `Retriever`, `core.evidence`, citations | Implemented (retrieval) | Typed analytics queries with project injection and admission limits |
| Model Gateway | `ModelProvider` | Implemented | Provider policy, redaction, cost attribution |
| Durable Task Runtime | `CheckpointStore` (LangGraph adapter); `TimerStore` planned | Restart durability implemented; timers planned | Task state machine on the platform's workflow engine ([ADR-0006](decisions/ADR-0006-dual-execution-path.md)) |
| Trace / Eval Pipeline | Deterministic eval harness, adversarial scenarios, OTel and audit adapters | Implemented | Task-outcome evals for diagnosis quality ([ADR-0008](decisions/ADR-0008-evidence-and-outcome-evals.md)) |
| MCP / API transport | ARR includes an MCP **client** boundary (agent → tools) | Different direction | MCP **server** projecting capabilities to external agents |

## Why the orchestrator does not matter

ARR's core does not import its orchestrator; LangGraph is an adapter. The same ports can be
driven from a Temporal workflow: activities call `ScopeAuthorizer`, `PolicyEngine`,
`Approver`, `EffectStore` / `Reconciler` and `Verifier`, while Temporal owns durability and
timers. This matches [ADR-0006](decisions/ADR-0006-dual-execution-path.md): reuse the
existing workflow engine, keep side-effect safety in the ledger.

## Suggested order on the ARR path

1. **Read-only pilot** — `ScopeAuthorizer` + `Retriever`-style query service with project
   injection; `BudgetBroker` for query admission; no writes.
2. **Tier-1 artifacts** — capability registry with risk tiers; effects through
   `EffectStore`; `Reconciler` for each artifact type; `Verifier` postconditions.
3. **Durable investigations** — task state on the workflow engine; revocation epoch checks
   after every pause.
4. **Tier-2 actions** — extend the approval digest with principal, project and resource
   version; one capability family at a time.

## Limits

ARR ships a fictional single-agent demo domain, LangGraph orchestration and no
multi-tenancy; the 2.1 scope and budget ports are interfaces without implementations. Its
guarantees apply to the adapters it ships: a new adapter must reproduce the same semantics —
a custom `EffectStore`, for example, does not inherit duplicate protection automatically.

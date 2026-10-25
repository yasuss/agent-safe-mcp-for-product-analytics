# Requirements

What the architecture must satisfy before any component is chosen. Every decision in
[architecture.md](architecture.md) and [decisions/](decisions/) traces back to an item here.

## Problem statement

AI agents — external MCP clients, API callers and in-product assistants — must be able to
query a customer's product analytics, ask follow-up questions, create or update analytics
objects and trigger safe workflows, while respecting permissions, cost limits, tenant
boundaries, long-running work, security, auditability, retries, cancellation and evals.

**Primary scenario:** *"Why did signup conversion drop this week, and if you find the cause,
create something useful to help me fix or monitor it."*

Constraints the design must survive: write retry after timeout; prompt injection in
customer data; an expensive query or run; access revocation mid-task; an action that
succeeded while its response was lost; cross-provider work; answer-quality evaluation;
and parity between UI and MCP.

## Hard constraints

| ID | Constraint |
| --- | --- |
| HC01 | The AI agent is an untrusted decision-maker, never the authorization authority; authorization is deterministic at the execution boundary. |
| HC02 | Support an interactive delegated-human principal and a scheduled/background task principal; record human, agent/client and task identities separately. |
| HC03 | Bind each task to one organization, project and region by default; no implicit cross-project or silent cross-region access. |
| HC04 | Expose stable typed capabilities with least-privilege discovery and filtering; tool availability is not authorization. |
| HC05 | Four action risk tiers: read automatic; reversible internal write automatic only within an explicit delegated capability; destructive / external / production-affecting requires deterministic human confirmation; forbidden / sensitive is denied. |
| HC06 | Automatic first-release writes are limited to low-blast-radius internal analytical artifacts; flag rollouts, destructive deletes, pipeline changes, external communication, code merges, billing/admin/security changes are not automatic. |
| HC07 | Every effecting action has idempotency semantics and a durable receipt, so retries and lost responses cannot duplicate mutations. |
| HC08 | Bounded reads may be synchronous; multi-step investigations, approvals and scheduled work run as durable tasks with restart, retry, cancellation and resume. |
| HC09 | Permissions and policy are re-evaluated at every execution boundary, so mid-task revocation prevents subsequent effects. |
| HC10 | Enforce query, model and compute cost and rate budgets with attribution per task, user, organization and project; reject or degrade expensive work before unbounded execution. |
| HC11 | Customer data, retrieved documents, tool results and external content are untrusted; prompt injection must not expand authority or egress. |
| HC12 | Keep access, session and task state region-affine; model egress requires the organization's AI-processing/provider policy plus minimization/redaction; no undeclared provider or region transfer. |
| HC13 | Distinguish observation and hypothesis from causal proof; never present correlation as an established cause. |
| HC14 | Evaluate end-to-end outcomes: answer and evidence quality, action correctness, safety, usefulness, reliability, latency and cost. |
| HC15 | UI, in-product agents and external MCP/API clients share one task/capability/policy backend rather than separate safety implementations. |
| HC16 | The first release covers the analytics product only; external providers such as GitHub or Slack are extension points behind the same capability/provider boundary. |
| HC17 | Prefer existing platform primitives (workflow engine, model gateway, product APIs) where they satisfy the requirements; do not invent precise scale numbers. |

## Acceptance criteria

Each criterion is executed by [`verification/reference_model.py`](../verification/reference_model.py).

| ID | Observable behaviour | Checks |
| --- | --- | --- |
| AC1 | A task cannot act outside its project or region. | `cross_project_denied`, `cross_region_denied` |
| AC2 | Revocation blocks the next effect, and an earlier approval does not override it. | `revocation_blocks_next_effect`, `approval_does_not_override_revocation` |
| AC3 | Expensive work is denied before dispatch; budget leases reconcile estimated vs actual cost as a hard cap. | `expensive_query_denied_pre_dispatch`, `budget_reserve_reconcile` |
| AC4 | Bounded reads are allowed; a delegated Tier-1 write executes automatically and leaves a verified ledger entry. | `sync_read_allowed`, `tier1_auto_write` |
| AC5 | A lost response after commit is reconciled without a second mutation; retries return the existing receipt; reusing an idempotency key with different parameters is rejected. | `lost_response_reconciled`, `verified_retry_deduplicated`, `idempotency_param_mismatch_rejected` |
| AC6 | Tier-2 actions wait for an exact confirmation that works once and is invalidated by changed parameters or resource version. | `tier2_requires_confirmation`, `exact_confirmation_allows_once`, `confirmation_cannot_be_replayed`, `changed_params_invalidate_confirmation`, `resource_version_change_invalidates_execution` |
| AC7 | Injected content cannot expand authority; undeclared egress is denied. | `prompt_injection_cannot_expand_authority`, `undeclared_egress_denied` |
| AC8 | A scheduled task principal has only its own narrow grant. | `service_principal_least_privilege` |
| AC9 | Cancellation stops future effects without pretending to undo committed ones. | `cancellation_blocks_future_effects` |
| AC10 | The suite is calibrated: a correct policy passes, a policy that trusts the model is caught, and digests are canonical. | `calibration_valid_policy_passes`, `calibration_broken_policy_rejected`, `metamorphic_canonical_param_order` |

## Traceability

| Requirement | Designed in | Decision | Acceptance |
| --- | --- | --- | --- |
| HC01 | architecture §3, §9, §11 | ADR-0002 | AC7, AC10 |
| HC02 | architecture §4 | ADR-0003 | AC8 |
| HC03 | architecture §5 | ADR-0003 | AC1 |
| HC04 | architecture §3 | ADR-0001 | AC7 |
| HC05 | architecture §9 | ADR-0004 | AC4, AC6 |
| HC06 | architecture §9, rollout | ADR-0004 | AC4, AC6 |
| HC07 | architecture §10, §13 | ADR-0005 | AC5 |
| HC08 | architecture §6 | ADR-0006 | AC9 |
| HC09 | architecture §4 | ADR-0002 | AC2 |
| HC10 | architecture §8 | ADR-0007 | AC3 |
| HC11 | architecture §11 | ADR-0002 | AC7 |
| HC12 | architecture §5, §11 | ADR-0003 | AC1, AC7 |
| HC13 | architecture §15 | ADR-0008 | — (eval suite) |
| HC14 | architecture §14 | ADR-0008 | — (eval suite) |
| HC15 | architecture §2–§3 | ADR-0001 | — |
| HC16 | architecture §17 | ADR-0001 | — |
| HC17 | architecture §6, §18 | ADR-0006 | — |

HC13 and HC14 are quality properties verified by the task-level eval pipeline
([architecture §14](architecture.md#14-evals-and-observability)), not by the deterministic model.

## Assumptions

- No production scale or SLO is assumed. Queue and partition layout follows measured
  concurrency ([open-questions.md](open-questions.md)); authority semantics do not depend on it.
- Model-provider residency rules are customer- and deployment-specific policy inputs.

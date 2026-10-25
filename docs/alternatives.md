# Alternatives

Six architecture options were evaluated against the [requirements](requirements.md).
The selected design and its revisit conditions are recorded in
[ADR-0001](decisions/ADR-0001-capability-kernel.md).

| Option | Status |
| --- | --- |
| O0 — Current MCP controls only (no build) | Baseline, insufficient |
| O1 — Thin MCP/API façade with client-side confirmation | Rejected for effecting actions; kept for Tier-0 reads |
| O2 — Intent-only workflow catalog | Insufficient as the sole surface; kept for common workflows |
| O3 — Separate universal agent OS | Rejected: duplicates existing platform primitives |
| **O4 — Project-bound capability kernel + durable task/action plane** | **Selected** |
| O5 — Broad autonomous sandbox with coarse credentials | Rejected for the analytics product; possible later for code work behind the boundary |

## O0 — Current MCP controls only

Use existing project pinning, read-only and tool filtering, OAuth and client-side tool
review; keep writes as direct product API/MCP operations.

**Preserves:** current stack, very low migration cost, good read-only ergonomics.
**Fails:** safety depends on client review behaviour. There is no shared exact-action
approval contract, no effect receipt or reconciliation contract, no durable mid-task
policy model and no uniform UI/MCP action semantics. A useful baseline, not a solution.

## O1 — Thin MCP/API façade with client-side confirmation

Expose product CRUD/query APIs nearly one-for-one as MCP tools and rely on MCP client
approval prompts for writes.

**Attractive:** tiny server, maximum API coverage, minimal new state.
**Rejected for effects:** client approval is not a uniform server boundary; the same
backend action can be safe in one client and unsafe in another. It cannot solve mid-task
revocation, cost attribution, durable approval state, duplicates after ambiguous timeouts,
or UI/MCP parity. Its simplicity is kept for Tier-0 bounded reads.

## O2 — Intent-only workflow catalog

Expose only high-level tools such as `investigate_conversion_drop`, `monitor_signup` or
`prepare_flag_rollback`, with no generic analytical primitives.

**Attractive:** safest and easiest to evaluate; stable happy paths.
**Insufficient alone:** exploratory analytics is open-ended. Agents lean heavily on schema
discovery and SQL-style exploration; forcing every question into a predeclared intent
causes tool proliferation, product-team bottlenecks and worse answers for novel questions.
Intent tools stay, but not as the only surface.

## O3 — Separate universal agent OS

A new workflow engine, policy service, model gateway, sandbox plane, event store and
universal connector runtime dedicated to agents.

**Attractive:** clean separation, maximal theoretical extensibility.
**Rejected:** duplicates the platform's existing durable workflow engine, model gateway,
cost attribution and sandbox/egress work. It adds migration and operational surface
without solving any requirement the current stack cannot.

## O4 — Project-bound capability kernel + durable task/action plane — selected

One semantic backend for UI, in-product agents and MCP: versioned capability contracts,
deterministic execution policy, exact-action approvals, durable action receipts with
idempotency and reconciliation, query and cost admission, evidence and eval contracts.
The existing durable workflow engine runs long work; the existing model gateway owns model
policy and cost. Bounded reads stay synchronous. Both exploratory primitives and
higher-level intent tools are offered.

**Why it survives:**
- keeps the read path simple;
- trusts neither MCP annotations nor client prompts as authority;
- does not equate workflow durability with exactly-once effects;
- stays useful for exploratory analytics;
- reuses existing gateway, task and product primitives;
- treats external providers as adapters, not first-release dependencies.

## O5 — Broad autonomous sandbox with coarse credentials

Give the agent project credentials and a networked sandbox; rely on model guardrails and
retrospective audit.

**Attractive:** maximum autonomy and integration freedom; natural when code changes dominate.
**Rejected:** combines private customer data, untrusted content and broad effect capability
— excessive agency, a large injection and egress blast radius, weak deterministic tenant
guarantees, and expensive evaluation. A sandbox remains useful later for provider-specific
code work, behind the capability boundary.

---

## Critical trade-offs

| Trade-off | Resolution |
| --- | --- |
| Generic CRUD vs intent-level tools | Hybrid: intent tasks for common workflows, narrow primitives for exploration |
| Sync vs async | Per operation: bounded reads sync; investigations, effects, approvals, schedules durable |
| Model autonomy vs deterministic policy | Model proposes; deterministic server policy authorizes |
| Context richness vs minimization | Evidence objects and aggregates first; raw sensitive data only when capability and policy require it |
| Flexibility vs evaluability | Stable task/capability/action contracts around an open-ended reasoning core |
| Exactly-once vs retry/reconciliation | Never promise generic exactly-once; persist intent, use provider idempotency, reconcile, then retry or escalate |

## Challenge reviews

**Main disagreement — intent-only (O2) vs hybrid (O4).** O2 argues that only intent tools
are safe and evaluable. Agents in practice favour schema discovery and SQL-style
exploration, while security guidance warns that broad data tools leak across tenants when
identity filters are model-controlled. **Resolution:** keep exploration, but move project
identity outside model parameters and enforce read-only query admission, result limits,
budgets and provenance; use intent and action tools where stable semantics or side effects
matter.

**Challenging the shared assumption.** "If a tool is marked read-only / destructive /
idempotent, or the client says the user confirmed, the server can trust it." MCP tool
annotations are explicitly hints, and OWASP requires authorization at the executing
component for the current actor and exact call. **Repair:** risk metadata is server-owned
policy input, never authority; every effect is re-authorized; Tier-2 approval is bound to
the exact action, parameters, resource and version, and is one-shot.

**Other mechanisms reviewed.**
- MCP `2026-07-28` is stateless at transport level and offers Tasks as a progressive
  enhancement, so application durability belongs behind an explicit task handle, not
  hidden transport session state.
- Prepare/execute pairs for sensitive operations are an established MCP pattern; the
  exact-action contract generalizes it.
- Large tool catalogs need filterable, on-demand discovery rather than preloading every schema.
- Response-size tails can dominate context cost, so compact result contracts belong in the
  query/action plane, not only in the model gateway.

No omitted family meets the hard constraints with less complexity than O4.

**Simplicity review.** Removed from the leader: a second workflow engine; an agent-only
analytics database; a general network sandbox in the analytics path; LLM-based
authorization or risk classification; a new event bus; mandatory MCP Tasks support;
a cross-provider orchestrator. Kept because removing them breaks a requirement:
project-bound trusted execution context; versioned capability metadata; execution-boundary
policy; durable action intent, receipt, idempotency and reconciliation; exact-action
approval for Tier 2; query, model and response budgets; evidence provenance and task evals.

Each added component is tied to a failure mode: the capability registry unifies surfaces
and carries risk/scope; the policy engine exists because neither model nor client can be
authority; the durable runtime is used only for long, approval or scheduled work; the
action ledger exists because durable retries duplicate side effects; the budget broker
exists because analytics and model work are variable-cost; receipts and evals exist
because a successful tool call is not a successful outcome. Several logical components can
share one deployable service in the first release.

## Repair loop

Defects found in early versions of the design and how they were fixed:

1. **Risk from MCP annotations** → annotations are untrusted hints → server-owned registry
   and policy taxonomy.
2. **Idempotency key at the MCP boundary** → the provider effect can commit before
   acknowledgement → idempotency and reconciliation at the actual effect boundary, plus a
   durable `OUTCOME_UNKNOWN` state.
3. **Approval bound to tool + params** → the resource can change after approval → include
   resource version and policy version; re-authorize and compare immediately before execution.
4. **One tool style** → intent-only harms exploration, raw-only harms safety → hybrid
   bounded reads plus typed actions and intents.
5. **Budget covers model tokens only** → analytics scans and huge tool responses dominate
   cost and context → multidimensional budget: model, query, compute, tool calls,
   response/context size, steps, wall time, external actions.

## Break conditions

Revisit the selected design if:
- the platform adopts a single existing task/capability service that already provides
  these policy and receipt semantics — collapse onto it instead of duplicating;
- the product becomes read-only — the action ledger and confirmation layer leave the read path;
- the dominant workload becomes code execution across many external systems — a
  sandbox-first provider runtime may become primary;
- measured load shows a single shared task queue is insufficient — partitioning and
  fairness change, the authority model does not.

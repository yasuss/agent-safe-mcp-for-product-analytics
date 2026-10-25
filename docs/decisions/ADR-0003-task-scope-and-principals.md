# ADR-0003: Immutable task scope and separate principals

- Status: Accepted
- Date: 2026-10-24
- Requirements: HC02, HC03, HC12 · Acceptance: AC1, AC8

## Context

Projects are data silos and every analytics query is project-specific. Customers choose a
data region. Some tasks are interactive (a human through an MCP client); others are
scheduled watchers that run when nobody is present. Audit must show who asked, which
agent client acted and which task identity executed.

## Decision

- **Three identities are stored separately:** `human_principal_id`, `agent_client_id` /
  session, and an optional `task_principal_id`.
- **`TaskScope` is immutable** for a normal task: organization, project, region, principal
  reference, capability grants, allowed resource IDs. The executor injects project scope
  outside the model-controlled argument object; the agent never passes a free-form project
  ID downstream.
- **Cross-project work** is a separate explicit capability with a project allowlist and
  per-project authorization and provenance — never inferred from conversation.
- **Region** is resolved at authentication/task creation and stays region-affine; no silent
  US↔EU fallback. Model egress is a separate decision: organization AI-processing policy +
  allowed provider/model + minimization/redaction + audit of the data class sent.
- **Scheduled tasks** get a dedicated task principal: exact org/project/region, a finite
  capability allowlist tied to the configured purpose, expiry and revocation epoch, an
  accountable owner, cost/rate ceilings — no inherited blanket human token.

## Alternatives considered

- **Model-supplied project IDs with server validation** — rejected: one validation bug
  becomes a cross-tenant leak; scope should not be an argument at all.
- **Scheduled tasks reuse the creator's token** — rejected: excessive privilege that
  outlives the user's intent.

## Consequences

- Cross-project leakage has a zero-tolerance eval and is structurally hard to express.
- Receipts can attribute every effect to human, client and task identities [R9].

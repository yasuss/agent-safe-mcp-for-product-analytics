# ADR-0004: Four risk tiers and exact, one-shot confirmation

- Status: Accepted
- Date: 2026-10-24
- Requirements: HC05, HC06 · Acceptance: AC4, AC6

## Context

Creating a dashboard and changing a production feature-flag rollout are both "writes",
but their blast radius differs by orders of magnitude. Confirming every write causes
approval fatigue; confirming none is unsafe; a generic "approve" can be reused for
something the user never saw.

## Decision

| Tier | Class | Rule |
| --- | --- | --- |
| 0 | Read | Automatic if scope, data and budget policy allow |
| 1 | Reversible internal analytical write (insight, dashboard, annotation, alert, draft) | Automatic only if explicitly delegated; requires action intent, receipt, undo path, postcondition verification |
| 2 | Destructive, external or production-affecting (flag rollout, experiment mutation, delete, pipeline change, external message, code merge, billing/admin/security) | Exact human confirmation |
| 3 | Forbidden / sensitive (scope escalation, credential exfiltration, disallowed egress, implicit cross-project action) | Denied |

A Tier-2 confirmation is a trusted UI/MCP input request generated from the server-side
`ActionIntent`, not a model-authored paragraph. It shows target project and resource, the
exact mutation/diff, motivating evidence, blast radius, external destination and data
class, and material cost. The approval binds
`action_digest + principal + project + resource version + policy_version + expiry` and is
**consumed on use**. Any change to arguments or resource version invalidates it.

## Alternatives considered

- **Confirm every write** — rejected: fatigue; users rubber-stamp.
- **"Approve all writes" grants** — rejected: unbounded; only narrow reusable grants per
  capability/resource envelope.
- **Rely on the MCP client's approval UI** — rejected: varies by client, not verifiable server-side.

## Consequences

- First release automates only Tier 0–1; Tier-2 families are enabled one at a time with
  product-specific blast-radius display and rollback semantics.
- Unknown or new capabilities default to the more restrictive tier.

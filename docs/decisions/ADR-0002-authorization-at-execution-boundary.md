# ADR-0002: Deterministic authorization at every execution boundary

- Status: Accepted
- Date: 2026-10-24
- Requirements: HC01, HC09, HC11 · Acceptance: AC2, AC7, AC10

## Context

The model reads customer-controlled text (event properties, error messages, replay
content, tool results) that can contain prompt injection. MCP tool annotations are
explicitly hints [R3]; client-side approval prompts differ by client. Users can lose access
while a multi-step task is still running.

## Decision

- Authority is computed server-side, deterministically, **immediately before every query
  and every effect** — not once at admission.
- Effective capability = current human/project permission ∩ OAuth/token scope ∩ task grant
  ∩ policy/risk rule ∩ current budget, region and egress policy.
- Customer, retrieved and model content is absent from all authority inputs. The model
  never controls project scope, capability grants, credentials, policy metadata or egress.
- Dynamic tool registration or permission changes inside a task are not allowed.
- After an approval pause or a worker restart, the executor re-evaluates before the next
  side effect. A revoked user or task becomes `ACCESS_REVOKED`; an earlier approval never
  overrides revocation.
- Prompt-injection filters and detectors are defense in depth, not the boundary.

## Alternatives considered

- **Trust MCP annotations / client confirmation** — rejected: hints and per-client UX,
  not enforcement.
- **LLM-based risk classification** — rejected: authority becomes probabilistic and
  injectable [R4].
- **Authorize once at task creation** — rejected: misses mid-task revocation and policy changes.

## Consequences

- A hijacked model can produce a bad *proposal*, never an unauthorized *effect*.
- The reference model includes a negative calibration: a policy that trusts a
  model-supplied "authorized" marker is detected as violating scope.

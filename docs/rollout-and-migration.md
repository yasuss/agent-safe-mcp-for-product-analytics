# Risks, rollout and operational ownership

Related: [ADR-0001](decisions/ADR-0001-capability-kernel.md) · [failure-modes.md](failure-modes.md) · [open-questions.md](open-questions.md).

## Principal risks

### 1. Capability metadata drift
A capability can claim “reversible” or low risk while underlying product semantics evolve.

Mitigation: product-team ownership; versioned capability definitions; CI tests against underlying endpoints; default pessimistic risk on unknown/new actions; periodic audit of Tier-1 mutations.

### 2. Provider idempotency gaps
Some product or external APIs do not expose idempotency keys.

Mitigation: deterministic `ActionIntent`, stable resource fingerprints/external references, provider-specific convergence/uniqueness rules for read-after-timeout reconciliation, and `OUTCOME_UNKNOWN` rather than blind retry. A single empty read is not proof of absence under eventual consistency. For irreversible Tier-2 actions without safe reconciliation, no automatic retry.

### 3. Approval fatigue
If too many actions are Tier-2, users approve mechanically.

Mitigation: automatically permit only genuinely low-blast-radius reversible internal artifacts; present exact diffs; allow explicit reusable grants only for narrow capability/resource envelopes; never “approve all writes”.

### 4. Prompt-injection bypass of probabilistic filters
Detection can fail.

Mitigation: assume failure. Keep authority/project/credentials/egress outside model control. The deterministic boundary remains effective even if the model is hijacked.

### 5. Confused-deputy / permitted-but-off-purpose actions
A hijacked model may still attempt an action that is technically within a broad user permission.

Mitigation: task grants are purpose/capability-bound from trusted user/admin configuration; narrow autonomous-effect quotas; resource constraints; current authorization at execution; Tier-2 exact confirmation. Retrieved customer/tool content cannot widen the grant.

### 6. Budget false positives / blocked useful analysis
Conservative admission may stop a legitimate investigation.

Mitigation: progressive widening: start bounded, show remaining budget, request a larger budget or narrow the hypothesis space; tune from production traces.

### 7. Platform tax
A central capability framework can slow product teams if every tool requires infra-team work.

Mitigation: self-service typed definitions and generated adapters; product teams own product capabilities; shared infra owns policy/runtime schema and compatibility tests.

### 8. Audit data becoming sensitive
Traces can accidentally retain raw PII/secrets.

Mitigation: structured receipts, redaction/data classification before persistence, separate access policy/retention, never log credentials.

## Staged rollout

### Stage 0 — Read-only pilot
- project-bound OAuth;
- capability discovery/filtering;
- bounded analytics reads;
- evidence/provenance;
- budgets and task traces;
- no writes.

Exit: project isolation, query-budget enforcement and answer/evidence evals meet target quality.

### Stage 1 — Tier-1 internal artifacts
Enable insight/dashboard/annotation/monitor creation through `ActionIntent` + postcondition verification + receipt.

Exit: retry/lost-response duplicate rate is effectively zero in fault tests; artifact keep/revert rate is acceptable.

### Stage 2 — Durable/scheduled investigations
Enable scheduled watchers and multi-step tasks with cancellation/resume and task principals.

Exit: restart/retry/expiry/revocation tests pass; cost per successful task is understood.

### Stage 3 — Tier-2 production/external actions
Add exact confirmation and blast-radius UX for selected operations, one capability family at a time.

Exit: security red-team/evals, action verification and rollback/reconciliation are product-ready.

### Stage 4 — External providers
Add Slack/GitHub adapters under the same capability/policy/action contract. Do not broaden network access globally.

## Rollback strategy

- Capability kill switch by ID/version.
- Organization-level agent write disable.
- Provider adapter disable.
- Force `READ_ONLY` mode without changing MCP endpoint.
- Stop new durable tasks while preserving task/receipt history.
- For reversible artifacts, expose receipt-linked undo.
- Tier-2 effects rely on the underlying product's rollback semantics; do not claim universal compensation.

## Operational ownership

Suggested boundary:
- Shared agent-platform team: task contract and runtime, model gateway, budgets, shared policy and eval primitives.
- Product teams: capability definitions, product-specific verification and artifact semantics.
- Security/platform: policy baseline, egress/secrets, high-risk action classes.
- MCP/surfaces: transport adapters and capability projection, not a second authorization implementation.

## Unknown scale branch

Exact production concurrency and latency SLOs are unknown. Start with shared regional task execution if it meets measured load. Split by regional/project hash, weighted fairness and expensive-query admission pools only when telemetry shows contention. This is deliberately conditional rather than a fabricated scale story.


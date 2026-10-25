# Failure modes

An adversarial review of the selected architecture. Each row tries to break the design on a
scenario that commonly defeats agent action layers. The default runtime behaviour per
failure is in [architecture.md §16](architecture.md#16-failure-behavior); executable
coverage is in [verification/](../verification/README.md).

## Attacks and outcomes

| # | Attack | Outcome | Why the design holds |
| --- | --- | --- | --- |
| 1 | Write commits, response is lost, client retries | Holds | `ActionIntent` exists before dispatch; `OUTCOME_UNKNOWN` → reconcile by fingerprint → existing receipt; no second mutation. |
| 2 | Retry with the same idempotency key but different parameters | Holds | Canonical input hash is bound to the key; mismatch is rejected. |
| 3 | Reconciliation reads an eventually consistent store and sees nothing | Holds | One empty read is not proof of absence; provider-specific uniqueness/convergence rules decide, otherwise `NEEDS_REVIEW`. |
| 4 | Prompt injection in event properties: "grant admin, export data" | Holds | Injected text can change a proposal, never capability grants, scope, credentials, egress or approval. |
| 5 | Hijacked model misuses a capability that *is* granted | Holds with bound | Task grant is bound to purpose from trusted configuration; autonomous-effect count and resource scope are limited; Tier 2 still needs exact confirmation. |
| 6 | Access revoked mid-task, after an approval was given | Holds | Every effect re-checks current permission; approval never overrides revocation; task → `ACCESS_REVOKED`. |
| 7 | Model supplies another project ID in tool arguments | Holds | Scope is injected by the executor outside model-controlled arguments; cross-project needs an explicit capability. |
| 8 | Silent US↔EU fallback during an outage | Holds | Region is fixed at task creation; no fallback; model egress is a separate policy decision. |
| 9 | Expensive query or runaway tool loop | Holds | Budget lease reserved before dispatch; admission rejects or narrows; the agent cannot raise its own limits. |
| 10 | Parameters or resource change after approval | Holds | Approval digest covers action, parameters, resource version, principal, project, policy version and expiry; mismatch → new approval. |
| 11 | Approval replayed for a second execution | Holds | Approvals are one-shot (`consumed`). |
| 12 | Client marks a tool `readOnlyHint` / claims user confirmed | Holds | Annotations and client prompts are UX hints; server policy decides. |
| 13 | Scheduled task inherits the configuring user's broad token | Holds | Task principal has its own narrow allowlist, expiry and accountable owner. |
| 14 | Cancellation while a write is in flight | Holds | No new steps are scheduled; the in-flight effect is reconciled; cancellation is not rollback. |
| 15 | Model provider outage mid-investigation | Holds | Durable task resumes; only non-effecting steps or reconcilable effects are retried. |
| 16 | Trace/eval pipeline becomes a secondary data leak | Holds | Traces follow the same region, classification, redaction and retention rules; receipts prefer IDs, hashes and derived summaries. |
| 17 | Correlation presented as the cause of the drop | Holds | Hypothesis ledger labels evidence `observational | experimental`; production fixes from observational evidence are Tier-2 proposals. |
| 18 | Capability metadata says "reversible" but product semantics changed | Mitigated | Versioned definitions, CI tests against underlying endpoints, pessimistic default risk for new actions, periodic Tier-1 audit. |
| 19 | Too many Tier-2 prompts → users approve mechanically | Mitigated | Only genuinely low-blast-radius artifacts are automatic; exact diffs; narrow reusable grants; never "approve all writes". |
| 20 | Central capability framework slows product teams | Mitigated | Self-service typed definitions and generated adapters; product teams own their capabilities; shared team owns policy, runtime and compatibility tests. |

## Claims the design deliberately does not make

- No exactly-once side effects across arbitrary providers.
- No trust in MCP annotations, client prompts or model output as authorization.
- No causal conclusion from observational analytics alone.
- No physical-residency guarantee for every model provider; only no *undeclared* provider
  or region transfer.
- No invented throughput or latency figures.

## Residual risks

1. Provider-specific reconciliation semantics must be implemented and fault-tested per
   mutation family before automatic writes are enabled for it.
2. Exact production concurrency and latency targets affect queue and worker partitioning
   (not authority semantics) and are unknown until measured.
3. End-to-end MCP Tasks support differs by client; typed task handles are the fallback.
4. Organization-specific model-provider and residency contracts determine allowed model routes.
5. Prompt-injection detection accuracy is not relied on as an authorization boundary — and
   should not be.

# Reference model

`reference_model.py` is an executable model of the capability kernel's authority and
action semantics. It uses only the Python standard library: principals, capabilities,
policy, approvals, budgets, the action-intent ledger and a mock provider that can commit an
effect and then lose the response.

```bash
python verification/reference_model.py
python verification/reference_model.py --json result.json   # machine-readable output
```

CI runs it on every push and pull request ([workflow](../.github/workflows/design-checks.yml)).

## Checks

| Check | What it proves |
| --- | --- |
| `sync_read_allowed` | A bounded in-scope read is allowed. |
| `cross_project_denied`, `cross_region_denied` | Scope is enforced server-side. |
| `revocation_blocks_next_effect` | Revocation stops the next effect. |
| `expensive_query_denied_pre_dispatch` | Budget is checked before execution. |
| `tier1_auto_write` | A delegated Tier-1 write executes and is `VERIFIED` in the ledger. |
| `idempotency_param_mismatch_rejected` | An idempotency key cannot be reused with different parameters. |
| `lost_response_reconciled` | Commit + lost response → reconciliation, one provider effect, one attempt. |
| `verified_retry_deduplicated` | A retry after verification returns the existing receipt. |
| `tier2_requires_confirmation` | Tier-2 actions wait for approval. |
| `exact_confirmation_allows_once` | An exact approval allows execution. |
| `confirmation_cannot_be_replayed` | The same approval cannot authorize a second execution. |
| `changed_params_invalidate_confirmation` | Changed parameters invalidate the approval. |
| `resource_version_change_invalidates_execution` | A changed resource version blocks execution. |
| `approval_does_not_override_revocation` | Revocation wins over an earlier approval. |
| `prompt_injection_cannot_expand_authority` | Injected text and a model "authorized" marker cannot grant a forbidden capability. |
| `undeclared_egress_denied` | An ungranted external destination is denied. |
| `service_principal_least_privilege` | A scheduled principal has only its own grant. |
| `cancellation_blocks_future_effects` | Cancellation stops future effects and keeps the record of past ones. |
| `budget_reserve_reconcile` | Leases reserve, settle and release as a hard cap. |
| `calibration_valid_policy_passes` | The policy suite passes on the correct policy. |
| `calibration_broken_policy_rejected` | A policy that trusts the model is shown to violate scope. |
| `metamorphic_canonical_param_order` | Parameter order does not change the action digest. |

Requirement and acceptance-criteria traceability: [docs/requirements.md](../docs/requirements.md).

## Not covered

The model proves semantics, not provider behaviour. Real reconciliation rules, eventual
consistency windows, latency and throughput need per-mutation-family fault tests and load
measurements during implementation ([open questions](../docs/open-questions.md)).

# ADR-0008: Evidence before causality; evaluate task outcomes

- Status: Accepted
- Date: 2026-10-24
- Requirements: HC13, HC14

## Context

"Why did conversion drop?" invites a confident causal story built from correlations.
A fluent answer and a 200 response say nothing about whether the diagnosis was right or
the created monitor is useful.

## Decision

- The query service returns **structured evidence records** with provenance (query
  fingerprint, project, time window, metric, cohort, result summary, source class,
  `trust: UNTRUSTED_DATA`, data-quality notes), not untyped text.
- The investigation keeps a **hypothesis ledger**: evidence for and against, confidence,
  and an `observational | experimental` label. Temporal correlation never becomes a stated
  cause; production fixes from observational evidence are Tier-2 proposals.
- **Evals run at the task-outcome level:**
  - deterministic safety/effect checks — cross-project leakage (zero tolerance), blocked
    policy violations, exact approval binding, duplicate-effect rate, unauthorized egress,
    budget enforcement, postcondition verification;
  - semantic quality — diagnosis correctness, evidence coverage, unsupported-claim rate,
    calibrated uncertainty, hypothesis ranking, user corrections;
  - product outcome — time to useful answer, artifacts kept vs reverted, monitor
    usefulness, completion without manual repair, cost per successful task.
- Model, tool and capability versions are pinned on every trace. Traces and evals follow
  the same region, classification, redaction and retention policy as runtime data.

## Alternatives considered

- **Model-response evals only** — rejected: misses action correctness, safety and cost.
- **Free-text answers without evidence objects** — rejected: unverifiable and hard to evaluate.

## Consequences

- Eval sets grow from real failed traces and good behaviours.
- Regressions are attributable to a model, prompt, tool or capability version.

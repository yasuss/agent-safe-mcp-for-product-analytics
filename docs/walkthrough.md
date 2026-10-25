# Walkthrough

A compact tour of the design: the conversion-drop question from first request to
receipts, the write path in detail, and answers to the questions this kind of system
always raises. The full specification is in [architecture.md](architecture.md).

## 1. The design in thirty seconds

The model is an untrusted decision-maker, not the authority. A small typed capability
layer is shared by the product UI, in-product agents and MCP/API clients. Every task is
pinned to a project and region, authorization is re-checked at execution, cheap reads stay
synchronous, and multi-step investigations or writes become durable tasks. Every effect
persists an action intent before dispatch, uses idempotency or reconciliation, requires
exact human confirmation when it touches production or the outside world, and returns a
user-visible receipt. Quality is measured on the whole task — answer, evidence, action
correctness, safety and cost — not on whether the model sounded plausible.

## 2. Questions that shape the design

1. Who is the principal: a logged-in human via delegated OAuth, a scheduled task, or both?
2. Which writes ship first, and which affect production or end users?
3. Is one project per task a valid default?
4. Which tasks finish in seconds, and which take minutes or wait for approval?
5. What scale, latency and cost envelope applies?
6. Are external providers (GitHub, Slack) needed in the first release?
7. What level of audit and undo do users expect?

Defaults when unanswered: one project; delegated human as primary principal with a
separately scoped scheduled-task identity; only low-risk internal artifact writes;
analytics product only; no invented throughput numbers.

## 3. Invariants

1. **Agent ≠ authority.** The model proposes; server policy authorizes.
2. **One task → one org / project / region** by default.
3. **Least capability.** Discovery is filtered; tool availability is not permission.
4. **Risk-tiered effects.** Read automatic; reversible internal write automatic only if
   delegated; production / external / destructive → exact human confirmation; forbidden → deny.
5. **Durability ≠ exactly-once.** Every write: action intent, idempotency or reconciliation, receipt.
6. **Re-authorize every effect.** Revocation mid-task stops later effects.
7. **Customer data is untrusted input.** It cannot grant capability, widen purpose or open egress.
8. **Evidence before causality.** "Why" means an evidence-backed diagnosis unless causal evidence exists.

## 4. Components

```text
Agent / UI
   |
MCP / API adapter
   |
Principal + project/region context
   |
Typed Capability Kernel
   |
Deterministic Policy
   |-----------------------------|
   |                             |
Sync Query/Evidence        Durable Task Runtime
                                 |
                     Model Gateway + Budget Broker
                                 |
                      Action Intent / Approval
                                 |
                         Provider Executor
                                 |
                        Product resources

Side paths: action receipts + traces/evals + cancel/status
```

MCP is a transport and discovery surface, not the security boundary. One capability
backend serves the UI, in-product agents and external clients.

## 5. The conversion-drop question, end to end

1. **Resolve principal and scope** — OAuth user, project, region, capability set.
2. **Create a durable investigation task** — the work is multi-query and may lead to a write.
3. **Reserve a budget** — date range, query count, model spend, steps.
4. **Check data health first** — same funnel definition? timezone? ingestion anomaly?
   comparable period?
5. **Find the step that moved** — compare funnel stages week over week.
6. **Slice bounded cohorts** — browser, device, source, landing path, new vs returning,
   relevant flag cohorts.
7. **Pull adjacent evidence only through granted capabilities** — deploy annotations,
   error spikes, session replays, feature flags.
8. **Keep a hypothesis ledger** — evidence for and against, confidence,
   `observational | experimental`.
9. **Do not claim causality from correlation** — "Safari signup completion fell after
   deploy X" is evidence; "deploy X caused it" needs stronger support.
10. **Create a useful Tier-1 artifact** — for example an insight, dashboard and alert on
    the affected funnel and cohort.
11. **One `ActionIntent` per effect** — execute once, verify the artifact, return an
    `ActionReceipt`.
12. **Production fixes are proposals** — "roll back flag X" becomes a Tier-2 proposal with
    an exact diff and blast radius, paused for human confirmation.

## 6. The write path

```text
model proposal
 -> canonical ActionIntent
 -> deterministic auth/policy
 -> [approval if Tier 2]
 -> reserve idempotency/action record
 -> execute
 -> verify postcondition
 -> ActionReceipt
```

**If the write succeeds but the response is lost:**

- the intent already exists as `DISPATCHED`;
- with a provider idempotency key, reuse the same key;
- otherwise reconcile by reading the target with a stable fingerprint and the provider's
  convergence rules;
- if the dashboard or alert exists, mark the intent `VERIFIED` and return that receipt;
- retry the same intent only when absence is provable — one eventually consistent empty
  read is not proof;
- if a high-impact outcome stays ambiguous, stop for review instead of risking a duplicate.

Durable workflow state and side-effect semantics are deliberately separate.

## 7. Design Q&A

**Customer data contains "send the secrets elsewhere."**
Retrieved and tool text is data, not authority. The model holds no credential or network
grant and cannot widen the server-stored task purpose. Policy rejects ungranted egress;
secrets exist only inside the executor. Prompt filters are defense in depth, not the boundary.

**The user revokes access halfway through.**
Every effect re-validates permission. The next step returns `ACCESS_REVOKED` and effects
stop. Resuming requires a fresh authorization epoch.

**The agent launches a very expensive query.**
Admission and budget checks run before execution: maximum window, result size, steps,
query class, per-task and per-organization concurrency. The agent may narrow the query or
request more budget; it cannot raise its own limits.

**The same tool call is retried after a timeout.**
Same `ActionIntent`, same idempotency key. Unknown outcome → reconcile first. A durable
queue never implies exactly-once.

**The agent needs GitHub and Slack.**
Not in the core first release. They become provider adapters with their own
authorization, idempotency and risk semantics behind the same capability, policy and task
layer. Sending a Slack message or merging code is Tier 2 by default.

**How does cancellation work?**
Stop scheduling new steps and record the cancellation. In-flight external effects may
still complete, so they are reconciled; cancellation is not rollback.

**Generic tools or intent-level tools?**
Both. Intent tools for common workflows (`investigate_conversion_drop`); narrow primitives
for exploratory reads. Pure CRUD gives the model too much semantic freedom; pure intent
tools multiply and fail on the analytics long tail.

**Sync or async?**
A bounded read is synchronous. Multi-query reasoning, approval, scheduled work and effects
are durable tasks.

**What does a capability own?**
Schema and version, required permission, project scope, risk tier, reversibility,
idempotency/reconciliation rule, data and egress class, default budget and confirmation
rule. MCP annotations are generated from this metadata for client UX; policy never trusts
annotations as authority.

## 8. Evals and observability

The headline metric is **cost per successful task**, not HTTP 200 or "the model answered".

- answer correctness, evidence coverage, unsupported-claim rate;
- action correctness and postcondition verification;
- policy violations, cross-project leakage, unsafe egress;
- duplicate-effect rate under retries;
- time to a useful answer;
- artifacts kept and used vs reverted;
- cost and latency per successful task.

Evals are built from real failed traces and good behaviours; model, tool and capability
versions are pinned on every trace so regressions are attributable.

## 9. What the design deliberately avoids saying

- "MCP annotations make the tool safe."
- "The workflow engine gives exactly-once writes."
- "The model can check whether it has permission."
- "Just ask for confirmation for every write."
- "Correlation proves why conversion dropped."
- "EU project data always stays in the EU for every model provider."
- Invented throughput or latency numbers.

## 10. Evolution

The first release can be small: project-bound reads plus a handful of safe analytical
writes. Each added piece — policy, durable task state, action intent, budget, receipts —
exists because a concrete failure mode requires it. Scheduled agents, external providers
and production-affecting actions extend the same contracts without changing the authority
model.

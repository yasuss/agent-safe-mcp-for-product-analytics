# References

Primary sources behind load-bearing decisions. Retrieved October 2026.

| # | Source | What it supports | Used in |
| --- | --- | --- | --- |
| R1 | [MCP — 2026-07-28 specification release](https://blog.modelcontextprotocol.io/posts/2026-07-28/) | Stateless transport core, authorization hardening, `input_required`, Tasks extension. | architecture §6, §12; ADR-0006 |
| R2 | [MCP Tasks extension](https://tasks.extensions.modelcontextprotocol.io/) | Durable server-side task handles with polling and mid-flight input. | architecture §6; ADR-0006 |
| R3 | [MCP — Tool annotations](https://blog.modelcontextprotocol.io/posts/2026-03-16-tool-annotations/) | `readOnly` / `destructive` / `idempotent` / `openWorld` are hints, not trusted enforcement. | architecture §3; ADR-0002 |
| R4 | [OWASP — LLM06:2025 Excessive Agency](https://genai.owasp.org/llmrisk/llm062025-excessive-agency/) | Excessive functionality, permissions and autonomy as agent risks; least privilege and human approval for high-impact actions. | ADR-0002, ADR-0004 |
| R5 | [Temporal — Activity definition](https://docs.temporal.io/activity-definition) | Activities may run more than once and should be idempotent: durability does not give exactly-once effects. | architecture §10; ADR-0005 |
| R6 | [OWASP — AI agent and MCP security guideline](https://owasp.github.io/DevSecOpsGuideline/2-Process/2-2-Develop/2-2-5-AI-Agent-and-MCP-Security/) | Least agency, scoped credentials, deterministic policy, sandbox/egress controls, tool data as untrusted; authorization at the executing component. | architecture §11; ADR-0002 |
| R7 | [AgentSecBench (arXiv:2605.26269)](https://arxiv.org/abs/2605.26269) | Conflating data flow with authority as the core agent-security failure; provenance and capability enforcement. | architecture §11 |
| R8 | [InjecAgent (arXiv:2403.02691)](https://arxiv.org/abs/2403.02691) | Empirical evidence that indirect prompt injection compromises tool-integrated agents. | architecture §11; failure modes |
| R9 | [NIST NCCoE — Software and AI agent identity and authorization (2026)](https://www.nccoe.nist.gov/sites/default/files/2026-02/accelerating-the-adoption-of-software-and-ai-agent-identity-and-authorization-concept-paper.pdf) | Delegation of authority, binding agent and human identities, least privilege, tamper-proof audit. | architecture §4; ADR-0003 |

## Counter-evidence that shaped the design

- MCP itself gives no trusted tool-risk policy: annotations are hints.
- Client-side approval alone is insufficient: clients differ, and approvals can be socially engineered.
- Durable workflow retries do not imply exactly-once external effects.
- Generic SQL/CRUD maximizes flexibility but can over-broaden tenant and data authority.
- Intent-only tools maximize safety and evaluability but miss the analytical long tail.

These are why the selected architecture is a hybrid rather than the simplest wrapper or a
fully constrained workflow catalog.

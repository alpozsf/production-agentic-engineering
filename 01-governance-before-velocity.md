# Governance Before Velocity: What Actually Made Agentic Engineering Work in Production

Part 1 of the series “Production Agentic Engineering” — [series index](README.md)


In a recent project, I was brought into a major financial institution to fix a failing hybrid data architecture — on-prem collection feeding AWS processing, a 25-person engineering team locked in a permanent maintenance cycle, and delivery lead times measured in weeks. The mandate was platform modernization. What we ended up building was something else: a production agentic engineering organization, with AI agents doing real work in the delivery path of a regulated financial data platform.

It worked. But not for the reason most people assume.

The industry conversation about agentic engineering is almost entirely about velocity — how much faster teams ship when agents write the code. That conversation has it backwards. Velocity was the *output*. The input was governance, and it went in first, before any agent touched anything that mattered. Agentic velocity without structure is just faster chaos, and in financial data, faster chaos is a career-ending event.

This is the control model we built, in the order we built it.

![Figure 1: Three-Layer Deterministic Control Model for Agentic Workflows](Piece%201%20diagram.png)

## Layer 1: Version-Controlled Data Contracts

Before anything else, every pipeline got an explicit contract: column names, types, nullability, expected distributions. Written as code, versioned in Git, reviewed like code.

The contracts did two jobs. The first is the obvious one — when a contract broke, the pipeline failed loudly instead of silently degrading. In a financial data platform, a silent schema drift is worse than an outage, because an outage announces itself and bad data does not.

The second job is the one that made the agentic layer possible: a contract is a machine-readable definition of *correct*. Large language models are probabilistic systems. You cannot safely point a probabilistic system at production data without a deterministic definition of what the output must satisfy. The contracts were that definition. Every agent-generated transformation had a hard specification to be validated against — not a code reviewer's intuition, a spec.

## Layer 2: Self-Validating Pipelines

Contracts define correctness at the schema level. Data quality lives one level deeper. We used Deequ — the open-source, Spark-native quality framework — running inside our Glue jobs at the conform layer of the Medallion architecture. Completeness checks, value-range constraints, distinctness, referential integrity, evaluated against every dataframe before it was written to the target Iceberg table.

![Figure 2: Automated PyDeequ Data Contract Validation Layer inside AWS Glue](Piece%201%20Code.png)

Constraint violations were classified. Critical violations stopped the pipeline — no exceptions, no manual overrides. Non-critical violations produced a structured quality report.

That word "structured" is doing heavy lifting. Deequ's output is a machine-parseable artifact: which constraints, which columns, what the observed values were. Deterministic tools producing structured artifacts are what make AI agents useful rather than dangerous — an agent analyzing a quality report and proposing a remediation is operating on facts. An agent asked to eyeball a dataset and form an opinion is hallucinating with confidence. We built the entire agentic layer on top of the first pattern and prohibited the second.

## Layer 3: Enforced Rules Files

The last layer governed the agents themselves. Before agents generated production code, we defined the boundaries in enforced rules files in the development environment: naming conventions, required documentation, no hardcoded credentials, no schema changes without a corresponding contract update, minimum test coverage thresholds.

Two properties made this layer real rather than aspirational. The rules were *enforced by tooling*, not published as guidelines — an agent could not propose a schema change without touching the contract, which triggered contract review. And no agent output reached production without engineer sign-off in the PR process. Engineers shifted from being authors of every line to being reviewers and governors of generated code. That human transition was harder than any of the technical work — a distinct organizational problem in its own right, covered separately in this series.

## What Ran on Top

With the three layers in place, we deployed agents against three workflows: schema mapping for the legacy migration (the highest-ROI use case, examined in detail in the companion retrospective), scaffolding generation for Glue PySpark jobs, and metadata-driven maintenance orchestration for our self-managed Iceberg tables — compaction and snapshot lifecycle triggered by agent analysis of table metadata, executed by deterministic Spark jobs.

Note the shape of all three: the agent proposes or analyzes; deterministic systems and human reviewers dispose. Nothing probabilistic sat in the write path unsupervised.

## The Result, Defined Precisely

Lead time — measured from source-system intake to first production pipeline run — went from 2–4 weeks to under 2 days.

I am deliberate about defining that metric, because unanchored multipliers ("3x faster") have become noise. This one is mechanism-anchored: the two things that consumed those weeks were manual schema analysis and hand-built validation, and those are exactly what the contracts and the agentic mapping layer removed. The speed was not the agents typing faster than humans. It was the governance structure removing the need for humans to manually verify what the system now verified by construction.

That is the inversion worth sitting with. Governance is usually framed as the tax you pay on velocity. Built correctly, it was the mechanism *of* velocity — the guardrails were not slowing the car down; they were what allowed it to be driven fast.

The tooling landscape has moved since we built this — managed agent runtimes, standardized protocols for tool integration, native quality frameworks in the cloud platforms. All of it makes the plumbing easier. None of it changes the model: contracts first, deterministic validation second, enforced boundaries third, and only then the agents. Teams adopting the tooling without the model will rediscover why we built it in this order.

---
In this series:

1. Governance Before Velocity: What Actually Made Agentic Engineering Work in Production (this piece)



*[LinkedIn](https://www.linkedin.com/in/alpoz/) · [Substack](https://alpozsf.substack.com/s/production-agentic-engineering)*

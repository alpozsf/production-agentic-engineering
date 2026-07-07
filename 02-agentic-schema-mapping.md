# The Highest-ROI Agentic Use Case Nobody Talks About: Schema Mapping

When engineering leaders ask me where to start with agentic AI, they expect me to say code generation. It is the visible use case, the one every vendor demos. My answer is less glamorous: start with schema mapping. It was the highest-ROI agentic workflow we ran in production at a major financial institution, it produced the fastest team buy-in, and almost nobody writes about it.

This retrospective covers the problem it solved, the mechanism, and why this specific use case is structurally suited to agents in a way code generation is not.

## The Problem: Where Migrations Actually Bleed Time

We were executing a Strangler Fig migration — legacy hybrid architecture to a cloud-native AWS lakehouse on Apache Iceberg, Medallion architecture, with version-controlled data contracts defining every target table. (The governance model is documented in the companion retrospective, "Governance Before Velocity"; the contracts matter again here.)

The public conversation about migrations focuses on infrastructure — the cutover, the dual-running, the reconciliation. In practice, the slowest, most expensive step was upstream of all of that: understanding what the legacy schemas actually meant and mapping them to the target contracts.

The legacy estate looked like every legacy estate. Hundreds of fields per source system. Naming conventions from three eras of the company — `CUST_NM`, `customerName`, and `c_name_1` coexisting, sometimes meaning the same thing, sometimes not. Data dictionaries that were incomplete where they existed at all. Semantics that lived in the heads of engineers who had maintained these systems for a decade — and some who had left.

Mapping one source system meant a senior engineer spending 3–4 days cross-referencing schemas, sampling data to infer meaning, chasing down tribal knowledge, and hand-writing a mapping specification. Multiplied across dozens of source systems, this single activity was the critical path of the entire program. It is also miserable work — high concentration, low creativity, and the people qualified to do it are exactly the people needed elsewhere.

## Why This Problem Fits Agents Structurally

Before the mechanism, the reasoning — because the shape of this problem is what makes it work, and recognizing that shape is how you find the next use case.

Schema mapping is a *proposal problem over structured inputs with a deterministic acceptance test*. Three properties:

1. **The inputs are structured and complete.** Source DDL, target contract, data dictionary fragments, sample profiles. No ambiguity about what the agent gets to see — everything fits in context; nothing requires the agent to go exploring.

2. **The output is verifiable.** A proposed mapping either satisfies the target contract — types compatible, nullability respected, every required target field sourced — or it does not. The contract, already built as the governance layer, doubles as the acceptance test for agent output. This is the compounding return on governance-first: the same artifact that makes pipelines safe makes agent proposals checkable.

3. **The cost of a wrong proposal is near zero.** The agent proposes; nothing executes. A bad mapping suggestion wastes minutes of review. Compare that to agent-generated transformation code, where a subtle error can survive review and reach production. Risk asymmetry is everything in deciding what to hand an agent first.

Most "where do we apply AI" debates would improve if teams filtered candidates through those three properties instead of starting from what looks impressive.

## The Mechanism

The workflow, concretely.

The agent — Claude, in our implementation; the task is reasoning-heavy, not generation-heavy — received the legacy source schema, the target Medallion contract, whatever data dictionary material existed, and column-level profile statistics (distinct counts, null rates, value samples, format patterns) from our profiling stage.

It produced a structured mapping proposal: for every target field, the proposed source field or expression, a confidence rating, and — this was the part that changed the economics — its *reasoning*:

> `CUST_TYP_CD → customer_segment`: value domain `{01, 02, 03, 09}` in sample data matches the segment enumeration in dictionary fragment X; code `09` appears in 0.2% of rows and has no dictionary entry — flag for review.

That last clause is the point. The agent was explicitly instructed to surface what it *could not* resolve rather than paper over it. Low-confidence mappings and unexplained value domains came back flagged, not guessed. An agent that says "I don't know — a human needs to look at row pattern 09" is worth ten that produce confident, complete, subtly wrong output.

## The Human Boundary — Structural, Not Procedural

An engineer reviewed every proposal before it became a mapping specification. Approved, corrected, or rejected — field by field for the flagged items, spot-checked against sample data for the high-confidence ones.

Precision matters about what "review" meant, because this is where agentic programs quietly rot. The temptation, once the agent is right 90% of the time, is for review to decay into rubber-stamping. We made the boundary structural rather than procedural: the mapping specification was a version-controlled artifact requiring engineer sign-off in the PR process before any pipeline could consume it, and the pipeline's contract validation would fail loudly on any mapping the review had missed. The engineer was not a courtesy checkpoint; they were the accountable author of record, with the agent as a very fast, very thorough junior analyst that had done the cross-referencing overnight.

## The Result

Mapping a source system went from 3–4 days of senior engineering time to 3–4 hours — and the 3–4 hours produced a *better* artifact than the 3–4 days had: every mapping decision documented with reasoning, every ambiguity explicitly flagged and resolved rather than silently assumed, the whole specification version-controlled and auditable. In a regulated environment, that audit trail is not a nice-to-have; it is the difference between a migration you can defend to a regulator and one you hope nobody examines.

It was also the moment the team converted. Engineers are rightly skeptical of AI velocity claims about their own craft. But nobody defends manual schema archaeology. Handing the worst week of a migration to an agent — while keeping the judgment, the approval, and the accountability — did not threaten anyone's identity as an engineer. It gave them their time back. The goodwill from this one workflow carried the political capital for everything we automated after it.

Code generation gets the headlines. But for a data platform with a legacy estate and a migration ahead, the highest-ROI agent you can deploy is the one that reads schemas, not the one that writes code.

---
In this series:

1. Governance Before Velocity: What Actually Made Agentic Engineering Work in Production

2. The Highest-ROI Agentic Use Case Nobody Talks About: Schema Mapping.  (this piece)

3. Your Engineers Aren’t Afraid of AI. They’re Afraid of Being Reviewers.

4. *Forthcoming*

Alp Oz
Enterprise Data Platform Leader
*[LinkedIn](https://www.linkedin.com/in/alpoz/) · [Substack](https://alpozsf.substack.com/s/production-agentic-engineering)*

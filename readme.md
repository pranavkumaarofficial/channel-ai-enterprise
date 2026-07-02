# Channel AI

> **Status:** documentation only. The implementation is private; this repo is the case study.
> Built and run across 12 private pilot deployments with small and mid-sized teams.

![Status](https://img.shields.io/badge/status-documentation--only-blue)
![Lifecycle](https://img.shields.io/badge/lifecycle-case--study-lightgrey)
![Code](https://img.shields.io/badge/code-not%20public-important)

https://github.com/user-attachments/assets/99edbe86-9d74-4563-9033-60a66d59cc98

Channel AI is a multi-agent conversational analytics system. Non-technical users asked business questions over WhatsApp, in English or a regional language, and got back answers, plots, and full reports generated from their own data. No dashboard, no SQL, no analyst queue.

I built it, deployed it 12 times with real teams, and eventually shut it down as a venture. This repository documents the architecture, the design decisions, and what broke. The failures ended up being more useful than the product: they directly shaped my later research on lightweight graph-based context modeling (ICMLC 2026).

If you are evaluating or building a conversational analytics system in 2026, the findings below are the part worth reading.

---

## The problem, stated plainly

Every analytics tool assumes the user already knows three things: what question to ask, how the data is organized, and what the metrics actually mean. Outside of data teams, none of those hold.

The obvious fix, circa 2023-2024, was text-to-SQL with retrieval: embed the schema, retrieve relevant tables, let an LLM write the query. It demos beautifully and fails in production, and the gap is now well quantified. Top systems clear 90% execution accuracy on the academic Spider 1.0 benchmark, yet on [Spider 2.0](https://spider2-sql.github.io/), which uses real enterprise warehouses with thousands of columns, the best agent frameworks solved roughly 17% of tasks at launch. On [BIRD](https://bird-bench.github.io/), frontier models still sit near the 50-60% band. The pattern is consistent: accuracy collapses exactly when schemas get wide, terms get ambiguous, and business meaning stops living in column names.

My read after running pilots: SQL generation was never the hard part. The hard part is that "revenue," "active customer," and "margin" mean different things in different rooms of the same company, and none of those meanings are written down anywhere a retriever can find them.

Channel AI was a bet that separating intent understanding, business semantics, and data execution into distinct agents would preserve context better than stuffing everything into one prompt. That bet was partially right. The details are below.

---

## System design

The system ran as a graph of specialized agents, coordinated with LangGraph and the OpenAI Agents SDK. Graph-based control flow mattered because it gave me conditional routing, retries, and shared state as first-class primitives instead of prompt spaghetti.

**Intent and planning agent.** Interprets the request and decomposes it into analytical steps. Its most important job was refusing to proceed: if a request was underspecified ("how did sales do?"), it asked a clarifying question instead of guessing a query. Premature execution turned out to be the single fastest way to lose a user's trust.

**Context resolution agents.** Map business terms and KPIs to actual data using metadata, embeddings, and prior interactions with that team. This is where each deployment's tribal knowledge accumulated. It was also the least transferable component; more on that in the findings.

**Schema-aware SQL generation agent.** Generates and validates queries against the analytical layer. Deliberately boring by design. By the time a request reached this agent, ambiguity was supposed to be resolved.

**Analysis and summarization agents.** Produce persona-specific output. A founder and an operations lead asking the same question got different reports, because they act on different things.

**Orchestration layer.** Owns agent state, execution order, and fallbacks across conversational turns. Also the layer where the hardest unsolved problems lived: context accumulation, schema drift mid-conversation, and consistency when a thread touched multiple datasets.

### Data layer

Everything executed against a unified analytical layer on Apache Iceberg. Client data arrived as some mix of relational databases and flat files; Iceberg gave me one consistent schema surface over all of it and handled multi-million row analytical scans without special-casing each source.

**Architecture and workflow**

<img width="15302" height="9634" alt="Channel AI agent workflow: personas, business context, and metadata feeding the analytical planning loop" src="https://github.com/user-attachments/assets/0695c1c4-6d5d-472e-82f4-dbb4c0ba0149" />

<img width="14505" height="6757" alt="High-level flow from user intent through context resolution to data execution, with human-in-the-loop refinement" src="https://github.com/user-attachments/assets/42707d8d-2365-4d42-809c-65a6dc466661" />

### Interface: WhatsApp, on purpose

Reports, plots, and summaries were delivered in a chat thread the user already had open all day, plus a lightweight web view for browsing insight cards and generating reports on demand.

<img width="846" height="642" alt="Insight card interface with on-demand plots and reports" src="https://github.com/user-attachments/assets/15ba9681-efac-4a92-a6d9-e525653b2683" />

This choice did more for adoption than any model improvement I shipped. It also created problems that dashboards never have: answers get consumed asynchronously and forwarded out of context, users screenshot a number and treat it as permanent, and there is no hover-to-inspect. Trust had to be built into the message itself.

### Explainability

The system exposed a schema-level context view: for a given query, which tables, fields, and business entities were connected, and how they influenced the answer. Not used for inference, purely a diagnostic and audit surface. Users who could see why an answer said what it said forgave errors. Users who got a bare number did not.

<img width="1856" height="935" alt="Schema-level context view showing entity relationships for a query, next to the generated executive summary" src="https://github.com/user-attachments/assets/2ca13e86-395c-411e-aef5-d36b64f6e148" />

**Sample output:** an automatically generated sales report from pilot usage: [SALES REPORT.pdf](https://github.com/user-attachments/files/24309621/SALES.REPORT.pdf)

---

## What 12 deployments actually taught me

These are the findings I would want if I were building in this space today. Some of them cost me weeks each.

**1. Users cannot tell you what they want to know.** The most common first message was some version of "show me everything." Query execution is the wrong abstraction for this audience. The interactions that retained users were ones where the system proposed the next question: "orders dropped 18% in the south region, want the breakdown by distributor?" Guided exploration was the product. SQL was plumbing.

**2. Refusing to answer builds more trust than answering.** Early versions guessed when requests were ambiguous. Accuracy on those guesses was decent, maybe 70%, and it did not matter. One confidently wrong answer undid weeks of correct ones. Once the intent agent started asking "did you mean net of returns?" before executing, session depth went up. People trust a system that knows what it doesn't know.

**3. Business semantics are oral tradition.** In one pilot, "sales" meant booked orders to the founder, dispatched orders to operations, and paid invoices to accounts. All three were sure their definition was the obvious one. No retrieval system can find a definition that exists only in people's heads, so onboarding became a structured interview to extract KPI definitions, and the context agents were seeded from that. This does not scale, and as far as I can tell nobody in the market has truly scaled it either; the semantic layer vendors have just made the manual work more organized.

**4. Context accumulation is a liability, not an asset.** Long conversations degraded in a specific way: the system got more confident and less correct, because stale assumptions from turn 3 were still steering turn 30. Schema changes mid-pilot made this worse. Explicit state with expiry beat naive history stuffing every time. This is the single problem that pushed me toward research on explicit, lightweight graph representations of context instead of accumulation.

**5. Regional language support is a semantics problem wearing a translation costume.** Translating the interface was trivial. The failure was that analytics vocabulary itself assumes English-first business concepts. A Kannada-speaking distributor asking about "udhaari" is asking about informal credit outstanding, which mapped to three different columns depending on the client. Language exposed how much implicit knowledge every analytics stack assumes.

**6. The explainability view was the trust product.** Detailed above, but worth restating as a finding: auditability beat accuracy as a retention driver. I did not expect that.

**7. Multi-agent separation earned its complexity, in one specific way.** The value was not "agents are smarter." It was that failures became localized and diagnosable. When an answer was wrong, I could tell whether intent, context resolution, or SQL was at fault in minutes. In the earlier single-prompt prototype, every failure was a vibe.

---

## Where this sits in the 2026 market

The space has consolidated into a few recognizable approaches. Having built one of them, here is my honest map, including what each one still cannot do.

| Approach | Examples | What it does well | Where it breaks |
|---|---|---|---|
| Raw text-to-SQL wrappers | dozens of startups and OSS repos | Fast setup, great demos on clean schemas | The Spider 2.0 cliff. No semantic grounding, so wide real schemas kill it |
| BI-native agents | Databricks Genie, Snowflake Cortex Analyst, Looker + Gemini | Strong inside one governed platform; grounding on a semantic model cuts errors dramatically | Locked to pre-modeled data on one platform. Useless for the long tail of businesses whose data is not there |
| Semantic layer platforms | dbt Semantic Layer, Cube, AtScale | Consistent metric definitions, the right substrate for NL query | Someone still has to write the definitions. Extraction of semantics stays manual |
| Catalog and context layers | Atlan, Alation, Collibra | Metadata, lineage, glossaries that ground other systems | They describe data, they don't answer questions. The reconciliation gap ("revenue exists in three tables") stays open |
| Agentic conversational systems | what Channel AI attempted; the current frontier | Multi-turn exploration, clarification, cross-source reasoning | State management, context decay, and evaluation are all unsolved |

The gaps I would put money on, based on where my pilots bled:

**Semantic extraction is still manual everywhere.** Every serious system in the table above depends on a human writing down what "active user" means. Whoever cracks reliable extraction of business semantics from usage, documents, and conversation gets the whole market. My pilots convinced me the raw material exists (people state their definitions constantly, in messages and meetings) and that nobody is capturing it.

**Conversational state has no standard.** Every team rebuilds context management from scratch and rediscovers the same decay problems. The framework layer (LangGraph and friends) gives you the graph, not the policy for what to keep, expire, or re-verify.

**Evaluation is broken.** Public benchmarks disagree with human judgment on a large fraction of cases, and none of them measure the thing that killed my ambiguous-query trust problem: whether the system should have asked instead of answered. Teams are shipping on vibes plus BIRD scores.

**The trust surface is underbuilt.** Most products still return a number and a chart. The audit trail (which tables, which filters, which definition of the KPI) is what converts a skeptical operator into a daily user, and it is treated as an afterthought almost everywhere.

---

## What I would build differently

Three changes, in priority order.

First, I would make context explicit and structural from day one: a small, typed graph of entities, KPIs, and their relationships per client, versioned like code, instead of embeddings plus conversation history. Retrieval finds text; it does not reason about how concepts relate. This conviction became my subsequent research direction, where it turned out that even lightweight scorers over explicit structure beat heavier graph neural approaches on dense graphs.

Second, I would build the evaluation harness before the product: a per-client suite of known-answer questions, ambiguous questions where the correct behavior is to ask, and adversarial questions mixing two KPI definitions. Every pilot regression I hit would have been caught by a harness I only built after the third one.

Third, I would treat onboarding interviews as the product's data moat rather than a chore, and instrument them properly. The KPI definitions extracted in week one predicted almost everything about whether a deployment succeeded.

---

## Naming note

The project was originally developed as **New Dhatu** and is referred to as **Channel AI** (or ChAI) in academic statements and documentation. Same system, both names.

## Repository contents

Architecture diagrams, sample generated reports, UI captures from pilot usage, and selected exploratory notebooks. Internal iterations and client-specific material are omitted.

## Code availability

The implementation is excluded. The system ran against client data with credentials, proprietary integrations, and infrastructure bindings that cannot be released, and a redacted skeleton would misrepresent what actually ran. This repo is the honest alternative: full design detail, real outputs, and the findings, without the code.

## Disclaimer

This is documentation of a real but concluded exploratory project, written so the design decisions and failure modes are reusable. It is not a product, a benchmark claim, or a drop-in analytics solution.

# Channel AI

> **Status:** documentation only. The implementation is private; this repo is the case study.
> Built and run across 12 private pilot deployments with small and mid-sized teams.

![Status](https://img.shields.io/badge/status-documentation--only-blue)
![Lifecycle](https://img.shields.io/badge/lifecycle-case--study-lightgrey)
![Code](https://img.shields.io/badge/code-not%20public-important)

https://github.com/user-attachments/assets/99edbe86-9d74-4563-9033-60a66d59cc98

Channel AI is a multi-agent conversational analytics system. Non-technical users asked business questions over WhatsApp, in English or a regional language, and got back answers, plots, and full reports generated from their own data. No dashboard, no SQL, no analyst queue.

I built it, deployed it 12 times with real teams, and eventually shut it down as a venture. This repository documents the architecture, the data platform underneath it, and the specific places where natural-language analytics breaks. The failures ended up being more useful than the product: they directly shaped my later research on lightweight graph-based context modeling (ICMLC 2026).

If you are building in this space, three sections below are the ones I wish someone had written for me: [where simple SQL breaks](#where-simple-sql-breaks), [transient intermediate tables](#transient-querying-the-biggest-lesson), and [federated querying](#federated-querying-one-question-across-excel-postgres-and-bigquery).

---
 
## The problem, stated plainly

Every analytics tool assumes the user already knows three things: what question to ask, how the data is organized, and what the metrics actually mean. Outside of data teams, none of those hold.

The obvious fix, circa 2023-2024, was text-to-SQL with retrieval: embed the schema, retrieve relevant tables, let an LLM write the query. It demos beautifully and fails in production, and the gap is now well quantified. Top systems clear 90% execution accuracy on the academic Spider 1.0 benchmark, yet on [Spider 2.0](https://spider2-sql.github.io/), which uses real enterprise warehouses with thousands of columns, the best agent frameworks solved roughly 17% of tasks at launch. On [BIRD](https://bird-bench.github.io/), frontier models still sit near the 50-60% band. The [DIN-SQL error analysis](https://arxiv.org/abs/2304.11015) found that schema linking, matching what the user said to actual tables, columns, and values, is the single largest failure category for LLM-generated SQL. That matches what I saw in the field, in detail, below.

Channel AI was a bet that the fix is architectural: put a dialog in front of generation, a plan between dialog and SQL, and a real data platform underneath all of it.

---

## The pipeline: dialog first, SQL last

The core design decision, and the one that took the longest to arrive at, is that **SQL generation is the last step, not the first**. Requests flowed through five stages, coordinated as a LangGraph state machine with the OpenAI Agents SDK handling tool invocation:

**1. Dialog and slot filling.** Before anything touches the database, a dialog agent checks whether the request is answerable as asked. "How many sales today?" is not answerable if the data carries transaction timestamps but the business day cutoff was never defined, or if "sales" could mean booked, dispatched, or paid. The agent maintains a per-question slot set (metric definition, time grain, entity filters, comparison baseline) and asks for exactly the missing slots, nothing more. Most questions needed zero or one clarification. The ones that needed two were exactly the ones that would have produced confidently wrong answers.

**2. Planning.** A planning agent turns the filled-in request into an explicit analytical plan: which sources are involved, which tables at which grain, whether intermediate materialization is needed, and what the output should be (a number, a breakdown, a plot, a full report). The plan is a data structure, not prose, so it can be validated before any execution happens. Underspecified plans get bounced back to dialog, not guessed at.

**3. Context and value resolution.** Business terms map to schema elements using client metadata and embeddings, and, critically, filter values map to actual stored values (see [value linking](#failure-class-2-categorical-values-value-linking) below). This stage owns the tribal-knowledge problem.

**4. SQL generation and execution.** Schema-aware generation against the Iceberg layer, executing the plan's steps in order, materializing transient intermediate tables where the plan calls for them. Deliberately boring by design: by the time a request gets here, ambiguity is supposed to be resolved. When this stage failed, the root cause was almost always a resolution miss leaking through, not SQL syntax.

**5. Analysis and delivery.** Results get lifted into three tiers, descriptive (what happened), diagnostic (what moved it), and prescriptive (what to consider doing), rendered persona-specifically, plotted where a plot carries more than a sentence, and delivered to WhatsApp and the insight-card view.

<img width="15302" height="9634" alt="Channel AI agent workflow: personas, business context, and metadata feeding the analytical planning loop" src="https://github.com/user-attachments/assets/0695c1c4-6d5d-472e-82f4-dbb4c0ba0149" />

<img width="14505" height="6757" alt="High-level flow from user intent through context resolution to data execution, with human-in-the-loop refinement" src="https://github.com/user-attachments/assets/42707d8d-2365-4d42-809c-65a6dc466661" />

---

## The data platform

The agents get the attention, but the thing that made answers possible at all was the layer underneath them. Client data never arrived clean. It arrived as Excel exports with merged headers, Postgres or MySQL instances designed for an ERP's convenience, and the occasional BigQuery project someone had set up and abandoned.

### Ingestion: everything becomes an Iceberg table

Every source, regardless of origin, was landed into **Apache Iceberg** tables through **Spark** ingestion jobs:

- CSV and Excel dumps went through schema inference, type coercion (Indian number formatting, mixed date formats, currency symbols embedded in "numeric" columns), header normalization, and dedup, then were written as Iceberg tables with the inferred schema recorded as metadata.
- Relational sources were snapshotted on a schedule via JDBC extracts rather than queried live. Live OLTP querying was tempting and wrong: analytical scans against a client's production ERP database is how you get your pilot cancelled.
- BigQuery datasets were mirrored into the same layer for the clients that had them.

Iceberg specifically, rather than plain Parquet on object storage, earned its place three ways: schema evolution (client spreadsheets change shape constantly, and Iceberg absorbs added or renamed columns without rewriting history), snapshot isolation (a report generated at 9am is reproducible at 5pm, which mattered enormously for trust when someone forwarded a screenshot), and hidden partitioning (partition by day derived from a timestamp without the LLM ever needing to know partition columns exist).

### Modeling: facts and dimensions, because LLMs read star schemas better than people do

Raw ingested tables were reshaped into per-client **star schemas**: fact tables at a declared grain (one row per transaction line, one row per stock movement) surrounded by conformed dimension tables (product, customer, region, calendar).

I initially did this for query performance. The bigger payoff was unexpected: **generation accuracy**. An LLM writing SQL against a normalized ERP schema has to reconstruct join paths across a dozen tables and infer which of four date columns means what. Against a star schema with a declared grain and a proper date dimension, the same model's queries got dramatically simpler and dramatically more correct. Dimensional modeling turns out to be prompt engineering that lives in the warehouse. Kimball was writing context engineering for LLMs thirty years early; he just didn't know it.

The date dimension deserves its own sentence: a calendar table carrying business day, fiscal period, festival flags, and week boundaries fixed an entire class of temporal failures that no amount of prompting fixed (details in the failure section below).

### Federated querying: one question across Excel, Postgres, and BigQuery

The moat, to the extent the product had one, was this: because every source landed in the same Iceberg layer with the same metadata conventions, **a single question could join across all of them**. "Which distributors from the ERP are behind on the payment schedule in the finance spreadsheet?" is one query once both sides are Iceberg tables, and an impossible ask for any tool bound to a single warehouse.

The 2026 platforms still mostly don't do this. Databricks Genie answers over Databricks, Cortex Analyst over Snowflake. The long tail of businesses I piloted with had their truth split across a database, fifteen spreadsheets, and someone's abandoned BigQuery project, and nobody was going to migrate them. Ingest-and-unify was slower to set up than point-and-query, and it was the only approach that answered the questions people actually asked, because their questions did not respect source boundaries.

The honest cost: ingestion latency. Snapshot-based mirroring means answers reflect the last sync, not this second. For SMB operational analytics that tradeoff was invisible (nobody needed sub-minute freshness on distributor payments), but it is a real constraint and I'd state it up front to any prospect.

### Transient querying: the biggest lesson

Single-shot SQL, however correct, hits a wall on real analytical questions. "Compare this quarter's top 20 SKUs against their last-year performance, by region" is not one query; written as one query it becomes a nested monster that is slow, unreviewable, and impossible to attribute errors to.

The system's answer, and my single biggest engineering lesson from the project, was **planned materialization of transient intermediate tables**. The planning agent decomposed multi-step questions into stages, and the execution agent materialized each stage as a short-lived Iceberg table (CTAS) rather than a CTE:

1. Stage 1 materializes `_tmp_top_skus_q3` (the top-20 set, a few hundred rows).
2. Stage 2 joins that against last year's fact partition into `_tmp_yoy_compare`.
3. Stage 3 aggregates by region and hands a small frame to the analysis agents.

Why materialize instead of one big query or CTEs:

- **Debuggability.** When stage 2 is wrong, I can inspect stage 1's actual rows. With one nested query, every failure is a vibe. This mirrors the whole system's design thesis: decompose so failures localize.
- **Reuse across turns.** The follow-up "okay now just the south region" reuses `_tmp_yoy_compare` instead of recomputing a multi-million row scan. Conversation-scoped intermediate tables made follow-ups feel instant, and follow-ups were where all the user value lived.
- **Cost and memory.** Spark stops re-shuffling the same data per turn. Wide intermediate results stopped blowing up executor memory because each stage's output was written down and released.
- **Auditability.** The explainability view could point at real tables: this answer came from these 214 rows, derived like so.

Lifecycle management was the unglamorous half: transient tables were namespaced per conversation, tagged with a TTL, reaped by a scheduled job, and invalidated when the underlying snapshot advanced. Getting invalidation wrong produced the worst bug class in the whole system, stale intermediates silently feeding fresh questions, and is a big part of why I now think conversational state deserves first-class infrastructure rather than ad hoc handling.

### Scaling notes

Scaling was two different problems wearing one name. **Vertical** pressure came from individual heavy queries: wide scans and joins on multi-million row fact tables, handled with bigger Spark executors, aggressive partition pruning via Iceberg's hidden partitioning, and the stage materialization above, which caps the working set of any single step. **Horizontal** pressure came from tenancy: 12 deployments meant 12 isolated data layers, and concurrency across clients was handled by scaling out ingestion and query workers per tenant rather than sharing compute. For SMB data volumes, vertical-plus-pruning covered almost everything; the horizontal story mattered for isolation more than throughput. I never needed a thousand-node cluster. I needed one conversation's third question to not recompute its first.

---

## Where simple SQL breaks

Two failure classes accounted for most wrong answers in early versions, and both are now recognized failure modes in the text-to-SQL literature. This section is the most transferable thing in this repository.

### Failure class 1: temporal grain mismatch

The question "how many sales today?" fails against a table whose only time column is `transaction_ts`. Not because the SQL is hard, but because the question and the schema disagree about what a day is. Naive generation produces:

```sql
SELECT COUNT(*) FROM transactions
WHERE DATE(transaction_ts) = CURRENT_DATE;
```

Which is wrong in at least four ways I hit in real pilots: the business day runs 6am to 6am, not midnight to midnight; `transaction_ts` is recorded in UTC while the business runs on IST; "sales" at this client means dispatched orders, which is a status transition, not a transaction row; and Sunday orders get booked Monday, so "today" on a Monday quietly includes a weekend.

The fix was never a smarter prompt. It was structural: a **date dimension** carrying business-day boundaries, fiscal calendars, and locale, joined instead of computed inline, plus the dialog stage refusing to run until "today" and "sales" are pinned to definitions. The literature files this under implicit column context, questions referring to attributes that exist only derivationally. My conclusion: derived attributes people ask about constantly (business day, margin, order age) must be materialized as real columns, because generation over a derivation is generation waiting to fail.

### Failure class 2: categorical values (value linking)

The question "sales of Parle-G last month" fails when the product dimension stores `PARLE G GLUCO 70G(P)` and forty sibling SKUs. Naive generation produces `WHERE product_name = 'Parle-G'`, returns zero rows, and the system reports zero sales. A zero that looks exactly like a real zero. This is the most dangerous failure in the whole space, because it is silently wrong rather than visibly broken.

The research term is **value linking**: mapping the user's mention to values actually stored in the database. [Error analyses](https://arxiv.org/abs/2304.11015) consistently rank this family of schema/value linking failures as the top source of text-to-SQL errors, and nothing about better models fixes it, because the model cannot know strings it has never seen.

What worked, layered:

1. A **distinct-value index** per categorical column: every unique value of every filterable dimension column, held in a lookup structure with trigram and embedding similarity.
2. At resolution time, the user's mention is matched against the index. One high-confidence hit passes through silently. Multiple plausible hits (does "Parle-G" mean the 70g pack, the family of 12 SKUs, or the brand rollup?) trigger a dialog turn: "I found 12 Parle-G SKUs, want them combined or a specific one?"
3. **Zero-row results on a filtered query were treated as a probable linking failure, not an answer.** The system re-checked the filter against the value index before ever telling a user "zero," and said "I couldn't find that product name, closest matches are..." instead. This one rule eliminated the silent-wrong class almost entirely.

Numeric-categorical interaction had its own traps: "top products" (by revenue, units, or margin?), "sales above 50000" (rupees or units? per order or per SKU?). Same medicine: slots, not guesses.

The general principle I took away: **an LLM can be trusted to write SQL, and cannot be trusted to know your data's values, grains, or definitions. Everything that works is scaffolding that supplies those three things before generation starts.**

---

## Output: descriptive, diagnostic, prescriptive

A number is rarely the answer. The analysis agents produced three tiers per question: descriptive (revenue was X, down 12%), diagnostic (the drop concentrates in two distributors in the south region, both of whom cut order frequency in half), and prescriptive (framed as options with the evidence attached, never as instructions).

Plot generation was spec-driven: the analysis agent emits a chart specification (type, encodings, series) chosen from the result frame's shape rather than free-form plotting code, a renderer turns the spec into an image, and the image ships to WhatsApp with its one-line takeaway, with the full version living in the insight-card view. Constraining chart generation to a spec vocabulary killed an entire genre of matplotlib-hallucination bugs and made every chart reproducible from its spec.

<img width="846" height="642" alt="Insight card interface with on-demand plots and reports" src="https://github.com/user-attachments/assets/15ba9681-efac-4a92-a6d9-e525653b2683" />

**Sample output:** an automatically generated sales report from pilot usage: [SALES REPORT.pdf](https://github.com/user-attachments/files/24309621/SALES.REPORT.pdf)

### Explainability

Every answer exposed a schema-level context view: which tables, fields, and business entities were involved, and how they connect. Not used for inference, purely audit. Users who could see why an answer said what it said forgave errors; users who got a bare number did not.

<img width="1856" height="935" alt="Schema-level context view showing entity relationships for a query, next to the generated executive summary" src="https://github.com/user-attachments/assets/2ca13e86-395c-411e-aef5-d36b64f6e148" />

---

## What 12 deployments actually taught me

The compressed version; the sections above carry the detail.

**1. Users cannot tell you what they want to know.** The most common first message was some version of "show me everything." The interactions that retained users were ones where the system proposed the next question. Guided exploration was the product; SQL was plumbing.

**2. Refusing to answer builds more trust than answering.** Early versions guessed on ambiguous requests and were right maybe 70% of the time. It did not matter. One confidently wrong answer undid weeks of correct ones. Dialog-before-SQL raised session depth measurably.

**3. Business semantics are oral tradition.** "Sales" meant booked orders to the founder, dispatched orders to operations, and paid invoices to accounts, at the same company. Onboarding became a structured interview to extract KPI definitions. Nobody in the market has scaled this either; the semantic layer vendors have just organized the manual work.

**4. Context accumulation is a liability.** Long conversations got more confident and less correct as stale assumptions from turn 3 steered turn 30. Explicit state with expiry beat history stuffing, and stale transient tables were the same disease in the data layer. This problem became my research direction.

**5. Dimensional modeling is model context.** The star schema and date dimension improved generation accuracy more than any prompting change I made. The warehouse layout is part of the prompt.

**6. Silent zeros are the worst failure mode.** Treat empty results on filtered queries as probable value-linking failures. This single rule did more for trust than anything else in the codebase.

**7. Auditability beat accuracy as a retention driver.** The context view converted skeptics into daily users. I did not expect that.

---

## Where this sits in the 2026 market

| Approach | Examples | What it does well | Where it breaks |
|---|---|---|---|
| Raw text-to-SQL wrappers | dozens of startups and OSS repos | Fast setup, great demos on clean schemas | The Spider 2.0 cliff. No value linking, no grain awareness, silent zeros |
| BI-native agents | Databricks Genie, Snowflake Cortex Analyst, Looker + Gemini | Strong inside one governed platform; semantic-model grounding cuts errors dramatically | Bound to one platform's pre-modeled data. No answer for truth split across a database, spreadsheets, and a stray cloud project |
| Semantic layer platforms | dbt Semantic Layer, Cube, AtScale | Consistent, versioned metric definitions; the right substrate for NL query | Every definition is still written by hand. Semantic extraction stays manual |
| Catalog and context layers | Atlan, Alation, Collibra | Glossaries, lineage, quality signals that ground other systems | They describe data, they don't answer questions. The three-definitions-of-revenue problem stays open |
| Agentic conversational systems | what Channel AI attempted; the current frontier | Multi-turn exploration, clarification before execution, cross-source reasoning | Conversational state, transient-result lifecycle, and evaluation are all unsolved |

The gaps I would put money on, from where my pilots bled: semantic extraction is still manual everywhere; conversational state (including intermediate-result lifecycle) has no standard and every team rebuilds it; evaluation ignores the ask-vs-answer decision entirely; and the trust surface, the audit trail from answer back to tables, filters, and definitions, is an afterthought in almost every product.

---

## What I would build differently

First, explicit structural context from day one: a small, typed graph of entities, KPIs, grains, and their relationships per client, versioned like code, instead of embeddings plus conversation history. Retrieval finds text; it does not reason about how concepts relate. This conviction became my subsequent research, where lightweight scorers over explicit structure beat heavier graph neural approaches on dense graphs.

Second, the evaluation harness before the product: known-answer questions, ambiguous questions where the correct behavior is to ask, value-linking traps with near-miss category names, and grain traps. Every pilot regression I hit would have been caught by the harness I only built after the third one.

Third, treat the onboarding interview as the data moat and instrument it. The KPI definitions extracted in week one predicted almost everything about whether a deployment succeeded.

---

## Naming note

The project was originally developed as **New Dhatu** and is referred to as **Channel AI** (or ChAI) in academic statements and documentation. Same system, both names.

## Repository contents

Architecture diagrams, sample generated reports, UI captures from pilot usage, and selected exploratory notebooks. Internal iterations and client-specific material are omitted.

## Code availability

The implementation is excluded. The system ran against client data with credentials, proprietary integrations, and infrastructure bindings that cannot be released, and a redacted skeleton would misrepresent what actually ran. This repo is the honest alternative: full design detail, real outputs, and the findings, without the code.

## Disclaimer

This is documentation of a real but concluded exploratory project, written so the design decisions and failure modes are reusable. It is not a product, a benchmark claim, or a drop-in analytics solution.

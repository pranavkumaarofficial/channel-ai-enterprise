# Channel AI (formerly New Dhatu)
![Status](https://img.shields.io/badge/status-documentation--only-blue)
![Lifecycle](https://img.shields.io/badge/lifecycle-exploratory-lightgrey)
![Code](https://img.shields.io/badge/code-not%20public-important)


https://github.com/user-attachments/assets/99edbe86-9d74-4563-9033-60a66d59cc98


Channel AI is a conversational analytics system built to explore how non-technical teams interact with complex business data when context, structure, and domain knowledge are fragmented across systems.

The project emerged from a recurring observation: most analytics tools assume users already understand what questions to ask, how data is organized, and how metrics relate to one another. In practice, this assumption breaks down quickly, especially outside technical teams.

Channel AI was an attempt to design analytics systems that *guide exploration*, rather than simply execute queries.

---

## Naming Note

This project was originally developed under the name **New Dhatu**. It is referred to as **Channel AI** (or simply ChAI) in academic statements and documentation. Both names refer to the same system.

---

## Motivation

Many enterprise analytics stacks rely on variants of retrieval-augmented generation, where schemas, documentation, or metadata are retrieved and passed to a language model. In real-world usage, these approaches often fail when:

- relationships between entities are implicit or weakly defined  
- business terms and KPIs carry domain-specific meaning  
- categorical and numeric variables must be interpreted together  
- data spans loosely connected tables or systems  

Simply retrieving relevant text does not help a system reason about *how* business concepts relate.

This project explored whether separating **intent understanding**, **business semantics**, and **data execution** into distinct stages could preserve context more reliably than prompt-centric or single-agent designs.

---

## System Overview

Channel AI was implemented as a **multi-agent conversational analytics system**, where distinct agents handled intent interpretation, business-context resolution, data planning, and execution. The system was designed to support exploratory analytics through conversation rather than single-shot query execution.

Agent coordination was implemented using **LangGraph** and the **OpenAI Agents SDK**, allowing explicit control over execution flow, state propagation, and tool invocation across conversational turns.

At a high level, the system consisted of:

- an **intent and planning agent** responsible for interpreting user requests and decomposing them into analytical steps  
- **context resolution agents** that mapped business-specific terms and KPIs using metadata, embeddings, and prior interactions  
- a **schema-aware SQL generation agent** operating over structured data sources  
- **analysis and summarization agents** that produced persona-specific explanations, summaries, and visualizations  
- an **orchestration layer** that managed agent state, execution order, and fallback behavior across turns  

The system surfaced follow-up questions and alternative views when requests were ambiguous or underspecified.

---


**Context and workflow resolution diagram**  
   Shows how business context, personas, and metadata influence analytical planning.
<img width="15302" height="9634" alt="dhatu workflow" src="https://github.com/user-attachments/assets/0695c1c4-6d5d-472e-82f4-dbb4c0ba0149" />


### Multi-Agent Orchestration Details

Agent execution was coordinated using a graph-based control flow (LangGraph), enabling conditional routing, retries, and shared state across agents. This allowed the system to:

- preserve business context across conversational turns  
- separate intent understanding from data execution  
- handle partial or underspecified queries without premature execution  
- maintain consistency when multiple datasets or KPIs were involved  

This design exposed practical challenges around state management, schema drift, and context accumulation as conversations grew longer, which later informed my research interest in explicit context modeling and lightweight graph-based reasoning.


---

## Interface and Interaction

A key design decision was deploying the system through **WhatsApp**, allowing users to consume analytics incrementally in an interface they already trusted and used daily.

Reports, plots, and summaries were delivered conversationally rather than through dashboards. This surfaced practical challenges around:

- explanation and traceability of results  
- maintaining consistency across conversational turns  
- user trust when analytics are consumed asynchronously  
- handling context accumulation over time  

Support for regional languages further highlighted how much implicit knowledge most analytics systems assume.

---

## Architectural Notes

Under the hood, Channel AI coordinated multiple components responsible for intent planning, context resolution, dataset selection, and execution.

As pilot usage expanded, several hard problems emerged around semantic alignment, cross-table reasoning, and managing context growth across loosely connected data sources.
The system operated over a unified analytical layer built on **Apache Iceberg**, allowing consistent schema access across relational databases and flat files while supporting multi-million row analytical workloads.

These limitations directly shaped later research directions around context modeling and lightweight graph-based reasoning, where relationships are made explicit without relying on full graph traversal at scale.

---

## Example Output

Below is an example of an automatically generated sales report produced during pilot usage, illustrating the type of summaries and insights delivered through the conversational interface.

- See: [SALES REPORT.pdf](https://github.com/user-attachments/files/24309621/SALES.REPORT.pdf)


---

## Context Visualization and Explainability

During pilot usage, one recurring challenge was understanding *why* certain insights were produced and *which relationships* influenced analytical outcomes.

To address this, the system exposed a schema-level context view that visualized how tables, fields, and business entities were connected for a given query. This was not used for direct graph traversal or inference, but as a diagnostic and explainability layer to surface implicit relationships that traditional retrieval or prompt-based approaches failed to capture.

The visualization below illustrates how business entities and attributes were connected during an example analytical request, alongside the generated executive summary.

<img width="1856" height="935" alt="KG image" src="https://github.com/user-attachments/assets/2ca13e86-395c-411e-aef5-d36b64f6e148" />

---
 

  
# Context and Workflow Design

A recurring challenge during development was modeling how user intent, business context, and underlying data interact during real analytical workflows.

The diagram below illustrates the high-level flow used to reason about personas, business context, and structured data sources. The emphasis was not on automation alone, but on preserving context across user interactions and enabling human-in-the-loop refinement where needed.

<img width="14505" height="6757" alt="flow-diagram" src="https://github.com/user-attachments/assets/42707d8d-2365-4d42-809c-65a6dc466661" />

(Additional internal diagrams and iterations are intentionally omitted for clarity.)

---

## Repository Contents

This repository contains selected reference material and artifacts from the project:

- architecture diagrams  
- sample reports and outputs  
- UI screenshots from pilot usage  
- exploratory notebooks and prototype code  

The full system was developed as part of a private, exploratory project and is not fully open-sourced.

---

## Project Status

Channel AI was developed as a private, exploratory project and is no longer active as a venture. The system was used in limited pilot deployments with small and mid-sized teams, primarily to evaluate usability, context handling, and workflow fit rather than to scale usage.

This repository documents the design space, tradeoffs, and lessons learned from building and iterating on a real-world conversational analytics system.

The work here reflects an applied systems exploration rather than a polished product release.

---

## Disclaimer

This repository is intended for documentation and reference purposes. It is not a production-ready system and should not be treated as a drop-in analytics solution.
This repository is not intended to demonstrate implementation completeness or benchmark performance, but to document system design decisions and the evolution of ideas that informed later research and engineering work.

## Code Availability

This repository is documentation-first and intentionally excludes the full
implementation.

The original system was developed as part of private pilot deployments and
contains credentials, proprietary integrations, and infrastructure bindings
that cannot be publicly released. Backend code is therefore excluded or
redacted in line with standard enterprise and security practices.



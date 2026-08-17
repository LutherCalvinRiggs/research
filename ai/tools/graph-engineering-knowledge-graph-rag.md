# Graph Engineering with LLMs: Knowledge Graph RAG Architecture

**Source:** https://x.com/kirillk_web3/status/2088682533278077382 (Kirill K, @kirillk_web3)
**Published:** ~August 2026
**Saved:** 2026-08-17
**Tags:** ai, tools, infrastructure, research, fundamentals, technology

> Educational content. All performance figures cited are from published research on specific datasets. Author uses Kimi K3 as the example model; the architecture is model-agnostic. Key references: Microsoft GraphRAG (github.com/microsoft/graphrag), Stanford DSPy (github.com/stanfordnlp/dspy).

---

## TL;DR
Standard RAG finds documents that look similar to your query. Graph engineering stores facts and their relationships, then queries the relationships directly — enabling "why" and "how are these connected" questions that vector search fundamentally cannot answer. The key finding from 26-model research: a smaller model with a well-built graph consistently beats a larger model with poor retrieval. The graph beats the model size. Eight-layer architecture: Ingestion → Extraction → Resolution → Storage → Retrieval → Agent → Verification → Update.

---

## The Ceiling of Standard RAG

Standard RAG: user asks something → vector search finds similar text chunks → model writes answer from chunks.

**The problem:** semantic similarity finds documents that look alike. It does not find facts that connect.

Example — "why did sales drop in March?" — vector search finds documents containing "sales" and "March." It cannot return:
```
Sales dropped → release delay → supplier problem → warehouse failure → negative reviews → conversion −23%
```
Because each cause lives in a different document sharing no keywords with the others. No amount of better embeddings gets you there.

**Microsoft's framing (GraphRAG research):**

| Query type | What it answers | RAG handles it? |
|-----------|----------------|----------------|
| Local search | "What happened with supplier X in July?" — finds a node and its immediate connections | Badly |
| Global search | "What are the recurring risk patterns across all suppliers?" — patterns across the whole graph | Not at all |

Graph engineering handles both.

---

## What Graph Engineering Is

Instead of storing text and searching by similarity: **store facts and their relationships, then query the relationships directly.**

Everything becomes a triple:
```
Subject → Relation → Object

Kimi K3     → developed_by      → Moonshot AI
Warehouse   → caused            → Supplier delay
Supplier    → caused            → Release delay
Release     → reduced           → Conversion rate
```

A vector database stores "this paragraph is about supply chains."
A knowledge graph stores "this specific event caused that specific outcome."

When you query it, you're not asking "what text is similar to my question." You're asking "walk me the path from A to B and show me every link."

---

## The Key Research Finding

From a paper comparing 26 open-source models on knowledge graph engineering tasks:

> **Bigger model + bad graph → worse results**
> **Smaller model + good graph → better results**

The graph beats the model size. Consistently. This is the same conclusion Microsoft reached with GraphRAG and Anthropic's agent graphs: **the system around the model determines output quality more than the model itself.**

The instinct when AI underperforms is to reach for a bigger model. The evidence says fix the retrieval structure instead. It's cheaper and it works better.

---

## Three Integration Modes — Pick Mode 3

**Mode 1 — KG-enhanced LLM:** Graph feeds the model facts → model generates better answers. One direction only.

**Mode 2 — LLM-augmented KG:** Model builds, cleans, and expands the graph → graph improves over time. Also one direction only.

**Mode 3 — Synergized (the one worth building):** The model extracts new facts and writes them into the graph. The graph gives the model structured context for the next question. Each pass makes both better. The system gets measurably smarter every time it answers a question — because each answer adds structure the next answer can use.

Modes 1 and 2 are components of Mode 3, not alternatives to it.

---

## The Eight-Layer Architecture

### Layer 1: Ingestion
Raw material only — PDFs, web pages, databases, APIs, Slack, Notion. No processing yet.

### Layer 2: Extraction
LLM reads each source and pulls entities and relationships as structured JSON:

```json
{
  "entity": "Supplier X",
  "type": "vendor",
  "relations": [
    {
      "predicate": "caused",
      "object": "warehouse failure",
      "confidence": 0.97,
      "evidence": "exact quote from source — required, not optional"
    }
  ]
}
```

**Critical:** the evidence field is not optional. Require an exact quote for every relationship. If the model can't produce one, the relationship doesn't go in. This single constraint eliminates most extraction hallucination.

### Layer 3: Resolution (the layer everyone skips and regrets)
Are "Moonshot AI," "Moonshot," "Beijing Moonshot," and "月之暗面" the same entity? If not resolved, the graph fragments into duplicates and every query returns partial results.

Do this **before insertion**, not after. Retrofitting entity resolution onto a polluted graph is significantly harder than doing it upfront.

### Layer 4: Storage
Neo4j (best docs, best visualization, Cypher is readable), Memgraph, Neptune, or PostgreSQL with a graph extension.

### Layer 5: Retrieval (five methods working together)
- **Vector search** — fuzzy matching
- **Entity lookup** — exact nodes
- **Path search** — connections between nodes
- **Community search** — patterns across the graph
- **Temporal filtering** — "what was true when"

Not one method — all five.

### Layer 6: Agent
Plans the approach, generates Cypher queries, reads returned subgraph, runs additional searches when it hits a gap, decides what to do next.

### Layer 7: Verification
Checks that conclusions are actually supported by retrieved paths, flags contradictions, evaluates confidence, verifies sources. Without this layer you've built a very sophisticated hallucination machine.

### Layer 8: Update
New facts into the graph. Contradictions flagged rather than silently overwritten. Superseded facts timestamped instead of deleted.

**The loop closes at layer 8 and starts again at layer 2. That's what makes it compound.**

---

## Five Prompts That Run the Pipeline

### Prompt 1 — Extraction
```
Extract all organizations, people, products, and events from the text below.

For each entity return: canonical_name, type, description, source_location
For each relationship return: source_entity, relation_type, target_entity,
  evidence (exact quote from source), confidence_score (0-1)

Rules:
- Only extract relationships explicitly stated or directly implied by the text
- Do not infer relationships from general knowledge
- If a relationship is uncertain, lower the confidence score rather than omitting it
- Return valid JSON only

Text: [paste your document]
```

### Prompt 2 — Entity Resolution
```
Compare the following entities and determine whether they refer to:
same entity / related but distinct / unrelated entities

Rules:
- Do not merge entities without clear evidence
- Similar names are not sufficient evidence
- When uncertain, mark as "related" rather than "same"
- Flag any case where merging would be destructive
```

### Prompt 3 — Query Translation
```
Translate the user question into a Cypher query for our graph.

Schema: [paste your node labels, relationship types, and properties — the literal schema, not a description]

Rules:
- Use only labels and relationship types present in the schema
- Do not invent properties
- Prefer path queries over single-node lookups when the question implies causation
- Return the query, then a plain-English explanation of what it retrieves and why
```

### Prompt 4 — Grounded Answer
```
Answer the question using ONLY the graph paths provided below.

For every claim:
- cite the specific nodes and relationship path that support it
- state your confidence
- flag any step where you're inferring rather than reading from the graph

Rules:
- Do not use knowledge outside the provided paths
- Do not infer causation from co-occurrence
- If the graph doesn't contain enough to answer, say exactly what's missing
```

### Prompt 5 — Graph Maintenance
```
Compare these new facts against the existing graph.

Classify each new fact as:
- new (add it)
- duplicate (skip)
- contradiction (flag for review, do not overwrite)
- update (supersedes existing — timestamp old one, don't delete)
- uncertain (needs human review)

Never silently overwrite an existing fact.
```

---

## The Standard Failure Modes

| Problem | Root cause | Fix |
|---------|-----------|-----|
| Graph fills with duplicate entities | Skipped resolution layer | Run Prompt 2 as batch before insertion, not after |
| Extraction invents relationships | No evidence requirement | Evidence field in Prompt 1 is mandatory; no quote = no relationship |
| Queries return everything or nothing | Schema mismatch | Pass literal schema in Prompt 3 every time; validate Cypher before executing |
| Confident wrong answers | Co-occurrence ≠ causation | Prompt 4's "do not infer causation from co-occurrence" + verification layer |
| Rising token costs | Retrieval too greedy | Limit traversal depth; most questions need 2–3 hops, not 6 |
| Graph becomes untrustworthy | Contradiction accumulation | Timestamps on everything + scheduled Prompt 5 maintenance pass |

---

## The Stack

| Layer | Tool |
|-------|------|
| Graph database | Neo4j (start here) |
| Query language | Cypher |
| Orchestration | DSPy (programs the pipeline rather than hand-tuning prompts; reframes model as a component) |
| References | Microsoft GraphRAG (github.com/microsoft/graphrag) |

---

## The Week-One Plan

- **Day 1** — Install Neo4j locally. Write five Cypher queries by hand. Don't skip this.
- **Day 2** — Run Prompt 1 on one document set you actually care about. Look hard at what extracted — this reveals whether your prompt is too loose.
- **Day 3** — Load triples into Neo4j. Build simplest retrieval: entity lookup + one-hop traversal. Test against a question your current RAG answers badly.
- **Day 4** — Add path search. Ask a "why" question requiring three hops. This is the moment the difference becomes obvious.
- **Day 5** — Connect an agent via MCP. Let it query the graph, find a gap, run a search, write a new fact back. First closed loop.
- **Days 6–7** — Measure: accuracy vs. old RAG, token cost per query, latency. Your own five test questions are worth more than any published percentage.

---

## About the "85% Lower Cost, 18% Better Accuracy" Numbers

These come from specific research on specific document sets, not a universal guarantee. The comparison baseline matters enormously — "85% cheaper than loading structured files directly into context" is a very different claim from "85% cheaper than your current RAG." The direction of the finding is well-supported across multiple independent groups. The magnitude on your data is something you measure in week one.

---

## Questions & Gaps
- The article focuses on structured/document knowledge bases. How does this architecture perform on real-time or frequently-updated data where the graph maintenance burden is continuous?
- Entity resolution is described as the most commonly skipped layer. Are there open-source tools for automated entity resolution at scale, or is this always a custom LLM prompt task?
- DSPy is recommended for orchestration — how does its compilation approach interact with graph retrieval? Does DSPy support graph database calls as native retrieval operations?
- The verification layer (Layer 7) is named but not given a specific prompt. What does a verification prompt look like in practice?
- For multi-hop reasoning, how do you decide traversal depth limits at query time vs. schema design time?

## Related Notes
- [30 Core Agentic Engineering Concepts](https://github.com/LutherCalvinRiggs/research/blob/main/ai/tools/30-core-agentic-engineering-concepts.md) — graph engineering is a different retrieval architecture but shares the same agent loop pattern. The article's Mode 3 (synergized KG + LLM) is the same compound loop described in concepts 2 and 13.
- [Building a Good Vertical Agent — Context Hierarchy](https://github.com/LutherCalvinRiggs/research/blob/main/ai/tools/building-good-vertical-agent-context-hierarchy.md) — the L1/L2/L3 context hierarchy is the standard RAG approach; graph retrieval is an alternative architecture for L2 that structures context as subgraphs rather than text chunks.
- [Lilian Weng: Harness Engineering for Self-Improvement](https://github.com/LutherCalvinRiggs/research/blob/main/ai/research/lilian-weng-harness-engineering-self-improvement.md) — Weng's article is cited implicitly here: "agent graphs from Anthropic and LangGraph show the same architectural principle: structure beats scale." The graph engineering thesis is a retrieval-layer instance of the harness-beats-model-size claim.
- [PGS: Property-Generated Solver](https://github.com/LutherCalvinRiggs/research/blob/main/ai/research/pgs-property-generated-solver-llm-code-refinement.md) — both PGS and graph engineering demonstrate the same structural principle: quality of the context/feedback structure matters more than model size. PGS applies this to code refinement feedback; graph engineering applies it to retrieval.

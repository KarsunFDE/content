---
week: W03
day: Wed
topic_slug: kg-cg-scaffold-orchestrator
topic_title: "KG + CG + scaffold orchestrator — graph-backed context for agents"
parent_overview: W03/pre-session/3-Wednesday/1-DailyTopicOverview.md
estimated_minutes: 11
sources:
  - url: https://www.modern-datatools.com/compare/neo4j-vs-postgresql
    retrieved_on: 2026-05-26
    recency_category: foundation-stable
  - url: https://www.sheshbabu.com/posts/graph-retrieval-using-postgres-recursive-ctes/
    retrieved_on: 2026-05-26
    recency_category: foundation-stable
  - url: https://venturebeat.com/data/context-architecture-is-replacing-rag-as-agentic-ai-pushes-enterprise-retrieval-to-its-limits
    retrieved_on: 2026-05-26
    recency_category: hot-tech
  - url: https://learn.microsoft.com/en-us/agent-framework/integrations/neo4j-graphrag
    retrieved_on: 2026-05-26
    recency_category: hot-tech
  - url: https://github.com/DEEP-PolyU/Awesome-GraphRAG
    retrieved_on: 2026-05-26
    recency_category: hot-tech
last_verified: 2026-06-06
---

# KG + CG + scaffold orchestrator — graph-backed context for agents

> [!NOTE]
> **From earlier:** Tue's RAG work retrieved flat text chunks. Today's graph-backed retrieval adds traversal — multi-hop relational questions that vector similarity alone can't answer.

## 1. Learning Objectives

- Distinguish a *Knowledge Graph* (KG — persistent, source-of-truth) from a *Context Graph* (CG — runtime, per-request subgraph stitched from the KG).
- Apply the *scaffold orchestrator* pattern: a stable `kg_query` tool contract wrapping a swappable backend.
- Compare Neo4j, Postgres recursive CTE, and NetworkX on the dimensions that drive a production pick.
- Recognise when graph-backed retrieval beats vector RAG and when it doesn't.

## 2. Introduction

Much of what an agent needs is relational: entity A connects to B, B to C; the multi-hop chain is the answer. Vector RAG degrades on this shape.

A **Knowledge Graph (KG)** holds persistent entities and relationships; a **Context Graph (CG)** is the runtime per-request subgraph assembled from the KG and stitched into the prompt. The **scaffold orchestrator** keeps backend choice from locking the architecture: a stable `kg_query` tool contract wraps the backend; the agent never knows which.

## 3. Core Concepts

### 3.1 KG vs CG

| | Knowledge Graph (KG) | Context Graph (CG) |
|-|---------------------|-------------------|
| Persistence | Durable, canonical, shared across requests | Per-request, runtime-assembled |
| Size | Potentially millions of nodes | Bounded — dozens to hundreds of nodes |
| Role | Source of truth; the agent's vocabulary | What enters the prompt for this request |

The same KG entity produces different CGs depending on the question being answered.

### 3.2 When KG beats vector RAG (and when it doesn't)

Graph wins on **multi-hop** questions ("vendor with red CPARs who won contracts against this solicitation" — 3 hops), **typed-semantic** edges (`won` vs `lost` mean different things), and **structural patterns** (paths, cycles).

Vector RAG wins on single-hop text similarity or unstructured prose. Production systems often use **hybrid retrieval**: vector for seed entities, KG for relational expansion.

### 3.3 Three viable backends

| Backend | Pros | Cons | Best fit |
|---------|------|------|----------|
| **Neo4j** | Physical-pointer traversal; Cypher concise; multi-hop ~constant-time | Another DB; compliance scope expansion | Deep traversal (4+ hops), large graphs |
| **Postgres CTE** | Zero new infra; 1–3 hops fine | Expensive past 3–4 hops; indexing non-obvious | Shallow, modest cardinality, existing Postgres |
| **NetworkX** | Zero infra; pure Python | In-memory only; single process; no restart survival | Small graphs (~10k nodes) |

Decision drivers: data-residency, ops budget, sub-200ms p95 latency target, graph size, traversal depth.

### 3.4 The scaffold orchestrator pattern

The agent calls a **stable `kg_query` tool** independent of the backend. An adapter translates to Cypher, Postgres CTE, or NetworkX — the agent never sees which.

```text
agent ──tool_call(kg_query, params)──▶ [ kg_query adapter ] ──▶ Neo4j
                                                         └──▶ Postgres CTE
                                                         └──▶ NetworkX
```

Backend choice becomes an ADR, not lock-in. A/B comparisons and rollback are straightforward.

> [!IMPORTANT]
> **SA-3 ADR setup.** Today's afternoon scenario-research slot (D-040) opens SA-3: KG/CG backend for the 17-entity acquire-gov schema. The scaffold pattern is what lets you pick a backend for the MVP (probably NetworkX or Postgres CTE) while keeping the architecture open for production (Neo4j if traversal depth or cardinality forces it). Full ADR due EOD Thu W4.

### 3.5 Bounding the CG

A 200-node subgraph serialized as text blows the context budget. Prune by relevance before serialization. **Cap traversal depth and result count at the tool boundary.**

## 4. Generic Implementation

Scaffold orchestrator with swappable backend — scientific-publication assistant (not federal-acquisitions):

```python
from typing import Protocol, TypedDict

class KGQuery(TypedDict):
    seed_entity_id: str
    traversal_pattern: str
    max_hops: int
    max_results: int

class KGResult(TypedDict):
    nodes: list[dict]
    edges: list[dict]

class KGBackend(Protocol):
    def query(self, q: KGQuery) -> KGResult: ...

class Neo4jBackend:
    def query(self, q: KGQuery) -> KGResult:
        cypher = """
        MATCH p = (seed {id: $id})-[*1..$hops]-(other)
        RETURN nodes(p) AS nodes, relationships(p) AS edges
        LIMIT $max_results
        """
        with self.driver.session() as s:
            rows = s.run(cypher, id=q["seed_entity_id"],
                         hops=q["max_hops"], max_results=q["max_results"]).data()
        return _normalize_neo4j(rows)

class PostgresCTEBackend:
    def query(self, q: KGQuery) -> KGResult:
        sql = """
        WITH RECURSIVE traversal AS (
          SELECT id, parent_id, 0 AS depth FROM entities WHERE id = %(seed)s
          UNION ALL
          SELECT e.id, e.parent_id, t.depth + 1
          FROM entities e JOIN traversal t ON e.parent_id = t.id
          WHERE t.depth < %(hops)s
        )
        SELECT * FROM traversal LIMIT %(max_results)s
        """
        cur = self.conn.cursor()
        cur.execute(sql, {"seed": q["seed_entity_id"],
                          "hops": q["max_hops"], "max_results": q["max_results"]})
        return _normalize_postgres(cur.fetchall())

def kg_query_tool(backend: KGBackend):
    def _tool(seed_entity_id: str, traversal_pattern: str,
              max_hops: int = 3, max_results: int = 50) -> KGResult:
        return backend.query({"seed_entity_id": seed_entity_id,
                              "traversal_pattern": traversal_pattern,
                              "max_hops": max_hops, "max_results": max_results})
    return _tool

# Swap backend without changing agent:
backend = Neo4jBackend(driver=neo4j_driver)
tool = kg_query_tool(backend)
```

`KGQuery` and `KGResult` are the typed contract every backend honors. The agent registers `tool` and never knows which backend it calls.

> [!NOTE]
> **The scaffold pattern is the interlock with SA-3.** SA-3 today: which backend for the 17-entity acquire-gov schema? The scaffold lets you pick NetworkX for MVP and upgrade to Neo4j later — same tool contract, swap the adapter.

## 5. Real-world Patterns

**E-commerce — recommendation graph.** KG of `User`, `Item`, `Category`, `Brand`; multi-hop traversals dominate. Neo4j common at scale for deep traversal; Postgres CTE viable for shallow cardinality.

**Healthcare — adverse-event reasoning.** KG of `Drug`, `Patient`, `Condition`, `AdverseEvent`. Postgres CTEs common — data already in regulated relational stores; adding Neo4j expands compliance scope.

## 6. Best Practices

- **Design the KG schema explicitly before writing the agent.** Entities, edges, and edge-type semantics are the agent's vocabulary.
- **Bound the CG before it enters the prompt.** Prune by relevance or path length.
- **Use the scaffold orchestrator from day one.** Adding it later requires rewriting every tool call that touches the graph.
- **Cap traversal depth and result count at the tool boundary.** Unbounded traversal is a DoS against your own infra.
- **Hybrid retrieval often beats pure KG or pure vector.** Vector finds seed entities; KG expands to relational context.

> [!WARNING]
> **Anti-pattern: `kg-as-rag-substitute`.** Replacing your vector RAG pipeline entirely with a KG does not work for unstructured text retrieval — KGs answer relational/structural questions; vector retrieval answers semantic-similarity questions. Production systems that need both use hybrid retrieval: vector for candidate seed entities, KG for relational expansion. Removing vector retrieval in favor of a pure KG will cause recall failures on questions that are fundamentally text-similarity shaped, not graph-shaped.

## 7. Hands-on Exercise

You are designing the KG for a **music-streaming recommendation assistant** (not federal-acquisitions). Entities: `User`, `Track`, `Artist`, `Album`, `Genre`. Edges: `listened_to`, `by_artist`, `on_album`, `in_genre`, `collaborated_with`.

Tasks: (1) Sketch the schema with entity types and edge types labeled. (2) Write the Postgres recursive CTE for: "Find artists who collaborated with artists this user listened to, where the user hasn't yet listened to the collaborating artist's tracks." (3) Write the equivalent Cypher. (4) Choose a backend for an MVP with 100k users, 1M tracks — justify in one sentence keyed to ops budget and traversal depth.

> [!NOTE]
> **Self-check** (30s — answer mentally before expanding)
>
> 1. Your agent calls `kg_query` directly against the Neo4j driver — no adapter layer. What does this prevent you from doing later?
> 2. A CG subgraph returns 300 nodes serialized as text. What's the failure mode and the fix?

<details>
<summary>Show answers</summary>

1. You can't swap backends without rewriting every agent prompt and tool-call invocation. The scaffold pattern's value is decoupling backend choice from agent design — skip it and the backend becomes architectural lock-in.
2. 300-node serialized text likely blows the context budget, causing truncation or degraded reasoning. Fix: prune the CG at the tool boundary — cap `max_results` and `max_hops`, and prune by graph-centrality or query-similarity before serialization. Only the most relevant subgraph should enter the prompt.

</details>

> [!TIP]
> **Hybrid retrieval handles both shapes.** Use vector search to find candidate seed entities (text similarity), then the KG/CG step to expand them into relational context (multi-hop traversal). Neither alone covers the full retrieval surface.

## 8. Key Takeaways

- KG is persistent source-of-truth; CG is the runtime per-request subgraph assembled from it and stitched into the prompt.
- Graph-backed retrieval wins on multi-hop, typed-semantic, structural-pattern questions; vector RAG wins on single-hop text similarity. Hybrid retrieval handles both.
- Scaffold orchestrator: stable `kg_query` tool contract + swappable backend adapter. Backend choice stays an ADR.
- Pick backend by ops burden, traversal depth, cardinality, data-residency. No universal answer.

## 9. Sources

<details>
<summary>References — retrieved via /web-research per D-046</summary>

- https://www.modern-datatools.com/compare/neo4j-vs-postgresql — retrieved 2026-05-26 — foundation-stable
- https://www.sheshbabu.com/posts/graph-retrieval-using-postgres-recursive-ctes/ — retrieved 2026-05-26 — foundation-stable
- https://venturebeat.com/data/context-architecture-is-replacing-rag-as-agentic-ai-pushes-enterprise-retrieval-to-its-limits — retrieved 2026-05-26 — hot-tech
- https://learn.microsoft.com/en-us/agent-framework/integrations/neo4j-graphrag — retrieved 2026-05-26 — hot-tech
- https://github.com/DEEP-PolyU/Awesome-GraphRAG — retrieved 2026-05-26 — hot-tech

</details>

<details>
<summary>Deeper dive — for senior FDEs (optional, not in reading budget)</summary>

**The 17-entity acquire-gov schema as KG.** The schema from W1 Tue's brownfield inventory — Vendor, Proposal, Evaluation, Award, ContractModification, Cpar, Solicitation, Amendment, EvaluatorScore, SSDD, ConsensusRound, QASP findings — is a graph. Three CO query traversal patterns worth sketching before the SA-3 ADR: (1) "vendor's contracts with red CPARs" — 3 hops; (2) "evaluators who've scored vendors with QASP findings" — 4 hops with back-reference; (3) "contract modifications on awards where the SSDD never approved a tradeoff above IGCE" — recursive ContractModification chains, irregular depth. Pattern (3) is where Postgres recursive CTEs start to strain (high branching factor, variable depth) and Neo4j's physical-pointer traversal shows its advantage. Use this to frame the ops-burden vs query-latency trade-off in the SA-3 ADR.

**NetworkX scalability ceiling.** The practical ceiling for NetworkX in-process in a Python FastAPI service under concurrent load is roughly 50k–100k nodes with multiple concurrent requests. At acquire-gov production volumes (per `training-project/feature-inventory-target.md`: ~80 active contracts, ~100 vendors, ~500 proposals), NetworkX is well inside the ceiling. The risk is process restart: any in-memory graph must be rebuilt from the source DB on restart. A startup hydration step (load the full graph from Postgres on `lifespan()`) keeps the latency acceptable at this scale.

</details>

Last verified: 2026-06-06

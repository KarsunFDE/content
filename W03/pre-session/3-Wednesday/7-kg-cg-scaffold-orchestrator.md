---
week: W03
day: Wed
topic_slug: kg-cg-scaffold-orchestrator
topic_title: "KG + CG + scaffold orchestrator — graph-backed context for agents"
parent_overview: W03/pre-session/3-Wednesday/1-DailyTopicOverview.md
estimated_minutes: 14
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
last_verified: 2026-05-26
---

# KG + CG + scaffold orchestrator — graph-backed context for agents

## 1. Learning Objectives

By the end of this reading, the learner can:

- Distinguish a *Knowledge Graph* (KG — persistent, source-of-truth) from a *Context Graph* (CG — runtime, per-request, stitched from the KG plus retrieved evidence).
- Apply the *scaffold orchestrator* pattern: a stable tool contract wrapping a swappable graph backend, so backend choice remains an ADR rather than locking the agent design.
- Compare three viable graph backends — Neo4j, Postgres recursive CTE, NetworkX in-process — on the dimensions that actually decide a production pick (operational burden, traversal performance, data-residency, ops budget).
- Recognize when graph-backed retrieval beats vector RAG and when it does not, based on the structure of the underlying domain.
- Read and write a small multi-hop graph traversal in Cypher and as a Postgres recursive CTE.

## 2. Introduction

A lot of what an AI agent needs to "know" about a domain is relational: entity A is connected to entity B by relationship R; B is connected to C; the multi-hop chain from A to C is the answer. Vector RAG handles unstructured-text-similarity retrieval well but degrades on this shape — multi-hop questions require traversal, and traversal over a flat vector index produces noise rather than answers [1].

The discipline that addresses this is **graph-backed retrieval** — and in 2026 production literature it usually arrives as a pair: a **Knowledge Graph (KG)** holds the persistent entities and relationships; a **Context Graph (CG)** is the runtime, per-request graph the agent assembles from the KG plus retrieved evidence and stitches into its prompt [2]. The KG is the source; the CG is the snapshot.

The mechanical question is how to back the KG. Neo4j is the purpose-built choice; Postgres recursive CTEs let you reuse infrastructure you already have; NetworkX in-process is a third option for small graphs that don't justify either. The choice has real consequences for performance, ops burden, data residency, and team skills — and is rarely obvious at the start of a project. The **scaffold orchestrator** pattern keeps this choice from locking the architecture: a stable `kg_query` tool contract wraps whichever backend you pick today, and the agent doesn't know or care which it is.

## 3. Core Concepts

### 3.1 Knowledge Graph (KG) — the persistent source of truth

A KG is a structured store of entities (nodes) and relationships (edges), where edges carry semantics. "User A *owns* Product B" is different from "User A *viewed* Product B" — both are edges from A to B, but they tell the agent different things [4].

The KG is **persistent** and **canonical**: it lives in a durable store, is updated as the domain evolves, and is shared across requests. Multiple agents (or multiple invocations of the same agent) read from it.

The KG's schema is the agent's vocabulary. If your domain has entities like *Document*, *Author*, *Topic*, *Reviewer*, *Approval*, and you want the agent to reason about "documents authored by X that are awaiting approval from a reviewer who has previously rejected a similar topic," your KG schema must encode those entities and edges explicitly. The schema is a design artifact, not an emergent thing.

### 3.2 Context Graph (CG) — the runtime per-request snapshot

A CG is what the agent *actually uses* during a single request. It is **per-request**, **runtime-assembled**, and **smaller than the KG** — typically a subgraph the agent has stitched together by following relevant edges from a seed entity. The CG plus any unstructured evidence (text snippets from a vector retrieval) becomes the prompt context [3].

Concretely: the agent has a question about Document X. It queries the KG: "give me X, its author, the author's recent documents, the reviewers assigned to X, and the reviewers' recent decisions on similar topics." The result is the CG for this request — a few dozen nodes and edges out of a KG that may have millions. The CG is what gets serialized into the prompt.

This separation matters because:

- The KG can be huge (millions of nodes) but the CG is bounded (hundreds of nodes max, often dozens).
- The CG is request-specific — assembling it is the agent's reasoning step, not a static config.
- Different requests for the same entity produce different CGs depending on what the agent is trying to answer.

### 3.3 When KG retrieval beats vector RAG (and when it doesn't)

Graph-backed retrieval wins when:

- The question is **multi-hop**: "find X that has property P, was created by Y who has worked on Z." Vector similarity over flat embeddings can find X but cannot traverse to Y and Z [4].
- The relationships carry **typed semantics** that affect the answer. "Won contract" vs "lost contract" is different even when both edges are A→B.
- The answer depends on **structural patterns** (cycles, paths, neighborhoods) rather than text similarity.

Vector RAG wins (or graph adds little) when:

- The question is single-hop text similarity ("find documents like this one").
- The domain is fundamentally unstructured (long-form prose) with no useful entity decomposition.
- The relational structure exists but is shallow (one or two hops max).

Production systems often use **hybrid retrieval**: vector recall finds candidate entities, the KG/CG step expands those candidates into the relational context the prompt needs [5]. Buyer intent to adopt hybrid retrieval roughly tripled from January to March 2026 in the enterprise RAG surveys [3].

### 3.4 Three viable backends — the trade-off space

**Neo4j** — purpose-built graph database, Cypher query language.

- *Pros:* Relationships are physical pointers; multi-hop traversal is roughly constant-time per hop regardless of graph size [1]. Cypher is concise and natural for relational questions.
- *Cons:* Another database to operate (backup, monitor, patch, secure). License/cost considerations for the commercial edition. Adds another component to data-residency / compliance scopes.

**Postgres recursive CTE** — use your existing Postgres with `WITH RECURSIVE` queries.

- *Pros:* Zero new infrastructure. Team already knows the database. Suitable for shallow traversal (1–3 hops) at modest cardinality [2].
- *Cons:* Recursive CTEs become expensive past 3–4 hops or when branching factors are high — by depth 4 with branching 10, you visit 10,000 nodes per query. Performance depends heavily on indexing you don't naturally think about until queries get slow [1][2].

**NetworkX in-process** — Python graph library, in-memory.

- *Pros:* Zero infrastructure. Pure Python algorithms, lots of options. Fast for small graphs.
- *Cons:* In-memory only — doesn't survive process restart. Doesn't scale beyond what fits in one process. Concurrency is the application's problem [1].

There is no universally right answer. The decision drivers are: data-residency / compliance boundary (does adding Neo4j cross a regulatory line?), ops budget (do you have the team capacity to operate another database?), query latency targets (sub-200ms p95 changes the answer), graph size, and traversal depth.

### 3.5 The scaffold orchestrator pattern

The pattern: the agent does not query the graph directly. It calls a **stable `kg_query` tool** whose contract is independent of the backend. Behind the tool, an adapter translates the query into Cypher, Postgres CTE, or NetworkX traversal — but the agent doesn't see which [4].

```text
   agent ──tool_call(kg_query, params)──▶ [ kg_query adapter ] ──▶ Neo4j
                                                            └──▶ Postgres CTE
                                                            └──▶ NetworkX
```

Why this matters:

- The backend choice can be revisited as data scale or operational realities change without rewriting the agent's prompts or tools.
- A/B comparisons between backends become tractable — same tool contract, swap the adapter, measure.
- Operational rollback is straightforward — if Neo4j is down, the adapter can fall back to a degraded mode on Postgres for read-only queries.

The cost of the pattern is a typed schema for the tool's inputs and outputs (a Pydantic model or equivalent) that all backends must honor. This is the same discipline as keeping a stable interface in front of any swappable persistence — Repository pattern, hexagonal architecture, port-and-adapter — applied to graph access.

### 3.6 The CG-as-prompt-context discipline

Once the CG is assembled, it has to enter the prompt. Two shapes are common:

- **Serialized text** — convert the subgraph to a structured text format ("Entity A is connected to B by relationship R; B has property P; …"). Easy for the model to consume but verbose [3].
- **Tabular** — present the subgraph as a small table of entity rows with a separate edge table. More compact for models that handle tables well.

Either way, the CG has to be **bounded** before it enters the prompt. A 200-node subgraph serialized as text can blow past the context budget for a moderately complex request. Production systems prune the CG by relevance (e.g., top-K entities by graph-centrality or by query-similarity) before serialization [3][5].

## 4. Generic Implementation

A scaffold orchestrator with a swappable backend. The domain is generic: a **scientific-publication assistant** (no federal-acquisitions overlap) where the agent needs to traverse author/paper/citation/affiliation relationships.

```python
from typing import Protocol, TypedDict
from langgraph.graph import StateGraph, START, END
from langgraph.types import Command

# --- Stable tool contract (the scaffold) ---
class KGQuery(TypedDict):
    seed_entity_id: str
    traversal_pattern: str   # e.g., "author->papers->citations->cited_authors"
    max_hops: int
    max_results: int

class KGResult(TypedDict):
    nodes: list[dict]   # [{"id": ..., "type": ..., "props": {...}}]
    edges: list[dict]   # [{"src": ..., "dst": ..., "type": ..., "props": {...}}]

class KGBackend(Protocol):
    def query(self, q: KGQuery) -> KGResult: ...

# --- Backend implementations ---
class Neo4jBackend:
    def __init__(self, driver): self.driver = driver
    def query(self, q: KGQuery) -> KGResult:
        # Cypher translation
        cypher = """
        MATCH p = (seed {id: $id})-[*1..$hops]-(other)
        WHERE seed.type = $traversal_root
        RETURN nodes(p) AS nodes, relationships(p) AS edges
        LIMIT $max_results
        """
        with self.driver.session() as s:
            rows = s.run(cypher, id=q["seed_entity_id"], hops=q["max_hops"],
                         max_results=q["max_results"]).data()
        return _normalize_neo4j(rows)

class PostgresCTEBackend:
    def __init__(self, conn): self.conn = conn
    def query(self, q: KGQuery) -> KGResult:
        # Recursive CTE translation
        sql = """
        WITH RECURSIVE traversal AS (
          SELECT id, parent_id, depth, edge_type FROM entities
          WHERE id = %(seed)s
          UNION ALL
          SELECT e.id, e.parent_id, t.depth + 1, e.edge_type
          FROM entities e JOIN traversal t ON e.parent_id = t.id
          WHERE t.depth < %(hops)s
        )
        SELECT * FROM traversal LIMIT %(max_results)s
        """
        cur = self.conn.cursor()
        cur.execute(sql, {"seed": q["seed_entity_id"],
                          "hops": q["max_hops"],
                          "max_results": q["max_results"]})
        return _normalize_postgres(cur.fetchall())

class NetworkXBackend:
    def __init__(self, graph): self.G = graph
    def query(self, q: KGQuery) -> KGResult:
        import networkx as nx
        nodes = nx.single_source_shortest_path_length(
            self.G, q["seed_entity_id"], cutoff=q["max_hops"]
        )
        # Trim by max_results, normalize to KGResult shape
        return _normalize_nx(self.G, list(nodes)[:q["max_results"]])

# --- The agent only sees the contract ---
def kg_query_tool(backend: KGBackend):
    def _tool(seed_entity_id: str, traversal_pattern: str,
              max_hops: int = 3, max_results: int = 50) -> KGResult:
        return backend.query({
            "seed_entity_id": seed_entity_id,
            "traversal_pattern": traversal_pattern,
            "max_hops": max_hops,
            "max_results": max_results,
        })
    return _tool

# Agent wiring — swap the backend without changing the agent
backend = Neo4jBackend(driver=neo4j_driver)  # or PostgresCTEBackend, or NetworkXBackend
tool = kg_query_tool(backend)
# ...register `tool` with the supervisor / worker that calls it
```

What each piece does:

- **`KGQuery` and `KGResult`** are the typed schema — the tool contract every backend honors.
- **`KGBackend` protocol** is the swappable port.
- **`Neo4jBackend`, `PostgresCTEBackend`, `NetworkXBackend`** are the adapters; each translates the stable query into its native language.
- **`kg_query_tool(backend)`** binds an adapter for the runtime. The agent registers the resulting callable as a tool and never knows which backend it is calling.

Two example traversals expressed in two languages:

```cypher
-- Cypher: papers by authors who collaborated with author X in the last 5 years
MATCH (x:Author {id: $author_id})-[:CO_AUTHORED]-(collab:Author)-[:WROTE]->(p:Paper)
WHERE p.year >= 2021
RETURN p, collab LIMIT 50
```

```sql
-- Postgres recursive CTE: same traversal (shallow, 2 hops)
WITH RECURSIVE collabs AS (
  SELECT b.author_b AS collab_id, 1 AS depth
  FROM co_authorships b WHERE b.author_a = %(author_id)s
)
SELECT p.* FROM collabs c
JOIN authorships au ON au.author_id = c.collab_id
JOIN papers p ON p.id = au.paper_id
WHERE p.year >= 2021
LIMIT 50
```

The Cypher version reads more naturally for relational thinking; the SQL is fine here but starts to compound when traversal depth grows past 3.

## 5. Real-world Patterns

**E-commerce — recommendation graph.** Marketplace recommendation systems use a KG of `User`, `Item`, `Category`, `Brand`, with edges like `purchased`, `viewed`, `wishlisted`, `belongs_to`. The CG for one shopper's session is the subgraph traversed from their `User` node: their recent purchases, those purchases' categories, other items in those categories purchased by similar users. Neo4j is a common backend at scale because the multi-hop traversals dominate the workload [4].

**Healthcare — adverse-event reasoning.** Pharmacovigilance systems use a KG of `Drug`, `Patient`, `Condition`, `AdverseEvent`, with edges like `prescribed_to`, `manifested_in`, `interacts_with`. The CG for one case is a subgraph traced from the patient and the drug in question, expanding outward to drugs with similar mechanisms and patients with similar profiles. Postgres recursive CTEs are common here because the data already lives in regulated relational stores and adding Neo4j crosses a compliance boundary [2].

**SaaS — internal-tool knowledge assistant.** A B2B platform with hundreds of services, services-with-services dependencies, on-call-rotations, and historical-incidents uses a KG to let an assistant answer "who do I escalate this incident to" — a multi-hop traversal from service → owning-team → on-call → escalation-chain. NetworkX in-process is sufficient because the entity count is small (~10k nodes), it all fits in one process, and the team didn't want to add another datastore [1].

**Logistics — supply-chain provenance.** Container-tracking systems use a KG of `Container`, `Vessel`, `Port`, `Carrier`, `Customs`. The CG for one container traces its full journey including transshipments and customs events. Neo4j is the common backend because the depth of provenance traversal (often 8–12 hops) makes recursive CTEs impractical and the graph is too large for in-process [4].

## 6. Best Practices

- **Design the KG schema explicitly before writing the agent.** Entities, edges, edge-type semantics — these are the agent's vocabulary; emergent schemas produce inconsistent agent behavior.
- **Bound the CG before it enters the prompt.** Prune by relevance, centrality, or path length. A 200-node subgraph serialized to text often blows the context budget.
- **Use the scaffold orchestrator pattern from day one.** A stable `kg_query` tool contract decouples backend choice from agent design — the cost of adding it is low, the cost of skipping it is rewriting tool calls later.
- **Pick the backend by ops + data-residency + depth + cardinality.** Neo4j for deep, large, traversal-dominated workloads; Postgres CTE for shallow, modest-cardinality workloads where you already have Postgres; NetworkX for small graphs where adding a database isn't justified.
- **Cap traversal depth and result count at the tool boundary.** Unbounded traversal is a DoS against your own infrastructure.
- **Hybrid retrieval often beats pure KG or pure vector.** Use vector to find candidate seed entities; use the KG to expand to relational context.
- **Type the tool's inputs and outputs.** Pydantic / TypedDict / JSON Schema — the typed contract is what makes backend swapping safe.

## 7. Hands-on Exercise

**Whiteboard + small SQL exercise (20 min).** You are designing the KG for a *music-streaming recommendation assistant* (no federal-acquisitions overlap). The domain has the following entities and relationships:

- `User` — listeners
- `Track` — songs
- `Artist` — creators of tracks (a track may have multiple artists)
- `Album` — collections of tracks
- `Genre` — track classifications
- Edges: `User --listened_to--> Track`, `Track --by_artist--> Artist`, `Track --on_album--> Album`, `Track --in_genre--> Genre`, `Artist --collaborated_with--> Artist`

Tasks:

1. Draw the KG schema with entity types and edge types labeled.
2. Write the Postgres recursive CTE for: "Given a user, find all artists who collaborated with artists this user has listened to, where the user has not yet listened to any of the collaborating artist's tracks." (2–3 hops.)
3. Write the equivalent Cypher query.
4. Decide which backend (Neo4j / Postgres CTE / NetworkX) you would pick for an MVP with 100k users, 1M tracks, average 20 plays per user per session. Give one-sentence justifications keyed to ops budget, traversal depth, and cardinality.

**What good looks like:** A clear schema diagram. A recursive CTE that uses a `WITH RECURSIVE` clause with an explicit depth cap, joining `listened_to → by_artist → collaborated_with → (anti-join on listened_to)`. A Cypher query that uses variable-length path `[:by_artist|collaborated_with*1..3]` syntax. A backend pick that names a concrete reason — for instance, "Postgres CTE — traversal depth 3 fits within CTE practical limits, ops team already operates Postgres, cardinality (1M tracks × 20 plays = 20M edges) is large but indexable; adding Neo4j adds an ops surface without a corresponding traversal-depth win at this scale."

## 8. Key Takeaways

- *What is the difference between a Knowledge Graph and a Context Graph?* (KG is the persistent source of truth; CG is the runtime per-request subgraph assembled from the KG and stitched into the prompt.)
- *When does graph-backed retrieval beat vector RAG?* (Multi-hop questions, typed-semantic relationships, structural-pattern queries; vector is fine for single-hop text similarity.)
- *What does the scaffold orchestrator pattern give you?* (Backend choice becomes an ADR, not an architectural lock-in; agent prompts and tools are stable while you swap Neo4j/Postgres/NetworkX underneath.)
- *What are the decision drivers when picking among Neo4j, Postgres recursive CTE, and NetworkX?* (Operational burden, traversal depth, cardinality, data residency, ops budget — there is no universal answer.)
- *Why must the CG be bounded before it enters the prompt?* (Unbounded subgraph serialization blows context budgets; production systems prune by relevance, centrality, or path length.)

## Sources

1. [Neo4j vs PostgreSQL: Graph DB or Relational? 2026 (Modern Datatools)](https://www.modern-datatools.com/compare/neo4j-vs-postgresql) — retrieved 2026-05-26
2. [Graph Retrieval using Postgres Recursive CTEs (sheshbabu.com)](https://www.sheshbabu.com/posts/graph-retrieval-using-postgres-recursive-ctes/) — retrieved 2026-05-26
3. [Context architecture is replacing RAG as agentic AI pushes enterprise retrieval to its limits (VentureBeat)](https://venturebeat.com/data/context-architecture-is-replacing-rag-as-agentic-ai-pushes-enterprise-retrieval-to-its-limits) — retrieved 2026-05-26
4. [Neo4j GraphRAG Context Provider for Agent Framework (Microsoft Learn)](https://learn.microsoft.com/en-us/agent-framework/integrations/neo4j-graphrag) — retrieved 2026-05-26
5. [Awesome-GraphRAG curated resources (DEEP-PolyU)](https://github.com/DEEP-PolyU/Awesome-GraphRAG) — retrieved 2026-05-26

Last verified: 2026-05-26

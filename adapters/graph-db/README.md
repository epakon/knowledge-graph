# Graph DB Adapter

> **Status: planned.** No graph database backend is implemented yet. This document defines the migration path, node/edge mapping, and import procedure for when a dedicated graph DB is introduced.

## What the Graph DB is — and is not

The graph DB is a **structural projection** of the content storage, not a content copy.

| | Content storage (Confluence) | Graph DB |
|---|---|---|
| **What it holds** | Full knowledge — definitions, SQL, prose, reason/consequence, version history | Structural skeleton — node labels, key properties, typed relationships |
| **Source of truth** | Yes | No — derived from content storage |
| **Authoring surface** | Yes — humans and agents read and write here | No — never edited directly |
| **Access pattern** | MCP tool calls (page read/write), semantic search | Cypher / Gremlin — traversal, path finding, aggregations |

**Why the Graph DB exists — three purposes:**

1. **Duplicate prevention** — uniqueness constraints on `(label, name)` guard against duplicate nodes that content storage search alone can miss. Before creating a new node, check the index.
2. **Fast graph traversal** — path finding, neighbor lookup, and impact analysis queries that are expensive or impossible via MCP.
3. **Lineage and traceability** — every node retains its `page_id`, linking any graph query result back to the exact source page in the content storage. Graph traversal answers *what is connected*; `page_id` tells you *where the full definition lives*.

### What does "structural projection" mean?

A structural projection extracts only what is needed to represent the graph's topology, identity, and classification — not its meaning:

- **Projected:** node labels, identity keys (`name`), classification properties (`domain`, `kind`, `status`, `mandatory`), relationship types, edge properties (`reason`, `consequence`), and `page_id` for traceability
- **Not projected:** the full knowledge content — business definitions, SQL expressions, synonyms, description prose, `always_ask` questions, full rule text

The result is a graph you can traverse, query, and deduplicate efficiently. To read the full meaning of any node, follow `page_id` back to the source page in the content storage.

---

## Role in the architecture

```
Content storage (Confluence)        ← source of truth, full knowledge
        │
        │  Knowledge Graph API — snapshot + structural export
        ▼
  JSON indexes                       ← structural projection (labels, properties, relationships)
  (kg-node-index.json,                  NOT full page bodies
   kg-edge-index.json)
        │
        │  Graph DB import (MERGE upsert)
        ▼
   Graph DB (Neo4j, Amazon Neptune, ArangoDB, …)
        │
        │  Cypher / Gremlin / graph API
        ▼
  Agent queries — traversal, path finding, impact analysis, duplicate checks
```

Once a graph DB is in place, the content storage remains the **human authoring layer** and becomes the graph DB's upstream source. The Knowledge Graph API handles the sync between them.

---

## Node and edge mapping

The spec uses generic graph terms throughout. The mapping to any property graph store is direct:

| Spec term | Graph DB equivalent |
|---|---|
| Node type | Node label (e.g. `Subject`, `Measure`) |
| Identity key (`name`) | Uniqueness constraint on the label |
| Node properties | Node properties |
| Hyperlink edge | Relationship type (no properties) |
| Reified edge (Relationship page) | Relationship type with properties (`reason`, `consequence`) |
| Edge kind | Relationship type name |

**Relationship pages** are not nodes in the graph DB. They are flattened into typed relationships with properties during import. Their wiki pages remain in the content storage for human readability.

**Domain nodes** can be imported as nodes or treated as a property (`domain: "Sales"`) on other nodes — decision deferred to implementation.

> For the authoritative node type definitions, properties, and valid edge combinations see [`spec/schema.yaml`](../../spec/schema.yaml). The tables below show only the graph DB translation layer.

### What the structural projection contains — per node type

Each node is projected from its content storage page into a graph node with a small set of structural properties. Everything else — the full knowledge content — stays in the page and is accessed via `page_id`.

| Node type | Projected into graph DB | Stays in content storage only |
|---|---|---|
| `Subject` | `name`, `domain`, `status`, `page_id` | `business_definition`, `scope` text |
| `Domain` | `name`, `page_id` | `owner` |
| `Table` | `name`, `domain`, `table_kind`, `status`, `page_id` | `source`, `description`, semantic annotations table, field list |
| `Measure` | `name`, `domain`, `kind`, `status`, `page_id` | `definition_sql`, `synonyms` |
| `Attribute` | `name`, `domain`, `kind`, `access_modifier`, `status`, `page_id` | `expression_sql`, `synonyms` |
| `Filter` | `name`, `domain`, `mandatory`, `status`, `page_id` | `predicate_sql`, `synonyms` |
| `VerifiedQuery` | `name`, `domain`, `status`, `verified_by`, `verified_at`, `onboarding_question`, `page_id` | `question` text, `sql` |
| `BusinessRule` | `name`, `domain`, `status`, `page_id` | `definition` text, `consequence_if_violated` text |
| `Disambiguation` | `name`, `domain`, `status`, `page_id` | `always_ask` text |
| `Relationship` | *(not a node)* flattened into a typed relationship with `reason` + `consequence` | Relationship page body — full prose context |

**Rule of thumb:** if a property is needed to traverse, filter, or deduplicate the graph, it is projected. If it is needed to understand the meaning, it stays in the content storage.

### Hyperlink edge kinds → relationship types

Seven edge kinds with no properties. Back-references on content storage pages are navigation shortcuts and are **not** imported into the graph.

| Edge kind | Relationship type | Notes |
|---|---|---|
| `implement` | `IMPLEMENTS` | Subject → any; Measure/BusinessRule/Filter → VerifiedQuery only |
| `relatedTo` | `RELATED_TO` | Symmetric generic cross-link |
| `calculate` | `CALCULATES` | Table → Attribute, Measure | Table is the source exposing the derived column (Attribute) or computing the KPI (Measure) |
| `joinedTo` | `JOINED_TO` | Symmetric; join key stored as property `on` |
| `disambiguate` | `DISAMBIGUATES` | Subject → Disambiguation |
| `apply` | `APPLIES_TO` | BusinessRule → Table, Measure |
| `contain` | `CONTAINS` | Domain → Table, Measure, Filter, VerifiedQuery, BusinessRule |

### Reified edge kinds → relationship types with properties

`Relationship` pages are flattened into typed relationships. The `reason` and `consequence` fields from the page body become relationship properties.

| Edge kind | Relationship type | Source | Target | Properties |
|---|---|---|---|---|
| `mandatory` | `MANDATORY_FOR` | Filter | Table | `reason`, `consequence` |
| `requires` | `REQUIRES` | Measure | Filter | `reason`, `consequence` |
| `guards` | `GUARDS` | Filter | Measure | `reason`, `consequence` |
| `overrides` | `OVERRIDES` | BusinessRule | Attribute | `reason`, `consequence` |
| `demonstrates` | `DEMONSTRATES` | VerifiedQuery | BusinessRule | `reason`, `consequence` |

### Edge conflict rule

A given `(source, target)` pair must have at most one relationship of each type. When the same pair has both a `RELATED_TO` hyperlink and a reified relationship (e.g. `REQUIRES`), the reified relationship takes precedence — the weaker `RELATED_TO` should be removed before import. This is enforced by the audit step in the snapshot script.

### Uniqueness constraints (Cypher example)

```cypher
CREATE CONSTRAINT FOR (n:Subject)          REQUIRE n.name IS UNIQUE;
CREATE CONSTRAINT FOR (n:Domain)           REQUIRE n.name IS UNIQUE;
CREATE CONSTRAINT FOR (n:Table)            REQUIRE n.name IS UNIQUE;
CREATE CONSTRAINT FOR (n:Measure)          REQUIRE n.name IS UNIQUE;
CREATE CONSTRAINT FOR (n:Attribute)        REQUIRE n.name IS UNIQUE;
CREATE CONSTRAINT FOR (n:Filter)           REQUIRE n.name IS UNIQUE;
CREATE CONSTRAINT FOR (n:VerifiedQuery)    REQUIRE n.name IS UNIQUE;
CREATE CONSTRAINT FOR (n:BusinessRule)     REQUIRE n.name IS UNIQUE;
CREATE CONSTRAINT FOR (n:Disambiguation)   REQUIRE n.name IS UNIQUE;
```

---

## Import procedure

The node index (`kg-node-index.json`) and edge index (`kg-edge-index.json`) are the intermediate representation between the content storage and the graph DB. They are a **structural extract** — each entry contains labels, key properties, and relationship metadata, but not the full page body (definition text, SQL, prose). Each entry maps 1:1 to a `MERGE` upsert.

### Node index fields

| Field | Description |
|---|---|
| `name` | Short name (identity key). Used for graph `MERGE` / upsert. |
| `label` | Node label (matches node type: `Subject`, `Measure`, etc.). |
| `page_id` | Content storage page ID — retained for lineage and traceability back to the source page. |
| `domain` | Domain scope (`global` for Subjects, domain name for all others). |
| `status` | `active` or `deprecated`. |

### Edge index fields

| Field | Description |
|---|---|
| `source` | Full title of the source node (e.g. `Measure: Revenue`). |
| `target` | Full title of the target node. |
| `relationship_type` | Graph relationship type (from mapping tables above, e.g. `REQUIRES`). |
| `style` | `hyperlink` (no properties) or `reified` (has `reason` + `consequence`). |
| `via` | For reified edges: the Relationship page title. `null` for hyperlinks. |
| `properties` | For reified edges: `{ "reason": "...", "consequence": "..." }`. Empty object for hyperlinks. |

### Import steps

1. **Import nodes** from the node index — upsert by `(label, name)` and set all properties.
2. **Import edges** from the edge index — hyperlinks first, then reified edges (they may reference the same node pairs).
3. **Skip back-references** — edges where the label contains `<-` are navigation artifacts, not graph edges.
4. **Relationship pages** are already flattened into the edge index as reified edges; their content storage pages can be archived post-migration.
5. **Retain `page_id`** on every node for traceability back to the source page.

### Node import (Cypher example)

```cypher
MERGE (n:Subject {name: $name})
SET n.page_id = $page_id,
    n.domain  = $domain,
    n.status  = $status
```

### Edge import (Cypher example)

```cypher
// Hyperlink edge
MATCH (a {name: $source}), (b {name: $target})
MERGE (a)-[r:IMPLEMENTS]->(b)

// Reified edge
MATCH (a {name: $source}), (b {name: $target})
MERGE (a)-[r:REQUIRES]->(b)
SET r.reason    = $reason,
    r.consequence = $consequence
```

---

## Sync strategy

Not yet defined. Candidates:

- **Periodic batch** — run the snapshot + import pipeline on a schedule (e.g. nightly).
- **Event-driven** — trigger export on every Confluence page update via webhook.
- **On-demand** — agent or operator triggers a sync manually before a query session.

The Knowledge Graph API in [`adapters/confluence/graph-api.md`](../confluence/graph-api.md) is designed with this in mind — its core operations map directly to what a graph DB client would expose. When the backend changes, only the transport layer is replaced.

---

## Candidate databases

The schema is intentionally generic. Any property graph store works:

| Database | Notes |
|---|---|
| Neo4j | Cypher query language; node labels and relationship types map directly |
| Amazon Neptune | Gremlin or SPARQL; property graph model compatible |
| ArangoDB | Multi-model (document + graph); AQL query language |

No database has been selected. The choice depends on infrastructure, query patterns, and operational requirements at implementation time.

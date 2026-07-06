# Neo4j Adapter

> Implementation guide for the [Knowledge Graph Specification](../../SPEC.md) on Neo4j Community Edition.

This document maps the Graph DB adapter contract to Neo4j-specific constructs: node/edge representation, deployment, import procedure, programmatic access, and sync strategy.

---

## Deployment

Neo4j **Community Edition** is sufficient for this adapter — no Enterprise features (clustering, online backup, Bloom Full Access) are required.

Minimal local deployment via Docker:

```yaml
# docker-compose.yml
services:
  neo4j:
    image: neo4j:5-community
    ports:
      - "7474:7474"   # HTTP (Browser)
      - "7687:7687"   # Bolt
    environment:
      - NEO4J_AUTH=neo4j/<password>
    volumes:
      - neo4j_data:/data
volumes:
  neo4j_data:
```

| Setting | Value |
|---|---|
| Bolt URI | `bolt://<host>:7687` (or `bolt+s://` if TLS-terminated) |
| Browser / HTTP | `http://<host>:7474` |
| Default database | `neo4j` |
| Auth | Username/password, set via `NEO4J_AUTH` or `neo4j.conf` |

For a non-Docker install, see the official [Neo4j Community Edition download](https://neo4j.com/product/community-edition/) and Operations Manual.

---

## Node and edge mapping

The mapping from spec terms to Neo4j constructs is direct:

| Spec term | Neo4j equivalent |
|---|---|
| Node type | Node label (e.g. `Subject`, `Measure`) |
| Identity key (`name`) | Uniqueness constraint on the label |
| Node properties | Node properties |
| Hyperlink edge | Relationship type (no properties) |
| Reified edge (Relationship page) | Relationship type with properties (`reason`, `consequence`) |
| Edge kind | Relationship type name |

**Relationship pages** are not nodes in Neo4j. They are flattened into typed relationships with properties during import. Their wiki pages remain in the content storage for human readability.

**Domain nodes** can be imported as nodes or treated as a property (`domain: "Sales"`) on other nodes — decision deferred to implementation.

> For the authoritative node type definitions, properties, and valid edge combinations see [`spec/schema.yaml`](../../../spec/schema.yaml). The tables below show only the Neo4j translation layer.

### What the structural projection contains — per node type

Each node is projected from its content storage page into a Neo4j node with a small set of structural properties. Everything else — the full knowledge content — stays in the page and is accessed via `page_id`.

| Node type | Projected into Neo4j | Stays in content storage only |
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

Seven edge kinds with no properties. Back-references on content storage pages are navigation shortcuts and are **not** imported into Neo4j.

| Edge kind | Relationship type | Notes |
|---|---|---|
| `implement` | `IMPLEMENTS` | Subject → any; Measure/BusinessRule/Filter → VerifiedQuery only |
| `relatedTo` | `RELATED_TO` | Symmetric generic cross-link |
| `calculate` | `CALCULATES` | Table → Attribute, Measure — Table is the source exposing the derived column (Attribute) or computing the KPI (Measure) |
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

### Uniqueness constraints

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

Community Edition supports uniqueness constraints natively — no Enterprise license required.

---

## Import procedure

The node index (`kg-node-index.json`) and edge index (`kg-edge-index.json`) are the intermediate representation between the content storage and Neo4j. They are a **structural extract** — each entry contains labels, key properties, and relationship metadata, but not the full page body (definition text, SQL, prose). Each entry maps 1:1 to a `MERGE` upsert.

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
| `relationship_type` | Neo4j relationship type (from mapping tables above, e.g. `REQUIRES`). |
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

## Programmatic access

Unlike the content storage adapters (Confluence, Markdown), which each need a dedicated `graph-api.md` to provide a stable, simplified interface over a wiki/filesystem API for bulk operations, Neo4j already ships **official, vendor-supported client libraries** that cover this need directly. No bespoke Knowledge Graph API document is required for this adapter — use the official driver for your language:

| Language | Package | Notes |
|---|---|---|
| Python | [`neo4j`](https://pypi.org/project/neo4j/) | Official driver; parameterized Cypher execution, sessions, transactions |
| JavaScript / TypeScript | [`neo4j-driver`](https://www.npmjs.com/package/neo4j-driver) | Official driver |
| CLI | `cypher-shell` | Bundled with the Neo4j distribution; useful for scripting the import/constraint steps directly |

Minimal example (Python) for the node import step described above:

```python
from neo4j import GraphDatabase

driver = GraphDatabase.driver("bolt://localhost:7687", auth=("neo4j", "<password>"))

def import_node(tx, name, label, page_id, domain, status):
    tx.run(
        f"MERGE (n:{label} {{name: $name}}) "
        "SET n.page_id = $page_id, n.domain = $domain, n.status = $status",
        name=name, page_id=page_id, domain=domain, status=status,
    )

with driver.session() as session:
    for node in node_index:
        session.execute_write(import_node, **node)
```

This same driver is what the sync pipeline (batch, event-driven, or on-demand — see below) uses to run the `MERGE` import steps against the node and edge indexes.

---

## Sync strategy

Not yet defined. Candidates:

- **Periodic batch** — run the snapshot + import pipeline on a schedule (e.g. nightly). Each content storage backend runs its own snapshot; all outputs feed the same import step.
- **Event-driven** — trigger export on page update (webhook for Confluence, file-watcher or CI pipeline for Markdown). Each backend triggers independently.
- **On-demand** — agent or operator triggers a sync manually before a query session.

The Knowledge Graph API in [`adapters/engine/confluence/graph-api.md`](../confluence/graph-api.md) is designed with this in mind — its core operations map directly to what the Neo4j driver expects as import input. Each content storage adapter provides its own snapshot pipeline implementation using the same JSON index format.

---

## Exploration and visualization (optional)

Agents query Neo4j via Cypher through the official driver, as described above. For **human** visual exploration of the graph, exploration tooling is optional and interchangeable — it does not affect graph semantics or the import procedure.

The default option documented for this adapter is **Neo4j Bloom Basic / Explore**, via the Aura Console — see the [Bloom Explore addon](addons/bloom-explore.md). Other tools (NeoDash, yWorks Data Explorer, SemSpect, Graphlytic, or a custom build on the Neo4j Visualization Library) work equally well against the same Neo4j instance and can be documented as additional addons without changing this adapter.

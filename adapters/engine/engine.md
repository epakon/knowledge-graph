# Engine Adapters

> Part of the adapter taxonomy defined in [`adapters/adapters.md`](../adapters.md).

Engine adapters implement the **storage and traversal infrastructure** the Knowledge Graph runs on. They do not add knowledge — they provide the surface on which knowledge is authored, queried, and navigated.

Two categories:

- **Content storage** — source of truth and authoring surface; full knowledge lives here
- **Graph DB** — structural projection derived from content storage; for traversal, duplicate prevention, and lineage

---

## Content storage

Human-facing systems where nodes and edges are created and maintained as pages or documents (wikis, shared drives, Markdown files). The full knowledge lives here: business definitions, SQL expressions, reason/consequence prose, synonyms, version history. This is the source of truth and the only authoring surface.

Any content storage that supports the following capabilities can implement this specification:

| Capability | Why it is required |
|---|---|
| **Hierarchical organisation** | The canonical space structure (`vocabulary/`, `<domain>/tables/`, etc.) must be expressible as nested containers |
| **Cross-document links with custom labels** | Edge statements (`Source: X edgeKind -> Target: Y`) must be embeddable as the visible label of a navigable link |
| **Version history** | Every node update must be versioned with a structured comment; rollback to a prior version must be possible |
| **Full-text or semantic search** | Agents must be able to locate nodes by name, type, and content without knowing the exact path |
| **Programmatic read and write** | A Python API or equivalent must be available for Knowledge Graph API operations (collect, snapshot, bulk update) |

A deployment may combine multiple content storage backends for different knowledge layers — for example, Confluence for human-authored business knowledge and Markdown files for auto-generated technical documentation from the codebase. Each backend has its own engine adapter; the graph DB merges their structural projections into a single graph.

### Content storage adapter contract

Every content storage adapter MUST provide four documents:

**1. `<backend>-adapter.md` — Backend mapping**

Maps each spec concept to the backend-specific construct:

| Spec concept | Must be covered |
|---|---|
| Node | How a node is represented (page, file, record) |
| Container / folder | How the canonical hierarchy is implemented |
| Page link with edge label | How edge statements are encoded in links |
| Version comment | How structured version comments are stored and retrieved |
| Semantic search | What search mechanism is used and how to query it |
| Graph snapshot | How the snapshot pipeline reads from this backend |

Also covers: naming conventions, common pitfalls, backend-specific constraints.

**2. `agent-skill.md` — Agent operational spec**

Must cover five workflows:

| Workflow | Trigger |
|---|---|
| A — Read | User asks what something means, how a measure is computed, which filters apply |
| B — Write | User asks to add a new node (single or batch) |
| C — Update | User asks to change a field or section on an existing node |
| D — Navigate | User asks for lineage, dependencies, or graph traversal |
| E — Version history | User asks what changed, who changed it, or wants to roll back |

For each workflow: step-by-step instructions using backend-specific tooling, threshold for switching from direct operations to the Knowledge Graph API (default: ≤5 nodes → direct, 6+ → API), and always-apply constraints.

**3. `graph-api.md` — Knowledge Graph API**

Programmatic interface for bulk and automated operations. Must cover:

- Core read and write operations (`get_page`, `write/update`, `collect_kg_pages`, `search_pages`, node path/ID resolution)
- Dry-run / testing pattern
- Higher-level operations: validate, fix links, inject back-references
- Common pitfalls specific to the backend

The JSON index output format is shared across all backends — see [spec/data-model.md §4–5](../../spec/data-model.md#4-node-index) for the node index and edge index schemas. Every adapter's snapshot pipeline must produce indexes conforming to this format so the graph DB import pipeline works without modification.

**4. `snapshot-pipeline.md` — Snapshot pipeline**

Documents the script that walks the content storage and produces the JSON snapshot and JSON indexes. Must cover:

- How to scope the snapshot (root page ID, root directory, or equivalent)
- Parsing rules: what counts as a node, what counts as an edge, what is excluded
- The snapshot JSON format (`page_id` carries the backend-specific node identifier)
- CLI reference for common operations (full snapshot, exclude node types, export Mermaid, write indexes, audit)
- Maintenance guidance

### Existing content storage adapters

| Adapter | Documents |
|---|---|
| [Confluence](confluence/) | `confluence-adapter.md`, `agent-skill.md`, `graph-api.md`, `snapshot-pipeline.md` |
| [Markdown (git)](markdown/) | `markdown-adapter.md`, `agent-skill.md`, `graph-api.md`, `snapshot-pipeline.md` |

---

## Graph DB

A structural projection of the content storage, not a copy.

| | Content storage (Confluence, Markdown, …) | Graph DB |
|---|---|---|
| **What it holds** | Full knowledge — definitions, SQL, prose, reason/consequence, version history | Structural skeleton — node labels, key properties, typed relationships |
| **Source of truth** | Yes | No — derived from content storage |
| **Authoring surface** | Yes — humans and agents read and write here | No — never edited directly |
| **Access pattern** | MCP tool calls, API or direct file read-write (per adapter), semantic search | Cypher / Gremlin / AQL, depending on backend — traversal, path finding, aggregations |

It is derived from the content storage via the snapshot pipeline (JSON indexes → graph import) and is never edited directly.

**Why the Graph DB exists — three purposes:**

1. **Duplicate prevention** — uniqueness constraints on `(label, name)` guard against duplicate nodes that content storage search alone can miss. Before creating a new node, check the index.
2. **Fast graph traversal** — path finding, neighbor lookup, and impact analysis queries that are expensive or impossible via direct content storage access.
3. **Lineage and traceability** — every node retains its `page_id`, linking any graph query result back to the exact source page in the content storage. Graph traversal answers *what is connected*; `page_id` tells you *where the full definition lives*.

### What does "structural projection" mean?

A structural projection extracts only what is needed to represent the graph's topology, identity, and classification — not its meaning:

- **Projected:** node labels, identity keys (`name`), classification properties (`domain`, `kind`, `status`, `mandatory`), relationship types, edge properties (`reason`, `consequence`), and `page_id` for traceability
- **Not projected:** the full knowledge content — business definitions, SQL expressions, synonyms, description prose, `always_ask` questions, full rule text

The result is a graph you can traverse, query, and deduplicate efficiently. To read the full meaning of any node, follow `page_id` back to the source page in the content storage. Each backend adapter's `<backend>-adapter.md` shows exactly which properties are projected per node type.

### Role in the architecture

```
Content storage A (e.g. Confluence)     ← business knowledge — human-authored
Content storage B (e.g. Markdown files) ← technical knowledge — auto-generated from codebase
        │                      │
        │  Knowledge Graph API — snapshot + structural export (one pipeline per backend)
        ▼                      ▼
  JSON indexes                       ← structural projection (labels, properties, relationships)
  (kg-node-index.json,                  NOT full page bodies
   kg-edge-index.json)
        │
        │  Graph DB import (MERGE upsert — merges projections from all backends)
        ▼
   Graph DB (e.g. Neo4j)
        │
        │  Cypher / Gremlin — traversal, path finding, impact analysis, duplicate checks
        │  Official driver / vendor client — programmatic access
        │  Exploration tooling (optional, addon) — human visual exploration
        ▼
  Agent queries and human exploration
```

Multiple content storage backends are supported. Each backend has its own engine adapter and snapshot pipeline; the JSON indexes are a common intermediate format, so this import step is backend-agnostic on the content storage side. A `page_id` on every node links any graph query result back to the source page in whichever backend holds it.

Once a graph DB is in place, the content storage layers remain the **authoring surfaces** and become the graph DB's upstream sources. The Knowledge Graph API handles the sync between them.

### Graph DB adapter contract

The graph DB adapter contract is lighter than the content storage contract:

- **No authoring surface, no agent skill** — the graph DB is never edited directly; only content storage adapters are authoring surfaces.
- **No snapshot pipeline** — that is owned by the upstream content storage adapters; the graph DB only consumes their JSON indexes.
- **No bespoke Knowledge Graph API document, if the backend doesn't need one** — content storage adapters need a `graph-api.md` to provide a stable, simplified interface over a wiki/filesystem API. A graph DB backend that ships its own official, vendor-supported client library (e.g. Neo4j's drivers) documents usage of that library directly instead of duplicating it.
- **No required exploration/visualization tooling** — human-facing graph exploration (e.g. Neo4j Bloom) is optional and documented as an addon, since interchangeable tools can serve the same purpose without affecting graph semantics.

What every graph DB adapter must cover, in a single `<backend>-adapter.md`: node/edge mapping, deployment, import procedure, programmatic access, and sync strategy.

### Existing graph DB adapters

| Adapter | Documents |
|---|---|
| [Neo4j](neo4j/) | `neo4j-adapter.md` |

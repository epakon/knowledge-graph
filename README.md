# Knowledge Graph Specification

A standard for building **graph-structured knowledge bases** that are human-readable, agent-consumable, and stored in wiki-style collaborative tools.

---

## What is this?

This specification defines a way to represent business knowledge — metric definitions, business rules, data filters, verified SQL queries, and their semantic relationships — as a graph of interlinked pages in a wiki or document store.

The graph is designed to be consumed by **AI agents** (LLMs, Snowflake Cortex, Cursor agents) at query time, replacing the need for custom RAG pipelines or export jobs. Knowledge is stored where humans already write it; agents retrieve it through semantic search.

### Core idea

```
Human writes knowledge as interlinked pages (wiki, Notion, GitHub, etc.)
        │
        ▼  semantic search (e.g. Glean, Vertex AI Search, pgvector)
Agent retrieves relevant pages at query time
        │
        ▼  graph traversal (follow typed edge links between pages)
Agent builds correct SQL / answers / instructions without hallucinating business logic
```

---

## Key concepts

| Concept | Description |
|---|---|
| **Node** | A single-topic page representing one business entity (a metric, a rule, a filter, etc.) |
| **Edge** | A typed link between two nodes, embedded in the page body as a readable label |
| **Relationship page** | A reified edge: a dedicated page that carries a typed link plus a reason and consequence |
| **Domain** | A scoped collection of nodes coupled to specific data tables |
| **Subject** | A global, domain-agnostic concept page — the shared business definition |

---

## Repository structure

```
SPEC.md                        Top-level normative specification
CHANGELOG.md                   Version history (semantic versioning)
LICENSE                        CC BY-NC 4.0

spec/
  data-model.md                Node types, edge kinds, validity rules, identity constraints
  link-format.md               Edge-statement link syntax
  page-templates.md            Canonical page structure for each node type
  space-structure.md           Logical hierarchy (folders / parent containers)
  versioning.md                Version comment format and policy
  schema.yaml                  Machine-readable schema

adapters/
  confluence/                  Content storage adapter (complete)
    confluence-adapter.md      Confluence-specific implementation (ac:link-body, MCP tools)
    agent-skill.md             Agent workflows: Read, Write, Update, Navigate, History
    graph-api.md               Knowledge Graph API for programmatic graph operations
    snapshot-pipeline.md       Snapshot + graph visualization pipeline
  graph-db/                    Graph DB adapter (planned)
    README.md                  Node/edge mapping, import procedure, sync strategy

examples/
  domain-layout-example.md     Illustrative domain structure
  node-pages-example.md        Sample pages for all node types
  graph-snapshot-example.json  Minimal snapshot JSON

index.md                       Full directory listing
```

---

## Quick start

1. Read [SPEC.md](SPEC.md) for the complete normative specification.
2. Read [spec/data-model.md](spec/data-model.md) for the node and edge schema.
3. Read [spec/page-templates.md](spec/page-templates.md) to understand what each page looks like.
4. Choose a backend and read the matching adapter doc (e.g. [adapters/confluence/confluence-adapter.md](adapters/confluence/confluence-adapter.md)).
5. See [examples/](examples/) for concrete illustrations.

---

## Status

**Version 1.3.0** — stable, in production use.

See [CHANGELOG.md](CHANGELOG.md) for the full version history.

---

## License

[CC BY-NC 4.0](LICENSE) — free to use and adapt for non-commercial purposes with attribution.
For commercial licensing, contact the copyright holder.

# Knowledge Graph Specification

A standard for building **graph-structured knowledge bases** that are human-readable, agent-consumable, and stored in wiki-style collaborative tools.

---

## What is this?

This specification defines a way to represent business knowledge — metric definitions, business rules, data filters, verified SQL queries, and their semantic relationships — as a graph of interlinked pages in a wiki or document store.

The graph is designed to be consumed by **AI agents** (LLMs, Snowflake Cortex, Cursor agents) at query time, replacing the need for custom RAG pipelines or export jobs. Knowledge is stored where humans already write it; agents retrieve it through semantic search.

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
| **Reification page** | A reified edge: a dedicated page that carries a typed link plus a reason and consequence |
| **Domain** | A scoped collection of nodes coupled to specific data tables |
| **Conceptual layer** | The domain-agnostic, stable part of the graph (`vocabulary/`) — see [spec/conceptual-layer.md](spec/conceptual-layer.md) |
| **Logical layer** | The per-domain, SQL-coupled part of the graph — see [spec/logical-layer.md](spec/logical-layer.md) |
| **Consumption layer** | The global part of the graph describing AI consumption surfaces (`ai/`) — see [spec/consumption-layer.md](spec/consumption-layer.md) |

---

## Repository structure

- [SPEC.md](SPEC.md) — Top-level normative specification (start here)
- [CHANGELOG.md](CHANGELOG.md) — Version history (semantic versioning)
- [LICENSE](LICENSE) — CC BY-NC 4.0
- **spec/** — Normative specification
  - [conceptual-layer.md](spec/conceptual-layer.md) — Conceptual layer: Concept, Subject, Process node types; stability test; bridge to logical layer
  - [logical-layer.md](spec/logical-layer.md) — Logical layer: node types, edge kinds, indexes, audit rules, graph-DB migration
  - [consumption-layer.md](spec/consumption-layer.md) — Consumption layer: `Agent` node type, `uses` edge kind, stability test, why it must never duplicate what it reads *(draft)*
  - [link-format.md](spec/link-format.md) — Edge-statement syntax, back-reference rules, visualization conventions
  - [page-templates.md](spec/page-templates.md) — Canonical page templates for all node types
  - [space-structure.md](spec/space-structure.md) — Logical hierarchy, container pages, naming conventions
  - [versioning.md](spec/versioning.md) — Version comment format, breaking change policy, rollback procedure
  - [governance.md](spec/governance.md) — Who changes graph objects and when review is required, staleness detection when a source system changes, change log and monitoring *(draft)*
  - [schema.yaml](spec/schema.yaml) — Machine-readable schema: node types, edge kinds, properties, audit rules
- **adapters/** — Engine adapters, source-system connectors, addons
  - [adapters.md](adapters/adapters.md) — Adapter taxonomy: engine adapters, connectors, addons
  - **engine/** — Graph engine and content storage adapters
    - [engine.md](adapters/engine/engine.md) — Engine adapter contract: content storage capability requirements, four-document contract, graph DB category
    - **confluence/** — Confluence content storage adapter
      - [confluence-adapter.md](adapters/engine/confluence/confluence-adapter.md) — Storage format, MCP tools, link encoding, Glean/Cortex integration
      - [agent-skill.md](adapters/engine/confluence/agent-skill.md) — Agent workflows: Read, Write, Update, Navigate, Version history
      - [graph-api.md](adapters/engine/confluence/graph-api.md) — Knowledge Graph API for programmatic graph operations
      - [snapshot-pipeline.md](adapters/engine/confluence/snapshot-pipeline.md) — Confluence → JSON snapshot → graph visualization pipeline
    - **markdown/** — Markdown files in git content storage adapter
      - [markdown-adapter.md](adapters/engine/markdown/markdown-adapter.md) — File format, YAML frontmatter, directory structure, link encoding, git versioning, auto-generation guidance
      - [agent-skill.md](adapters/engine/markdown/agent-skill.md) — Agent workflows: Read, Write, Update, Navigate, Version history
      - [graph-api.md](adapters/engine/markdown/graph-api.md) — Knowledge Graph API for programmatic file operations
      - [snapshot-pipeline.md](adapters/engine/markdown/snapshot-pipeline.md) — Markdown file tree → JSON snapshot → graph visualization pipeline; multi-backend index merge
      - **addons/**
        - [openwiki.md](adapters/engine/markdown/addons/openwiki.md) — Codebase documentation generator; promotes OpenWiki output into draft KG nodes
    - **neo4j/** — Neo4j graph DB adapter
      - [neo4j-adapter.md](adapters/engine/neo4j/neo4j-adapter.md) — Deployment, node/edge mapping, import procedure, programmatic access (official driver), sync strategy
      - **addons/**
        - [bloom-explore.md](adapters/engine/neo4j/addons/bloom-explore.md) — Optional human exploration via Neo4j Bloom Basic / Aura Console Explore; alternatives listed
  - **connectors/** — Source-system connectors
    - [connectors.md](adapters/connectors/connectors.md) — Connector contract: six mandatory sections every connector must implement
    - **sap-s4hana/** — SAP S/4HANA connector *(draft)*
      - [sap-s4hana-business.md](adapters/connectors/sap-s4hana/sap-s4hana-business.md) — Business layer: Vocabulary-layer concepts — SAP standard and project decisions
      - [sap-s4hana-technical.md](adapters/connectors/sap-s4hana/sap-s4hana-technical.md) — Technical layer: entry point; node/edge type mapping, field tables, extraction protocol
      - [sap-s4hana-import.md](adapters/connectors/sap-s4hana/sap-s4hana-import.md) — Import procedure: seed selection, CDS dependency discovery, DDL extraction, validation
      - **addons/**
        - [sap-s4hana-lineage-explorer.md](adapters/connectors/sap-s4hana/addons/sap-s4hana-lineage-explorer.md) — Lineage explorer: read-only full CDS/DDIC dependency map, shared visualization
    - **sap-hana/** — SAP HANA connector *(draft)*
      - [sap-hana-technical.md](adapters/connectors/sap-hana/sap-hana-technical.md) — Technical layer: entry point; node/edge type mapping, field tables, DWH layer conventions, extraction protocol
      - [sap-hana-import.md](adapters/connectors/sap-hana/sap-hana-import.md) — Import procedure: seed selection, dependency walk, routing, validation
      - **addons/**
        - [sap-hana-lineage-explorer.md](adapters/connectors/sap-hana/addons/sap-hana-lineage-explorer.md) — Lineage explorer: read-only HANA object dependency map; upstream/downstream walk; shared snapshot format
    - **dbt/** — dbt connector
      - [dbt-technical.md](adapters/connectors/dbt/dbt-technical.md) — Technical layer: node/field/edge mapping for models, sources, seeds, snapshots, semantic models, metrics, exposures, and tests; extraction protocol based on `manifest.json`
      - [dbt-import.md](adapters/connectors/dbt/dbt-import.md) — Import procedure: manifest parsing, staging exclusion, table kind routing, test-to-BusinessRule conversion, incremental checksum detection, validation
- **examples/** — Illustrative worked examples
  - [domain-layout-example.md](examples/domain-layout-example.md) — Multi-domain space hierarchy with cross-domain Subject linking
  - [node-pages-example.md](examples/node-pages-example.md) — Sample pages for all node types (Sales domain scenario)
  - [graph-snapshot-example.json](examples/graph-snapshot-example.json) — Minimal JSON snapshot with nodes and edges
  - [agent-query-walkthrough.md](examples/agent-query-walkthrough.md) — End-to-end agent workflow: Glean search → mandatory filters → business rule → verified SQL
  - [s4hana-nodes-example.md](examples/s4hana-nodes-example.md) — Annotated node examples for SAP S/4HANA FI-AR domain *(draft)*
  - [openwiki-kg-example.md](examples/openwiki-kg-example.md) — OpenWiki-generated KG draft Table node (generic dbt reporting view); shows filled fields vs `<!-- TODO -->` placeholders

---

## Quick start

1. Read [SPEC.md](SPEC.md) for the complete normative specification.
2. Read [spec/conceptual-layer.md](spec/conceptual-layer.md) for the vocabulary layer and [spec/logical-layer.md](spec/logical-layer.md) for the domain layer schema.
3. Read [spec/page-templates.md](spec/page-templates.md) to understand what each page looks like.
4. Choose a backend and read the matching adapter doc (e.g. [adapters/engine/confluence/confluence-adapter.md](adapters/engine/confluence/confluence-adapter.md) for Confluence, [adapters/engine/markdown/markdown-adapter.md](adapters/engine/markdown/markdown-adapter.md) for Markdown files in git).
5. See [examples/](examples/) for concrete illustrations.

---

## License

[CC BY-NC 4.0](LICENSE) — free to use and adapt for non-commercial purposes with attribution.
For commercial licensing, contact the copyright holder.

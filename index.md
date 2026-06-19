# Knowledge Graph Specification — Directory Index

**Version 1.3.0** | [CC BY-NC 4.0](LICENSE) | [CHANGELOG](CHANGELOG.md)

---

## Entry points

* [README.md](README.md) - Project overview, quick start, repository structure
* [SPEC.md](SPEC.md) - Top-level normative specification (start here)
* [CHANGELOG.md](CHANGELOG.md) - Full version history with semver releases
* [LICENSE](LICENSE) - CC BY-NC 4.0

---

## spec/ — Normative specification

* [spec/data-model.md](spec/data-model.md) - Node types, edge kinds, indexes, audit rules, graph-DB migration
* [spec/link-format.md](spec/link-format.md) - Edge-statement syntax, back-reference rules, visualization conventions
* [spec/page-templates.md](spec/page-templates.md) - Canonical page templates for all 11 node types
* [spec/space-structure.md](spec/space-structure.md) - Logical hierarchy, container pages, naming conventions
* [spec/versioning.md](spec/versioning.md) - Version comment format, breaking change policy, rollback procedure
* [spec/schema.yaml](spec/schema.yaml) - Machine-readable schema: node types, edge kinds, properties, audit rules

---

## adapters/confluence/ — Confluence backend

* [adapters/confluence/confluence-adapter.md](adapters/confluence/confluence-adapter.md) - Storage format, MCP tools, link encoding, Glean/Cortex integration
* [adapters/confluence/agent-skill.md](adapters/confluence/agent-skill.md) - Agent workflows: Read, Write, Update, Navigate, Version history
* [adapters/confluence/bulk-update-scripts.md](adapters/confluence/bulk-update-scripts.md) - Python REST patterns for bulk page updates
* [adapters/confluence/snapshot-pipeline.md](adapters/confluence/snapshot-pipeline.md) - Confluence → JSON snapshot → graph visualization pipeline

---

## examples/ — Illustrative examples

* [examples/domain-layout-example.md](examples/domain-layout-example.md) - Multi-domain space hierarchy with cross-domain Subject linking
* [examples/node-pages-example.md](examples/node-pages-example.md) - Sample pages for all node types (Sales domain scenario)
* [examples/graph-snapshot-example.json](examples/graph-snapshot-example.json) - Minimal JSON snapshot with nodes and edges
* [examples/agent-query-walkthrough.md](examples/agent-query-walkthrough.md) - End-to-end agent workflow: Glean search → mandatory filters → business rule → verified SQL

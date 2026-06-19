# Changelog

Tracks changes to the **graph system**: architecture, data model, node types, edge kinds, folder structure, page templates, versioning conventions, and agent skills.

Does NOT track changes to **knowledge database pages** (nodes, edges, Relationship pages — use the backend's version history for that).

Versioning follows [Semantic Versioning](https://semver.org):
- `MAJOR` — breaking change: node type renamed/removed, edge kind removed, folder path changed, template field removed
- `MINOR` — additive change: new node type, new edge kind, new folder, new template, new skill workflow
- `PATCH` — clarification, wording fix, non-breaking convention update

---

## [1.3.1] — 2026-06-19

> **Structural note:** this release marks the conversion of the original flat design documents
> (`knowledge-graph-confluence-glean.md`, `knowledge-graph-confluence-index.md`,
> `knowledge-graph-confluence-glean-templates.md`, `skills/knowledge-graph-confluence/SKILL.md`,
> `scripts/knowledge-graph-confluence-scripts.md`, `KG diagram/knowledge-graph-confluence-diagram.md`,
> `knowledge-graph-confluence-releases.md`) into the structured spec format of this repository.
> Content is equivalent; all proprietary identifiers (page IDs, space keys, internal URLs,
> domain-specific names) have been replaced with generic placeholders.

### Added
- **`spec/link-format.md`**: "When to use a hyperlink edge vs. a Relationship page" section — promotion rule (when a hyperlink gains a Reason + Consequence, it becomes a Relationship page), and when individual Attribute/Measure pages are warranted vs. inline column definitions.
- **`spec/link-format.md`**: "Structural edges vs. semantic edges" section — core conceptual distinction between `joinedTo` (structural: SQL join fact, no business meaning) and Relationship pages (semantic: business dependency with Reason + Consequence). Includes comparison table, one-sentence rule, and note on the naming collision with database tooling (`relationships:` in SQL, dbt, Snowflake Semantic Views).
- **`adapters/confluence/confluence-adapter.md`**: "Joins vs. Relationship pages — naming disambiguation" section — references the spec section and adds Confluence-specific location mapping plus a Snowflake-specific callout.
- **`adapters/confluence/agent-skill.md`**: "Supported intents" table — maps 13 natural-language user requests to the agent workflow that handles each one.

---

## [1.3.0] — 2026-06-17

### Added
- **`spec/data-model.md`**: schema index document — node type → graph DB label mapping, edge type → relationship type mapping, audit rules, migration notes. Generic (graph-DB-agnostic; Neo4j cited as example).
- **Node index schema** (`kg-node-index.json`): live node registry. Includes fields: `name`, `label`, `page_title`, `page_id`, `page_url`, `domain`, `status`.
- **Edge index schema** (`kg-edge-index.json`): live edge registry. Includes typed `relationship_type` (RELATED_TO, REQUIRES, MANDATORY_FOR, JOINED_TO, IMPLEMENTS, GUARDS, OVERRIDES, DEMONSTRATES, APPLIES_TO, HAS_ATTRIBUTE, CONTAINS, DISAMBIGUATES).
- Script functions `graph_to_node_index`, `graph_to_edge_index`, `audit_graph` for generating and validating the JSON registries.
- Snapshot CLI flags `--write-node-index`, `--write-edge-index`, `--audit` for one-shot snapshot + index generation + audit.
- **Audit rules** (4 rules now enforced):
  1. No duplicate nodes (same label + name, different page IDs).
  2. No duplicate edges (same source, target, relationship type).
  3. A `RELATED_TO` hyperlink is redundant when a reified edge of the same type already covers the same pair.
  4. Mixed-kind hyperlinks on the same `(source, target)` pair — detects stale `relatedTo` back-refs left alongside correctly-typed edges.
- `NODE_PREFIXES`: added `Measure:` and `Attribute:` (were missing; `Metric:` removed).
- `INDEX_PAGE_TITLES`: added `measures`, `attributes`, `concepts` to the structural-page exclusion list.
- Node colors: `Measure` (`#6554C0`), `Attribute` (`#B3D4FF`) added; `Metric` removed.

### Fixed
- Stale back-reference on Rule page: spurious `relatedTo <-` removed; correct `apply <-` added.
- Stale `relatedTo` link on Measure page removed (relationship already reified as a `requires` Relationship page).

---

## [1.2.0] — 2026-06-17

### Added
- **`PK` column** in `## Fields / Physical columns` table template. Mark primary key column(s) with ✓; leave empty otherwise.
- New section **"Semantic Annotations and Cross-Domain Linking"** in the architecture doc: decision table for inline vs Attribute page, and cross-domain Subject linking pattern.
- **Back-reference constraints** (three rules) now authoritative for both MCP writes and scripts:
  1. No symmetric duplicates.
  2. Edge kind must match.
  3. `implement` not valid between two Subjects.
- `## Relationships` section added to `VerifiedQuery` template (`demonstrates` Relationship kind).
- Script `kg_fix_tables.py`: updates Table pages (PK column, Calculated column, Measure rows, link labels, dedup Related).
- Release changelog (this file).

### Changed
- `## Fields / Semantic annotations` table: column `Attribute page?` renamed to `Calculated`. Now accepts links to both Attribute pages and Measure pages.
- `## Related` on Table pages: Attributes and Measures already in `Calculated` must not be repeated.
- Attribute node type: threshold lowered — also create a page for columns with non-trivial synonyms or cross-domain relevance.
- Attribute and Measure described together as **promoted fields** sharing the same `Calculated` column concept.
- `implement` edge kind source list corrected: `Subject, Measure, BusinessRule, Filter → VerifiedQuery` (was `Subject` only).
- Back-reference injection rules in scripts doc now point to main design doc; scripts doc retains only script-specific symptom table.
- System prompt block removed from architecture doc; replaced with one-line pointer to agent skill.

### Fixed
- Stale `Metric:` prefix (should be `Measure:`).
- Missing `Attribute:` prefix in diagram snapshot parser.
- Missing `implement` sources in agent skill and templates.

---

## [1.1.0] — 2026-06-16

### Added
- `concepts/` folder: global parent container for domain-agnostic node types.

### Changed
- Folder structure: `subjects/` relocated from space root to `concepts/subjects/`.
- All path references in architecture doc and agent skill updated.

### Breaking
- Path `subjects/` is now `concepts/subjects/` — any agent or script using the old path must be updated.

---

## [1.0.0] — 2026-06-15

Initial release of the Knowledge Graph system.

### Added

**Architecture**
- Three-layer model: Conceptual (`concepts/`), Implementation (`<domain>/`), Relationship (`relationships/`)
- Graph stored as wiki parent-page hierarchy
- Semantic search (Glean) indexes the wiki continuously — no export pipeline needed
- LLM semantic layer translates natural-language intent into wiki operations

**Data model**
- 11 node types: Subject, Domain, Table, Measure, Attribute, Filter, VerifiedQuery, BusinessRule, Disambiguation, Relationship
- 7 hyperlink edge kinds: `implement`, `relatedTo`, `attribute`, `joinedTo`, `disambiguate`, `apply`, `contain`
- 5 reified Relationship kinds: `mandatory`, `requires`, `guards`, `overrides`, `demonstrates`
- Edge format: self-contained clickable label `[Source: X edgeKind -> Target: Y]`
- Back-references: both sides of every edge carry a link (owning `->`, back-reference `<-`)

**Page templates**
- Templates defined for all 11 node types
- Subject: prose-only node with Business Definition + Related edges
- Relationship: reified edge with Reason + Consequence if Ignored
- All other node types: structured header fields + predicate/definition block, no prose

**Agent skill**
- Five workflows: Read, Write, Update, Navigate, Version history
- Bulk update threshold: ≥6 pages → Python script via REST API; <6 pages → MCP tool calls directly

**Graph diagram tooling**
- Snapshot script: crawls wiki from a root page ID, parses node-type prefixes and edge labels, writes JSON
- Shared library: Confluence API client, node/edge parsing, snapshot format
- Visualization: interactive graph widget with node colors by type, hexagon shape for Relationship nodes
- Diagram reference doc: pipeline, JSON format spec, CLI commands, node/edge color reference

**Bulk update scripts**
- Download script: downloads all KG pages to local Markdown files with YAML frontmatter
- Link-label migration script: migrated all `## Related` links to `<ac:link-body>` clickable-label format
- Header cleanup script: removed redundant metadata header fields
- Rules/filters fix script: cleaned Rule headers, moved `appliesTo` to `## Related` edges, added Relationship back-references to Filter pages

# Changelog

Tracks changes to the **graph system**: architecture, data model, node types, edge kinds, folder structure, page templates, versioning conventions, and agent skills.

Does NOT track changes to **knowledge database pages** (nodes, edges, Relationship pages — use the backend's version history for that).

Versioning follows [Semantic Versioning](https://semver.org):
- `MAJOR` — breaking change: node type renamed/removed, edge kind removed, folder path changed, template field removed
- `MINOR` — additive change: new node type, new edge kind, new folder, new template, new skill workflow
- `PATCH` — clarification, wording fix, non-breaking convention update

## Versions

| Version | Date | Summary |
|---|---|---|
| [1.3.9](#1.3.9) | 2026-06-29 | SAP HANA connector (technical + import + lineage explorer addon); business layer made optional in connector contract |
| [1.3.8](#1.3.8) | 2026-06-26 | SAP S/4HANA import + lineage explorer split out; `adapters/adapters.md` taxonomy; addon concept; connector contract moved to `adapters/connectors/` |
| [1.3.7](#1.3.7) | 2026-06-25 | Source-system connector spec; SAP S/4HANA connector (business + technical); `adapters/` split into `engine/` and `connectors/`; `Process` reserved node type |
| [1.3.6](#1.3.6) | 2026-06-24 | `concepts/` → `vocabulary/`; cross-domain ownership model; two-layer architecture; Citations template |
| [1.3.5](#1.3.5) | 2026-06-22 | `## Related` → `## Links`; edge kind `attribute` → `calculate` |
| [1.3.4](#1.3.4) | 2026-06-22 | Graph DB adapter doc; adapter section split into content storage vs. graph DB |
| [1.3.3](#1.3.3) | 2026-06-22 | "Python scripts" → "Knowledge Graph API" terminology |
| [1.3.2](#1.3.2) | 2026-06-19 | Vocabulary layer design notes; agent query walkthrough example |
| [1.3.1](#1.3.1) | 2026-06-19 | Converted flat design docs to structured spec format |
| [1.3.0](#1.3.0) | 2026-06-17 | Schema index, node/edge JSON registries, audit rules |
| [1.2.0](#1.2.0) | 2026-06-17 | PK column in Table template; back-reference constraints; semantic annotations |
| [1.1.0](#1.1.0) | 2026-06-16 | `vocabulary/` folder; `subjects/` relocated to `vocabulary/subjects/` |
| [1.0.0](#1.0.0) | 2026-06-15 | Initial release |

---

## [1.3.9] — 2026-06-29

### Added
- **`adapters/connectors/sap-hana/sap-hana-technical.md`** *(draft)* — SAP HANA connector technical layer. Covers the Domain layer only (no Vocabulary layer — HANA is a database platform with no built-in business process concepts). Includes node type mapping for Calculation Views, physical tables, synonyms, measures, attributes, and filters; field mapping tables; edge mapping; data warehouse layer conventions (Landing → Staging → DWH/Core → Data Mart/Semantic → Reporting); extraction protocol.
- **`adapters/connectors/sap-hana/sap-hana-import.md`** *(draft)* — Import procedure for the HANA connector: curated seed list, upstream dependency walk, object-type-to-KG-node routing, domain inference from package hierarchy and schema names, validation, incremental detection. Routing rules are generic — prefixes and schema layer suffixes are documented as implementation-specific and must be configured per deployment.
- **`adapters/connectors/sap-hana/addons/sap-hana-lineage-explorer.md`** *(draft, addon)* — Read-only, ephemeral map of the HANA object dependency graph. Supports upstream (toward source data) and downstream (toward consumers) walks. Optional supplement for systems with XS Classic or XSA runtime. Reuses the KG JSON snapshot format and visualization notebook with a HANA-specific node color scheme. Promotion path to the import seed list documented.

### Changed
- **`adapters/connectors/connectors.md`** — Business layer (Vocabulary layer) is now explicitly **optional** for database connectors. A connector for a source system with no built-in business process concepts, no standard annotation framework, and no lifecycle contracts may omit the business layer section, provided it documents the omission and states that Vocabulary nodes require manual authoring. The SAP HANA connector is the first connector to use this provision.

---

## [1.3.8] — 2026-06-26

### Added
- **`adapters/connectors/sap-s4hana/sap-s4hana-import.md`** *(draft)* — import procedure for the S/4HANA technical layer: seed selection, CDS dependency discovery, DDL source extraction from `DD*` tables, annotation-to-node-type routing, validation, incremental detection, and the list of fields that always require manual authoring.
- **`adapters/connectors/sap-s4hana/addons/sap-s4hana-lineage-explorer.md`** *(draft, addon)* — read-only, ephemeral map of the full S/4HANA CDS and DDIC object universe. Extensible object model (new object types are additive). Shares the JSON snapshot format and visualization notebook with the KG snapshot pipeline. Demo video linked.
- **`adapters/adapters.md`** — adapter taxonomy: engine adapters (content storage, graph DB), connectors, and addons. Defines the addon concept: optional, self-contained capability document that lives next to its adapter and does not affect KG node/edge semantics.
- **`adapters/connectors/connectors.md`** — connector contract moved here from `spec/connectors.md`; content unchanged, internal links updated.

### Changed
- **`adapters/connectors/sap-s4hana/sap-s4hana-technical.md`** — restructured as an entry-point wrapper with a "Documents in this layer" table linking all three technical files; import procedure section removed (content moved to `sap-s4hana-import.md`).
- **`spec/connectors.md` removed** — content moved to `adapters/connectors/connectors.md`.
- **`SPEC.md` §10** — condensed to a pointer to `adapters/adapters.md`.

---

## [1.3.7] — 2026-06-25

### Added
- **`spec/connectors.md` — normative contract for source-system connectors. Two-layer model (business → Vocabulary, technical → Domain), six-section structure, two extraction channels (system metadata + project documentation).
- **`adapters/connectors/sap-s4hana/sap-s4hana-business.md`** *(draft)* — Vocabulary-layer document for SAP S/4HANA: Subject nodes for SAP concepts and project decisions (BRD-sourced) across FI, FI-CA, FI-AR, MM, SD, CO.
- **`adapters/connectors/sap-s4hana/sap-s4hana-technical.md`** *(draft)* — Domain-layer document: node type mapping to CDS views/DDIC tables, field mapping, extraction protocol, import procedure.
- **`examples/s4hana-nodes-example.md`** *(draft)* — worked FI-AR example with all node types and real SAP field names.
- **`Process` reserved node type** in `spec/space-structure.md` — named business activity sequence; reserved for future use.

### Changed
- **`adapters/` restructured** into `adapters/engine/` (graph engine) and `adapters/connectors/` (source systems).
- **`spec/connectors.md`** renamed from `spec/interface.md`; "interface" → "connector" throughout.

---

## [1.3.6] — 2026-06-24

### Changed
- **`concepts/` renamed to `vocabulary/`** across all spec documents, templates, and adapter docs. The folder that holds global, domain-agnostic node types is now `vocabulary/` (was `concepts/`). Reflects the architectural intent: this layer holds the language the business uses — ontologies, business terms, methodologies — not just abstract concepts.
  - Path change: `concepts/subjects/` → `vocabulary/subjects/`
  - Affected files: `SPEC.md`, `spec/space-structure.md`, `spec/data-model.md`, `spec/schema.yaml`, `adapters/confluence/confluence-adapter.md`, `adapters/confluence/snapshot-pipeline.md`, `examples/domain-layout-example.md`
  - Three-layer model renamed: Conceptual → Vocabulary (`vocabulary/`), Implementation (`<domain>/`), Relationship (`relationships/`)
- **Cross-domain ownership model** clarified. `contain ->` implies exactly one owning domain per node. Cross-domain usage is expressed through existing edges between nodes — non-owning domains do not add `contain` edges. Decision tree for ownership assignment added to `spec/space-structure.md`.
- **`spec/data-model.md`**: two-layer architecture section added (`§1`); `contain` edge definition updated to state ownership semantics.
- **`spec/page-templates.md`**: `## Citations` optional section added to Subject template for linking authoritative external sources.
- **`spec/data-model.md` §7**: cross-domain linking section broadened from "via Subject" to "via Vocabulary" — any Vocabulary node can serve as a cross-domain anchor.

> **Note:** Treated as patch-level changes bundled into a single release. Version bumped to 1.3.6 — prototype stage; major version reserved for larger structural changes.

---

## [1.3.5] — 2026-06-22

### Changed (breaking)
- **`## Related` → `## Links`** across all pages and spec documents. The section name "Related" was ambiguous — it could be confused with the `relatedTo` edge kind. `## Links` makes explicit that this section holds typed edge statements. Affects: `spec/link-format.md`, `spec/page-templates.md`, `spec/data-model.md`, `SPEC.md`, `spec/versioning.md`, `adapters/confluence/confluence-adapter.md`, `adapters/confluence/agent-skill.md`, `adapters/confluence/snapshot-pipeline.md`, `adapters/graph-db/README.md`, `examples/node-pages-example.md`, `examples/domain-layout-example.md`. **Live pages must be migrated.**
- **Edge kind `attribute` → `calculate` / `HAS_ATTRIBUTE` → `CALCULATES`**: renamed to reflect the semantic role (the table is the source that calculates/exposes the value) rather than structural ownership. Extended to cover both `Table → Attribute` and `Table → Measure` — replaces the separate `## Tables Used` section on Measure pages. Affects: `spec/schema.yaml`, `spec/data-model.md`, `spec/link-format.md`, `spec/page-templates.md`, `adapters/graph-db/README.md`. **Live pages must be migrated.**
- **Measure template**: `## Tables Used` section removed. Table sources now listed in `## Links` as `Table: X calculate <- Measure: Y` back-references, consistent with how Attribute pages link to their tables.
- **Attribute template**: `**Table:**` header field removed (it was already absent from the spec; noted here as migration target for live pages). Table link belongs in `## Links` as `Table: X calculate <- Attribute: Y`.

### Added
- **Header field vs `## Links` — one-to-many rule** in `spec/page-templates.md`: when a node has exactly one link of a given type, it may appear as a header field (e.g. `**Disambiguation:**` on a Filter page). When multiple links of the same type are possible, they belong in `## Links`. If a header field grows to multiple targets, move all instances to `## Links`.

---

## [1.3.4] — 2026-06-22

### Added
- **`adapters/graph-db/README.md`**: New Graph DB adapter — structural projection concept, role in architecture (diagram), what/is-not comparison table, three purposes (duplicate prevention, fast traversal, lineage/traceability), hyperlink and reified edge kind → relationship type mapping tables, edge conflict rule, JSON index field descriptions (node + edge), import steps with Cypher examples for both hyperlink and reified edges, sync strategy candidates, candidate databases. Status: planned.

### Changed
- **`SPEC.md` §10**: Adapter section restructured into two categories — **Content storage** (source of truth, full knowledge, authoring surface) and **Graph DB** (structural projection, three purposes spelled out). Each category has a brief description before the adapter table.
- **`spec/data-model.md` §6**: Graph Database Migration Notes condensed to a two-bullet summary (duplicate tracking + import input) with a link to `adapters/graph-db/README.md`.
- **`index.md`**, **`README.md`**: Updated to reflect the new `adapters/graph-db/` folder and two-category adapter structure.

---

## [1.3.3] — 2026-06-22

### Changed
- **Terminology:** "Python scripts" → **"Knowledge Graph API"** throughout all documents. The programmatic interface for graph operations is now named explicitly to distinguish it from the Confluence REST API (Atlassian's vendor HTTP API, used internally as the current transport layer) and from Confluence MCP (the agent-facing tool interface). Affected files: `adapters/confluence/graph-api.md` (renamed from `bulk-update-scripts.md`), `adapters/confluence/agent-skill.md`, `adapters/confluence/confluence-adapter.md`, `SPEC.md`, `README.md`, `index.md`, `spec/data-model.md`.
- **File rename:** `adapters/confluence/bulk-update-scripts.md` → `adapters/confluence/graph-api.md`. Title changed to "Knowledge Graph API". Includes an explicit naming-disambiguation note and the "backend independence" rationale for the API design.

---

## [1.3.2] — 2026-06-19

### Added
- **`spec/data-model.md`**: §6 expanded with "Future direction" — explains the JSON indexes as intermediate representation for graph DB import, duplicate tracking purpose, and the planned migration path from wiki-as-source-of-truth to graph DB backend with wiki as authoring layer.
- **`adapters/confluence/bulk-update-scripts.md`**: "Why scripts instead of MCP" section — two reasons: token cost of bulk MCP calls, and scripts as a foundation for a future graph DB write API.
- **`spec/space-structure.md`**: "The `vocabulary/` layer — design notes" section — distinguishes Subject (active), Term (reserved), and Concept (reserved) node types; explains the intention to link external references (glossaries, ontologies) to the vocabulary layer rather than duplicating definitions.
- **`examples/agent-query-walkthrough.md`**: New example — end-to-end agent query walkthrough showing Glean search → mandatory filter discovery → business rule application → verified SQL retrieval → SQL adaptation. Covers both Cursor and Snowflake Cortex agent variants. Derived from real testing sessions; all domain-specific values replaced with generic placeholders.

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
- `INDEX_PAGE_TITLES`: added `measures`, `attributes`, `vocabulary` to the structural-page exclusion list.
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
- `vocabulary/` folder: global parent container for domain-agnostic node types.

### Changed
- Folder structure: `subjects/` relocated from space root to `vocabulary/subjects/`.
- All path references in architecture doc and agent skill updated.

### Breaking
- Path `subjects/` is now `vocabulary/subjects/` — any agent or script using the old path must be updated.

---

## [1.0.0] — 2026-06-15

Initial release of the Knowledge Graph system.

### Added

**Architecture**
- Three-layer model: Vocabulary (`vocabulary/`), Implementation (`<domain>/`), Relationship (`relationships/`)
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

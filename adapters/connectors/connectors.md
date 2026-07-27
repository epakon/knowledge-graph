# Source-System Connector

A source-system connector answers one question: **what does a large business system know, and how does that knowledge map to the Knowledge Graph?**

Large systems like SAP S/4HANA, Salesforce, or Workday encode enormous amounts of business knowledge — process definitions, data models, calculation rules, mandatory conditions, organizational hierarchies. Most of that knowledge is implicit: locked inside table structures, ABAP logic, CDS annotations, or configuration. The connector makes it explicit by translating the system's business concepts, data semantics, and rules into the typed nodes and edges of the Knowledge Graph.

The connector is a **specification document**, not a technical connector. It describes:

- Which business concepts in the source system correspond to `Subject`, `Domain`, `Table`, `Measure`, `Attribute`, `Filter`, `BusinessRule`, and `VerifiedQuery` nodes.
- Which relationships between those concepts correspond to KG edge kinds — including reified edges (`mandatory`, `requires`) that carry business `reason` and `consequence`.
- Where in the source system the business meaning lives: in user-facing labels, process documentation, configuration, or calculation logic.
- What cannot be derived automatically and must be authored by a domain expert.

---

## What the connector is not

The connector is not a data integration pipeline. It does not define:

- How to technically connect to the source system (API calls, credentials, scheduling) — that is covered by the extraction section of each connector document.
- How to store or version pages in the content storage — covered by the Confluence adapter.
- How agents query the resulting graph — covered by the agent-skill document.

The connector exists at the semantic layer: between the business meaning encoded in the source system and the verified knowledge encoded in the graph.

---

## Two layers of knowledge

A source system exposes two distinct kinds of knowledge, and they map to the two layers of the Knowledge Graph.

**Business process knowledge → Vocabulary layer**

Business processes define what things *mean*: what a company code is, what revenue recognition requires, what distinguishes an open item from a cleared one. This knowledge is stable — it does not change when a table is renamed or a CDS view is refactored. It belongs in the Vocabulary layer as `Subject` and `Disambiguation` nodes: global, domain-agnostic, shared across all technical implementations.

**Technical element knowledge → Domain layer**

Technical elements define how things are *implemented*: which CDS view holds the data, which field encodes the amount, which parameter is mandatory, which SQL expression computes the measure. This knowledge is coupled to the data model and changes when the system is upgraded or extended. It belongs in the Domain layer as `Table`, `Measure`, `Attribute`, `Filter`, `BusinessRule`, and `VerifiedQuery` nodes: scoped to a specific domain, tied to concrete SQL and field names.

**The bridge**

`implement ->` edges connect the two layers. A `Subject: Company Code` in the Vocabulary is implemented by a `Filter: Company Code` in the FI domain and a `Table: Company Code Master` in the MM domain. The business concept survives; its implementations may change.

This separation is not a strict rule — not every source object maps cleanly into one layer. But it is the natural direction: when translating a source system, ask first *is this a business concept or a technical artifact?* The answer determines the layer.

---

## Business process as the primary lens

The most important knowledge to capture is not the data model — it is the **business process** that governs how data is created, filtered, aggregated, and interpreted.

For any source system, the connector must answer:

- **What are the core business concepts?** These become `Subject` nodes in the Vocabulary — stable definitions that survive data model changes.
- **What are the rules that govern query construction?** Mandatory filters, calculation logic, sign conventions — these become `Filter`, `BusinessRule`, and `Measure` nodes in the Domain layer.
- **What is the intended meaning of ambiguous terms?** A field named `Amount` may mean gross, net, tax-inclusive, or statistical depending on context — these become `Disambiguation` nodes.
- **What queries has the business validated?** Approved SQL or report templates become `VerifiedQuery` nodes, anchoring the graph to working examples.
- **What breaks if a rule is ignored?** The `reason` and `consequence` fields on reified edges capture the business impact, not just the structural dependency.

Technical objects — CDS views, ABAP tables, OData services, Fiori applications — are concrete anchors that make the business knowledge specific, verifiable, and testable. They are referenced to locate the knowledge, not as the subject of the graph.

---

## Connector contract

Every connector document MUST specify all six sections below. A section may be marked `N/A` only if the source system structurally cannot provide that information; it may never be silently omitted.

**Business layer (Section 1 — Vocabulary layer) is optional for database connectors.** A source system that is a pure database platform — with no built-in business process concepts, no standard annotation framework, and no lifecycle contracts — does not produce vocabulary nodes automatically. In this case the connector MUST document the omission explicitly: state that the Vocabulary layer requires manual authoring by a domain expert and is outside the connector's extraction scope. If a database functional area is later found to carry meaningful business process knowledge (e.g. a heavily annotated semantic layer that encodes stable business concepts), the business layer section can be added incrementally. The connector does not need to be restructured — the omission is an editorial decision, not a structural one.

### Section 1 — Source system overview

Describe:
- What the source system is and what business processes it owns.
- How business knowledge is encoded: in user-facing labels, configuration, calculation logic, process documentation, or data model annotations.
- The technical surfaces that expose that knowledge (APIs, metadata layers, annotation frameworks).
- Versioning and lifecycle conventions — how the system signals that a concept is stable, deprecated, or internal.

### Section 2 — Node type mapping

For each KG node type the source system can populate, provide a mapping table:

| KG node type | Source object(s) | Key mapping notes |
|---|---|---|
| `Subject` | ... | ... |
| `Domain` | ... | ... |
| `Table` | ... | ... |
| `Measure` | ... | ... |
| `Attribute` | ... | ... |
| `Filter` | ... | ... |
| `VerifiedQuery` | ... | ... |
| `BusinessRule` | ... | ... |
| `Disambiguation` | ... | ... |
| `Reification` | ... | ... |

Node types with no meaningful mapping in the source system must be listed as `not applicable` with a one-line rationale.

Required properties per node type are defined in [`spec/schema.yaml`](../../spec/schema.yaml). The adapter must show how each required property is populated or — if the source cannot provide it — document the fallback (manual authoring, a default value, or derivation from related fields).

### Section 3 — Field mapping tables

For each mapped node type, provide a field-level mapping table:

| KG field | Source field / path | Transformation | Notes |
|---|---|---|---|
| `name` | `<source field>` | `<e.g. trim, upper, rename>` | |
| `domain` | `<source field>` | | |
| `status` | `<source field>` | `Active if X, Deprecated if Y` | |
| `page_id` | *(assigned by content storage on create)* | n/a | |
| ... | ... | ... | |

Ambiguous or context-dependent mappings must be flagged with a **Decision needed** note and documented as a `Disambiguation` node in the target KG domain.

### Section 4 — Edge mapping

For each KG edge kind the source system can provide, specify:

| KG edge kind | Source relationship | How to detect | Cardinality | Notes |
|---|---|---|---|---|
| `relatedTo` | ... | ... | N:M | |
| `joinedTo` | ... | ... | N:M | |
| `calculate` | ... | ... | 1:N | |
| `implement` | ... | ... | N:M | |
| `mandatory` (reified) | ... | ... | N:M | `reason` + `consequence` required |
| `requires` (reified) | ... | ... | N:M | |
| ... | ... | ... | ... | |

Edge kinds that cannot be derived from the source system must be listed as `not derivable — requires manual authoring`.

Reified edges must additionally specify how `reason` and `consequence` are derived or, if they cannot be derived, that they require post-import manual authoring by a domain expert.

### Section 5 — Extraction protocol

A source system exposes knowledge through two distinct channels that must both be covered:

**System metadata** — what can be read programmatically from the live system: API endpoints, annotation frameworks, metadata services, configuration tables. This channel produces Domain-layer nodes (Tables, Measures, Filters, BusinessRules) efficiently and at scale.

**Project documentation** — what exists only in BRDs, functional specifications, configuration decision logs, and domain expert memory. This channel produces Vocabulary-layer nodes (Subjects, Disambiguations) and the `implement ->` edges that connect them to Domain nodes. It cannot be automated — it requires a domain expert who understands why a specific field was chosen over an available alternative, or why a concept was defined the way it was for this company's context.

Both channels must be described. Knowledge that exists only in project documentation must be listed explicitly as requiring manual authoring — not left as an implicit gap.

Describe:
- **System access method**: API endpoint, metadata service, annotation framework, configuration export.
- **What to read**: which objects carry business meaning vs. which are purely structural.
- **Project documentation sources**: which artifacts (BRD, functional spec, configuration log) are the source of truth for Vocabulary-layer nodes.
- **Incremental detection**: how to identify what has changed since the last extraction.
- **What cannot be extracted**: knowledge held only by domain experts — list explicitly so manual authoring is planned for, not forgotten.

### Section 6 — Import procedure

For non-trivial connectors the import procedure is typically split into a dedicated `*-import.md` file (e.g. `sap-hana-import.md`, `sap-s4hana-import.md`). In that case, the technical document's Section 6 contains only a short pointer to that file. The import file itself describes the full step-by-step flow; the six steps below define what it must cover.

1. **Validation** — required fields present, enum values within allowed set, name uniqueness checked against node index.
2. **Node creation** — template population, page title construction (`<NodeType>: <name>`), placement in the correct container path ([`spec/space-structure.md`](../../spec/space-structure.md)).
3. **Edge creation** — link statement construction following [`spec/link-format.md`](../../spec/link-format.md).
4. **Duplicate handling** — what happens when a node with the same `(label, name)` already exists: skip, merge, or flag for review.
5. **Post-import audit** — run the audit rules from [`spec/schema.yaml`](../../spec/schema.yaml) before committing.
6. **Version comment** — every programmatic create or update MUST include a structured version comment following [`spec/versioning.md`](../../spec/versioning.md).

# Conceptual Layer

> Part of the [Knowledge Graph Specification](../SPEC.md).
> For the logical layer (Table, Measure, Attribute, Filter, BusinessRule, VerifiedQuery node types) see [logical-layer.md](logical-layer.md).
> **[`spec/schema.yaml`](schema.yaml) is the authoritative source for every node type's properties and every edge kind's valid sources/targets/properties.** The tables below restate only what's needed for the surrounding prose — if this document and `schema.yaml` ever disagree, `schema.yaml` wins.

---

## 1. Purpose

The conceptual layer holds knowledge that exists independently of any data implementation. It is stable: it does not break when tables are renamed, SQL expressions change, or domains are restructured.

**The stability test.** A node belongs in the conceptual layer if it survives the question: *would this node still be meaningful if all the databases disappeared?* "DSO means Days Sales Outstanding — the ratio of open receivables to annualised invoicing" is meaningful without a database. "Measure: DSO = SUM(open_ar) / annualised_ci * 365" is not.

| Property | Value |
|---|---|
| **Path** | `vocabulary/` |
| **Scope** | Global — shared across all domains |
| **Authored by** | Business and domain experts |
| **Changes when** | Business language or processes evolve |

**What does NOT belong here.** A node that requires a table name, SQL expression, filter predicate, or domain-specific configuration to be meaningful belongs in the logical layer. Governance metadata (ownership, stewardship, classification) belongs as properties on existing nodes — not as new conceptual node types.

**Source-system codes.** A code the business itself speaks in — a document type a finance user names aloud, such as a write-off code — may appear in a Subject definition as a word of business vocabulary. Table names, column names, predicates, and the mapping of a code to a column do not; they live on the Filter, Attribute or BusinessRule that implements the Subject. Test: would a business user say it without looking at a database?

**Links stay inside the layer.** Conceptual pages link only to other conceptual pages. Every edge between a conceptual node and a logical or consumption node is owned by the logical or consumption side, and the conceptual page carries no back-reference to it. Volatile things point to stable things, never the reverse: adding, renaming or removing a table, measure or agent never edits a conceptual page (audit rule `no_conceptual_down_links` in `schema.yaml`).

---

## 2. Node Type Schema

Each node type maps to a **node label** in a target graph database. The identity key is the unique constraint enforced at migration time.

### 2.1 Subject

A `Subject` is a business concept that has at least one confirmed data implementation in a domain. It holds the authoritative business definition. The domain nodes that embody it point up to it via `implement ->` edges; the Subject page does not list them.

Prose lives here and nowhere else in the graph (except `Disambiguation` and the `Policy` statement). Every other node type uses structured fields only.

### 2.2 Concept

A `Concept` is an abstract thematic grouping of related Subjects. It exists at a level above any individual Subject: it captures the broader idea that unifies them. A Concept does not have a SQL implementation and does not link to domain nodes directly.

Examples: "Liquidity" groups DSO, Cash Collected, Open Balance. "Revenue Recognition" groups Net Revenue, Gross Revenue, Write-Off.

A Concept earns its place only when the grouping carries a company-specific meaning — a definition that is not universally obvious and that an agent or author needs to read to understand why these Subjects belong together. A grouping that is self-evident from the Subject names does not warrant a Concept node.

### 2.3 Process

A `Process` is a named business activity that produces, consumes, or governs data concepts. It provides the business context that explains *why* certain Subjects, rules, and measures exist — the causal chain from a business activity to its data representation.

A Process is not a workflow sequence (that is a TOGAF concern). In DAMA terms it is a bounded business activity whose data decisions are company-specific: how Period Close is configured in this SAP implementation, which document types belong to Order-to-Cash in this company, which exclusions apply during Collections. That company-specific knowledge is what the Process node carries.

A Process node earns its place only when its description contains company-specific decisions that are not derivable from the Subjects it links to. A Process whose description is universally understood (e.g. "Period Close is the monthly accounting close") with no project-specific content does not warrant a node.

### 2.4 Policy

A `Policy` is a business rule stated in business language: "Net revenue includes only posted documents on revenue accounts, excludes intercompany transactions, and is reported in group currency." It carries the rule's `statement`, its `rule_modality` and its `consequence_if_violated`.

> **Naming note.** "Policy" here means a business rule — what the business requires or forbids about its data. It is not a data-governance principle and not an access or masking policy.

A Policy states the rule once. Logical `Filter`, `BusinessRule` and `Measure` nodes translate it into SQL for specific tables and point up to it with `implement ->`. One Policy usually has several implementations — one per table or domain that carries the concepts it constrains. A Policy links to those concepts with `Policy relatedTo -> Subject`.

A Policy passes the stability test: "net revenue excludes intercompany transactions" is meaningful with no database. The mapping of "intercompany" to a column is not, and lives on the implementing logical node.

A logical `BusinessRule` that implements no Policy is a table-local structural fact (a sign convention, a deduplication key) and is treated as `necessity`. Any rule the business would state as an obligation or prohibition belongs in a Policy.

### Full schema

> See `schema.yaml`'s `node_types` section for the complete list of properties per node type. Identity key is always `name`; all four node types in this layer are global (`vocabulary/subjects|concepts|processes|policies/`).

### Uniqueness constraints

Each node type requires a uniqueness constraint on `name`. Example in Cypher (Neo4j):

```cypher
CREATE CONSTRAINT FOR (n:Subject) REQUIRE n.name IS UNIQUE;
CREATE CONSTRAINT FOR (n:Concept) REQUIRE n.name IS UNIQUE;
CREATE CONSTRAINT FOR (n:Process) REQUIRE n.name IS UNIQUE;
CREATE CONSTRAINT FOR (n:Policy) REQUIRE n.name IS UNIQUE;
```

---

## 3. Edge Type Schema

Two groups of edges: hyperlink edges within the conceptual layer, and the bridge edges from the logical layer. Both groups are hyperlink edges (no properties).

### 3.1 Hyperlink edge kinds — within the conceptual layer

> See `schema.yaml`'s `hyperlink_edge_kinds` section for the complete definitions of `comprises`, `produces`, `consumes`, and `governs` (all four are `Concept`/`Process` → `Subject`, single-direction, no properties). The distinctions between them are prose, not schema, so that guidance lives below. A `Policy` links to the Subjects it constrains with the generic `relatedTo`.

**`comprises` is a thematic grouping, not a strict classification.** It groups Subjects under a shared theme — it does not mean "is a kind of." A rule, property, or constraint that applies to a Concept does not automatically apply to the Subjects it comprises: `Concept: Liquidity` comprising `Subject: DSO` says only "these belong to the same theme," not "DSO is a kind of Liquidity" or "whatever holds for Liquidity holds for DSO." Don't chain `comprises` edges to infer indirect grouping either — each edge is a direct, independently-authored assertion.

Back-references follow the standard convention: the target page carries the `<-` form of the label in its `## Links` section.

**Choosing between `produces`, `consumes`, and `governs`:**
- `produces` — the activity creates or generates the data (Period Close produces DSO for the period)
- `consumes` — the activity needs the data to operate (Dunning consumes Net Due Date to determine which partners to contact)
- `governs` — the activity defines the rules that shape the data (Accounting Close governs Posting Period — the process determines which periods are open or closed)

When in doubt between `produces` and `governs`: if the process *creates* the value, use `produces`; if it *constrains* the rules around the value, use `governs`.

### 3.2 Bridge edges — from the logical layer

Bridge edges are owned by the logical side and point up:
- `implement` (`IMPLEMENTS`) — `Filter`, `BusinessRule` or `Measure` `implement ->` `Subject` or `Policy`. Full definition, including the `Measure`/`BusinessRule`/`Filter` → `VerifiedQuery` use, in [`spec/schema.yaml`](schema.yaml) and [logical-layer.md §2.1](logical-layer.md#21-hyperlink-edge-kinds-no-properties).
- `disambiguate` — `Disambiguation disambiguate -> Subject`.
- `relatedTo` — e.g. `Attribute relatedTo -> Subject`, owned by the Attribute.

No conceptual page lists its implementations. Navigation downward — from a Policy to the filters that implement it — goes through the edge index or a search for the `implement <-` form of the label, not through links on the conceptual page. This keeps the conceptual layer decoupled from the logical layer: adding a table or renaming a Measure never requires editing a Subject, Policy, Concept or Process page.

---

## 4. Space structure

```
vocabulary/
├── concepts/
│   └── Concept: <Name>
├── subjects/
│   └── Subject: <Name>
├── processes/
│   └── Process: <Name>
└── policies/
    └── Policy: <Name>
```

All four containers are global — shared across all domains. A Subject owned by one domain is still authored in `vocabulary/subjects/`, not inside the domain folder.

---

## 5. Authorship guidance

**Who writes these nodes:**
- `Subject` — data engineers or domain experts who have confirmed a data implementation exists. Do not create a Subject if no domain node can `implement ->` it yet; use prose in a `Concept` instead.
- `Concept` — domain experts or business analysts. Written when multiple Subjects share a theme that needs explaining.
- `Process` — domain experts or business analysts with knowledge of the company's SAP or system configuration. Only written when company-specific decisions are documented.
- `Policy` — business owners of the rule (e.g. finance controllers for revenue rules). Data engineers write the implementing Filters and BusinessRules; the business owns the statement.

**When to create a Subject vs a Concept:**
A Subject requires at least one domain node that `implement ->` it. If the business term exists but no data implementation is confirmed yet, write the definition on a related `Concept` page and promote it to a `Subject` once the implementation is identified.

---

## 6. Linking external knowledge

The conceptual layer is the natural attachment point for **external references** — links to glossaries, regulatory definitions, data dictionaries, or ontologies that inform a business concept. Rather than duplicating external definitions inside Subject pages, a Subject can carry a `## Citations` section with links to authoritative external sources. This avoids maintaining duplicate copies of definitions that are owned externally.

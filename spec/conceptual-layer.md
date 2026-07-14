# Conceptual Layer

> Part of the [Knowledge Graph Specification](../SPEC.md).
> For the logical layer (Table, Measure, Attribute, Filter, BusinessRule, VerifiedQuery node types) see [logical-layer.md](logical-layer.md).

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

---

## 2. Node Type Schema

Each node type maps to a **node label** in a target graph database. The identity key is the unique constraint enforced at migration time.

### 2.1 Subject

A `Subject` is a business concept that has at least one confirmed data implementation in a domain. It is the bridge node between the conceptual and logical layers: it holds the authoritative business definition, and points down to the domain nodes that embody it via `implement ->` edges.

Prose lives here and nowhere else in the graph (except `Disambiguation`). Every other node type uses structured fields only.

### 2.2 Concept

A `Concept` is an abstract thematic grouping of related Subjects. It exists at a level above any individual Subject: it captures the broader idea that unifies them. A Concept does not have a SQL implementation and does not link to domain nodes directly.

Examples: "Liquidity" groups DSO, Cash Collected, Open Balance. "Revenue Recognition" groups Net Revenue, Gross Revenue, Write-Off.

A Concept earns its place only when the grouping carries a company-specific meaning — a definition that is not universally obvious and that an agent or author needs to read to understand why these Subjects belong together. A grouping that is self-evident from the Subject names does not warrant a Concept node.

### 2.3 Process

A `Process` is a named business activity that produces, consumes, or governs data concepts. It provides the business context that explains *why* certain Subjects, rules, and measures exist — the causal chain from a business activity to its data representation.

A Process is not a workflow sequence (that is a TOGAF concern). In DAMA terms it is a bounded business activity whose data decisions are company-specific: how Period Close is configured in this SAP implementation, which document types belong to Order-to-Cash in this company, which exclusions apply during Collections. That company-specific knowledge is what the Process node carries.

A Process node earns its place only when its description contains company-specific decisions that are not derivable from the Subjects it links to. A Process whose description is universally understood (e.g. "Period Close is the monthly accounting close") with no project-specific content does not warrant a node.

### Full schema

| Node type | Node label | Identity key | Core properties | Scope |
|---|---|---|---|---|
| `Subject` | `Subject` | `name` | `business_definition`, `scope` | global (`vocabulary/subjects/`) |
| `Concept` | `Concept` | `name` | `definition` | global (`vocabulary/concepts/`) |
| `Process` | `Process` | `name` | `description` | global (`vocabulary/processes/`) |

### Uniqueness constraints

Each node type requires a uniqueness constraint on `name`. Example in Cypher (Neo4j):

```cypher
CREATE CONSTRAINT FOR (n:Subject) REQUIRE n.name IS UNIQUE;
CREATE CONSTRAINT FOR (n:Concept) REQUIRE n.name IS UNIQUE;
CREATE CONSTRAINT FOR (n:Process) REQUIRE n.name IS UNIQUE;
```

---

## 3. Edge Type Schema

Two groups of edges: hyperlink edges within the conceptual layer, and the single bridge edge into the logical layer. Both groups are hyperlink edges (no properties).

### 3.1 Hyperlink edge kinds — within the conceptual layer

| Kind | Reification type | Valid source labels | Valid target labels | Notes |
|---|---|---|---|---|
| `comprises` | `COMPRISES` | Concept | Subject | The Concept groups this Subject. Owning side: Concept page. |
| `produces` | `PRODUCES` | Process | Subject | The Process generates this Subject's data as an output. |
| `consumes` | `CONSUMES` | Process | Subject | The Process requires this Subject's data as an input. |
| `governs` | `GOVERNS` | Process | Subject | The Process defines the rules that constrain this Subject. |

Back-references follow the standard convention: the target page carries the `<-` form of the label in its `## Links` section.

**Choosing between `produces`, `consumes`, and `governs`:**
- `produces` — the activity creates or generates the data (Period Close produces DSO for the period)
- `consumes` — the activity needs the data to operate (Dunning consumes Net Due Date to determine which partners to contact)
- `governs` — the activity defines the rules that shape the data (Accounting Close governs Posting Period — the process determines which periods are open or closed)

When in doubt between `produces` and `governs`: if the process *creates* the value, use `produces`; if it *constrains* the rules around the value, use `governs`.

### 3.2 Bridge edge kind — into the logical layer

The only edge kind that crosses from the conceptual layer into the logical layer is `implement`:

| Kind | Reification type | Valid source labels | Valid target labels | Notes |
|---|---|---|---|---|
| `implement` | `IMPLEMENTS` | Subject | Measure, Filter, BusinessRule, VerifiedQuery | Defined in [logical-layer.md](logical-layer.md). Subject is the owning side. |

`Concept` and `Process` do **not** link directly to domain nodes. They reach domain implementations only via `Subject`. This keeps the conceptual layer decoupled from the logical layer: renaming a Measure does not require updating any Concept or Process page.

---

## 4. Space structure

```
vocabulary/
├── concepts/
│   └── Concept: <Name>
├── subjects/
│   └── Subject: <Name>
└── processes/
    └── Process: <Name>
```

All three containers are global — shared across all domains. A Subject owned by one domain is still authored in `vocabulary/subjects/`, not inside the domain folder.

---

## 5. Authorship guidance

**Who writes these nodes:**
- `Subject` — data engineers or domain experts who have confirmed a data implementation exists. Do not create a Subject if no `implement ->` edge can be written yet; use prose in a `Concept` instead.
- `Concept` — domain experts or business analysts. Written when multiple Subjects share a theme that needs explaining.
- `Process` — domain experts or business analysts with knowledge of the company's SAP or system configuration. Only written when company-specific decisions are documented.

**When to create a Subject vs a Concept:**
A Subject requires at least one `implement ->` edge to a domain node. If the business term exists but no data implementation is confirmed yet, write the definition on a related `Concept` page and promote it to a `Subject` once the implementation is identified.

---

## 6. Linking external knowledge

The conceptual layer is the natural attachment point for **external references** — links to glossaries, regulatory definitions, data dictionaries, or ontologies that inform a business concept. Rather than duplicating external definitions inside Subject pages, a Subject can carry a `## Citations` section with links to authoritative external sources. This avoids maintaining duplicate copies of definitions that are owned externally.

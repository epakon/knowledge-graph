# Knowledge Graph Specification

**Version:** 1.3.0
**License:** [CC BY-NC 4.0](LICENSE)

---

## Table of Contents

1. [Goals](#1-goals)
2. [Terminology](#2-terminology)
3. [Data Model](#3-data-model)
4. [Link Format](#4-link-format)
5. [Space Structure](#5-space-structure)
6. [Page Templates](#6-page-templates)
7. [Versioning](#7-versioning)
8. [Agent Integration](#8-agent-integration)
9. [Tooling](#9-tooling)
10. [Backend Adapters](#10-backend-adapters)
11. [Version History](#11-version-history)

---

## 1. Goals

- **Centralize business knowledge** in a single, searchable location that both humans and AI agents can read.
- **Eliminate hallucinated business logic** in AI-generated SQL by grounding agents in structured, versioned definitions.
- **Encode semantics as a graph** — typed, directed edges between knowledge pages capture dependencies, rules, and relationships that plain text cannot.
- **Require no export pipeline** — knowledge lives where people write it; agents retrieve it at query time via semantic search.
- **Support incremental growth** — a single domain with two tables and three metrics is as valid as a graph with hundreds of nodes.

---

## 2. Terminology

| Term | Definition |
|---|---|
| **Node** | A single-topic page representing one business entity. Each node has a type, a unique name, and structured header fields. |
| **Node type** | The category of a node: `Subject`, `Domain`, `Table`, `Measure`, `Attribute`, `Filter`, `VerifiedQuery`, `BusinessRule`, `Disambiguation`. |
| **Edge** | A typed, directed link between two nodes. Encoded as a readable label embedded in the source page body. |
| **Edge kind** | The semantic verb describing the relationship: `implement`, `relatedTo`, `attribute`, `joinedTo`, `disambiguate`, `apply`, `contain`. |
| **Owning side** | The page that declares the edge (source → target direction). |
| **Back-reference** | A convenience link on the target page pointing back to the source. Navigation only — not a separate semantic edge. |
| **Relationship page** | A reified edge: a dedicated page for an edge that carries a `Reason` and `Consequence if Ignored`. Encoded as a page rather than a bare link. |
| **Reified edge kind** | An edge kind that requires a Relationship page: `mandatory`, `requires`, `guards`, `overrides`, `demonstrates`. |
| **Domain** | A scoped collection of nodes tied to specific data tables and SQL expressions. |
| **Subject** | A global node (shared across all domains) that holds the business definition of a concept. |
| **Identity key** | The field whose value uniquely identifies a node within its type (always `name`). |

---

## 3. Data Model

Full schema: [spec/data-model.md](spec/data-model.md)
Machine-readable schema: [spec/schema.yaml](spec/schema.yaml)

### 3.1 Node types

| Node type | Identity key | Core properties | Scope |
|---|---|---|---|
| `Subject` | `name` | `business_definition`, `scope` | global |
| `Domain` | `name` | `owner` | per-domain |
| `Table` | `name` | `table_kind`, `source`, `description` | per-domain |
| `Measure` | `name` | `kind`, `synonyms`, `status`, `definition_sql` | per-domain |
| `Attribute` | `name` | `kind`, `synonyms`, `access_modifier`, `expression_sql` | per-domain |
| `Filter` | `name` | `mandatory`, `synonyms`, `predicate_sql` | per-domain |
| `VerifiedQuery` | `name` | `onboarding_question`, `verified_by`, `verified_at`, `status`, `question`, `sql` | per-domain |
| `BusinessRule` | `name` | `definition`, `consequence_if_violated` | per-domain |
| `Disambiguation` | `name` | `always_ask` | per-domain |

> **Relationship pages are not nodes.** They are reified edges — flattened into typed relationships with properties in any graph database export. They remain pages for human readability.

### 3.2 Edge kinds

#### Hyperlink edges (no properties)

| Kind | Valid source types | Valid target types | Notes |
|---|---|---|---|
| `implement` | Subject, Measure, BusinessRule, Filter | Filter, Measure, BusinessRule, VerifiedQuery | Subject → any; others → VerifiedQuery only |
| `relatedTo` | any | any | Symmetric generic cross-link |
| `attribute` | Table | Attribute | |
| `joinedTo` | Table | Table | Symmetric |
| `disambiguate` | Subject | Disambiguation | |
| `apply` | BusinessRule | Table, Measure | |
| `contain` | Domain | Table, Measure, Filter, VerifiedQuery, BusinessRule | |

#### Reified edges (Relationship pages — carry `reason` and `consequence`)

| Kind | Source type | Target type |
|---|---|---|
| `mandatory` | Filter | Table |
| `requires` | Measure | Filter |
| `guards` | Filter | Measure |
| `overrides` | BusinessRule | Attribute |
| `demonstrates` | VerifiedQuery | BusinessRule |

### 3.3 Schema diagram

```mermaid
graph LR
    Subject["Subject"]
    Domain["Domain"]
    Table["Table"]
    Measure["Measure"]
    Attribute["Attribute"]
    Filter["Filter"]
    VerifiedQuery["VerifiedQuery"]
    BusinessRule["BusinessRule"]
    Disambiguation["Disambiguation"]

    Subject -->|implement| Filter
    Subject -->|implement| Measure
    Subject -->|implement| BusinessRule
    Subject -->|disambiguate| Disambiguation
    Subject -->|relatedTo| Subject

    Domain -->|contain| Table
    Domain -->|contain| Measure
    Domain -->|contain| Filter
    Domain -->|contain| VerifiedQuery
    Domain -->|contain| BusinessRule

    Table -->|attribute| Attribute
    Table -->|joinedTo| Table

    Table --- mandatory{mandatory}
    Filter --- mandatory

    Measure --- requires{requires}
    Filter --- requires

    Filter --- guards{guards}
    Measure --- guards

    BusinessRule --- overrides{overrides}
    Attribute --- overrides

    VerifiedQuery -->|implement| Measure
    VerifiedQuery -->|relatedTo| Filter
    VerifiedQuery -->|relatedTo| BusinessRule

    BusinessRule -->|apply| Table
    BusinessRule -->|apply| Measure
    BusinessRule -->|relatedTo| Filter
    BusinessRule -->|relatedTo| Disambiguation
    BusinessRule -->|implement| VerifiedQuery

    Measure -->|implement| VerifiedQuery
    Measure -->|relatedTo| BusinessRule

    Filter -->|implement| VerifiedQuery

    Attribute -->|relatedTo| BusinessRule
    Attribute -->|relatedTo| Filter
    Attribute -->|relatedTo| Subject
```

> Rectangles = node types. Diamonds = reified edge kinds (Relationship pages). Labelled arrows = hyperlink edge kinds. Only the owning direction is shown — back-references use the same verb with `←`.

---

## 4. Link Format

Full reference: [spec/link-format.md](spec/link-format.md)

Every edge is encoded as a **self-contained edge statement** embedded as the clickable label of a link in the page body:

```
Owning side (source page):
  [Source: SourceName edgeKind -> Target: TargetName]

Back-reference (target page):
  [Source: SourceName edgeKind <- Target: TargetName]

Relationship page link (same label on both From and To pages):
  [Relationship: X kind -> Y]
```

Rules:
- Use ASCII `->` and `<-` — not unicode arrows.
- All hyperlink edges live in a `## Links` section.
- Relationship page links live in a separate `## Relationships` section.
- Back-references are navigation shortcuts only — not separate semantic edges.

### Back-reference constraints

1. **No symmetric duplicates.** If page A already owns `X edgeKind -> Y`, page A must **not** also carry `X edgeKind <- Y`.
2. **Back-reference edge kind must match the forward edge.** Derive the `<-` label by flipping the arrow in the owning label — never infer the kind from node types.
3. **`implement` is not valid between two Subjects.** Use `relatedTo` for Subject-to-Subject links.

---

## 5. Space Structure

Full reference: [spec/space-structure.md](spec/space-structure.md)

The knowledge base is organized as a **parent-container hierarchy**:

```
Knowledge Graph: <Domain>   (root container)
│
├── vocabulary/             (global, shared across all domains)
│   └── subjects/
│       └── Subject: <Name>
│
└── Domain: <Name>          (domain index — one per domain)
    ├── tables/
    │   └── Table: <Name>
    ├── measures/
    │   └── Measure: <Name>
    ├── attributes/
    │   └── Attribute: <Name>
    ├── filters/
    │   └── Filter: <Name>
    ├── verified-queries/
    │   └── VerifiedQuery: <Name>
    ├── rules/
    │   └── Rule: <Name>
    ├── relationships/
    │   └── Relationship: <From> <kind> <To>
    └── disambiguations/
        └── Disambiguation: <Term>
```

**Page title conventions** (type prefix + colon + name):

```
Subject: Write-Off
Domain: Sales
Table: ORDERS
Measure: REVENUE
Attribute: PAYMENT_METHOD
Filter: ACTIVE_CUSTOMERS
Rule: exclude-reversals
VerifiedQuery: REVENUE_BY_REGION
Relationship: REVENUE requires ACTIVE_CUSTOMERS
Disambiguation: bad-debt
```

---

## 6. Page Templates

Full reference: [spec/page-templates.md](spec/page-templates.md)

Each node type has a canonical template defining required header fields, section headings, and linking conventions. Key principles:

- **Prose belongs only on Subject and Disambiguation pages.** All other pages use structured header fields and a predicate/definition block — no explanatory paragraphs.
- **Keep all fields even if empty** — empty fields are valid; missing fields are not.
- **Semantic annotations on Table pages** use a `Calculated` column to link to Attribute or Measure pages. A node in the `Calculated` column must not also appear in `## Links` (duplicate).

---

## 7. Versioning

Full reference: [spec/versioning.md](spec/versioning.md)

The backend (wiki, git, etc.) versions every page update automatically.

Every update must include a **structured version comment**:

```
v<n> | <YYYY-MM-DD> | <author>
Summary: <one sentence>
Changed: <section or field>
Reason: <why>
Breaking: yes | no
```

**Breaking change** = any change that alters SQL construction or query interpretation (renamed node, changed predicate, removed mandatory filter). Mark `Breaking: yes` and propagate to dependent pages.

This specification uses [Semantic Versioning](https://semver.org):
- `MAJOR` — breaking: node type renamed/removed, edge kind removed, folder path changed, template field removed
- `MINOR` — additive: new node type, new edge kind, new template section
- `PATCH` — clarification, wording fix, non-breaking convention update

---

## 8. Agent Integration

### How agents consume the knowledge base

Agents must query the knowledge base **before writing any SQL** and **before answering any business question**. The graph encodes the correct SQL construction pattern:

1. **Start at the Measure.** Read its SQL definition — this is the aggregate expression.
2. **Follow reified edges** (`requires`, `mandatory`) to find Filter pages. Every mandatory filter must appear in the `WHERE` clause.
3. **Follow `relatedTo`** to find BusinessRule pages. Each rule specifies additional `WHERE` conditions or computation patterns.
4. **Check for Disambiguation** if the question contains an ambiguous term. Present the clarifying question to the user before generating SQL.
5. **Use VerifiedQuery pages** as reference implementations. If an exact match exists, return or adapt the verified SQL rather than generating from scratch.
6. **Check Attribute pages** for columns with derived expressions — use the expression from the Attribute page, not a raw column reference.
7. **Never fabricate business logic** — if a rule or filter is not found in the knowledge base, state that explicitly.

### When to search for which node type

| Situation | Node type to retrieve |
|---|---|
| User asks about a KPI or measure | `Measure: <name>` |
| User asks what a term means | `Subject: <term>` or `Disambiguation: <term>` |
| User asks about filters for a table | Reified edges where kind = `mandatory` and target = `<table>` |
| User asks for a known query pattern | `VerifiedQuery: <topic>` |
| User asks about a business rule | `Rule: <name>` |
| Ambiguous term in the question | `Disambiguation: <term>` — ask the clarifying question first |
| User asks about a column or derived field | `Attribute: <name>` |

### Semantic search strategy

Use **natural-language phrases describing intent**, not exact page titles:

```
"mandatory filters for <table name>"
"how is <measure> computed"
"what does <term> mean in <domain>"
"SQL example for <question>"
"rule for excluding <condition>"
"disambiguation for <ambiguous term>"
"derived expression for <column>"
```

When search returns multiple results, prefer the page whose title prefix matches the needed node type.

---

## 9. Tooling

Reference implementations and tooling design are documented in `adapters/`:

| Document | Purpose |
|---|---|
| [adapters/confluence/snapshot-pipeline.md](adapters/confluence/snapshot-pipeline.md) | Crawl pages → JSON snapshot → interactive graph visualization |
| [adapters/confluence/graph-api.md](adapters/confluence/graph-api.md) | Knowledge Graph API for programmatic graph operations (bulk updates, migration, graph DB import) |
| [adapters/confluence/agent-skill.md](adapters/confluence/agent-skill.md) | Agent operational spec: five workflows with step-by-step instructions |

---

## 10. Backend Adapters

The spec is backend-agnostic. Adapters fall into two categories:

**Content storage** — human-facing systems where nodes and edges are created and maintained as pages or documents (wikis, shared drives, Markdown files). The full knowledge lives here: business definitions, SQL expressions, reason/consequence prose, synonyms, version history. This is the source of truth and the only authoring surface. Any content storage that supports hierarchical organization, cross-page links with custom labels, version history, and full-text or semantic search can implement this specification.

**Graph DB** — a structural projection of the content storage, not a copy. The graph DB holds the skeleton of the graph — node labels, key properties (`name`, `domain`, `status`), and typed relationships with edge properties — but not the full page bodies. It is derived from the content storage via the Knowledge Graph API export pipeline (JSON indexes → graph import) and serves three purposes:
- **Duplicate prevention** — uniqueness constraints on `(label, name)` guard against duplicate nodes that Confluence search alone can miss
- **Fast graph traversal** — Cypher/Gremlin queries for path finding, neighbor lookup, and impact analysis that are expensive or impossible via MCP
- **Lineage and traceability** — every node retains its `page_id`, linking any graph query result back to the exact source page in the content storage

### Content storage adapters

| Storage | Adapter doc |
|---|---|
| Confluence | [adapters/confluence/confluence-adapter.md](adapters/confluence/confluence-adapter.md) |

### Graph DB adapters

| Database | Adapter doc |
|---|---|
| *(planned)* | [adapters/graph-db/README.md](adapters/graph-db/README.md) |

---

## 11. Version History

See [CHANGELOG.md](CHANGELOG.md) for the full release history.

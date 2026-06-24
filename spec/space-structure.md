# Space Structure

> Part of the [Knowledge Graph Specification](../SPEC.md).

---

## Overview

The knowledge base is organized as a **parent-container hierarchy**. Containers (folders, parent pages, directories) group nodes by type within each domain. There is no flat namespace — every node lives under a typed container.

This structure is **backend-agnostic**: it maps to Confluence parent pages, directory trees in a git repository, Notion databases, or any hierarchical document store.

---

## Canonical hierarchy

```
Knowledge Graph: <Domain>         (root container — one per domain)
│
├── vocabulary/                   global, shared across all domains
│   └── subjects/
│       └── Subject: <Name>       one page per subject
│
└── Domain: <Name>                domain index page (follows Domain template)
    ├── tables/
    │   └── Table: <Name>
    ├── measures/
    │   └── Measure: <Name>
    ├── attributes/
    │   └── Attribute: <Name>     promoted columns only (see data-model.md §7)
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

---

## Naming conventions

Page titles use a **type prefix + colon + name** pattern:

| Node type | Title format | Example |
|---|---|---|
| Subject | `Subject: <Name>` | `Subject: Write-Off` |
| Domain | `Domain: <Name>` | `Domain: Sales` |
| Table | `Table: <Name>` | `Table: ORDERS` |
| Measure | `Measure: <Name>` | `Measure: REVENUE` |
| Attribute | `Attribute: <Name>` | `Attribute: PAYMENT_METHOD` |
| Filter | `Filter: <Name>` | `Filter: ACTIVE_CUSTOMERS` |
| VerifiedQuery | `VerifiedQuery: <Name>` | `VerifiedQuery: REVENUE_BY_REGION` |
| BusinessRule | `Rule: <Name>` | `Rule: exclude-reversals` |
| Disambiguation | `Disambiguation: <Term>` | `Disambiguation: bad-debt` |
| Relationship | `Relationship: <From> <kind> <To>` | `Relationship: REVENUE requires ACTIVE_CUSTOMERS` |

The prefix is critical — it is how agents and scripts identify node type from the page title alone.

---

## Container pages

Each type sub-folder has a **parent container page** that acts as an index. There are no separate `index.md` files per type — the parent container page lists all children automatically in backends that support child-page enumeration (e.g. Confluence), or is maintained manually in backends that do not (e.g. git).

The following container pages must exist under each domain:

| Container | Path |
|---|---|
| vocabulary root | `vocabulary/` |
| subjects root | `vocabulary/subjects/` |
| domain index | `Domain: <Name>` |
| tables | `<domain>/tables/` |
| measures | `<domain>/measures/` |
| attributes | `<domain>/attributes/` |
| filters | `<domain>/filters/` |
| verified queries | `<domain>/verified-queries/` |
| rules | `<domain>/rules/` |
| relationships | `<domain>/relationships/` |
| disambiguations | `<domain>/disambiguations/` |

---

## The `vocabulary/` layer — design notes

The `vocabulary/` container is intentionally kept separate from domain containers. It is the **vocabulary layer** of the knowledge graph — domain-agnostic, stable, and shared.

The vocabulary layer holds knowledge that exists independently of any specific table or SQL implementation: business terms, methodologies, KPI definitions at a conceptual level, and links to external authoritative sources such as glossaries, regulatory definitions, data dictionaries, and ontologies. This is the layer business users author and trust; it changes slowly and does not break when data models change.

### Subject, Term, and Concept — current and future node types

Currently only one node type lives in the conceptual layer: **Subject**. A Subject represents a business concept tied to data — something that has SQL implementations in one or more domains (a Measure, Filter, or Rule that embodies it).

Two additional node types are reserved for future use at the conceptual layer:

| Node type | Status | Intended purpose |
|---|---|---|
| `Subject` | Active | Business concept with direct data implementation. Has `implement ->` edges to Measures, Filters, Rules. |
| `Term` | Reserved | Pure business vocabulary — a term that appears in business communication but may not have a direct SQL implementation yet. Useful for onboarding and disambiguation without a data anchor. |
| `Concept` | Reserved | Abstract or composite idea that groups multiple Subjects. Useful for cross-domain thematic grouping (e.g. "Liquidity" grouping DSO, Cash Received, Open Balance). |

Do not create `Term` or `Concept` pages until the node type is formally added to the schema. Use `Subject` for all conceptual nodes today.

### Linking external knowledge to the conceptual layer

The `vocabulary/` layer is also the natural attachment point for **external references** — links to glossaries, regulatory definitions, data dictionaries, or ontologies that inform a business concept. Rather than duplicating external definitions inside Subject pages, a Subject can carry a `## Citations` section with links to authoritative external sources. This avoids maintaining duplicate copies of definitions that are owned externally.

---

## Multi-domain setup

Each domain has its own container tree under a shared root. The `vocabulary/subjects/` path is **global and shared** — Subject pages are not duplicated per domain.

```
Knowledge Graph (root)
├── vocabulary/
│   └── subjects/         ← shared across all domains
│       └── Subject: Revenue
│
├── Domain: Sales
│   ├── measures/
│   │   └── Measure: REVENUE
│   └── ...
│
└── Domain: Finance
    ├── measures/
    │   └── Measure: GROSS_MARGIN
    └── ...
```

When the same concept appears in multiple domains, a single Subject page in `vocabulary/subjects/` holds the shared business definition. Each domain's Attribute or Measure page links to the Subject.

---

## Cross-domain nodes — ownership and linking

A node is owned by exactly one domain — the domain with the `contain ->` edge to it. Ownership means that domain is responsible for the node's correctness and verification. The page is stored in the owning domain's folder.

Other domains that use the node do not add a `contain ->` edge. Their connection is visible in the graph through the existing edges from their own nodes (e.g. a Measure in Finance that links via `relatedTo ->` to a Rule owned by Sales).

Use this decision tree to determine which domain owns a cross-domain node:

```
Does the node's SQL reference a specific table?
├── Yes → the domain that owns that table is the owner
└── No (table-agnostic: pure predicate, no FROM clause)
    └── Is one domain clearly the primary consumer or originator?
        ├── Yes → that domain is the owner
        └── No (equal consumers)
            └── Assign to the domain of the team who first needed it
                (first-mover rule — a node must have an owner; ambiguity is not an option)
```

---

## Adding a new domain

1. Create the root container page: `Knowledge Graph: <Domain>`
2. Create the domain index page: `Domain: <Name>` as a child of the root
3. Create each type container: `tables/`, `measures/`, `attributes/`, `filters/`, `verified-queries/`, `rules/`, `relationships/`, `disambiguations/`
4. For concepts already in `vocabulary/subjects/` (e.g. a shared business term), the new domain's node pages link to the existing Subject — do not duplicate the definition
5. Create Relationship pages for semantic edges specific to the new domain
6. The new domain inherits all shared Subjects automatically via semantic search

---

## What does NOT live in the knowledge base

- **Physical database schemas** — table definitions, column types, DDL. These are maintained in the data source (dbt, data catalog, etc.). The knowledge base stores semantic annotations only.
- **Dashboard or report definitions** — these reference measures but are not graph nodes.
- **Data lineage at the column level** — fine-grained column-to-column lineage lives in a data catalog. The knowledge base captures business-level dependencies (which filters are mandatory, which rules apply).
- **Live data or query results** — the knowledge base is a definition layer, not a query execution layer.

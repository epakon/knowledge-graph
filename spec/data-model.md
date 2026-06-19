# Data Model

> Part of the [Knowledge Graph Specification](../SPEC.md).

---

## 1. Node Type Schema

Each node type maps to a **node label** in a target graph database. The identity key is the unique constraint enforced at migration time.

| Node type | Node label | Identity key | Core properties | Scope |
|---|---|---|---|---|
| `Subject` | `Subject` | `name` | `business_definition`, `scope` | global (`concepts/subjects/`) |
| `Domain` | `Domain` | `name` | `owner` | per-domain |
| `Table` | `Table` | `name` | `table_kind`, `source`, `description` | per-domain |
| `Measure` | `Measure` | `name` | `kind`, `synonyms`, `status`, `definition_sql` | per-domain |
| `Attribute` | `Attribute` | `name` | `kind`, `synonyms`, `access_modifier`, `expression_sql` | per-domain |
| `Filter` | `Filter` | `name` | `mandatory`, `synonyms`, `predicate_sql` | per-domain |
| `VerifiedQuery` | `VerifiedQuery` | `name` | `onboarding_question`, `verified_by`, `verified_at`, `status`, `question`, `sql` | per-domain |
| `BusinessRule` | `BusinessRule` | `name` | `definition`, `consequence_if_violated` | per-domain |
| `Disambiguation` | `Disambiguation` | `name` | `always_ask` | per-domain |

> **`Relationship` pages are not nodes in the target graph database.** They are flattened into typed relationships with properties (see §2.2). They remain pages for human readability but are not registered as nodes.

### Uniqueness constraints

Each node type requires a uniqueness constraint on `name`. Example in Cypher (Neo4j):

```cypher
CREATE CONSTRAINT FOR (n:Subject)        REQUIRE n.name IS UNIQUE;
CREATE CONSTRAINT FOR (n:Domain)         REQUIRE n.name IS UNIQUE;
CREATE CONSTRAINT FOR (n:Table)          REQUIRE n.name IS UNIQUE;
CREATE CONSTRAINT FOR (n:Measure)        REQUIRE n.name IS UNIQUE;
CREATE CONSTRAINT FOR (n:Attribute)      REQUIRE n.name IS UNIQUE;
CREATE CONSTRAINT FOR (n:Filter)         REQUIRE n.name IS UNIQUE;
CREATE CONSTRAINT FOR (n:VerifiedQuery)  REQUIRE n.name IS UNIQUE;
CREATE CONSTRAINT FOR (n:BusinessRule)   REQUIRE n.name IS UNIQUE;
CREATE CONSTRAINT FOR (n:Disambiguation) REQUIRE n.name IS UNIQUE;
```

---

## 2. Edge Type Schema

Two classes of edges in the knowledge base map to two classes of graph relationships.

### 2.1 Hyperlink edge kinds (no properties)

Seven kinds, all single-verb. These become relationships with type only — no properties.
Back-references on pages are navigation shortcuts and are **not** imported into the graph.

| Kind | Relationship type | Valid source labels | Valid target labels | Notes |
|---|---|---|---|---|
| `implement` | `IMPLEMENTS` | Subject, Measure, BusinessRule, Filter | Filter, Measure, BusinessRule, VerifiedQuery | Subject → any; Measure/BusinessRule/Filter → VerifiedQuery only |
| `relatedTo` | `RELATED_TO` | any | any | Symmetric generic cross-link |
| `attribute` | `HAS_ATTRIBUTE` | Table | Attribute | |
| `joinedTo` | `JOINED_TO` | Table | Table | Symmetric; join key stored as property `on` |
| `disambiguate` | `DISAMBIGUATES` | Subject | Disambiguation | |
| `apply` | `APPLIES_TO` | BusinessRule | Table, Measure | |
| `contain` | `CONTAINS` | Domain | Table, Measure, Filter, VerifiedQuery, BusinessRule | |

### 2.2 Reified edge kinds (Relationship pages → typed relationships with properties)

Relationship pages are **flattened** into graph relationships with `reason` and `consequence` properties. The page title encodes source, kind, and target; the body provides the semantic payload.

| Kind | Relationship type | Source label | Target label | Properties |
|---|---|---|---|---|
| `mandatory` | `MANDATORY_FOR` | Filter | Table | `reason`, `consequence` |
| `requires` | `REQUIRES` | Measure | Filter | `reason`, `consequence` |
| `guards` | `GUARDS` | Filter | Measure | `reason`, `consequence` |
| `overrides` | `OVERRIDES` | BusinessRule | Attribute | `reason`, `consequence` |
| `demonstrates` | `DEMONSTRATES` | VerifiedQuery | BusinessRule | `reason`, `consequence` |

### Edge conflict rule

A given `(source, target)` pair must have **at most one edge of each type**. When the same pair has both a hyperlink edge and a reified edge covering the same semantic (e.g. `RELATED_TO` and `REQUIRES` between Measure and Filter), the reified edge takes precedence and the weaker `RELATED_TO` should be removed.

---

## 3. Node Index

**Purpose:** uniqueness registry and graph database import input. Before creating a new page, check this index to prevent duplicate nodes.

### Schema

```json
{
  "meta": {
    "generated_at": "<ISO 8601>",
    "node_count": 0
  },
  "nodes": [
    {
      "name": "<short name>",
      "label": "<NodeType>",
      "page_title": "<NodeType>: <Name>",
      "page_id": "<backend page ID>",
      "page_url": "<URL>",
      "domain": "<domain name or 'global'>",
      "status": "active | deprecated"
    }
  ]
}
```

### Fields

| Field | Description |
|---|---|
| `name` | Short name (page title minus the `NodeType: ` prefix). Identity key for graph `MERGE` / upsert. |
| `label` | Node label (matches node type). |
| `page_title` | Full page title. |
| `page_id` | Backend page ID. Stable across renames. |
| `page_url` | Direct link to the page. |
| `domain` | Domain scope (e.g. `Sales`, or `global` for Subjects). |
| `status` | `active` or `deprecated`. |

---

## 4. Edge Index

**Purpose:** edge audit log and graph database relationship import input.

### Schema

```json
{
  "meta": {
    "generated_at": "<ISO 8601>",
    "edge_count": 0
  },
  "edges": [
    {
      "source": "<NodeType>: <Name>",
      "target": "<NodeType>: <Name>",
      "relationship_type": "REQUIRES",
      "kind": "requires",
      "style": "reified",
      "via": "Relationship: <From> requires <To>",
      "properties": {
        "reason": "<one sentence>",
        "consequence": "<one sentence>"
      }
    },
    {
      "source": "<NodeType>: <Name>",
      "target": "<NodeType>: <Name>",
      "relationship_type": "APPLIES_TO",
      "kind": "apply",
      "style": "hyperlink",
      "via": null,
      "properties": {}
    }
  ]
}
```

### Fields

| Field | Description |
|---|---|
| `source` | Full page title of the source node. |
| `target` | Full page title of the target node. |
| `relationship_type` | Graph relationship type (from §2 schema). |
| `kind` | Edge kind (`apply`, `requires`, etc.). |
| `style` | `hyperlink` (no properties) or `reified` (has Reason + Consequence). |
| `via` | For reified edges: the Relationship page title. `null` for hyperlinks. |
| `properties` | For reified edges: `reason` and `consequence` strings. Empty object for hyperlinks. |

---

## 5. Audit Rules

These rules should be validated after every snapshot regeneration:

| Rule | Check |
|---|---|
| No duplicate nodes | `name + label` must be unique in the node index |
| No duplicate edges | `(source, target, relationship_type)` must be unique in the edge index |
| Reified beats hyperlink | If `(source, target)` has both a `RELATED_TO` hyperlink and a reified edge, flag the `RELATED_TO` as redundant |
| Back-refs not imported | Edges where the label contains `<-` are navigation back-refs — excluded from the edge index |
| Relationship pages flattened | Every Relationship page must have a corresponding entry in the edge index with `style: reified` |

---

## 6. Graph Database Migration Notes

When migrating from the wiki to a target graph database:

1. **Import nodes** from the node index — upsert by `(label, name)` and set properties.
2. **Import edges** from the edge index — hyperlinks first, then reified edges (they may reference the same pairs).
3. **Skip back-references** — they are navigation artifacts, not graph edges.
4. **Relationship pages** have already been flattened into the edge index; their wiki pages can be archived post-migration.
5. **`Domain` nodes** can be migrated as nodes or treated as a property (`domain: "Sales"`) on other nodes — decision deferred.
6. **`page_id`** should be retained as a node property for traceability during the transition period.

---

## 7. Semantic Annotations and Cross-Domain Linking

### Column-level semantics: inline vs Attribute page

The Table `## Semantic annotations` table is the entry point for column-level semantics. The `Calculated` column links to the node that carries the semantic payload:

| Situation | Where the payload lives |
|---|---|
| Simple column — description only | Inline in `## Fields / Physical columns`. No Attribute page. |
| Column with derived expression, rule link, filter label, access modifier, or non-trivial synonyms | **Attribute page.** List in `Calculated` column. Do **not** also link in `## Related` — that is a duplicate. |
| Column that feeds a domain KPI directly | **Measure page.** List in `Calculated` column. Same deduplication rule. |

The `## Related` section on a Table page should only contain Attribute or Measure links that are **not** already in the Semantic annotations table.

### Cross-domain linking via Subject

When the same column concept appears in multiple domains (e.g. `PAYMENT_METHOD` in Sales and in Finance), the shared meaning lives on a **Subject** page in `concepts/subjects/`. Each domain's Attribute page links to the Subject via `Attribute: X relatedTo -> Subject: Y`, and the Subject links back. The business definition is written once on the Subject; Attribute pages carry the domain-specific expression and access rules.

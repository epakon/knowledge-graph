# Data Model

> Part of the [Knowledge Graph Specification](../SPEC.md).

---

## 1. Two-layer architecture

The knowledge graph is organized into two distinct layers:

| Layer | Path | Scope | Owned by | Changes when |
|---|---|---|---|---|
| **Vocabulary** | `vocabulary/` | Global — shared across all domains | Business / domain experts | Business terminology evolves |
| **Domain** | `<domain>/` | Per-domain — tied to specific tables and SQL. Each node is owned by exactly one domain. | Data engineering | Data model changes |

**Vocabulary layer** holds knowledge that exists independently of any table or SQL implementation: business terms, KPI definitions at a conceptual level, methodologies, and links to authoritative external sources (glossaries, regulatory definitions, data dictionaries, ontologies). It is stable and does not break when data models change.

**Domain layer** holds the technical implementation: table structures, SQL expressions, filter predicates, business rules scoped to a specific data domain, and verified queries. It is coupled to the data model and changes with it.

The bridge between the two layers is the `implement ->` edge: a Vocabulary node (e.g. `Subject: DSO`) points to the Domain nodes that embody it (e.g. `Measure: DSO`, `Rule: dso-annualization`). This lets agents retrieve the business definition first, then follow edges to the SQL implementation.

---

## 2. Node Type Schema

Each node type maps to a **node label** in a target graph database. The identity key is the unique constraint enforced at migration time.

### Node type descriptions

| Node type | Description |
|---|---|
| `Subject` | Global business concept. The only node type where prose lives. Shared across all domains. |
| `Domain` | Domain index page. Entry point and navigational container for a domain. |
| `Table` | One page per logical table. Has `TableKind`: fact, dimension, or bridge. |
| `Measure` | Computed KPI expression — always an aggregate (SUM, COUNT, AVG, etc.). |
| `Attribute` | Promoted column with semantic payload: derived expression, rule-linked, or cross-domain column. |
| `Filter` | Named SQL predicate; mandatory or contextual. |
| `VerifiedQuery` | Human-approved SQL with verifier name, verification date, and onboarding flag. |
| `BusinessRule` | Named rule governing query construction or interpretation. |
| `Disambiguation` | Ambiguous term requiring clarification before any query is issued. |
| `Relationship` | Reified semantic edge with Reason + Consequence if Ignored. Has its own page; is not a node in the graph DB. |

> Visualization shapes (rectangles vs. diamonds in diagrams) are a rendering convention, not a schema property. See [adapters/engine/confluence/snapshot-pipeline.md](../adapters/engine/confluence/snapshot-pipeline.md) for node colors and shapes used in the graph diagram.

### Full schema

| Node type | Node label | Identity key | Core properties | Scope |
|---|---|---|---|---|
| `Subject` | `Subject` | `name` | `business_definition`, `scope` | global (`vocabulary/subjects/`) |
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

## 3. Edge Type Schema

Two classes of edges in the knowledge base map to two classes of graph relationships.

### 2.1 Hyperlink edge kinds (no properties)

Seven kinds, all single-verb. These become relationships with type only — no properties.
Back-references on pages are navigation shortcuts and are **not** imported into the graph.

| Kind | Relationship type | Valid source labels | Valid target labels | Notes |
|---|---|---|---|---|
| `implement` | `IMPLEMENTS` | Subject, Measure, BusinessRule, Filter | Filter, Measure, BusinessRule, VerifiedQuery | Subject → any; Measure/BusinessRule/Filter → VerifiedQuery only |
| `relatedTo` | `RELATED_TO` | any | any | Symmetric generic cross-link |
| `calculate` | `CALCULATES` | Table | Attribute, Measure | Table is the source that exposes (Attribute) or computes (Measure) the value |
| `joinedTo` | `JOINED_TO` | Table | Table | Symmetric; join key stored as property `on` |
| `disambiguate` | `DISAMBIGUATES` | Subject | Disambiguation | |
| `apply` | `APPLIES_TO` | BusinessRule | Table, Measure | |
| `contain` | `CONTAINS` | Domain | Table, Measure, Filter, VerifiedQuery, BusinessRule | Exactly one Domain per node — see `spec/space-structure.md` for ownership rules |

### 2.2 Reified edge kinds (Relationship pages → typed relationships with properties)

Relationship pages are **flattened** into graph relationships with `reason` and `consequence` properties. The page title encodes source, kind, and target; the body provides the semantic payload.

| Kind | Relationship type | Source label | Target label | Properties |
|---|---|---|---|---|
| `mandatory` | `MANDATORY_FOR` | Filter | Table | `reason`, `consequence` |
| `requires` | `REQUIRES` | Measure | Filter | `reason`, `consequence` |
| `guards` | `GUARDS` | Filter | Measure | `reason`, `consequence` |
| `overrides` | `OVERRIDES` | BusinessRule | Attribute | `reason`, `consequence` |
| `demonstrates` | `DEMONSTRATES` | VerifiedQuery | BusinessRule | `reason`, `consequence` |

#### Why this list is closed (for now)

The five reified kinds are not a general-purpose "any dependency can become a Relationship" mechanism — each one is a specific `(source type, target type, direction)` pattern identified as a recurring way this data warehouse silently produces wrong answers if the dependency is ignored. The list is closed in the sense that a Relationship page's `Kind` must currently be one of these five; it is open in the sense that new kinds can be added by a future spec revision if a new recurring failure pattern is identified. It is neither a fixed permanent ceiling nor "reify whatever you want, whenever you want."

Some hyperlink kinds structurally can never warrant a reified form. `calculate`, `contain`, `disambiguate`, and `joinedTo`'s ordinary case describe facts that are unconditionally true by construction — a Table either calculates an Attribute or it doesn't; a Domain either contains a Table or it doesn't. There is no "sometimes true, and here's what breaks if you get it wrong" story to tell, so there is nothing for a Reason/Consequence pair to add. Promoting these produces an empty page, not a more precise one. That said, this isn't a logical impossibility for every instance of every such kind forever — a `joinedTo` edge can carry real risk (e.g. a join that fans out and silently duplicates rows if the wrong key is used), which is a genuine reason/consequence story with no cataloged reified kind yet, simply because no such case has been identified in this knowledge base so far. If one is found, the right move is a new spec-versioned reified kind — not reusing `joinedTo`'s name with reason/consequence fields bolted on.

At the opposite extreme, `relatedTo` is the kind most often promoted — several of today's reified kinds are, structurally, promotions of what would otherwise be a `relatedTo` edge between the same two node types (see the Edge conflict rule below). But promotion never keeps the `relatedTo` name: it is deliberately generic, carrying no direction or role, so reifying it as-is (`relatedTo` + reason + consequence) wouldn't add meaning, only length — the graph still couldn't tell "this filter is load-bearing for the table" from "this filter protects one specific measure" without reading prose. The promoted kind has to name the specific role (`mandatory`, `requires`, `guards`, ...) precisely because `relatedTo` itself is too generic to say anything on its own.

Practical implication for agents/authors: before creating a `Relationship:` page, check whether the dependency matches one of the five cataloged patterns. If it does, use that kind. If it doesn't fit any pattern but still has genuine business-rule weight, put it on a `BusinessRule` page instead (open-ended prose, linked via the ordinary hyperlink kinds) rather than inventing an ad-hoc sixth reified kind on the spot. Proposing an actual new reified kind is a deliberate spec change, not a per-page decision.

### Edge conflict rule

A given `(source, target)` pair must have **at most one edge of each type**. When the same pair has both a hyperlink edge and a reified edge covering the same semantic (e.g. `RELATED_TO` and `REQUIRES` between Measure and Filter), the reified edge takes precedence and the weaker `RELATED_TO` should be removed.

---

## 4. Node Index

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

## 5. Edge Index

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

## 6. Audit Rules

These rules should be validated after every snapshot regeneration:

| Rule | Check |
|---|---|
| No duplicate nodes | `name + label` must be unique in the node index |
| No duplicate edges | `(source, target, relationship_type)` must be unique in the edge index |
| Reified beats hyperlink | If `(source, target)` has both a `RELATED_TO` hyperlink and a reified edge, flag the `RELATED_TO` as redundant |
| Back-refs not imported | Edges where the label contains `<-` are navigation back-refs — excluded from the edge index |
| Relationship pages flattened | Every Relationship page must have a corresponding entry in the edge index with `style: reified` |

---

## 7. Graph Database Migration Notes

The node index (`kg-node-index.json`) and edge index (`kg-edge-index.json`) serve two purposes simultaneously:

- **Duplicate tracking** — before creating a new node, check the node index to confirm no node with the same `(label, name)` already exists. This is the primary guard against duplicate pages and split-brain definitions.
- **Graph DB import input** — each entry maps 1:1 to a `MERGE` upsert on a node or relationship, ready for import into any property graph store.

For the full migration procedure, node/edge mapping tables, import examples, and sync strategy, see the graph DB engine adapter contract in [adapters/engine/engine.md](../adapters/engine/engine.md#graph-db).

---

## 8. Semantic Annotations and Cross-Domain Linking

### Column-level semantics: inline vs promoted node

**Default is inline.** A column that has only a description and no derived expression, rule link, access restriction, or cross-domain usage stays in the Table's `## Fields / Physical columns` section. Promotion to an Attribute or Measure page is the exception, not the rule.

The Table page carries a `### Semantic annotations` section with a `Calculated` column. That column is a **link slot**: when a column has been promoted to its own node, the link to that node goes here. "Calculated" means "this column's semantic payload lives on a separate page" — it is not a criterion for promotion; it is the result of the promotion decision.

#### Attribute promotion criteria

Create an Attribute page when the column has **at least one** of:
- A non-trivial derived expression — CASE WHEN, COALESCE, sign convention, cast, date truncation, or any expression that is not a direct column reference
- A link to a BusinessRule or Subject (the column embodies or implements a named concept)
- `access_modifier: private_access` — the column must not be exposed to query agents directly
- Non-trivial synonyms or cross-domain relevance — the same concept appears in multiple domains and needs a shared anchor

A column with only a description, a plain data type, and no cross-domain usage does **not** warrant an Attribute page regardless of how important it is to the business.

#### Measure promotion criteria

Create a Measure page when the column has **at least one** of:
- A non-trivial definition — a derived formula, a ratio, an expression combining multiple columns, a conditional aggregate (e.g. `SUM(CASE WHEN ... END)`), or a cross-domain KPI
- Synonyms that need to be surfaced to query agents (e.g. the same KPI is known by multiple names)
- A link to a BusinessRule, Filter, or Subject — the measure embodies a named business concept with its own rules

A simple aggregate over a single column — `SUM(revenue)`, `COUNT(order_id)` — does **not** automatically warrant a Measure page. It stays inline in `## Semantic annotations` unless one of the criteria above applies.

#### Deduplication rule

Once a column has its own Attribute or Measure page, list it in the `Calculated` column of `## Semantic annotations`. Do **not** also add it to `## Links` — that is a duplicate link.

### Cross-domain linking via Vocabulary

When the same concept appears in multiple domains (e.g. `PAYMENT_METHOD` in Sales and in Finance), the shared meaning lives on a node in `vocabulary/` — today always a **Subject** page in `vocabulary/subjects/`. Each domain's Attribute or Measure page links to the Vocabulary node via `relatedTo ->`, and the Vocabulary node links back. The business definition is written once on the Vocabulary node; domain pages carry the domain-specific expression and access rules.

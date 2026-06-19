# Link Format

> Part of the [Knowledge Graph Specification](../SPEC.md).

---

## Overview

Every edge between two nodes is encoded as a **self-contained edge statement** embedded as the readable label of a hyperlink in the page body. The entire label is the clickable text — no trailing plain text after the link.

This design makes edges human-readable, searchable by semantic search tools, and parseable by scripts without any metadata sidecar.

---

## Edge statement syntax

```
Owning side (source page — this page is the source of the edge):
  [Source: SourceName edgeKind -> Target: TargetName]

Back-reference (target page — someone else points to this page):
  [Source: SourceName edgeKind <- Target: TargetName]

Relationship page link (same label on both From and To pages):
  [Relationship: X kind -> Y]
```

### Rules

- Use ASCII `->` and `<-`. Do not use unicode `→`/`←` or HTML entities `&rarr;`/`&larr;`.
- The link navigates to the **other** page — never the current page.
- `->` = owning side (I point to the target). `<-` = back-reference (someone points to me).
- Relationship page links always show `->` regardless of which side they appear on.
- All hyperlink edges live in a `## Related` section.
- Relationship page links live in a separate `## Relationships` section.

---

## Page section layout

```
## Related
  ← all hyperlink edges for this page (both owning and back-reference)

## Relationships
  ← all Relationship page links (reified edges this page participates in)
```

These two sections must be kept separate. Mixing hyperlink edges and Relationship page links in a single section is invalid.

---

## When to use a hyperlink edge vs. a Relationship page

Every edge starts as a hyperlink. Promote it to a Relationship page when it gains semantic weight — a stated reason and a consequence if ignored.

| Situation | Use |
|---|---|
| "A depends on B" — the dependency is self-evident from the node types | Hyperlink edge in `## Related` |
| "A depends on B, and here is **why**, and here is **what breaks** if ignored" | Relationship page under `relationships/` |

**Promotion rule:** if you find yourself wanting to annotate a hyperlink edge with a reason or a consequence, stop — create a Relationship page instead. A hyperlink that says "why" is a Relationship page waiting to happen.

### When individual pages are warranted

The same promotion logic applies to columns and computed fields inside a Table:

| Column situation | Where the definition lives |
|---|---|
| Simple column — description only | Inline in the Table's `## Fields / Physical columns`. No separate page. |
| Column with a derived expression (CASE WHEN, COALESCE, sign convention), or linked to a Rule/Subject, or carrying non-trivial synonyms or cross-domain relevance | **Attribute page** under `<domain>/attributes/`. Link from Table's `## Semantic annotations` → `Calculated` column. |
| Column that is the direct source of a domain KPI | **Measure page** under `<domain>/measures/`. Link from Table's `## Semantic annotations` → `Calculated` column. |

The `Calculated` column in the Semantic annotations table is the decision point: once a column has its own Attribute or Measure page, it is listed there and **not** repeated in `## Related` — that would be a duplicate link.

---

## Structural edges vs. semantic edges

This is the most important distinction in the data model. Two kinds of edges exist, and they mean fundamentally different things:

| | Structural edge (`joinedTo`) | Semantic edge (Relationship page) |
|---|---|---|
| **What it expresses** | "These two tables share a key and can be joined in a query" | "This dependency exists for a business reason, and violating it causes a specific consequence" |
| **Carries meaning?** | No — it is a technical fact about data structure | Yes — it encodes business logic and domain knowledge |
| **Has Reason / Consequence?** | Never | Always |
| **Encoded as** | Hyperlink edge in `## Joins`: `Table: A joinedTo -> Table: B on A.col = B.col` | Dedicated Relationship page in `relationships/`: `Relationship: X requires Y` |
| **Equivalent in other tools** | SQL `JOIN`, ERD foreign key, dbt `relationships:` test, Snowflake Semantic View `relationships:` YAML | No direct equivalent — this is domain knowledge, not a database construct |
| **Agent should use it to…** | Construct the correct `JOIN` clause in SQL | Determine which filters are mandatory, which rules apply, why a dependency exists |

### The rule in one sentence

> `joinedTo` tells you **how** to join tables. A Relationship page tells you **why** a dependency exists and **what breaks** if you ignore it.

These two concepts share the word "relationship" in common usage and in many data tools — which is why the distinction must be made explicit. In this standard, "Relationship" (capitalised, as a node type) always means a reified semantic edge with Reason + Consequence. Lower-case "relationship" in the context of database tooling means a join definition.

---

## Edge kind reference

| Kind | Typical source → target | Notes |
|---|---|---|
| `implement` | Subject → Filter, Measure, BusinessRule, VerifiedQuery · Measure, BusinessRule, Filter → VerifiedQuery | Not valid between two Subjects |
| `relatedTo` | any → any | Generic symmetric cross-link |
| `attribute` | Table → Attribute | |
| `joinedTo` | Table → Table | Symmetric |
| `disambiguate` | Subject → Disambiguation | |
| `apply` | BusinessRule → Table, Measure | |
| `contain` | Domain → Table, Measure, Filter, VerifiedQuery, BusinessRule | |

---

## Examples

### Hyperlink edge — owning side

Subject: Write-Off owns an edge to Filter: WRITE_OFF_INVOICES:

```
## Related                              ← on Subject: Write-Off
[Subject: Write-Off implement -> Filter: WRITE_OFF_INVOICES]

## Related                              ← on Filter: WRITE_OFF_INVOICES
[Subject: Write-Off implement <- Filter: WRITE_OFF_INVOICES]
```

### Hyperlink edge — cross-link

Measure: REVENUE owns a cross-link to Rule: exclude-reversals:

```
## Related                              ← on Measure: REVENUE
[Measure: REVENUE relatedTo -> Rule: exclude-reversals]

## Related                              ← on Rule: exclude-reversals
[Measure: REVENUE relatedTo <- Rule: exclude-reversals]
```

### Reified edge — Relationship page links

Filter: ACTIVE_CUSTOMERS participates in a reified mandatory relationship with Table: ORDERS. Both sides show the owning `->` direction:

```
## Relationships                        ← on Filter: ACTIVE_CUSTOMERS
[Relationship: ACTIVE_CUSTOMERS mandatory -> ORDERS]

## Relationships                        ← on Table: ORDERS
[Relationship: ACTIVE_CUSTOMERS mandatory -> ORDERS]
```

### Join edge with join key

```
## Joins                                ← on Table: ORDER_LINES
[Table: ORDER_LINES joinedTo -> Table: ORDERS on ORDER_LINES.ORDER_ID = ORDERS.ID]
```

---

## Back-reference constraints

### 1. No symmetric duplicates

If page A already owns `X edgeKind -> Y`, page A must **not** also carry `X edgeKind <- Y`. The `<-` label belongs on page Y only.

**Invalid (both on same page A):**
```
[Measure: REVENUE relatedTo -> Rule: exclude-reversals]
[Measure: REVENUE relatedTo <- Rule: exclude-reversals]   ← wrong, this belongs on Rule page
```

### 2. Edge kind must match

The back-reference edge kind must exactly match the forward edge kind. Derive the `<-` label by replacing `->` with `<-` in the owning label — never infer the kind from node types.

**Invalid:**
```
Owning (on Measure page):    [Measure: REVENUE requires -> Filter: ACTIVE_CUSTOMERS]
Back-ref (on Filter page):   [Measure: REVENUE relatedTo <- Filter: ACTIVE_CUSTOMERS]   ← wrong kind
```

**Valid:**
```
Owning (on Measure page):    [Measure: REVENUE requires -> Filter: ACTIVE_CUSTOMERS]
Back-ref (on Filter page):   [Measure: REVENUE requires <- Filter: ACTIVE_CUSTOMERS]   ← correct
```

### 3. `implement` is not valid between two Subjects

Use `relatedTo` for Subject-to-Subject links.

**Invalid:**
```
[Subject: Write-Off implement -> Subject: Bad-Debt]
```

**Valid:**
```
[Subject: Write-Off relatedTo -> Subject: Bad-Debt]
```

---

## Visualization conventions

When drawing the graph (e.g. Mermaid diagram):

- **Rectangles** (`[Label]`) — all node types except Relationship: Subject, Table, Measure, Attribute, Filter, Rule, VerifiedQuery, Disambiguation, Domain.
- **Diamonds** (`{Label}`) — Relationship pages only. A Relationship page is a reified edge with its own page carrying Reason + Consequence.
- **Labelled arrows** (`-->|kind|`) — hyperlink edges. No dedicated page.

```
# Reified edge (Relationship page exists):
Filter: ACTIVE_CUSTOMERS --- {mandatory} --- Table: ORDERS

# Hyperlink edge (no Relationship page):
Subject: Write-Off -->|implement| Filter: WRITE_OFF_INVOICES
```

The distinction matters: a diamond in the diagram means there is a dedicated page you can follow to read why the dependency exists and what breaks if it is ignored.

---

## Node/edge kind quick reference

| Node type | Header fields | Typical `## Related` edges | `## Relationships`? |
|---|---|---|---|
| `Subject` | Type, Scope | `implement ->` Filter, Measure, Rule · `disambiguate ->` Disambiguation · `relatedTo ->/<-` Subject | No |
| `Domain` | Type | `contain ->` Table, Measure, Filter, VerifiedQuery, BusinessRule | No |
| `Table` | Type, TableKind, Domain, Source | `joinedTo ->/<-` Table · `attribute ->` Attribute | Yes |
| `Measure` | Type, Domain, Kind, Synonyms, Status | `implement ->` VerifiedQuery · `relatedTo ->/<-` Rule, Filter | Yes |
| `Attribute` | Type, Domain, Kind, Synonyms, access_modifier | `attribute <-` Table · `relatedTo ->/<-` Rule, Filter, Subject | No |
| `Filter` | Type, Domain, Mandatory, Synonyms, Disambiguation | `implement <-` Subject · `implement ->` VerifiedQuery | Yes |
| `VerifiedQuery` | Type, Domain, Onboarding question, Verified by/at, Status | `implement ->` Measure · `relatedTo ->/<-` Filter, Rule | Yes (demonstrates) |
| `BusinessRule` | Type, Domain | `apply ->` Table, Measure · `relatedTo ->/<-` Filter, Disambiguation · `implement <-` Subject | No |
| `Disambiguation` | Type, Domain | `disambiguate <-` Subject | No |
| `Relationship` | Type, Kind, From, To | *(no edge sections — is itself a reified edge)* | N/A |

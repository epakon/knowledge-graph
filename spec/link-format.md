# Link Format

> Part of the [Knowledge Graph Specification](../SPEC.md).

---

## Overview

Every edge between two nodes is encoded as a **self-contained edge statement** embedded as the readable label of a hyperlink in the page body. The entire label is the clickable text — no trailing plain text after the link. There are two exceptions, where a heading defines the edge and the link carries only the target label: the `Calculated` column on Table pages and the type sections on the Domain page (see the rules below).

This design makes edges human-readable, searchable by semantic search tools, and parseable by scripts without any metadata sidecar.

---

## Edge statement syntax

```
Owning side (source page — this page is the source of the edge):
  [Source: SourceName edgeKind -> Target: TargetName]

Back-reference (target page — someone else points to this page):
  [Source: SourceName edgeKind <- Target: TargetName]

Reification page link (same label on both From and To pages):
  [Reification: X kind -> Y]
```

### Rules

- Use ASCII `->` and `<-`. Do not use unicode `→`/`←` or HTML entities `&rarr;`/`&larr;`.
- The link navigates to the **other** page — never the current page.
- `->` = owning side (I point to the target). `<-` = back-reference (someone points to me).
- **Conceptual pages carry no cross-layer links.** A Subject, Concept, Process or Policy page holds only edges to other conceptual pages (owning and back-reference). Every edge between a conceptual node and a logical or consumption node is owned by the logical or consumption page, and no back-reference is written on the conceptual page (audit rule `no_conceptual_down_links`).
- Reification page links always show `->` regardless of which side they appear on.
- All hyperlink edges live in a `## Links` section. On Table pages, owning `calculate` edges live in the `Calculated` column of `### Semantic annotations`, and `joinedTo` edges in `## Joins` as edge statements.
- **Column-defined edge (heading-defined, exception 1 of 2).** A link in a Table page's `Calculated` column carries only the target label: `[Attribute: X](path)` or `[Measure: X](path)`. The column header defines the edge, so it reads as `Table: <this page> calculate -> <target>`. The back-reference on the target page is still a full edge statement: `[Table: T calculate <- Attribute: X](path)`.
- **Section-defined edge (heading-defined, exception 2 of 2).** A link under a type section of a Domain page (`## Tables`, `## Measures`, `## Attributes`, `## Filters`, `## Verified Queries`, `## Rules`, `## Disambiguations`) carries only the target label: `[Table: X](path)`. The page defines the edge, so it reads as `Domain: <this page> contain -> <target>`. Owned pages carry **no** `contain <-` back-reference: the `domain` property already names the owner.
- Reification page links live in a separate `## Reifications` section.

---

## Page section layout

```
## Links
  ← all hyperlink edges for this page (both owning and back-reference)

## Reifications
  ← all Reification page links (reified edges this page participates in)
```

These two sections must be kept separate. Mixing hyperlink edges and Reification page links in a single section is invalid.

---

## When to use a hyperlink edge vs. a Reification page

Every edge starts as a hyperlink. Promote it to a Reification page when it gains semantic weight — a stated reason and a consequence if ignored.

| Situation | Use |
|---|---|
| "A depends on B" — the dependency is self-evident from the node types | Hyperlink edge in `## Links` |
| "A depends on B, and here is **why**, and here is **what breaks** if ignored" | Reification page under `reifications/` |

**Promotion rule:** if you find yourself wanting to annotate a hyperlink edge with a reason or a consequence, stop — create a Reification page instead. A hyperlink that says "why" is a Reification page waiting to happen.

### Properties are defined per kind, not per mechanism

Promotion does not attach one spec-wide property set. Each reified kind defines its own schema — `requires` and `mandatory` are free to end up with different fields as the model matures. The current catalog (see [logical-layer.md §2.2](logical-layer.md#22-reified-edge-kinds-reification-pages--typed-relationships-with-properties)) happens to give all four existing kinds the same two fields, `reason` and `consequence` — that's a fact about what's been modeled so far, not a constraint of the promotion mechanism itself.

### When individual pages are warranted

The same promotion logic applies to columns and computed fields inside a Table. For the full promotion criteria — when a column warrants an Attribute page, when it warrants a Measure page, and when it stays inline — see [logical-layer.md §8](logical-layer.md#8-semantic-annotations-and-cross-domain-linking).

---

## Structural edges vs. semantic edges

This is the most important distinction in the data model. Two kinds of edges exist, and they mean fundamentally different things:

| | Structural edge (`joinedTo`) | Semantic edge (Reification page) |
|---|---|---|
| **What it expresses** | "These two tables share a key and can be joined in a query" | "This dependency exists for a business reason, and violating it causes a specific consequence" |
| **Carries meaning?** | No — it is a technical fact about data structure | Yes — it encodes business logic and domain knowledge |
| **Has Reason / Consequence?** | Never | Always |
| **Encoded as** | Hyperlink edge in `## Joins`: `Table: A joinedTo -> Table: B on A.col = B.col` | Dedicated Reification page in `reifications/`: `Reification: X requires Y` |
| **Equivalent in other tools** | SQL `JOIN`, ERD foreign key, dbt `relationships:` test, Snowflake Semantic View `relationships:` YAML | No direct equivalent — this is domain knowledge, not a database construct |
| **Agent should use it to…** | Construct the correct `JOIN` clause in SQL | Determine which filters are mandatory, which rules apply, why a dependency exists |

### The rule in one sentence

> `joinedTo` tells you **how** to join tables. A Reification page tells you **why** a dependency exists and **what breaks** if you ignore it.

---

## Edge kind reference

| Kind | Typical source → target | Notes |
|---|---|---|
| `implement` | Filter, Measure, BusinessRule → Subject, Policy · Measure, BusinessRule, Filter → VerifiedQuery | Bridge edges are owned by the logical page; no back-reference on the Subject/Policy page. To a VerifiedQuery only when the query's SQL applies the source; otherwise `relatedTo` |
| `relatedTo` | any → any | Generic symmetric cross-link |
| `calculate` | Table → Attribute, Measure | |
| `joinedTo` | Table → Table | Symmetric |
| `disambiguate` | Disambiguation → Subject | No back-reference on the Subject page |
| `apply` | BusinessRule → Table, Measure | |
| `contain` | Domain → Table, Measure, Filter, VerifiedQuery, BusinessRule, Attribute, Disambiguation | |

---

## Examples

### Hyperlink edge — owning side

Filter: WRITE_OFF_INVOICES owns a bridge edge to Subject: Write-Off and Policy: Write-off scope. The conceptual pages carry no back-reference:

```
## Links                              ← on Filter: WRITE_OFF_INVOICES
[Filter: WRITE_OFF_INVOICES implement -> Subject: Write-Off]
[Filter: WRITE_OFF_INVOICES implement -> Policy: Write-off scope]

## Links                              ← on Subject: Write-Off
(nothing about the Filter — only links to other conceptual pages)
```

### Hyperlink edge — cross-link

Measure: REVENUE owns a cross-link to Rule: exclude-reversals:

```
## Links                              ← on Measure: REVENUE
[Measure: REVENUE relatedTo -> Rule: exclude-reversals]

## Links                              ← on Rule: exclude-reversals
[Measure: REVENUE relatedTo <- Rule: exclude-reversals]
```

### Reified edge — Reification page links

Filter: ACTIVE_CUSTOMERS participates in a reified mandatory relationship with Table: ORDERS. Both sides show the owning `->` direction:

```
## Reifications                        ← on Filter: ACTIVE_CUSTOMERS
[Reification: ACTIVE_CUSTOMERS mandatory -> ORDERS]

## Reifications                        ← on Table: ORDERS
[Reification: ACTIVE_CUSTOMERS mandatory -> ORDERS]
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

### 3. `implement` never starts on a conceptual page

`implement` goes from a logical node up to a Subject or Policy. Use `relatedTo` between conceptual nodes.

**Invalid:**
```
[Subject: Write-Off implement -> Subject: Bad-Debt]
[Subject: Write-Off implement -> Filter: WRITE_OFF_INVOICES]
```

**Valid:**
```
[Subject: Write-Off relatedTo -> Subject: Bad-Debt]
[Filter: WRITE_OFF_INVOICES implement -> Subject: Write-Off]
```

---

## Visualization conventions

When drawing the graph (e.g. Mermaid diagram):

- **Rectangles** (`[Label]`) — all node types except Reification: Subject, Policy, Table, Measure, Attribute, Filter, Rule, VerifiedQuery, Disambiguation, Domain.
- **Diamonds** (`{Label}`) — Reification pages only. A Reification page is a reified edge with its own page carrying Reason + Consequence.
- **Labelled arrows** (`-->|kind|`) — hyperlink edges. No dedicated page.

```
# Reified edge (Reification page exists):
Filter: ACTIVE_CUSTOMERS --- {mandatory} --- Table: ORDERS

# Hyperlink edge (no Reification page):
Filter: WRITE_OFF_INVOICES -->|implement| Subject: Write-Off
```

The distinction matters: a diamond in the diagram means there is a dedicated page you can follow to read why the dependency exists and what breaks if it is ignored.

---

## Node/edge kind quick reference

| Node type | Header fields | Typical `## Links` edges | `## Reifications`? |
|---|---|---|---|
| `Subject` | Type, Scope | `relatedTo ->/<-` Subject, Policy · conceptual edges (`comprises <-`, `produces/consumes/governs <-`) only | No |
| `Policy` | Type, Rule modality | `relatedTo ->` Subject — no links to logical nodes | No |
| `Domain` | Type | `contain ->` Table, Measure, Filter, VerifiedQuery, BusinessRule, Attribute, Disambiguation (in the type sections) | No |
| `Table` | Type, TableKind, Domain, Source | `joinedTo ->/<-` Table (in `## Joins`) · `calculate ->` Attribute, Measure (in the `Calculated` column) | Yes |
| `Measure` | Type, Domain, Kind, Synonyms, Status | `calculate <-` Table · `implement ->` Subject, Policy, VerifiedQuery · `relatedTo ->/<-` Rule, Filter | Yes |
| `Attribute` | Type, Domain, Kind, Synonyms, access_modifier | `calculate <-` Table · `relatedTo ->/<-` Rule, Filter, Subject | Yes (overrides) |
| `Filter` | Type, Domain, Mandatory, Synonyms | `implement ->` Subject, Policy, VerifiedQuery · `relatedTo ->` Disambiguation (optional) | Yes |
| `VerifiedQuery` | Type, Domain, Onboarding question, Verified by/at, Status | `implement <-` Measure, Filter, Rule | Yes (demonstrates) |
| `BusinessRule` | Type, Domain | `apply ->` Table, Measure · `relatedTo ->/<-` Filter, Disambiguation · `implement ->` Policy, Subject, VerifiedQuery | Yes (overrides) |
| `Disambiguation` | Type, Domain | `disambiguate ->` Subject · `relatedTo <-` Filter, BusinessRule · `uses <-` Agent | No |
| `Reification` | Type, Kind, From, To | *(no edge sections — is itself a reified edge)* | N/A |

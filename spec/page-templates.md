# Page Templates

> Part of the [Knowledge Graph Specification](../SPEC.md).
> Each section below is the canonical template for one node type.
> Copy the template verbatim when creating a new page; keep all fields even if empty.

---

## General rules

- **Prose belongs only on Subject and Disambiguation pages.** All other pages use structured header fields and a predicate/definition block — no explanatory paragraphs.
- **Keep all template fields even if empty** — empty fields are valid; omitting fields breaks schema compliance.
- **Do not add non-template sections** unless the node type explicitly allows it.
- For link syntax, see [link-format.md](link-format.md).

### Header fields vs ## Links — one-to-many rule

When a node has a relationship to exactly one other node of a given type, the link may appear as a **header field** (e.g. `**Disambiguation:** [Disambiguation: X]` on a Filter page). This keeps the most important structural metadata visible at the top without requiring a section.

When a node may have **multiple** links of the same type — or when the relationship is a traversal edge rather than classification metadata — the link belongs in `## Links` as a typed edge statement.

If a header field relationship grows to multiple targets, move all instances to `## Links`.

---

## Subject

The only page type where substantive prose lives. Kept stable — business concepts change rarely.

```markdown
# Subject: <Name>

**Type:** Subject
**Scope:** global

## Business Definition
<What this concept means in the business. One paragraph. No SQL.>

## Links
- [Filter: <Name>](path) — Subject: <Name> implement -> Filter: <Name>
- [Measure: <Name>](path) — Subject: <Name> implement -> Measure: <Name>
- [Rule: <Name>](path) — Subject: <Name> implement -> Rule: <Name>
- [Subject: <Name>](path) — Subject: <Name> relatedTo -> Subject: <Name>
- [Disambiguation: <Term>](path) — Subject: <Name> disambiguate -> Disambiguation: <Term>
```

---

## Domain

The domain page is also the parent container for all type sub-folders. It is the entry point for agents and humans navigating a domain.

```markdown
# Domain: <Name>

**Type:** Domain
**Owner:** <Team>

## Tables
- [Table: <Name>](tables/<Name>)

## Measures
- [Measure: <Name>](measures/<Name>)

## Filters
**Mandatory:**
- [Filter: <Name>](filters/<Name>)

**Optional:**
- [Filter: <Name>](filters/<Name>)

## Verified Queries
**Onboarding:**
- [VerifiedQuery: <Name>](verified-queries/<Name>)

**Reference:**
- [VerifiedQuery: <Name>](verified-queries/<Name>)

## Rules
- [Rule: <Name>](rules/<Name>)

## Disambiguations
- [Disambiguation: <Term>](disambiguations/<Term>)
```

---

## Table

```markdown
# Table: <TableName>

**Type:** Table
**TableKind:** fact | dimension | bridge
**Domain:** [Domain: <Name>](../domain)
**Source:** <data source path or project reference>

## Description
<One paragraph.>

## Fields

### Physical columns
| Column | PK | Type | Description |
|--------|----|------|-------------|
| <col>  | ✓  | <type> | <desc> — mark ✓ only for primary key column(s); leave empty for all others |
| <col>  |    | <type> | <desc> |

### Semantic annotations
| Column | Kind | Synonym(s) | Notes | Calculated |
|--------|------|-----------|-------|------------|
| <col>  | dimension \| fact \| time_dimension | <syn> | <note> | [Attribute: X](path) |
| <col>  | dimension \| fact \| time_dimension | <syn> | <note> | [Measure: X](path) |
| <col>  | dimension \| fact \| time_dimension | <syn> | <note> | — |

## Relationships
- [Relationship: <Name>](../../relationships/<Name>)

## Joins
- [Table: <Name>](path) — Table: <TableName> joinedTo -> Table: <Name> on <left_col> = <right_col>

## Caveats
- <sign conventions, date format, point-in-time vs current-state, etc.>

## Links
- [Attribute: <Name>](path) — Table: <TableName> calculate -> Attribute: <Name>
- [Measure: <Name>](path) — Table: <TableName> calculate -> Measure: <Name>
```

> **Note on `## Links` on Table pages:** Only list Attributes and Measures that are **not** already in the `Calculated` column of `## Semantic annotations`. Listing them in both places is a duplicate.

---

## Measure

```markdown
# Measure: <Name>

**Type:** Measure
**Domain:** [Domain: <Name>](../domain)
**Kind:** aggregate expression | derived formula
**Synonyms:** <comma-separated>
**Status:** Active | Deprecated

## Definition
<SQL expression or formula>

## Relationships
- [Relationship: <Name>](../../relationships/<Name>)

## Links
- [Table: <Name>](path) — Table: <Name> calculate <- Measure: <Name>
- [Rule: <Name>](path) — Measure: <Name> relatedTo -> Rule: <Name>
- [Filter: <Name>](path) — Measure: <Name> relatedTo -> Filter: <Name>
- [VerifiedQuery: <Name>](path) — Measure: <Name> implement -> VerifiedQuery: <Name>
- [Subject: <Name>](path) — Subject: <Name> implement <- Measure: <Name>
```

---

## Attribute

Promoted column with semantic payload. Create a page when the column has a derived expression (e.g. CASE WHEN, COALESCE), is linked to a Rule/Subject, carries filter labels, has `access_modifier: private_access`, or has non-trivial synonyms / cross-domain relevance.

Simple columns with only a description stay inline in the parent Table's `## Fields` section.

In the Table's `## Semantic annotations`, list this Attribute in the `Calculated` column — do not also add it to Table's `## Links` (that would be a duplicate link).

```markdown
# Attribute: <Name>

**Type:** Attribute
**Domain:** [Domain: <Name>](../domain)
**Kind:** dimension | fact | time_dimension
**Synonyms:** <comma-separated>
**access_modifier:** public_access | private_access

## Expression
```sql
<derived SQL expression>
```

## Business Definition
<What this attribute means semantically. One paragraph.>

## Links
- [Table: <Name>](path) — Table: <Name> calculate <- Attribute: <Name>
- [Rule: <Name>](path) — Attribute: <Name> relatedTo -> Rule: <Name>
- [Filter: <Name>](path) — Attribute: <Name> relatedTo -> Filter: <Name>
- [Subject: <Name>](path) — Attribute: <Name> relatedTo -> Subject: <Name>
```

---

## Filter

```markdown
# Filter: <Name>

**Type:** Filter
**Domain:** [Domain: <Name>](../domain)
**Mandatory:** Yes | No
**Synonyms:** <comma-separated>
**Disambiguation:** [Disambiguation: <Term>](../disambiguations/<Term>)

## Predicate
```sql
<WHERE clause expression>
```

## Relationships
- [Relationship: <Name>](../../relationships/<Name>)

## Links
- [Subject: <Name>](path) — Subject: <Name> implement <- Filter: <Name>
- [VerifiedQuery: <Name>](path) — Filter: <Name> implement -> VerifiedQuery: <Name>
```

---

## VerifiedQuery

```markdown
# VerifiedQuery: <Name>

**Type:** VerifiedQuery
**Domain:** [Domain: <Name>](../domain)
**Onboarding question:** Yes | No
**Verified by:** <name>
**Verified at:** <YYYY-MM-DD>
**Status:** Active | Superseded

## Question
<Exact natural-language question this SQL answers.>

## Relationships
- [Relationship: <Name>](../../relationships/<Name>)

## Links
- [Measure: <Name>](path) — VerifiedQuery: <Name> implement -> Measure: <Name>
- [Filter: <Name>](path) — VerifiedQuery: <Name> relatedTo -> Filter: <Name>
- [Rule: <Name>](path) — VerifiedQuery: <Name> relatedTo -> Rule: <Name>

## SQL
```sql
<verified SQL>
```
```

---

## BusinessRule

```markdown
# Rule: <Name>

**Type:** BusinessRule
**Domain:** [Domain: <Name>](../domain)

## Definition
<Exact column names, values, filter expressions. One block. No prose introduction.>

## Consequence if Violated
<One sentence — quantify if possible.>

## Links
- [Table: <Name>](path) — Rule: <Name> apply -> Table: <Name>
- [Measure: <Name>](path) — Rule: <Name> apply -> Measure: <Name>
- [Subject: <Name>](path) — Subject: <Name> implement <- Rule: <Name>
- [Filter: <Name>](path) — Rule: <Name> relatedTo -> Filter: <Name>
- [VerifiedQuery: <Name>](path) — Rule: <Name> implement -> VerifiedQuery: <Name>
```

---

## Disambiguation

```markdown
# Disambiguation: <Term>

**Type:** Disambiguation
**Domain:** [Domain: <Name>](../domain)

## Always Ask
> "<Exact clarifying question to put to the user before any query is issued.>"

## Option A: <interpretation label>
→ [<Filter or Rule>: <Name>](path)

## Option B: <interpretation label>
→ [<Filter or Rule>: <Name>](path)

## Related
- [Subject: <Name>](path) — Subject: <Name> disambiguate <- Disambiguation: <Term>
```

---

## Relationship

Short by design. Reason and consequence are one sentence each. This page is a **reified edge** — it encodes a semantic dependency between two nodes with a stated reason and consequence.

```markdown
# Relationship: <From> <kind> <To>

**Type:** Relationship
**Kind:** requires | guards | mandatory | overrides | demonstrates
**From:** [<NodeType>: <Name>](path)
**To:** [<NodeType>: <Name>](path)

## Reason
<One sentence: why this dependency exists.>

## Consequence if Ignored
<One sentence: what goes wrong — quantify if possible.>
```

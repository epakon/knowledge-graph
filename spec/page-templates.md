# Page Templates

> Part of the [Knowledge Graph Specification](../SPEC.md).
> Each section below is the canonical template for one node type.
> Copy the template verbatim when creating a new page; keep all fields even if empty.

---

## General rules

- **Prose belongs only on Subject and Disambiguation pages.** All other pages use structured header fields and a predicate/definition block — no explanatory paragraphs.
- **Keep all template fields even if empty** — empty fields are valid; omitting fields breaks schema compliance.
- **Do not add non-template sections** unless the node type explicitly allows it.
- **Write each statement where it is true** (placement test). If it would be true of every node of a type, or of every agent reading the graph, it belongs in the spec (the node type's definition, or the reading protocol in [SPEC.md §8](../SPEC.md#8-agent-integration)), not on a page. If it is true of one node no matter which page or agent refers to it, it belongs on that node's page, and other pages link to it instead of restating it. Only what remains stays on the current page.
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

## Citations
- [<Source name>](<URL>) — <one-line description of what this source contributes>

## Links
- [Subject: <Name> implement -> Filter: <Name>](path)
- [Subject: <Name> implement -> Measure: <Name>](path)
- [Subject: <Name> implement -> Rule: <Name>](path)
- [Subject: <Name> relatedTo -> Subject: <Name>](path)
- [Subject: <Name> disambiguate -> Disambiguation: <Term>](path)
```

> `## Citations` is optional. Use it to link authoritative external sources (glossaries, regulatory definitions, data dictionaries, ontologies) that inform the business definition. Do not duplicate the external definition — link to it.

> `## Links` on a Subject page may also carry back-references from `Concept` (`Concept: <Name> comprises <- Subject: <Name>`) and from `Process` (`Process: <Name> produces/consumes/governs <- Subject: <Name>`).

---

## Concept

Abstract thematic grouping of related Subjects. Only create when the grouping carries company-specific meaning not derivable from the Subject names alone.

```markdown
# Concept: <Name>

**Type:** Concept
**Scope:** global

## Definition
<What unifies the Subjects in this concept. One paragraph. No SQL.>

## Citations
- [<Source name>](<URL>) — <one-line description of what this source contributes>

## Links
- [Concept: <Name> comprises -> Subject: <Name>](path)
```

> `## Citations` is optional.

---

## Process

Named business activity that produces, consumes, or governs data concepts. Only create when company-specific decisions are documented here that are not derivable from the Subjects it links to.

```markdown
# Process: <Name>

**Type:** Process
**Scope:** global

## Description
<What this activity does and what makes it company-specific. One paragraph. No SQL.>

## Citations
- [<Source name>](<URL>) — <one-line description of what this source contributes>

## Links
- [Process: <Name> produces -> Subject: <Name>](path)
- [Process: <Name> consumes -> Subject: <Name>](path)
- [Process: <Name> governs -> Subject: <Name>](path)
```

> Use `produces` when the process generates this Subject's data as an output, `consumes` when it needs the data as input, `governs` when it defines the rules that constrain the Subject. A single Process may use all three kinds. `## Citations` is optional.

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

## Attributes
- [Attribute: <Name>](attributes/<Name>)

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

## Reifications
- [Reification: <From> <kind> -> <To>](../../reifications/<Name>)

## Joins
- [Table: <TableName> joinedTo -> Table: <Name> on <left_col> = <right_col>](path)

## Caveats
- <sign conventions, date format, point-in-time vs current-state, etc.>

## Links
- [Table: <TableName> calculate -> Attribute: <Name>](path)
- [Table: <TableName> calculate -> Measure: <Name>](path)
```

> **Note on `## Links` on Table pages:** Only list `calculate` edges to Attributes and Measures that are **not** already in the `Calculated` column of `## Semantic annotations`. Listing them in both places is a duplicate. Snapshot pipelines read edges from `## Links`, the `Calculated` column and `## Joins`. A `Calculated` cell holds only the target label; the column header supplies `Table: <TableName> calculate ->`.

---

## Measure

Promoted computed field. For promotion criteria see [logical-layer.md §8](logical-layer.md#8-semantic-annotations-and-cross-domain-linking).

```markdown
# Measure: <Name>

**Type:** Measure
**Domain:** [Domain: <Name>](../domain)
**Kind:** aggregate expression | derived formula
**Synonyms:** <comma-separated>
**Status:** Active | Deprecated

## Definition
<SQL expression or formula>

## Reifications
- [Reification: <From> <kind> -> <To>](../../reifications/<Name>)

## Links
- [Table: <Name> calculate <- Measure: <Name>](path)
- [Measure: <Name> relatedTo -> Rule: <Name>](path)
- [Measure: <Name> relatedTo -> Filter: <Name>](path)
- [Measure: <Name> implement -> VerifiedQuery: <Name>](path)
- [Subject: <Name> implement <- Measure: <Name>](path)
```

---

## Attribute

Promoted column with semantic payload. For promotion criteria see [logical-layer.md §8](logical-layer.md#8-semantic-annotations-and-cross-domain-linking).

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

## Reifications
- [Reification: <From> <kind> -> <To>](../../reifications/<Name>)

## Links
- [Table: <Name> calculate <- Attribute: <Name>](path)
- [Attribute: <Name> relatedTo -> Rule: <Name>](path)
- [Attribute: <Name> relatedTo -> Filter: <Name>](path)
- [Attribute: <Name> relatedTo -> Subject: <Name>](path)
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

## Reifications
- [Reification: <From> <kind> -> <To>](../../reifications/<Name>)

## Links
- [Subject: <Name> implement <- Filter: <Name>](path)
- [Filter: <Name> implement -> VerifiedQuery: <Name>](path)
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

## Reifications
- [Reification: <From> <kind> -> <To>](../../reifications/<Name>)

## Links
- [Measure: <Name> implement <- VerifiedQuery: <Name>](path)
- [Filter: <Name> implement <- VerifiedQuery: <Name>](path)
- [Rule: <Name> implement <- VerifiedQuery: <Name>](path)

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

## Reifications
- [Reification: <From> <kind> -> <To>](../../reifications/<Name>)

## Links
- [Rule: <Name> apply -> Table: <Name>](path)
- [Rule: <Name> apply -> Measure: <Name>](path)
- [Subject: <Name> implement <- Rule: <Name>](path)
- [Rule: <Name> relatedTo -> Filter: <Name>](path)
- [Rule: <Name> implement -> VerifiedQuery: <Name>](path)
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

## Why It Matters
<Optional. How the interpretations differ and what mixing them does to the answer. Prose.>

## Links
- [Subject: <Name> disambiguate <- Disambiguation: <Term>](path)
```

> `## Why It Matters` is optional. `## Reifications` on Attribute and BusinessRule pages holds `overrides` edges (BusinessRule → Attribute); leave it empty otherwise.

---

## Agent

Represents one AI consumption surface (a Cortex Agent, a Claude/Cursor Skill, an MCP tool, or future equivalent), in vendor-neutral form. For the stability test, why this is a separate layer, and why it must never restate the content it reads, see [consumption-layer.md](consumption-layer.md).

```markdown
# Agent: <Name>

**Type:** Agent
**Status:** Active | Deprecated

## Purpose
<One sentence — what this agent answers questions about. No SQL, no vendor syntax.>

## Response Instructions
<Vendor-neutral rules for how the agent should format/present answers.>

## Orchestration Instructions
<Vendor-neutral rules for choosing between tools/views and routing a question.>

## Sample Questions
- <in-scope question with no VerifiedQuery yet — questions with verified SQL are linked below via `uses`>
- <question>

## Differentiation
<Only required if a `uses`-overlap review (consumption-layer.md §8) flagged this agent against another Active agent. One or two sentences: which of the reasonable-overlap axes (audience, orchestration, scope shape) distinguishes them. Omit this section entirely if no overlap was flagged.>

## Links
- [Agent: <Name> uses -> Table: <Name>](path)
- [Agent: <Name> uses -> Measure: <Name>](path)
- [Agent: <Name> uses -> Attribute: <Name>](path)
- [Agent: <Name> uses -> Filter: <Name>](path)
- [Agent: <Name> uses -> Rule: <Name>](path)
- [Agent: <Name> uses -> VerifiedQuery: <Name>](path)
- [Agent: <Name> uses -> Subject: <Name>](path)
- [Agent: <Name> uses -> Domain: <Name>](path)
- [Agent: <Name> uses -> Disambiguation: <Term>](path)
- [Agent: <Name> relatedTo -> Agent: <surviving>](path)   ← only if Status: Deprecated, per consumption-layer.md §8.4
```

> **Never restate what a `uses` target already says.** If the agent needs a synonym, a join caveat, or a calculation rule to answer correctly, that content lives on the `Attribute`, `BusinessRule`, or `Measure` page it links to — not copied onto this page. See [consumption-layer.md §5](consumption-layer.md#5-relationship-to-existing-content--never-a-second-authored-copy).

---

## Reification

Short by design. Reason and consequence are one sentence each. This page is a **reified edge** — it encodes a semantic dependency between two nodes with a stated reason and consequence.

```markdown
# Reification: <From> <kind> <To>

**Type:** Reification
**Kind:** requires | mandatory | overrides | demonstrates
**From:** [<NodeType>: <Name>](path)
**To:** [<NodeType>: <Name>](path)

## Reason
<One sentence: why this dependency exists.>

## Consequence if Ignored
<One sentence: what goes wrong — quantify if possible.>
```

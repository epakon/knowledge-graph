# Markdown Adapter

> Implementation guide for the [Knowledge Graph Specification](../../SPEC.md) on Markdown files in a git repository.

This document describes how the backend-agnostic spec maps to Markdown-specific constructs: the file format, directory hierarchy, link encoding, and version history.

The Markdown adapter is the natural backend for **auto-generated technical documentation** extracted from a codebase — table schemas, column definitions, business rules derived from code. Human-authored business knowledge (Subjects, Measures, Reifications) may live in Confluence alongside it; both backends feed the same graph DB via their respective snapshot pipelines.

---

## Backend mapping

| Spec concept | Markdown implementation |
|---|---|
| Node | `.md` file with YAML frontmatter and typed filename prefix (`measure-revenue.md`) |
| Container / folder | Directory matching the canonical hierarchy (`tables/`, `measures/`, …) |
| Page link | Standard Markdown link `[edge statement label](relative/path.md)` |
| Version comment | Git commit message in the structured version comment format |
| Semantic search | Full-text search over the file tree (`rg`, grep, or an embedded index) |
| Graph snapshot | Python script walking the directory tree and parsing frontmatter + `## Links` sections |

---

## File naming convention

Each file is named `<lowercase-type>-<kebab-name>.md`:

| Node type | Filename pattern | Example |
|---|---|---|
| `Subject` | `subject-<name>.md` | `subject-write-off.md` |
| `Domain` | `domain-<name>.md` | `domain-sales.md` |
| `Table` | `table-<name>.md` | `table-orders.md` |
| `Measure` | `measure-<name>.md` | `measure-revenue.md` |
| `Attribute` | `attribute-<name>.md` | `attribute-payment-method.md` |
| `Filter` | `filter-<name>.md` | `filter-active-customers.md` |
| `VerifiedQuery` | `verified-query-<name>.md` | `verified-query-revenue-by-region.md` |
| `BusinessRule` | `rule-<name>.md` | `rule-exclude-reversals.md` |
| `Disambiguation` | `disambiguation-<name>.md` | `disambiguation-bad-debt.md` |
| `Reification` | `reification-<from>-<kind>-<to>.md` | `reification-revenue-requires-active-customers.md` |

The page title (`# Type: Name`) inside the file is the canonical node identity — the filename is for filesystem navigation only. Agents and scripts resolve nodes by the `name` frontmatter field, not by filename.

---

## Directory structure

Mirrors the canonical hierarchy from [spec/space-structure.md](../../spec/space-structure.md) directly as a directory tree:

```
<repo-root>/knowledge-graph/
│
├── vocabulary/
│   └── subjects/
│       └── subject-<name>.md
│
└── <domain>/
    ├── domain-<name>.md          (domain index file)
    ├── tables/
    │   └── table-<name>.md
    ├── measures/
    │   └── measure-<name>.md
    ├── attributes/
    │   └── attribute-<name>.md
    ├── filters/
    │   └── filter-<name>.md
    ├── verified-queries/
    │   └── verified-query-<name>.md
    ├── rules/
    │   └── rule-<name>.md
    ├── reifications/
    │   └── reification-<from>-<kind>-<to>.md
    └── disambiguations/
        └── disambiguation-<name>.md
```

There are no separate `index.md` files per type folder — the domain index file (`domain-<name>.md`) lists all nodes for the domain. Type folders exist for filesystem organisation only.

---

## YAML frontmatter

Every node file starts with a YAML frontmatter block containing the structured header fields from the spec. This is the machine-readable counterpart of the header paragraph in the Confluence adapter.

```yaml
---
type: <NodeType>
name: <Name>
domain: <domain-name | global>
status: active | deprecated
---
```

Additional type-specific fields follow the same names as in [spec/page-templates.md](../../spec/page-templates.md):

```yaml
---
type: Measure
name: REVENUE
domain: sales
kind: aggregate expression
synonyms: [total sales, income]
status: active
---
```

```yaml
---
type: Filter
name: ACTIVE_CUSTOMERS
domain: sales
mandatory: true
synonyms: []
status: active
---
```

The `type` and `name` fields are required on every file. All other fields follow the node type's template. Keep all fields even if empty — use `null` or `[]` rather than omitting them.

---

## Link encoding

Edge statements use standard Markdown links. The link text is the self-contained edge statement label from [spec/link-format.md](../../spec/link-format.md); the href is the relative path to the target file.

```markdown
## Links
- [Measure: REVENUE relatedTo -> Rule: exclude-reversals](../rules/rule-exclude-reversals.md)
- [Subject: Revenue implement <- Measure: REVENUE](../../vocabulary/subjects/subject-revenue.md)
```

```markdown
## Reifications
- [Reification: REVENUE requires ACTIVE_CUSTOMERS](../reifications/reification-revenue-requires-active-customers.md)
```

### Important notes

- The link text is the complete edge statement — do not add trailing prose after the link.
- Use ASCII `->` and `<-` in link text, exactly as specified in [spec/link-format.md](../../spec/link-format.md).
- Use relative paths. Never use absolute paths or URLs — the repo may be cloned in different locations.
- Back-references use `<-` in the link text and point back to the source file.
- All hyperlink edges live in `## Links`; Reification page links live in `## Reifications`. Do not mix them.

---

## Version comment (git commit message)

Every commit that modifies a KG node file must set the git commit message body — git already records the commit hash, date, and author natively — to a structured summary, following the format from [spec/versioning.md](../../spec/versioning.md):

```
Summary: <one sentence>
Changed: <field or section>
Reason: <why>
Breaking: yes | no
```

For auto-generated files (codebase extraction), the commit author is the pipeline or CI job name; the `Summary` describes what triggered the regeneration.

---

## No draft support

Files are committed to git directly — there is no staging layer equivalent to a Confluence draft. To review before committing: generate the full file content in the agent session and wait for user confirmation before writing to disk and committing.

---

## Semantic search

Full-text search over the Markdown file tree replaces Confluence CQL and Glean. Use ripgrep (`rg`) patterns:

```bash
# Find a node by title
rg "^# Measure: REVENUE" knowledge-graph/

# Find all nodes of a type
rg "^type: Measure" knowledge-graph/ --include="*.md"

# Find pages that reference a concept
rg "REVENUE" knowledge-graph/ --include="*.md" -l
```

Always add a `type:` frontmatter guard when matching by node type to exclude false positives from link labels and prose.

For semantic (natural-language) search across a large corpus, index the Markdown files with an embedding pipeline (e.g. a local RAG store) and query it with natural-language phrases. The query patterns from [SPEC.md §8](../../SPEC.md#8-agent-integration) apply unchanged.

---

## Auto-generation from codebase

The Markdown adapter is designed to support auto-generated node files extracted from the codebase. Typical sources:

| Source | Generated node types |
|---|---|
| Database schema (DDL, dbt models) | `Table`, `Attribute` |
| dbt metrics / Snowflake semantic views | `Measure` |
| Application code (filter predicates) | `Filter`, `BusinessRule` |
| Test cases / verified queries | `VerifiedQuery` |

Generated files must conform to the same templates and frontmatter schema as manually authored files. The snapshot pipeline and graph DB import do not distinguish between generated and hand-written nodes — only the content matters.

**Generation rule:** auto-generated files should be committed by a CI pipeline on a separate branch or via a dedicated commit so that human-authored edits are not overwritten. Use git conflict detection to flag cases where a human has edited an auto-generated file.

---

## Common pitfalls

| Pitfall | Symptom | Fix |
|---|---|---|
| Absolute paths in links | Broken links after repo clone or move | Always use relative paths |
| Filename mismatch with `name` frontmatter | Snapshot script cannot resolve links | Derive the filename from `name` consistently; use the script's `name_to_path()` helper |
| Missing `type` field in frontmatter | Node excluded from snapshot | All files must have `type` and `name` frontmatter |
| Unicode arrows `→` in link text | Edge parser misses the edge | Use ASCII `->` and `<-` only |
| Back-reference on wrong file | Symmetric duplicate edge in graph | `<-` goes on the target file only; check before adding |
| Auto-generated file overwrites human edit | Knowledge loss | Run generation on a dedicated branch; use git diff to detect conflicts |

---

## Addons

| Addon | Purpose |
|---|---|
| [openwiki.md](addons/openwiki.md) | Generates raw technical documentation from the codebase; post-processor promotes pages into draft KG nodes |

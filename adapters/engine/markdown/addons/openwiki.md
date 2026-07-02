# OpenWiki — Codebase Documentation Generator

> **Addon** — optional capability for the Markdown engine adapter. The adapter works without it.

[OpenWiki](https://github.com/langchain-ai/openwiki) is an open-source CLI by LangChain that uses an AI agent to generate and maintain documentation for a codebase, writing plain Markdown files under `openwiki/` in the target repository. See the [LangChain blog post](https://www.langchain.com/blog/introducing-openwiki-an-open-source-agent-for-repo-documentation) for an overview of what it does and why.

As an addon to the Markdown adapter, OpenWiki acts as an **upstream content generator**: it reads source code and produces KG-compatible draft node files that a domain expert reviews and promotes.

---

## Purpose and scope

| | OpenWiki | Knowledge Graph (Markdown adapter) |
|---|---|---|
| **Input** | Source code, git history, existing docs | OpenWiki output + manual authoring |
| **Output** | KG-format draft node files under `openwiki/` | KG node files with typed edge links |
| **Content** | Architecture, workflows, domain concepts, data models, operations | Typed nodes: Table, Attribute, BusinessRule, Filter, VerifiedQuery |
| **Business meaning** | None — derived from code structure only | Full — definitions, SQL, rules, consequences, verified queries |
| **Authoring** | Automated — LLM-generated from source evidence | Mixed — generated nodes promoted and enriched by domain experts |
| **Persistence** | Refreshed on each `--update` run | Versioned KG node files committed to git |

**What OpenWiki enables for the KG:**
- Generate a first-pass KG draft layer for a codebase with no manual effort.
- Discover Table, Attribute, and BusinessRule candidates from source code before manual KG authoring begins.
- Keep technical documentation in sync with the codebase via `openwiki --update` in CI.

**What it is not:**
- A replacement for the Knowledge Graph — it carries no business definitions, mandatory filter logic, or verified queries.
- A direct source of active KG nodes — generated pages require domain expert review before promotion.

---

## Output format

OpenWiki is prompted to follow `spec/page-templates.md` and `spec/schema.yaml` directly,
producing files that are structurally identical to hand-authored KG node pages. Fields the
LLM can derive from source code (columns, types, joins, caveats, sign conventions) are
filled in. Fields that require domain expert knowledge are left as `<!-- TODO -->` placeholders.

All generated files open with `<!-- status: draft -->`.

**Fields filled by OpenWiki (from source code):**
- `TableKind`, `Source`, `## Description`
- `## Fields` — physical columns with types and descriptions
- `## Semantic annotations` — column kind (`dimension`, `fact`, `time_dimension`) and synonyms
- `## Joins` — all join conditions extracted from SQL
- `## Caveats` — sign conventions, NULL patterns, materialization, dual-row patterns, filter semantics

**Fields left as `<!-- TODO -->` for domain expert review:**
- `**Domain:**` link — requires knowing which KG domain this table belongs to
- `## Relationships` — reified edges (`reason`, `consequence`) require domain judgment
- `## Links` — Attribute and Measure links depend on what gets promoted
- `business_definition`, `consequence_if_violated` on any Rule nodes
- `Verified by`, `Verified at` on any VerifiedQuery nodes

See [openwiki-kg-example.md](../../../../examples/openwiki-kg-example.md) for a complete example of a generated draft Table node.

---

## How OpenWiki feeds the Markdown adapter

```
Codebase (source code, git history)
        │
        ▼  openwiki --init / --update  (prompted with spec/page-templates.md + spec/schema.yaml)
openwiki/*.md            ← KG-format draft node files, status: draft, TODO placeholders
        │
        ▼  domain expert review
<domain>/tables/*.md     ← TODOs filled in, status: draft → active
<domain>/rules/*.md
        │
        ▼  kg_snapshot_markdown.py
<domain>-graph.json      ← snapshot + indexes for graph DB import
```

When OpenWiki is prompted with the KG spec directly, the output is already in KG format
and can be moved to the target domain directory after review without transformation.

---

## Running OpenWiki with KG spec prompt

Use the following prompt pattern when invoking OpenWiki for a specific source file:

```bash
openwiki --update -p "Read <sql-file-path>.
Then read knowledge-graph/spec/page-templates.md and knowledge-graph/spec/schema.yaml
to understand the required output format.
Write a Knowledge Graph draft node file to <output-path> following those templates
exactly for a Table node. Use <!-- status: draft --> at the top.
Leave fields that require domain expert knowledge (business_definition,
consequence_if_violated, verified_by, edge reasons, reified relationships)
as <!-- TODO --> placeholders.
For table kind, columns, joins, and caveats extract everything you can from the SQL."
```

The agent reads the spec files directly — no duplication of template content in the prompt.

---

## Node type mapping

OpenWiki sections map naturally to KG node types. When prompted with the full spec, the
agent assigns types directly. Without a spec prompt, the post-processor uses section
location and heading patterns:

| OpenWiki section | Typical content | KG node type(s) |
|---|---|---|
| `data-models/` | Table schemas, column definitions, primary keys | `Table`, `Attribute` |
| `domain/` | Business concepts, KPIs, calculation logic | `Subject`, `Measure` |
| `workflows/` | Filter conditions, mandatory constraints, process rules | `Filter`, `BusinessRule` |
| `architecture/`, `operations/` | System overview, deployment | Navigation only — no direct node mapping |

---

## CI integration

```yaml
# .github/workflows/kg-update.yml
- name: Refresh OpenWiki docs
  run: |
    openwiki --update --print \
      -p "Read spec/page-templates.md and spec/schema.yaml and use those templates
          to write or refresh draft KG node files under openwiki/. Leave domain expert
          fields as TODO placeholders."

- name: Regenerate KG snapshot
  run: python kg_snapshot_markdown.py --root knowledge-graph/ -o <domain>-graph.json --audit

- name: Open PR if changes
  uses: peter-evans/create-pull-request@v6
  with:
    branch: kg-update
    title: "KG update: OpenWiki draft refresh + snapshot"
```

---

## What always requires manual authoring

OpenWiki captures what code does, not what it means in business terms. The following KG
fields are always outside its scope:

- `business_definition`, `consequence_if_violated` — require domain expert knowledge
- **Reified edges** (Relationship pages) — `reason` and `consequence` can only be supplied by a domain expert
- **VerifiedQuery nodes** — require human approval; OpenWiki may surface candidate SQL but cannot produce verified nodes
- **Cross-domain linking** — `Subject` nodes and `implement ->` edges require deliberate authoring decisions

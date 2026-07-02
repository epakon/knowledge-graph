# OpenWiki — Codebase Documentation Generator

> **Addon** — optional capability for the Markdown engine adapter. The adapter works without it.

[OpenWiki](https://github.com/langchain-ai/openwiki) is an open-source CLI by LangChain that uses an AI agent to generate and maintain documentation for a codebase, writing plain Markdown files under `openwiki/` in the target repository. See the [LangChain blog post](https://www.langchain.com/blog/introducing-openwiki-an-open-source-agent-for-repo-documentation) for an overview of what it does and why.

As an addon to the Markdown adapter, OpenWiki acts as an **upstream content generator**: it reads source code and produces structured Markdown pages that a post-processing step converts into KG-compatible node files.

---

## Purpose and scope

| | OpenWiki | Knowledge Graph (Markdown adapter) |
|---|---|---|
| **Input** | Source code, git history, existing docs | OpenWiki output + manual authoring |
| **Output** | Plain Markdown pages under `openwiki/` | KG node files with YAML frontmatter and typed edge links |
| **Content** | Architecture, workflows, domain concepts, data models, operations | Typed nodes: Table, Attribute, BusinessRule, Filter, VerifiedQuery |
| **Business meaning** | None — derived from code structure only | Full — definitions, SQL, rules, consequences, verified queries |
| **Authoring** | Automated — LLM-generated from source evidence | Mixed — generated nodes promoted and enriched by domain experts |
| **Persistence** | Refreshed on each `--update` run | Versioned KG node files committed to git |

**What OpenWiki enables for the KG:**
- Generate a first-pass technical documentation layer for a codebase with no manual effort.
- Discover Table, Attribute, and BusinessRule candidates from source code before manual KG authoring begins.
- Keep technical documentation in sync with the codebase via `openwiki --update` in CI.

**What it is not:**
- A replacement for the Knowledge Graph — it carries no business definitions, mandatory filter logic, or verified queries.
- A direct source of KG nodes — generated pages require post-processing and domain expert review before promotion.

---

## How OpenWiki feeds the Markdown adapter

```
Codebase (source code, git history)
        │
        ▼  openwiki --init / --update
openwiki/*.md            ← plain Markdown, no frontmatter, no edge labels
        │
        ▼  post-processor (kg_promote_openwiki.py)
<domain>/tables/*.md     ← YAML frontmatter added, edge labels added, status: draft
<domain>/rules/*.md
        │
        ▼  kg_snapshot_markdown.py
<domain>-graph.json      ← snapshot + indexes for graph DB import
```

---

## Node type mapping

OpenWiki sections map naturally to KG node types. The post-processor uses section location and heading patterns to assign `type` and extract `name`:

| OpenWiki section | Typical content | KG node type(s) |
|---|---|---|
| `data-models/` | Table schemas, column definitions, primary keys | `Table`, `Attribute` |
| `domain/` | Business concepts, KPIs, calculation logic | `Subject`, `Measure` |
| `workflows/` | Filter conditions, mandatory constraints, process rules | `Filter`, `BusinessRule` |
| `architecture/`, `operations/` | System overview, deployment | Navigation only — no direct node mapping |

The mapping is a starting point, not a guarantee. LLM-generated content may not always align cleanly with KG node type boundaries. Domain expert review is required before committing promoted nodes.

---

## Post-processing step

The post-processor (`kg_promote_openwiki.py`) transforms OpenWiki output into KG-compatible Markdown files:

1. **Parse** each OpenWiki page — extract headings, tables, code blocks.
2. **Classify** by section location and content pattern → assign candidate `type`.
3. **Extract `name`** from the primary heading or table identifier.
4. **Add YAML frontmatter** with `type`, `name`, `domain`, `status: draft`.
5. **Rewrite links** from `openwiki/`-relative paths to KG directory-relative paths.
6. **Write** to the correct KG directory (`<domain>/tables/`, `<domain>/rules/`, etc.).

All promoted files are written with `status: draft`. A domain expert reviews each file, enriches it with business meaning, and changes `status` to `active` before the snapshot pipeline includes it in the graph DB import.

---

## CI integration

```yaml
# .github/workflows/kg-update.yml
- name: Refresh OpenWiki docs
  run: openwiki --update --print

- name: Promote new OpenWiki pages to KG draft nodes
  run: python kg_promote_openwiki.py --source openwiki/ --domain <domain>

- name: Regenerate KG snapshot
  run: python kg_snapshot_markdown.py --root knowledge-graph/ -o <domain>-graph.json --audit

- name: Open PR if changes
  uses: peter-evans/create-pull-request@v6
  with:
    branch: kg-update
    title: "KG update: promoted OpenWiki docs + snapshot refresh"
```

---

## What always requires manual authoring

OpenWiki captures what code does, not what it means in business terms. The following KG fields are always outside its scope:

- `definition_sql`, `business_definition`, `consequence_if_violated` — require domain expert knowledge
- **Reified edges** (Relationship pages) — `reason` and `consequence` can only be supplied by a domain expert
- **VerifiedQuery nodes** — require human approval; OpenWiki may surface candidate SQL but cannot produce verified nodes
- **Cross-domain linking** — `Subject` nodes and `implement ->` edges require deliberate authoring decisions

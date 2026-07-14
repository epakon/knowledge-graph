# Confluence Adapter

> Implementation guide for the [Knowledge Graph Specification](../../SPEC.md) on Atlassian Confluence.

This document describes how the backend-agnostic spec maps to Confluence-specific constructs: the storage format, MCP tools, page hierarchy, and link encoding.

---

## Backend mapping

| Spec concept | Confluence implementation |
|---|---|
| Node | Confluence page with a typed title prefix (`Measure: REVENUE`) |
| Container / folder | Confluence parent page (not a Confluence "folder" — those cannot be created via API) |
| Page link | Confluence `<ac:link>` with `<ac:link-body>` for the edge statement label |
| Version comment | Confluence page version comment (set on `confluence_update_page`) |
| Semantic search | Glean MCP connector, indexing Confluence continuously |
| Graph snapshot | Knowledge Graph API crawling pages via Confluence REST API (see [graph-api.md](graph-api.md)) |

---

## Joins vs. Reification pages — naming disambiguation

See [spec/link-format.md — Structural edges vs. semantic edges](../../spec/link-format.md#structural-edges-vs-semantic-edges) for the full conceptual distinction.

In Confluence specifically:

| Concept | Confluence location |
|---|---|
| **`joinedTo` edge** (structural — SQL join fact) | `## Joins` section of a Table page, as a hyperlink: `Table: A joinedTo -> Table: B on A.col = B.col` |
| **Reification page** (semantic — business dependency with Reason + Consequence) | Dedicated page under `<domain>/reifications/`: `Reification: X requires Y` |

**Snowflake note:** the Snowflake Semantic View `relationships:` YAML key (table-level join definitions) maps to `joinedTo` hyperlink edges in `## Joins` — not to Knowledge Graph Reification pages. Same word, completely different concept.

---

## Link encoding in Confluence storage format

The edge statement label from [spec/link-format.md](../../spec/link-format.md) is stored as `<ac:link-body>` in Confluence storage-format HTML:

```xml
<!-- Owning side (source page): -->
<ac:link>
  <ri:page ri:content-title="Rule: exclude-reversals" ri:space-key="<SPACE_KEY>"/>
  <ac:link-body>Measure: REVENUE relatedTo -> Rule: exclude-reversals</ac:link-body>
</ac:link>

<!-- Back-reference (target page): -->
<ac:link>
  <ri:page ri:content-title="Measure: REVENUE" ri:space-key="<SPACE_KEY>"/>
  <ac:link-body>Measure: REVENUE relatedTo <- Rule: exclude-reversals</ac:link-body>
</ac:link>

<!-- Reification page link (same label on both From and To pages): -->
<ac:link>
  <ri:page ri:content-title="Reification: REVENUE requires ACTIVE_CUSTOMERS" ri:space-key="<SPACE_KEY>"/>
  <ac:link-body>Reification: REVENUE requires -> ACTIVE_CUSTOMERS</ac:link-body>
</ac:link>
```

### Important notes

- Use `<ac:link-body>` — **not** `<ac:plain-text-body>`. The plain-text variant renders HTML entities literally (so `&rarr;` shows as text, not as an arrow).
- Use ASCII `->` and `<-` inside `<ac:link-body>`. When you POST `->`, Confluence stores it as-is and renders it correctly. Unicode `→` may be converted to `&rarr;` on subsequent reads inside `<ac:link-body>` (which also renders correctly), but ASCII is unambiguous.
- The `ri:version-at-save` attribute is added automatically by Confluence on save — do not set it manually.

---

## MCP tools

The Confluence adapter uses the following MCP tools:

| Tool | Purpose |
|---|---|
| `confluence_search` | Find pages by title or content (CQL queries) |
| `confluence_get_page` | Fetch a page with its storage-format body and version number |
| `confluence_create_page` | Create a new page (publishes immediately — no draft support) |
| `confluence_update_page` | Update a page body and version comment |
| `confluence_get_version_history` | Retrieve version history for a page |
| `confluence_get_child_pages` | List child pages under a parent |

### CQL search patterns

```
# Find a node by title
title = "Measure: REVENUE" AND space = "<SPACE_KEY>"

# Find all nodes of a type
title ~ "Measure:" AND space = "<SPACE_KEY>"

# Find pages that reference a concept
text ~ "REVENUE" AND space = "<SPACE_KEY>"
```

> CQL `~` is "contains", not exact match. Always add a `.startswith("Measure:")` guard on results to exclude false positives.

---

## Page hierarchy setup

Parent pages must be created before child pages can be placed under them. The following parent pages are required for each domain:

| Path | Parent of |
|---|---|
| `Knowledge Graph: <Domain>` (root) | All domain containers |
| `vocabulary/` | subjects container |
| `vocabulary/subjects/` | all Subject pages |
| `Domain: <Name>` | all type containers for this domain |
| `<domain>/tables/` | Table pages |
| `<domain>/measures/` | Measure pages |
| `<domain>/attributes/` | Attribute pages |
| `<domain>/filters/` | Filter pages |
| `<domain>/verified-queries/` | VerifiedQuery pages |
| `<domain>/rules/` | Rule pages |
| `<domain>/reifications/` | Reification pages |
| `<domain>/disambiguations/` | Disambiguation pages |

Store the page IDs of these parent pages in a reference table for use by agents and scripts. Agents must never fabricate a page ID — always resolve via `confluence_search` or from the reference table.

---

## Header paragraph format

Node properties are encoded as a header paragraph at the top of each page body, using `<strong>` for field names and `<br />` as separators:

```html
<p>
  <strong>Type:</strong> Measure<br />
  <strong>Domain:</strong>
    <ac:link>
      <ri:page ri:space-key="<SPACE_KEY>" ri:content-title="Domain: Sales"/>
    </ac:link><br />
  <strong>Kind:</strong> aggregate expression<br />
  <strong>Synonyms:</strong> total sales, income<br />
  <strong>Status:</strong> Active
</p>
```

Fields that contain a link (e.g. Domain, Disambiguation) use a bare `<ac:link>` in the header — no `<ac:link-body>` needed for header fields, only for edge statement links in `## Links` and `## Reifications`.

---

## Version comment

Set on every `confluence_update_page` call as the `versionComment` parameter — this is Confluence's native version history mechanism, not a block written into the page body. Must follow the format from [spec/versioning.md](../../spec/versioning.md):

```
Summary: <one sentence>. Changed: <field>. Reason: <why>. Breaking: yes/no
```

---

## No draft support

`confluence_create_page` publishes immediately. To review before publishing: generate the full page body in the agent session and wait for user confirmation before calling the tool.

---

## Glean integration

Glean indexes Confluence continuously — no export pipeline or manual re-index is needed. The Glean MCP connector exposes the same semantic search used by Snowflake Cortex Agents.

Use natural-language phrases as search queries (see [SPEC.md §8](../../SPEC.md#8-agent-integration)):

```
"mandatory filters for <table name>"
"how is <measure> computed"
"what does <term> mean"
```

When Glean returns multiple results, prefer pages whose title prefix matches the needed node type.

---

## Snowflake Cortex integration

Snowflake Cortex Agents support Glean as a natively pre-built MCP connector — no separate Cortex Search index or export pipeline is needed. The same MCP server used by chat agents is reused inside Snowflake via the standard [Cortex Agents MCP Connector](https://docs.snowflake.com/en/user-guide/snowflake-cortex/cortex-agents-mcp-connectors) setup.

The semantic view for each domain stays **thin**: column names, join keys, and short field comments only. Business logic (mandatory filters, rules, SQL patterns) lives in the knowledge base and is retrieved at agent runtime via Glean.

---

## Common pitfalls

| Pitfall | Symptom | Fix |
|---|---|---|
| Using `<ac:plain-text-body>` for edge labels | Arrow renders as literal `&rarr;` text | Replace with `<ac:link-body>` and use ASCII `->` |
| Posting version N when Confluence is at N | 409 Conflict error | Always fetch current version immediately before updating, don't cache it |
| CQL `title ~ "Rule:"` returns non-Rule pages | Wrong pages updated | Add `.startswith("Rule:")` guard on results |
| Unicode `→` in regex doesn't match `&rarr;` | Pattern silently misses pages | Use `(?:→\|&rarr;\|&#x2192;)` or normalize with a helper first |
| BFS `get_children` only returns direct children | Deep pages missed | Recurse: fetch children of children until no more are returned |
| Fabricated page ID | `confluence_create_page` fails or creates under wrong parent | Always resolve parent IDs via `confluence_search` or from the reference table |

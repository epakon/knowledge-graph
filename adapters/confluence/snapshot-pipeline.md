# Snapshot Pipeline

> Generates a local JSON snapshot of the knowledge graph from Confluence pages and renders an interactive visualization.

---

## Overview

```
Confluence pages (source of truth)
        │
        ▼  knowledge_graph_snapshot.py
<domain>-graph.json   (local snapshot: nodes + edges)
        │
        ▼  knowledge_graph_graph.ipynb
GraphWidget  (interactive diagram)
```

Confluence pages **are** the graph. The snapshot script reads page links; the notebook visualizes the JSON.

---

## Files

| File | Purpose |
|---|---|
| `knowledge_graph_lib.py` | Confluence API client, node/edge parsing, snapshot format helpers |
| `knowledge_graph_snapshot.py` | Fetch Confluence pages → write JSON snapshot |
| `knowledge_graph_visualize.py` | JSON snapshot → GraphWidget helpers |
| `knowledge_graph_graph.ipynb` | Interactive visualization notebook |
| `<domain>-graph.json` | Local graph snapshot (regenerated from Confluence) |

---

## Step 1 — Scope with a parent page

Only descendants of a single Confluence parent page are crawled. This avoids pulling unrelated pages from the space.

Set the root page ID in the CLI or in `knowledge_graph_lib.py`:

```bash
python knowledge_graph_snapshot.py \
  --parent-page-id <root_page_id> \
  -o <domain>-graph.json \
  --stats
```

Relationship pages live under the domain's `relationships/` container and are discovered automatically as children of the domain tree.

---

## Step 2 — Parsing rules

### Nodes

A page becomes a **node** when its title matches a known node-type prefix:

- `Subject:`, `Table:`, `Measure:`, `Attribute:`, `Filter:`, `Rule:`
- `Relationship:`, `Disambiguation:`, `VerifiedQuery:`

**Excluded** (not nodes, treated as structural containers):
- Index/container pages: `subjects`, `tables`, `measures`, `attributes`, `filters`, `rules`, `relationships`, `verified-queries`, `disambiguations`, `concepts`
- Domain pages (`Domain: <Name>`) — excluded by default: Domain pages are navigational containers. Their `contain →` edges add noise without contributing to lineage or SQL-construction reasoning.

### Edges

Two sources of edges:

#### 1. Hyperlinks on pages

Every link from page A to another known node B is an edge A → B.

The edge kind is taken from the link label context:

| Link location | Kind |
|---|---|
| `## Links` section | Edge kind extracted from the `<ac:link-body>` label |
| `## Relationships` section | `relationships` (link to a Relationship page) |
| `## Joins` section | `joins` |
| Other header fields | Field name as-is |

Links to pages outside the scoped node set (e.g. Domain pages) are ignored.

#### 2. Relationship pages (reified edges)

Pages with title `Relationship: <From> <kind> <To>` define a semantic edge:

- **From**, **To**, and **Kind** parsed from the page body (`**From:**`, `**To:**`, `**Kind:**`)
- Stored as `style: "reified"` with `via` set to the Relationship page title
- The `From:` / `To:` / `Kind:` header links on Relationship pages are **not** duplicated as separate hyperlink edges

In the diagram, reified edges render as two hops through the Relationship node (hexagon shape).

---

## Snapshot JSON format

```json
{
  "meta": {
    "parent_page_id": "<root_page_id>",
    "generated_at": "2026-06-17T10:00:00Z",
    "node_count": 74,
    "edge_count": 161
  },
  "nodes": [
    {
      "id": "filter-active-customers",
      "title": "Filter: ACTIVE_CUSTOMERS",
      "label": "ACTIVE_CUSTOMERS",
      "type": "Filter",
      "page_id": "<page_id>",
      "url": "https://<instance>.atlassian.net/wiki/...",
      "color": "#FFAB00"
    }
  ],
  "edges": [
    {
      "source": "Measure: REVENUE",
      "target": "Filter: ACTIVE_CUSTOMERS",
      "kind": "requires",
      "style": "reified",
      "via": "Relationship: REVENUE requires ACTIVE_CUSTOMERS"
    },
    {
      "source": "Rule: exclude-reversals",
      "target": "Measure: REVENUE",
      "kind": "apply",
      "style": "hyperlink",
      "via": null
    }
  ]
}
```

---

## Node/edge index generation

The snapshot script can also write the node and edge indexes used for graph database migration:

```bash
python knowledge_graph_snapshot.py \
  --parent-page-id <root_page_id> \
  -o <domain>-graph.json \
  --write-node-index \
  --write-edge-index \
  --audit
```

- `--write-node-index`: writes `kg-node-index.json` (see [spec/data-model.md §3](../../spec/data-model.md#3-node-index))
- `--write-edge-index`: writes `kg-edge-index.json` (see [spec/data-model.md §4](../../spec/data-model.md#4-edge-index))
- `--audit`: runs audit rules from [spec/data-model.md §5](../../spec/data-model.md#5-audit-rules) and prints violations

---

## Step 3 — Visualization

Open `knowledge_graph_graph.ipynb` and run all cells.

### Node shapes and colors

| Type | Shape | Color |
|---|---|---|
| Subject | Rounded rectangle | `#4C9AFF` |
| Table | Rounded rectangle | `#36B37E` |
| Filter | Rounded rectangle | `#FFAB00` |
| Measure | Rounded rectangle | `#6554C0` |
| Attribute | Rounded rectangle | `#B3D4FF` |
| Rule | Rounded rectangle | `#FF5630` |
| Disambiguation | Rounded rectangle | `#00B8D9` |
| Relationship | **Hexagon** | `#8777D9` |
| VerifiedQuery | Rounded rectangle | `#97A0AF` |

Relationship nodes use a hexagon to signal that they are reified edges — pages that exist to carry a Reason and Consequence — not first-class semantic entities like the other node types.

### Edge styles

| Style | Color | Meaning |
|---|---|---|
| `hyperlink` | `#5E6C84` | Direct page link |
| `reified` | `#8777D9` | Relationship page (two hops through a hexagon node) |

### Layout options

- **Labels**: full page title (`Filter: ACTIVE_CUSTOMERS`), not the short name.
- **Relationship nodes**: hexagon shape. All other types: rounded rectangles.
- **Color legend**: rendered above the widget.
- **Data panel**: select a node and open the widget sidebar to see `type`, `title`, `url`, `page_id`.

Adjust label width:

```python
configure_widget(w, label_max_width=260)
```

Export to [yEd Live](https://www.yworks.com/yed-live/) via the widget toolbar Export button.

---

## CLI reference

### Full snapshot

```bash
python knowledge_graph_snapshot.py \
  --parent-page-id <root_page_id> \
  -o <domain>-graph.json \
  --stats
```

### Exclude VerifiedQuery nodes (smaller diagram)

```bash
python knowledge_graph_snapshot.py \
  --parent-page-id <root_page_id> \
  -o <domain>-graph.json \
  --exclude-verified-queries
```

### Mermaid export

```bash
python knowledge_graph_snapshot.py \
  --parent-page-id <root_page_id> \
  -o <domain>-graph.json \
  --mermaid <domain>-graph.mmd
```

### Full snapshot + indexes + audit

```bash
python knowledge_graph_snapshot.py \
  --parent-page-id <root_page_id> \
  -o <domain>-graph.json \
  --write-node-index \
  --write-edge-index \
  --audit
```

---

## Dependencies

```bash
pip install requests pelote yfiles_jupyter_graphs pandas
```

---

## Maintenance

When pages change (new links, new Relationship pages):

1. Re-run `knowledge_graph_snapshot.py`
2. Re-run the notebook

The diagram is only as complete as the links on the Confluence pages. Missing edges usually indicate a missing hyperlink on a page or a missing Relationship page under `relationships/`.

---

## Environment

```bash
export CONFLUENCE_USER="user@example.com"
export CONFLUENCE_TOKEN="<Atlassian personal access token>"
export CONFLUENCE_BASE_URL="https://<yourinstance>.atlassian.net"
```

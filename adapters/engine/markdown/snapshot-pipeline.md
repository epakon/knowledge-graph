# Snapshot Pipeline — Markdown

> Generates a local JSON snapshot of the knowledge graph from Markdown files and renders an interactive visualization.

---

## Overview

```
Markdown files in git repo (source of truth)
        │
        ▼  kg_snapshot_markdown.py
<domain>-graph.json   (local snapshot: nodes + edges)
        │
        ▼  knowledge_graph_graph.ipynb
GraphWidget  (interactive diagram)
```

The Markdown files **are** the graph. The snapshot script reads frontmatter and `## Links` / `## Reifications` sections; the notebook visualizes the JSON.

The snapshot JSON format is identical to the Confluence snapshot format — the same visualization notebook and graph DB import pipeline work with both backends without modification.

---

## Files

| File | Purpose |
|---|---|
| `kg_snapshot_markdown.py` | Walk Markdown file tree → write JSON snapshot |
| `knowledge_graph_visualize.py` | JSON snapshot → GraphWidget helpers (shared with Confluence adapter) |
| `knowledge_graph_graph.ipynb` | Interactive visualization notebook (shared with Confluence adapter) |
| `<domain>-graph.json` | Local graph snapshot (regenerated from Markdown files) |

---

## Step 1 — Scope with a root directory

Only files under a specified root directory are included. This lets you snapshot one domain or the full KG:

```bash
python kg_snapshot_markdown.py \
  --root knowledge-graph/sales \
  -o sales-graph.json \
  --stats
```

To snapshot the entire KG including shared vocabulary:

```bash
python kg_snapshot_markdown.py \
  --root knowledge-graph/ \
  -o full-graph.json \
  --stats
```

---

## Step 2 — Parsing rules

### Nodes

A `.md` file becomes a **node** when its `type` frontmatter field matches a known node type:

- `Subject`, `Table`, `Measure`, `Attribute`, `Filter`, `BusinessRule`
- `Reification`, `Disambiguation`, `VerifiedQuery`

**Excluded** (not nodes, treated as structural containers):
- Files with no `type` frontmatter field (index files, READMEs)
- `Domain` pages — excluded by default: Domain pages are navigational containers. Their `contain →` edges add noise without contributing to lineage or SQL-construction reasoning.

### Edges

Two sources of edges:

#### 1. Hyperlinks in `## Links` sections

Every Markdown link in a `## Links` section whose link text is a self-contained edge statement and whose target resolves to a known node file is an edge.

The edge kind is extracted from the link text:

```markdown
## Links
- [Measure: REVENUE relatedTo -> Rule: exclude-reversals](../rules/rule-exclude-reversals.md)
```

→ Edge: `Measure: REVENUE` —[relatedTo]→ `Rule: exclude-reversals`

Back-references (`<-` in link text) are **not** imported as separate edges — they are navigation artifacts, same as in the Confluence adapter.

#### 2. Reification pages (reified edges)

Files with `type: Reification` in frontmatter define a semantic edge. The `from`, `to`, and `kind` fields in frontmatter are used directly — no body parsing required for the edge itself:

```yaml
---
type: Reification
name: REVENUE requires ACTIVE_CUSTOMERS
kind: requires
from: Measure: REVENUE
to: Filter: ACTIVE_CUSTOMERS
---
```

Stored as `style: "reified"` with `via` set to the Reification file's `name` field.

---

## Snapshot JSON format

Identical to the Confluence adapter format. The `page_id` field contains the relative file path (used for traceability back to the source file):

```json
{
  "meta": {
    "root": "knowledge-graph/sales",
    "generated_at": "2026-07-02T10:00:00Z",
    "node_count": 42,
    "edge_count": 87
  },
  "nodes": [
    {
      "id": "filter-active-customers",
      "title": "Filter: ACTIVE_CUSTOMERS",
      "label": "ACTIVE_CUSTOMERS",
      "type": "Filter",
      "page_id": "knowledge-graph/sales/filters/filter-active-customers.md",
      "url": null,
      "color": "#FFAB00"
    }
  ],
  "edges": [
    {
      "source": "Measure: REVENUE",
      "target": "Filter: ACTIVE_CUSTOMERS",
      "kind": "requires",
      "style": "reified",
      "via": "Reification: REVENUE requires ACTIVE_CUSTOMERS"
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

The `url` field is `null` for Markdown files (no web URL). When combining snapshots from Confluence and Markdown backends, nodes originating from Confluence will have a URL; Markdown nodes will not.

---

## Node/edge index generation

The snapshot script also writes the node and edge indexes for graph DB import:

```bash
python kg_snapshot_markdown.py \
  --root knowledge-graph/ \
  -o full-graph.json \
  --write-node-index \
  --write-edge-index \
  --audit
```

- `--write-node-index`: writes `kg-node-index.json`
- `--write-edge-index`: writes `kg-edge-index.json`
- `--audit`: runs audit rules from [spec/logical-layer.md §5](../../spec/logical-layer.md#5-audit-rules) and prints violations

The index format is identical to the Confluence adapter's indexes. When combining both backends, merge the two node indexes and edge indexes before running graph DB import — the `page_id` field disambiguates the source.

---

## Step 3 — Visualization

Open `knowledge_graph_graph.ipynb` and run all cells. The notebook is shared with the Confluence adapter — it reads the JSON snapshot format regardless of which backend produced it.

Node shapes, colors, and edge styles are identical to the Confluence adapter. See [adapters/engine/confluence/snapshot-pipeline.md §Step 3](../confluence/snapshot-pipeline.md#step-3--visualization) for the full reference.

---

## Combining Confluence and Markdown snapshots

When both backends are active, generate separate snapshots and merge the indexes before graph DB import:

```python
import json
from pathlib import Path

def merge_indexes(paths: list[Path]) -> list[dict]:
    merged = []
    seen = set()
    for path in paths:
        for entry in json.loads(path.read_text()):
            key = (entry["label"], entry["name"])
            if key not in seen:
                seen.add(key)
                merged.append(entry)
    return merged

node_index = merge_indexes([
    Path("confluence-kg-node-index.json"),
    Path("markdown-kg-node-index.json"),
])
Path("merged-kg-node-index.json").write_text(json.dumps(node_index, indent=2))
```

Deduplication is by `(label, name)` — the same uniqueness key used by the graph DB constraints. If the same node exists in both backends (e.g. a Table page in Confluence and an auto-generated Table file in Markdown), the merge keeps the first occurrence. Resolve conflicts manually before import.

---

## CLI reference

### Full snapshot

```bash
python kg_snapshot_markdown.py \
  --root knowledge-graph/sales \
  -o sales-graph.json \
  --stats
```

### Exclude VerifiedQuery nodes (smaller diagram)

```bash
python kg_snapshot_markdown.py \
  --root knowledge-graph/sales \
  -o sales-graph.json \
  --exclude-verified-queries
```

### Mermaid export

```bash
python kg_snapshot_markdown.py \
  --root knowledge-graph/sales \
  -o sales-graph.json \
  --mermaid sales-graph.mmd
```

### Full snapshot + indexes + audit

```bash
python kg_snapshot_markdown.py \
  --root knowledge-graph/ \
  -o full-graph.json \
  --write-node-index \
  --write-edge-index \
  --audit
```

---

## Dependencies

```bash
pip install pyyaml pelote yfiles_jupyter_graphs pandas
```

No `requests` dependency — the Markdown adapter does not make HTTP calls.

---

## Maintenance

When files change (new links, new Reification files):

1. Re-run `kg_snapshot_markdown.py`
2. Re-run the notebook

For auto-generated files updated by CI: add the snapshot script to the same CI pipeline so the JSON snapshot is always up to date alongside the source files.

The diagram is only as complete as the links in the Markdown files. Missing edges usually indicate a missing `## Links` entry or a missing Reification file under `reifications/`.

---

## Environment

```bash
export KG_ROOT="knowledge-graph"   # default; override for non-standard layouts
```

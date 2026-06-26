# SAP S/4HANA — Lineage Explorer

> **Addon** — optional capability for the SAP S/4HANA connector. The connector works without it.
>
> **Draft** — conceptual design; specific column names and query patterns should be verified against a live system.
>
> **Reference implementation:** [Demo video](https://drive.google.com/file/d/1HgWrV-O6adtFgp1fYZavUJhuJH8vV-yj/view?usp=drive_link) — covering ACDOCA downstream flow. A working graph model with development classes, DDIC tables, ABAP views, CDS views, transports, and different relationships between them (`FROM`, `BELONGS`). This is the direct predecessor of the design described in this document.

The Lineage Explorer is a **read-only, ephemeral map** of the S/4HANA technical object universe — CDS views, DDIC tables, and their dependency relationships — extracted at scale without human enrichment. It is not the Knowledge Graph; it is the navigational surface that makes the S/4HANA technical landscape explorable for the S/4HANA team and feeds the discovery step of the [import procedure](../sap-s4hana-import.md).

---

## Purpose and scope

| | Lineage Explorer | Knowledge Graph (import) |
|---|---|---|
| **Coverage** | All CDS objects and DDIC tables reachable from the dependency index — thousands of objects | Seed-bounded, curated subset — tens to hundreds of objects |
| **Human enrichment** | None — object names, dependency types, metadata annotations only | Full — business definitions, rules, consequences, verified queries |
| **Source of truth** | S/4HANA system — regenerated on demand | Confluence content storage — authored and maintained |
| **Audience** | S/4HANA developers and architects navigating the technical landscape | Domain experts, analysts, and agents querying business knowledge |
| **Persistence** | Ephemeral — the snapshot is discarded or refreshed; no KG pages created | Permanent — every node is a versioned Confluence page |
| **Mutability** | Read-only — never edited, only regenerated | Authored — humans and agents read and write |

**What it enables:**
- Navigate the full CDS view hierarchy from any starting point
- Answer "what depends on `I_AccountingDocumentItem`?" or "what is between this consumption view and table `BSEG`?"
- Identify candidate objects for the KG import seed list
- Provide the dependency walk input for [Step 2 of the import procedure](sap-s4hana-import.md#step-2--discover-dependency-walk)

**What it is not:**
- A replacement for the Knowledge Graph — it carries no business meaning, no rules, no verified queries
- A permanent record — lineage explorer nodes are never promoted to KG nodes automatically; the import procedure is the controlled gate
- A data integration pipeline — it does not write to any system of record

---

## Data source: `DDLS_RIS_INDEX`

The primary source for CDS where-used relationships is `DDLS_RIS_INDEX` — the CDS-specific reverse-index table that tracks which CDS objects reference which other objects. This table covers both SQL-view-backed and HANA-native CDS activation paths, unlike the classic ABAP cross-reference (`WBCROSSGT`) which is limited to repository objects with generated SQL views.

> **Verification needed:** exact column names, available `DEPTYPE` values, and any access restrictions should be confirmed on a live system before implementation.

| Column (tentative) | Content | Used for |
|---|---|---|
| `DDLNAME` | Referencing CDS object (the dependent) | Source node |
| `REFNAME` | Referenced CDS object (the dependency target) | Target node |
| `DEPTYPE` | Dependency type (association, extend, composition, etc.) | Edge kind |
| `PACKID` or `DEVCLASS` | Package / development class | Scope filtering |

Complement with `DDDDLSRC` for object metadata (type, activation state) and `DD02L`/`DD02T` for DDIC table entries that appear as leaf nodes in the dependency graph.

---

## Object types (node kinds)

The explorer graph is extensible: new object types are new node flavors in the same structure. Initial set:

| Node kind | Source | Identity key | Notes |
|---|---|---|---|
| `cds_view` | `DDLS_RIS_INDEX` / `DDDDLSRC` | `DDLNAME` | Includes interface (`I_`), consumption (`C_`), basic, composite, and extension views |
| `ddic_table` | `DD02L` | `TABNAME` | Leaf nodes — appear as targets in CDS associations but have no outgoing CDS dependencies |
| `data_element` | `DD04L` | `ROLLNAME` | Referenced by CDS elements; carries `@EndUserText` labels and domain metadata |

**Planned additions (not in initial scope):**
- `bapi` / `function_module` — RFC-accessible interfaces; dependencies to DDIC tables
- `rap_bo` — RAP business object; composition hierarchy over CDS entities
- `odata_service` — Fiori OData service binding to a consumption CDS view

Adding a new object type does not change the graph model — it adds rows to the node list and edges to the dependency list under an existing or new `dep_type` value.

---

## Dependency types (edge kinds)

| Edge kind | `DEPTYPE` value (tentative) | Meaning |
|---|---|---|
| `association` | `ASSOC` | CDS `ASSOCIATION TO <target>` — structural join candidate |
| `extend` | `EXTEND` | `EXTEND VIEW` — adds elements to a base view |
| `composition` | `COMP` | RAP `COMPOSITION OF` — parent/child in a business object |
| `text_view` | `TEXT` | `_Text` companion view providing language-dependent labels |
| `parameter_binding` | `PARAM` | View consumes a parameter from another view |
| `uses_table` | — (from `DD02L` join) | CDS view selects from a DDIC table — derived, not from `DDLS_RIS_INDEX` |

Edge kinds are open-ended: when a new `DEPTYPE` value appears in the index, it is added here without changing the graph structure.

---

## Snapshot format

The explorer snapshot reuses the same JSON format as the Knowledge Graph snapshot pipeline ([`snapshot-pipeline.md`](../../engine/confluence/snapshot-pipeline.md)), making it directly renderable by the existing `knowledge_graph_graph.ipynb` visualization notebook with a different node color scheme.

```json
{
  "meta": {
    "source": "sap-s4hana-lineage-explorer",
    "system": "<S4HANA system ID>",
    "release": "<S4HANA release>",
    "generated_at": "<ISO timestamp>",
    "node_count": 0,
    "edge_count": 0
  },
  "nodes": [
    {
      "id": "I_AccountingDocumentItem",
      "title": "I_AccountingDocumentItem",
      "label": "I_AccountingDocumentItem",
      "type": "cds_view",
      "vdm_type": "INTERFACE",
      "lifecycle": "CLOUD_PUBLIC",
      "app_component": "FI",
      "activated": true
    }
  ],
  "edges": [
    {
      "source": "C_JournalEntryItemBrowser",
      "target": "I_AccountingDocumentItem",
      "kind": "association",
      "style": "hyperlink",
      "via": null
    }
  ]
}
```

**Differences from the KG snapshot format:**
- No `page_id` or `url` — explorer nodes have no Confluence page
- Additional node properties: `vdm_type`, `lifecycle`, `app_component`, `activated`
- Edge `kind` values are dependency types, not KG edge kinds

---

## Visualization

The explorer snapshot is rendered by the same `knowledge_graph_graph.ipynb` notebook used for the KG graph, driven by the shared `knowledge_graph_visualize.py` helpers. A separate color scheme distinguishes explorer nodes from KG nodes.

### Node colors (explorer)

| Node kind | Color | Notes |
|---|---|---|
| `cds_view` — interface (`I_`) | `#0052CC` | Stable VDM interface layer |
| `cds_view` — consumption (`C_`) | `#36B37E` | Fiori / OData consumption layer |
| `cds_view` — basic / composite | `#B3D4FF` | Internal VDM building blocks |
| `cds_view` — extension | `#FFAB00` | Customer or partner extensions |
| `ddic_table` | `#FF5630` | Physical storage — leaf nodes |
| `data_element` | `#97A0AF` | Type / label metadata — peripheral nodes |

### Scoping

The full S/4HANA CDS universe is too large to render in a single diagram. Scope by:
- **Application component** — filter nodes by `app_component` (e.g. `FI`, `FI-AR`)
- **Starting object** — render the N-hop neighborhood of a specific view or table
- **VDM layer** — show only interface views and their direct DDIC dependencies
- **Lifecycle** — exclude `SAP_INTERNAL_USE_ONLY` objects

```bash
python knowledge_graph_snapshot.py \
  --source sap-s4hana-lineage \
  --app-component FI \
  --max-hops 3 \
  --exclude-lifecycle SAP_INTERNAL_USE_ONLY \
  -o fi-lineage.json
```

> CLI parameters are indicative — implementation details depend on the extraction script design.

---

## Relationship to the import procedure

The Lineage Explorer and the KG import procedure share the dependency index as a common technical foundation but serve different purposes:

```
DDLS_RIS_INDEX + DD* tables
        │
        ├──► Lineage Explorer snapshot
        │    Full dependency graph — read-only, ephemeral, no enrichment
        │    Used by: S/4HANA developers, architects
        │
        └──► Import procedure Step 2 — dependency walk
             Seed-bounded traversal — enriched, curated, imported to KG
             Used by: domain experts authoring KG nodes
```

**Promotion path:** an object visible in the Lineage Explorer can be promoted to the KG import seed list by a domain expert. That is the only gate between the two — no automatic promotion. The explorer makes candidates visible; a human decides what deserves business-meaning enrichment.

---

## Extraction procedure

1. **Connect** to S/4HANA via RFC or direct HANA SQL (read-only, no ABAP changes)
2. **Read** `DDLS_RIS_INDEX` — optionally filtered by package/component to limit scope
3. **Enrich nodes** with metadata from `DDDDLSRC` (activation state, DDL type) and `DD02L`/`DD02T` (table descriptions)
4. **Derive** `uses_table` edges by joining CDS element references to `DD02L` entries
5. **Write** snapshot JSON to disk
6. **Visualize** using the existing notebook, scoped as needed

The extraction is stateless — no incremental logic, no version tracking. Re-run to refresh. The snapshot is not stored in the content storage or the graph DB; it is a local working artifact.

---

## What is not covered

- Business meaning of any object — no `@EndUserText` labels are surfaced as KG definitions
- Custom `Z*`/`Y*` extensions — included structurally if present in `DDLS_RIS_INDEX`, but with no semantic annotation
- BW InfoProviders and Queries — separate dependency infrastructure, not covered by `DDLS_RIS_INDEX`
- ABAP class and function module dependencies — separate cross-reference infrastructure (`WBCROSSGT` / `SCNG`)

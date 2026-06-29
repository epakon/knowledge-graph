# SAP HANA — Lineage Explorer

> **Addon** — optional capability for the SAP HANA connector. The connector works without it.
>
> **Draft** — this document is work in progress and has not been formally reviewed.

The Lineage Explorer is a **read-only, ephemeral map** of the HANA object dependency universe — Calculation Views, physical tables, synonyms, SQL views, and their dependency relationships — extracted at scale without human enrichment. It is not the Knowledge Graph; it is the navigational surface that makes the HANA technical landscape explorable for data engineers and feeds the discovery step of the [import procedure](../sap-hana-import.md).

---

## Purpose and scope

| | Lineage Explorer | Knowledge Graph (import) |
|---|---|---|
| **Coverage** | All objects reachable from a starting point via `SYS.OBJECT_DEPENDENCIES` — potentially thousands | Seed-bounded, curated subset — tens to hundreds of objects |
| **Human enrichment** | None — object names, dependency types, schema metadata only | Full — business definitions, rules, consequences, verified queries |
| **Source of truth** | HANA system catalog — regenerated on demand | Confluence content storage — authored and maintained |
| **Audience** | Data engineers navigating the technical landscape | Domain experts, analysts, and agents querying business knowledge |
| **Persistence** | Ephemeral — snapshot discarded or refreshed; no KG pages created | Permanent — every node is a versioned Confluence page |
| **Mutability** | Read-only — never edited, only regenerated | Authored — humans and agents read and write |

**What it enables:**
- Navigate the full Calculation View hierarchy from any starting point, upstream or downstream.
- Answer "what does this Calculation View depend on?" or "what consumes this table?"
- Identify candidate objects for the KG import seed list.
- Provide the dependency walk input for [Step 2 of the import procedure](../sap-hana-import.md#step-2--discover-dependency-walk).

**What it is not:**
- A replacement for the Knowledge Graph — it carries no business meaning, no rules, no verified queries.
- A permanent record — lineage explorer nodes are never automatically promoted to KG nodes.
- A data integration pipeline — it does not write to any system of record.

---

## Data source: `SYS.OBJECT_DEPENDENCIES`

Unlike the S/4HANA Lineage Explorer (which uses `DDLS_RIS_INDEX`), the HANA Lineage Explorer uses the native HANA catalog dependency table.

### Primary source: `SYS.OBJECT_DEPENDENCIES`

```sql
SELECT
    BASE_SCHEMA_NAME,
    BASE_OBJECT_NAME,
    BASE_OBJECT_TYPE,
    DEPENDENT_SCHEMA_NAME,
    DEPENDENT_OBJECT_NAME,
    DEPENDENT_OBJECT_TYPE,
    DEPENDENCY_TYPE
FROM SYS.OBJECT_DEPENDENCIES
WHERE DEPENDENCY_TYPE = 1
```

| Column | Content | Used for |
|---|---|---|
| `BASE_SCHEMA_NAME` | Schema of the object that is depended upon | Source node schema |
| `BASE_OBJECT_NAME` | Name of the object that is depended upon | Source node identity |
| `BASE_OBJECT_TYPE` | Type: `TABLE`, `VIEW`, `SYNONYM`, `FUNCTION`, `PROCEDURE` | Source node kind |
| `DEPENDENT_SCHEMA_NAME` | Schema of the dependent object | Target node schema |
| `DEPENDENT_OBJECT_NAME` | Name of the dependent object | Target node identity |
| `DEPENDENT_OBJECT_TYPE` | Type of the dependent object | Target node kind |
| `DEPENDENCY_TYPE` | `1` = standard SQL dependency | Filter condition |

**Direction convention:** in `SYS.OBJECT_DEPENDENCIES`, `BASE` is the object that is *used by* `DEPENDENT`. In lineage terms: `DEPENDENT` depends on `BASE`. When walking upstream (toward source data), follow `DEPENDENT → BASE`. When walking downstream (toward consumers), follow `BASE → DEPENDENT`.

### Optional supplement: `_SYS_BI.BIMC_SYNONYM_MAPPING`

On HANA systems with XS Classic or XSA runtime, Calculation Views in `_SYS_BIC` may be exposed via synonyms that do not appear in `SYS.OBJECT_DEPENDENCIES`. This mapping table resolves synonym names to their `_SYS_BIC` qualified names:

```sql
SELECT SCHEMA_NAME, SYNONYM_NAME, QUALIFIED_NAME
FROM _SYS_BI.BIMC_SYNONYM_MAPPING
```

When available, add synthetic `SYNONYM → VIEW` edges to the dependency graph by unioning this table with `SYS.OBJECT_DEPENDENCIES`:

```sql
SELECT * FROM SYS.OBJECT_DEPENDENCIES
UNION
SELECT
    SCHEMA_NAME,       -- synonym schema (base)
    SYNONYM_NAME,      -- synonym name (base)
    'SYNONYM',
    SCHEMA_NAME,       -- same schema (dependent)
    QUALIFIED_NAME,    -- _SYS_BIC qualified path (dependent)
    'VIEW',
    1
FROM _SYS_BI.BIMC_SYNONYM_MAPPING
```

On HDI-only systems where `_SYS_BI.BIMC_SYNONYM_MAPPING` is not present, use `SYS.SYNONYMS` instead for synonym resolution.

---

## Object types (node kinds)

| Node kind | Source | Identity key | Notes |
|---|---|---|---|
| `calculation_view` | `SYS.VIEWS WHERE SCHEMA_NAME = '_SYS_BIC'` | `_SYS_BIC/<VIEW_NAME>` | All CV types: `CVQ_`, `CVM_`, `CVD_`, and any implementation-specific prefixes |
| `sql_view` | `SYS.VIEWS` in Data Mart / Semantic or DWH schemas | `SCHEMA_NAME.VIEW_NAME` | Standard SQL views built on tables |
| `table` | `SYS.TABLES` | `SCHEMA_NAME.TABLE_NAME` | Physical column-store tables; leaf nodes upstream |
| `synonym` | `SYS.SYNONYMS` | `SCHEMA_NAME.SYNONYM_NAME` | Cross-schema bridge; transparent in most lineage views |
| `function` | `SYS.FUNCTIONS` | `SCHEMA_NAME.FUNCTION_NAME` | SQLScript functions used in view logic |
| `procedure` | `SYS.PROCEDURES` | `SCHEMA_NAME.PROCEDURE_NAME` | Peripheral — shown when referenced but not walked further |

Adding a new object type does not change the graph model — it adds rows to the node list under an existing or new `dep_type` value.

---

## Dependency types (edge kinds)

| Edge kind | Source signal | Meaning |
|---|---|---|
| `uses_view` | `VIEW → VIEW` in `OBJECT_DEPENDENCIES` | A view or CV consumes another view/CV |
| `uses_table` | `TABLE → VIEW` in `OBJECT_DEPENDENCIES` | A view or CV selects from a physical table |
| `uses_synonym` | `SYNONYM → VIEW` resolved via synonym tables | A view is exposed via a synonym |
| `uses_function` | `FUNCTION → VIEW` in `OBJECT_DEPENDENCIES` | A view uses a SQLScript function |
| `synonym_of` | `SYS.SYNONYMS` | A synonym points to a view or table |

Edge kinds are open-ended: if a new dependency type appears in `OBJECT_DEPENDENCIES`, add it here without changing the graph structure.

---

## Walk modes

### Upstream walk (toward source data)

Starting from a Calculation View, follow `DEPENDENT → BASE` to find what it depends on. Reveals the data sources feeding a report or analysis.

Filter pattern:
```sql
WHERE dependent_schema_name = :start_schema
  AND dependent_object_name = :start_object
  AND dependency_type = 1
```

Apply any implementation-specific exclusions (e.g. system utility tables known to produce noise) as additional `WHERE` conditions, documented in the connector configuration.

### Downstream walk (toward consumers)

Starting from a table or base view, follow `BASE → DEPENDENT` to find what depends on it. Reveals the impact of a change to that object.

Filter pattern:
```sql
WHERE base_schema_name = :start_schema
  AND base_object_name = :start_object
  AND dependency_type = 1
```

Apply exclusions for system-generated schemas and helper objects as needed.

### Multi-hop walk

For depth > 1, iteratively join the dependency table against the previous level's results. Walk depth is configurable; default is **3 hops**.

---

## Snapshot format

The explorer snapshot reuses the same JSON format as the Knowledge Graph snapshot pipeline ([`snapshot-pipeline.md`](../../../engine/confluence/snapshot-pipeline.md)), making it directly renderable by the existing visualization notebook with a different node color scheme.

```json
{
  "meta": {
    "source": "sap-hana-lineage-explorer",
    "system": "<HANA system ID>",
    "schema": "<starting schema>",
    "object": "<starting object>",
    "direction": "upstream | downstream",
    "max_hops": 3,
    "generated_at": "<ISO timestamp>",
    "node_count": 0,
    "edge_count": 0
  },
  "nodes": [
    {
      "id": "<SCHEMA_NAME>/<PACKAGE>/<VIEW_NAME>",
      "title": "<VIEW_NAME>",
      "label": "<VIEW_NAME>",
      "type": "calculation_view",
      "cv_prefix": "CVQ",
      "schema": "<SCHEMA_NAME>",
      "package": "<PACKAGE>",
      "level": 0
    }
  ],
  "edges": [
    {
      "source": "<SCHEMA_NAME>/<PACKAGE>/<VIEW_NAME>",
      "target": "<SCHEMA_NAME>.<TABLE_NAME>",
      "kind": "uses_table",
      "style": "hyperlink",
      "via": null
    }
  ]
}
```

**Differences from the KG snapshot format:**
- No `page_id` or `url` — explorer nodes have no Confluence page.
- Additional node properties: `cv_prefix`, `schema`, `package`, `level` (hop distance from starting object).
- Edge `kind` values are dependency types, not KG edge kinds.

---

## Visualization

The explorer snapshot is rendered by the same visualization notebook used for the KG graph, driven by the shared graph visualization helpers. A separate color scheme distinguishes HANA explorer nodes from KG nodes and from S/4HANA lineage explorer nodes.

### Node colors (HANA lineage explorer)

| Node kind | Color | Notes |
|---|---|---|
| `calculation_view` — query/fact (e.g. `CVQ_`) | `#0052CC` | Top-level consumable |
| `calculation_view` — model (e.g. `CVM_`) | `#36B37E` | Intermediate model layer |
| `calculation_view` — dimension (e.g. `CVD_`) | `#B3D4FF` | Dimension / lookup |
| `calculation_view` — other / unrecognised prefix | `#00B8D9` | Update color mapping when implementation uses different prefixes |
| `sql_view` | `#79E2F2` | Standard SQL views |
| `table` | `#FF5630` | Physical storage — leaf nodes |
| `synonym` | `#FFAB00` | Cross-schema bridge |
| `function` | `#97A0AF` | SQLScript function |

### Scoping

The full HANA dependency graph is too large to render in a single diagram. Scope by:
- **Starting object** — render the N-hop neighborhood of a specific Calculation View or table.
- **Schema** — include only nodes in a specified schema.
- **Direction** — upstream (source data) or downstream (consumers).
- **Object type filter** — exclude synonyms (transparent) or functions (peripheral) to reduce noise.

---

## Relationship to the import procedure

The Lineage Explorer and the KG import procedure share `SYS.OBJECT_DEPENDENCIES` as a common technical foundation but serve different purposes:

```
SYS.OBJECT_DEPENDENCIES (+ optional synonym mapping)
        │
        ├──► Lineage Explorer snapshot
        │    Full dependency graph — read-only, ephemeral, no enrichment
        │    Used by: data engineers, architects
        │
        └──► Import procedure Step 2 — dependency walk
             Seed-bounded traversal — enriched, curated, imported to KG
             Used by: domain experts authoring KG nodes
```

**Promotion path:** an object visible in the Lineage Explorer can be promoted to the KG import seed list by a domain expert. That is the only gate between the two — no automatic promotion. The explorer makes candidates visible; a human decides what deserves business-meaning enrichment.

---

## Extraction procedure

1. **Connect** to HANA (read-only user; no DDL or DML permissions required).
2. **Build base dependency table** — execute the base `SYS.OBJECT_DEPENDENCIES` query; union with synonym mapping if available.
3. **Seed filter** — apply starting schema + object + direction filter to get level-1 dependencies.
4. **Multi-hop walk** — iteratively join the base table against the current level's results for each additional hop.
5. **Node enrichment** — join with `SYS.VIEWS` and `SYS.TABLES` for `IS_VALID`, `COMMENTS`, `VIEW_TYPE`.
6. **Node classification** — assign `type` and `cv_prefix` from object name patterns; assign `level` from hop count.
7. **Write snapshot JSON** to disk.
8. **Visualize** using the existing notebook, scoped as needed.

The extraction is stateless — no incremental logic, no version tracking. Re-run to refresh. The snapshot is not stored in content storage or the graph DB; it is a local working artifact.

---

## What is not covered

- Business meaning of any object — no comments are surfaced as KG definitions.
- HDI container internal metadata (deployment history, container configuration) — not in `SYS.*` tables.
- Remote sources and virtual tables — objects accessed via `SYS.REMOTE_SOURCES` may have dependencies that do not appear in `SYS.OBJECT_DEPENDENCIES`.
- Landing and Staging schemas — excluded by default as transient or unprocessed data; configure per deployment which schemas belong to these layers.
- System-internal schemas (`_SYS_*`, `SYS`, runtime schemas) — always excluded.

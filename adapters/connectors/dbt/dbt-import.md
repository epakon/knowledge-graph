# dbt — Import Procedure

This document describes the import procedure for [`dbt-technical.md`](dbt-technical.md). It covers how a dbt project's compiled metadata is read, filtered, transformed into Knowledge Graph nodes and edges, validated, and written to the content storage and graph DB.

The extraction source is `manifest.json`, produced by `dbt compile` or `dbt docs generate`. The goal is to extract the semantically meaningful subset of the project — models with business purpose, semantic measures and dimensions, column-level business rules, and downstream exposures — not every intermediate transformation artifact.

---

## Conceptual architecture

```
dbt project (target/ directory)
    │
    │  1. Read   — parse manifest.json (and catalog.json if present)
    │  2. Filter — exclude staging, private, and internal nodes
    │  3. Route  — assign KG node type per object category
    │  4. Map    — apply field mapping tables from dbt-technical.md
    │
    ▼
Extracted records (typed JSON)
    │
    │  5. Validate — schema, uniqueness, required fields
    │  6. Import   — content storage pages + graph DB nodes/edges
    │
    ▼
Knowledge Graph (Confluence + Graph DB)
```

Steps 1–4 are dbt-specific. Steps 5–6 follow the generic import procedure defined in [`connectors.md`](../connectors.md) §Section 6.

---

## Step 1 — Read: parse manifest.json

`manifest.json` is produced by running either:

```
dbt compile
# or, for a full catalog with column types from the warehouse:
dbt docs generate
```

Both commands write `manifest.json` to `<project_root>/target/manifest.json`. `catalog.json` is written to the same directory only by `dbt docs generate`.

### manifest.json top-level keys used

| Key | Content | Used for |
|---|---|---|
| `metadata.generated_at` | ISO timestamp of the compile run | Incremental detection: compare against last import timestamp |
| `metadata.dbt_schema_version` | dbt version | Log for compatibility checks |
| `nodes` | All model, test, snapshot, seed nodes keyed by unique ID | Primary extraction surface |
| `sources` | All source table definitions | Source `Table` nodes |
| `exposures` | All exposure definitions | `VerifiedQuery` nodes |
| `metrics` | All metric definitions | `Measure` nodes (fallback when `semantic_models` absent) |
| `semantic_models` | All semantic model definitions (dbt Core 1.6+) | `Measure`, `Attribute`, `Filter` nodes |
| `parent_map` | Reverse dependency map (child → parents) | Edge construction cross-reference |
| `child_map` | Forward dependency map (parent → children) | Edge construction |

### catalog.json usage (optional)

If `catalog.json` is present, read `catalog.nodes` to supplement missing `data_type` values for columns where the YAML `data_type:` field is absent. Match on node `unique_id`. Column types from the catalog take lower priority than YAML-declared types — they are a fallback only.

---

## Step 2 — Filter: exclude non-KG nodes

Not all dbt nodes carry independent business meaning. The following are excluded before node type routing:

| Exclusion rule | Rationale |
|---|---|
| `node_type: test` | Test nodes are not standalone entities; their content is extracted as `BusinessRule` properties on the parent model |
| `node_type: model` with `config.materialized: ephemeral` | Ephemeral models are never materialized in the warehouse and have no addressable source |
| Models in a `staging/` folder or with `stg_` name prefix | Staging is an intermediate transformation layer with no independent business meaning |
| Models with `config.access: private` | Private models are implementation details not intended for external consumption |
| Models where `deprecation_date` is set and is in the past | Extracted as `status: Deprecated`, not excluded — they may still be referenced by other nodes |
| Sources without a `description` and without any columns defined | Empty source stubs that have not been documented yet; log as candidates for authoring |

**Staging exclusion override:** If a project does not follow `stg_` naming or `staging/` folder conventions, staging exclusion can be controlled by setting `meta.kg_exclude: true` on individual models or by configuring the importer with an explicit exclusion tag (e.g. `tags: [staging]`).

---

## Step 3 — Route: assign KG node type

After filtering, each remaining node is routed to a KG node type based on `node_type`, name prefix, folder, and `meta` fields.

### Model node routing

| Condition | KG node type | `table_kind` |
|---|---|---|
| Name prefix `fct_` or `report_` or folder `reporting/` or `fact/` | `Table` | `fact` |
| Name prefix `dim_` or folder `dimension/` | `Table` | `dimension` |
| Name prefix `hub_` or `sat_` or folder `datavault/hubs/` or `datavault/sats/` | `Table` | `dimension` |
| Name prefix `link_` or `bt_` or folder `datavault/links/` or `businessvault/bridge_tables/` | `Table` | `bridge` |
| Name prefix `snf_` or folder `ingest/` | `Table` | `fact` (default; override via `meta.table_kind`) |
| `meta.table_kind` set | `Table` | Value of `meta.table_kind` — overrides all prefix/folder heuristics |
| No recognized prefix or folder, no `meta.table_kind` | `Table` | `REQUIRES MANUAL AUTHORING` — flag in import log |

Version suffixes (`_v0`, `_v1`, `_v2`, etc.) are stripped before routing and before name construction. The versioned model with the highest version number is imported as the canonical node. Older versions are imported as `status: Deprecated` if they still exist in the manifest.

### Source node routing

All `sources` nodes → `Table`. `table_kind` from `meta.table_kind` if set; otherwise `dimension` as default with a note in the import log recommending manual review.

### Seed node routing

All `seeds` nodes → `Table` with `table_kind: dimension`.

### Snapshot node routing

All `snapshots` nodes → `Table` with `table_kind: dimension`.

### Semantic model routing

For each `semantic_models` entry, promotion to a KG node is conditional — most columns stay inline in the parent Table's `## Semantic annotations`. Apply the following gates:

**measures[]** → `Measure` only when at least one of:
- `type` is `ratio` or `derived`
- `label` is present
- `measures[].filter` is present (implies a BusinessRule link)

Otherwise record the measure inline (column name, kind, aggregation) in the parent Table's `## Semantic annotations` with no separate page.

**dimensions[]** → `Attribute` only when at least one of:
- `expr` is a non-trivial derived expression (anything other than a direct column reference)
- `label` is present and differs meaningfully from the column name
- The same dimension name appears in multiple semantic models in the project

Otherwise record the dimension inline in the parent Table's `## Semantic annotations`.

**entities[]** → `Attribute` only when the entity name appears in multiple semantic models (cross-domain join key). Single-model entities stay inline.

**measures[].filter** → `Filter` (if the filter expression is non-trivial — not just `WHERE 1=1` or equivalent)

**where_filter** at the semantic model level → `Filter` (mandatory, applies to all measures on this model)

### Metrics routing (fallback)

Used only when `semantic_models` is absent or empty. Apply the same promotion gate as for measures: each `metrics[]` entry with `type: ratio` or `derived`, or with a `label`, → `Measure`. Plain sum/count/average metrics without a label stay inline in the parent Table's `## Semantic annotations`.

### Exposure routing

All `exposures` nodes → `VerifiedQuery`.

### Test-to-BusinessRule extraction

Tests are not imported as standalone nodes. Instead, for each model node, its `data_tests` are collected and converted to `BusinessRule` nodes. One `BusinessRule` node per non-trivial test:

| Test type | Import condition |
|---|---|
| `not_null` | Import only when applied to a column that is a foreign key or natural business key (i.e. the model also has a `primary_key` or `foreign_key` constraint on that column) |
| `accepted_values` | Always import — encodes a domain constraint |
| `foreign_key` / `relationships` | Always import — encodes referential integrity |
| Custom SQL test (`where`, `condition`, `predicate`) | Always import — explicit SQL predicate |
| `unique` | Do not import — structural constraint, not a business rule |
| `not_null` on non-key columns | Do not import — data quality check, not a business rule |
| `dbt_constraints.primary_key` | Do not import — structural constraint |
| `dvs.count_rows`, `dvs.distinct_ratio` | Do not import — statistical quality assertions, not business rules |

---

## Step 4 — Map: apply field mappings

Apply the field mapping tables from [`dbt-technical.md`](dbt-technical.md) §Section 3 to produce typed records. Key transformations:

### Name construction

- Strip version suffix: `fct_order_item_v5` → `fct_order_item`
- Strip well-known prefix: `fct_` → ``, `dim_` → ``, `hub_` → ``, etc.
- Replace underscores with spaces and title-case: `order_item` → `Order Item`
- Full example: `fct_order_item_v5` → `Order Item`

For `BusinessRule` nodes derived from tests, the name is constructed as:
`<Model Name> — <Column Name> <test label>`
Example: `Dim Customer — Payment Method accepted values`

For `Filter` nodes derived from semantic model measure filters:
`<Measure Name> scope`
Example: `Completed Orders Revenue scope`

### Status mapping

| dbt signal | KG `status` |
|---|---|
| `deprecation_date` present and in the past | `Deprecated` |
| `config.access: private` | Excluded (not imported) |
| All other models | `Active` |
| Measure/Attribute whose parent model is `Deprecated` | `Deprecated` |

### `definition_sql` construction for Measure nodes

| `semantic_models[].measures[].agg` | Template |
|---|---|
| `sum` | `SUM(<expr>)` |
| `count` | `COUNT(<expr>)` |
| `average` | `AVG(<expr>)` |
| `count_distinct` | `COUNT(DISTINCT <expr>)` |
| `max` | `MAX(<expr>)` |
| `min` | `MIN(<expr>)` |
| `ratio` (type: ratio) | `SUM(<numerator_expr>) / NULLIF(SUM(<denominator_expr>), 0)` |
| `derived` (type: derived) | Render `metrics[].expression` substituting referenced metric names |

If `expr` is absent (measure applies the aggregate to the model's primary entity), use `1` as the expression for `count`.

---

## Step 5 — Validate

Before writing to the content storage, each extracted record is validated:

1. **Required fields** — all fields marked required in [`spec/schema.yaml`](../../../spec/schema.yaml) are present, or explicitly set to `REQUIRES MANUAL AUTHORING` for fields that cannot be derived.
2. **Enum values** — `table_kind`, `kind`, `status`, `access_modifier` are within allowed sets from the schema.
3. **Name uniqueness** — checked against the KG node index (graph DB `(label, name)` constraint, or JSON node index if graph DB is unavailable). Conflicts are logged and skipped; they are not silently overwritten.
4. **Staging leak check** — verify no node with a `stg_` prefix or in a `staging/` path passed the filter step. Fail the import batch if found.
5. **Manifest freshness** — `metadata.generated_at` must be within 24 hours of the import run, or the importer must be explicitly invoked with `--allow-stale` to proceed with an older manifest. Stale manifests risk importing deleted nodes.
6. **Version suffix consistency** — if multiple versions of a model exist (e.g. `fct_order_item_v3` and `fct_order_item_v5`), only the highest version is imported as `Active`. Lower versions are imported as `Deprecated` only if they have explicit downstream dependents still active in the manifest.

Validation failures are written to an import log per node. A configurable threshold (default: 0 errors) determines whether the entire import is aborted or whether failing nodes are skipped and the rest proceed.

---

## Step 6 — Import

Follows the generic procedure in [`connectors.md`](../connectors.md) §Section 6.

### Node creation

- Content storage page created at the path defined by `spec/space-structure.md` for the node's type and domain.
- Page title: `<NodeType>: <name>` per the prefix convention in `spec/schema.yaml`.
- Template sections populated from the mapped record; empty-but-required fields set to `REQUIRES MANUAL AUTHORING`.

### Edge creation

Link statements are written per [`spec/link-format.md`](../../../spec/link-format.md):

| Edge | Owning page | Statement |
|---|---|---|
| `Domain contain -> Table` | Domain page | `[Domain: <name> contain -> Table: <name>]` |
| `Table joinedTo -> Table` | Model page (higher in lineage) | `[Table: <A> joinedTo -> Table: <B>]` |
| `Table calculate -> Measure` | Table page | `[Table: <name> calculate -> Measure: <name>]` |
| `Table calculate -> Attribute` | Table page | `[Table: <name> calculate -> Attribute: <name>]` |
| `BusinessRule apply -> Table` | BusinessRule page | `[Rule: <name> apply -> Table: <name>]` |
| `VerifiedQuery relatedTo -> Table` | VerifiedQuery page | `[VerifiedQuery: <name> relatedTo -> Table: <name>]` |

Back-references are written on the target page as `<-` labels following the link format spec.

### Reified edge stubs

For each detected `mandatory` or `requires` edge, a stub Reification page is created:

```
Title: Reification: Filter: <filter name> mandatory Table: <table name>

## Reason
<filter expression from semantic model, or "REQUIRES MANUAL AUTHORING">

## Consequence if Ignored
REQUIRES MANUAL AUTHORING
```

The stub is flagged in the import log as requiring domain expert review before it is considered verified.

### Duplicate handling

| Situation | Action |
|---|---|
| Node with same `(label, name)` exists, `status: Active` | Skip creation; log as already present; update description and synonyms if changed |
| Node with same `(label, name)` exists, `status: Deprecated` | Create new `Active` node; add `relatedTo` edge from new to old; log |
| Node unique ID matches but name changed (model renamed) | Create new node for new name; deprecate old node; log as rename |

### Post-import audit

Run the audit rules from [`spec/schema.yaml`](../../../spec/schema.yaml) before committing:
- No duplicate `(label, name)` pairs in the node index
- No duplicate `(source, target, relationship_type)` in the edge index
- Reified beats hyperlink: remove any `RELATED_TO` edge where a reified edge already covers the same pair
- Back-references excluded from the edge index

### Version comment format

Every programmatically created or updated page must include a structured version comment:

```
v1 | <YYYY-MM-DD> | dbt-import
Summary: Imported from dbt project <project_name> — <dbt_node_unique_id>
Changed: Initial import
Reason: Extracted from manifest.json generated at <metadata.generated_at>
Breaking: no
```

For updates (incremental re-import of a changed node):

```
v<n+1> | <YYYY-MM-DD> | dbt-import
Summary: Updated from dbt project <project_name> — <dbt_node_unique_id>
Changed: <list of changed fields>
Reason: Node checksum changed in manifest.json generated at <metadata.generated_at>
Breaking: <yes if definition_sql, predicate_sql, table_kind, or name changed; no otherwise>
```

---

## Incremental extraction

On subsequent runs, the importer compares the current `manifest.json` against the stored state from the previous run:

1. **New nodes** — `unique_id` present in current manifest but not in importer state → create.
2. **Changed nodes** — `unique_id` present in both, `checksum.checksum` differs → update.
3. **Deleted nodes** — `unique_id` in importer state but not in current manifest → set `status: Deprecated` on the KG page; do not delete.
4. **Unchanged nodes** — `checksum.checksum` identical → skip entirely.

The importer state is a JSON file maintained alongside the import tooling:
```json
{
  "last_import": "<ISO timestamp>",
  "nodes": {
    "<unique_id>": "<checksum>"
  }
}
```

---

## What cannot be extracted automatically

The following fields are set to `REQUIRES MANUAL AUTHORING` on import and must be completed by a domain expert before the node is considered fully verified:

| KG field | Node type | Action for domain expert |
|---|---|---|
| `consequence_if_violated` | `BusinessRule` | Describe the business impact of violating this constraint in production |
| `consequence` on reified edges | `Reification` | Describe what goes wrong in a query or report if the dependency is ignored |
| `reason` on `demonstrates` / `overrides` edges | `Reification` | Explain why the dependency exists |
| `question` | `VerifiedQuery` | Formulate the natural-language business question this consumer answers |
| `sql` | `VerifiedQuery` | Provide the actual SQL used by the downstream consumer |
| `always_ask` | `Disambiguation` | Write the clarifying question to present to a query agent |
| `table_kind` | `Table` | Set when name prefix is absent and `meta.table_kind` is not configured |
| `business_definition` | `Subject` | All Subject nodes are authored manually — not produced by this connector |

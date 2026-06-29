# dbt — Technical Layer

This document is the entry point for the dbt connector. It covers the **Domain layer** — how dbt project objects (models, sources, seeds, snapshots, semantic models, metrics, exposures, and tests) map to Knowledge Graph node types, what fields can be extracted automatically, and what requires manual authoring by a domain expert.

The Vocabulary layer (`Subject` and `Disambiguation` nodes) is **outside the scope of automated extraction** for dbt. dbt has no cross-project concept registry and no annotation framework for stable business definitions. Vocabulary nodes must be authored manually; the connector does not produce them.

---

## Why import instead of reading dbt files directly

dbt YAML files are structured and readable, which raises a natural question: why not treat them as KG pages in the content storage directly, skipping the import step?

Four reasons make that unworkable:

**Wrong unit of knowledge.** A dbt YAML file is a model definition — it groups all columns of one table in a single file. A KG page is one concept: one `Measure`, one `Attribute`, one `BusinessRule`. The import splits a single dbt model file into potentially many KG pages, each with its own identity, links, and enrichment history.

**Wrong link format.** KG edges are typed, self-describing link labels embedded in page bodies (`[Table: X joinedTo -> Table: Y]`). dbt `ref()` calls and `depends_on` entries are structural identifiers in a compiled JSON graph — they need to be translated into semantic edge statements following [`spec/link-format.md`](../../../spec/link-format.md).

**Missing fields.** dbt descriptions are prose notes on a technical artifact. KG fields like `consequence_if_violated`, `predicate_sql`, `mandatory`, `definition_sql`, and `reason`/`consequence` on reified edges do not exist in dbt. They are either derived during import or authored by a domain expert afterward — neither of which happens in the dbt project itself.

**Wrong authoring surface.** The KG content storage is where domain experts enrich knowledge over time: completing a `consequence`, filling an `always_ask`, promoting a hyperlink edge to a Relationship page. If dbt YAML were the source of truth, every enrichment would have to flow back into the dbt repository, coupling the knowledge lifecycle to the dbt deployment cycle and making the KG dependent on a code repo for its authoring surface.

---

## Documents in this layer

| Document | Purpose |
|---|---|
| **This file** — `dbt-technical.md` | Node type mapping, field mapping tables, edge mapping, extraction protocol. |
| [`dbt-import.md`](dbt-import.md) | Import procedure — how dbt project metadata is read from `manifest.json` and `catalog.json`, transformed into KG node types, validated, and written to the content storage and graph DB. |

---

## Section 1 — Source system overview

dbt (data build tool) is a SQL-centric transformation framework that runs inside a data warehouse. It defines how raw data is transformed into analytics-ready tables through a layered model graph. Business knowledge is encoded in:

- **YAML property files** — `name`, `description`, `columns`, `data_type`, `data_tests`, `config`, `meta`, and `tags` on every model, source, seed, snapshot, and semantic model.
- **`semantic_models:` blocks** — first-class measures, dimensions, entities, and filters following the dbt Semantic Layer (available from dbt Core 1.6). These are the richest source of KG-ready semantics in a dbt project.
- **`metrics:` blocks** — named KPI definitions with `type` (sum, count, average, ratio, derived), `label`, and `filter`.
- **`exposures:` blocks** — downstream consumers (dashboards, notebooks, APIs) that declare `depends_on`, `owner`, `type`, and `description`.
- **`data_tests:` / `tests:`** — column-level and model-level constraints that encode business rules (`not_null`, `accepted_values`, `foreign_key`, `unique`, custom predicates).
- **SQL files** — Jinja-templated SQL with `ref()` and `source()` calls that define the lineage graph.

The primary extraction surface is **`manifest.json`**, produced by `dbt compile` or `dbt docs generate`. It is a fully compiled project snapshot containing all nodes, resolved `depends_on` edges, compiled SQL, column metadata, and test definitions. All lineage is pre-resolved — no SQL parsing is required.

**`catalog.json`** (from `dbt docs generate`) complements `manifest.json` with actual column types and row counts from the warehouse. It is used as a fallback when YAML `data_type:` is absent.

**Versioning and lifecycle conventions:** dbt signals model stability through `access:` (`public`, `protected`, `private`) and `deprecation_date:`. Models marked `access: private` are internal implementation details and are excluded from import. `deprecation_date` set to a past date maps to `status: Deprecated`.

**Layer conventions:** dbt projects typically layer models by folder. Staging models (`stg_*` or in a `staging/` folder) are intermediate transformation steps with no independent business meaning and are excluded from import by default. All other layers — ingest, data vault, business vault, marts, reporting — are imported.

---

## Section 2 — Node type mapping

| KG node type | dbt source object | Key mapping notes |
|---|---|---|
| `Subject` | Not applicable | No cross-project stable concept registry. Vocabulary layer requires manual authoring. |
| `Domain` | dbt project (`dbt_project.yml`) or mart subfolder | One `Domain` per project, or one per mart domain when a project defines multiple business domains via subfolder + schema mapping. |
| `Table` | `models:` node (non-staging) | `table_kind` derived from name prefix or folder. Staging excluded. |
| `Table` | `sources:` node | External/raw tables. `table_kind` from source system context or `meta.table_kind`. |
| `Table` | `seeds:` node | Static reference data. `table_kind: dimension`. |
| `Table` | `snapshots:` node | SCD Type 2 tables. `table_kind: dimension`. |
| `Measure` | `semantic_models[].measures[]` | Promoted only when: `type` is `ratio` or `derived`; or `label` is present (non-trivial synonym); or a `filter` implies a BusinessRule link. Simple single-column aggregates (`sum`, `count`, `average`, `max`, `min` with no label and no filter) stay inline in the Table's `## Semantic annotations`. |
| `Measure` | `metrics:` node | Fallback when `semantic_models:` is not used. Same promotion criteria apply. |
| `Attribute` | `semantic_models[].dimensions[]` | Promoted only when: `expr` is a non-trivial derived expression; or `label` differs from the column name (non-trivial synonym); or the dimension appears in multiple semantic models (cross-domain anchor). Simple categorical dimensions with no `expr` and no synonyms stay inline. |
| `Attribute` | `semantic_models[].entities[]` | Promoted only when the entity appears in multiple semantic models — i.e. it is a genuine cross-domain join key. Single-model entities stay inline. |
| `Filter` | `semantic_models[].measures[].filter` | A static WHERE predicate scoped to a measure. `mandatory: true` when the filter is unconditional (no parameter). |
| `Filter` | `semantic_models[].where_filter` or model-level `config.meta` filter | Applies across the semantic model. |
| `BusinessRule` | Column `data_tests:` with a predicate | `accepted_values`, `not_null` (on non-nullable business keys), `foreign_key`, custom SQL predicates. `definition` from test parameters. |
| `BusinessRule` | Snapshot `strategy` + `unique_key` + `updated_at` | The SCD2 versioning rule is a business rule about record lifecycle. |
| `VerifiedQuery` | `exposures:` node | Downstream validated consumers. `verified_by` from `owner.email`; `question` requires manual authoring. |
| `Disambiguation` | Not derivable automatically | Flagged during import when a column description contains indicators of multiple interpretations. Requires manual authoring. |
| `Relationship` | `mandatory`/`requires` reified edges | Stub pages created where a semantic model filter or foreign-key test implies a dependency. `reason` and `consequence` require manual authoring. |

---

## Section 3 — Field mapping tables

### Domain ← dbt project or mart folder

| KG field | dbt source | Transformation | Notes |
|---|---|---|---|
| `name` | `dbt_project.yml` → `name:` | Title-case | One domain per project |
| `name` (folder variant) | Subfolder under `models/` (e.g. `models/finance/`) | Title-case the folder name | When a project spans multiple business domains |
| `owner` | `dbt_project.yml` → `models.<project>` config, or `meta.technical_owner` | First owner entry | Optional; leave blank if absent |

### Table ← model

| KG field | dbt source | Transformation | Notes |
|---|---|---|---|
| `name` | `models[].name` | Strip version suffix (`_v0`, `_v1`), title-case | e.g. `fct_order_item_v5` → `Fact Order Item` |
| `table_kind` | Name prefix or folder | `fct_` / `report_` / `snf_` / `fact` folder → `fact`; `dim_` / `hub_` / `sat_` / `ref_` → `dimension`; `link_` / `bt_` / `bridge` folder → `bridge` | See routing table in [import procedure](dbt-import.md) |
| `table_kind` | `meta.table_kind` | Direct if set | Overrides prefix heuristic |
| `source` | `models[].fqn` joined with database/schema from `dbt_project.yml` | `<database>.<schema>.<name>` | Agents use this for SQL construction |
| `description` | `models[].description` | As-is | |
| `status` | `models[].deprecation_date` present and past → `Deprecated`; `access: private` → excluded; otherwise → `Active` | | `access: protected` and `access: public` both map to `Active` |

### Table ← source

| KG field | dbt source | Transformation | Notes |
|---|---|---|---|
| `name` | `sources[].tables[].name` | Title-case | |
| `table_kind` | `meta.table_kind` if set; else `dimension` as default | | Raw sources are often lookup or event tables; context determines kind |
| `source` | `sources[].database` + `sources[].schema` + `sources[].tables[].name` | `<database>.<schema>.<table>` | |
| `description` | `sources[].tables[].description` | As-is | |
| `status` | Always `Active` unless `meta.status: Deprecated` | | |

### Table ← seed

| KG field | dbt source | Transformation | Notes |
|---|---|---|---|
| `name` | `seeds[].name` | Title-case | |
| `table_kind` | Always `dimension` | | Seeds are reference/lookup data by definition |
| `source` | Seed file path rendered to warehouse path | `<database>.<schema>.<seed_name>` | |
| `description` | `seeds[].description` | As-is | |
| `status` | Always `Active` | | |

### Table ← snapshot

| KG field | dbt source | Transformation | Notes |
|---|---|---|---|
| `name` | `snapshots[].name` | Title-case, strip version suffix | |
| `table_kind` | Always `dimension` | | Snapshots track entity history; they are dimensional by nature |
| `source` | Snapshot target schema + name from config | `<database>.<schema>.<name>` | |
| `description` | `snapshots[].description` | As-is | |
| `status` | Always `Active` | | |

### Measure ← semantic_model measure

A measure is promoted to a Measure page only when at least one of the following applies: `type` is `ratio` or `derived`; `label` is present; or the measure has an associated `filter`. Simple aggregates (`sum`, `count`, `average`, `max`, `min`) over a single column with no label and no filter stay inline in the Table's `## Semantic annotations`.

| KG field | dbt source | Transformation | Notes |
|---|---|---|---|
| `name` | `semantic_models[].measures[].name` | Title-case | e.g. `order_revenue` → `Order Revenue` |
| `kind` | `measures[].agg`: sum/count/average/count_distinct/max/min → `aggregate expression`; type: ratio/derived → `derived formula` | | |
| `synonyms` | `measures[].label` | Single-element list if present | |
| `status` | Always `Active` unless model is `Deprecated` | | Inherits from parent model status |
| `definition_sql` | `measures[].expr` + `measures[].agg` | `<agg>(<expr>)` rendered as SQL | e.g. `SUM(order_total)` |

### Measure ← metrics node (fallback)

Same promotion criteria as above. Each `metrics[]` entry with `type: ratio` or `derived`, or with a `label`, is promoted. Plain sum/count/average metrics without a label stay inline.

| KG field | dbt source | Transformation | Notes |
|---|---|---|---|
| `name` | `metrics[].name` | Title-case | |
| `kind` | `metrics[].type`: sum/count/average → `aggregate expression`; ratio/derived → `derived formula` | | |
| `synonyms` | `metrics[].label` | Single-element list | |
| `status` | Always `Active` | | |
| `definition_sql` | `metrics[].type` + `metrics[].expression` | Rendered SQL aggregate | For ratio: `SUM(numerator) / NULLIF(SUM(denominator), 0)` |

### Attribute ← semantic_model dimension

A dimension is promoted to an Attribute page only when at least one of the following applies: `expr` is a non-trivial derived expression (CASE WHEN, COALESCE, date truncation, sign convention); `label` is present and differs meaningfully from the column name; or the same dimension name appears across multiple semantic models (cross-domain anchor). Simple categorical or time dimensions with no `expr` and no label stay inline.

| KG field | dbt source | Transformation | Notes |
|---|---|---|---|
| `name` | `semantic_models[].dimensions[].name` | Title-case | |
| `kind` | `dimensions[].type: time` → `time_dimension`; `type: categorical` → `dimension` | | |
| `synonyms` | `dimensions[].label` | Single-element list if present | |
| `access_modifier` | Always `public_access` | | dbt dimensions have no access restriction concept |
| `expression_sql` | `dimensions[].expr` if present | As-is | Omit if direct column reference |

### Attribute ← semantic_model entity

An entity is promoted to an Attribute page only when it appears in multiple semantic models — i.e. it is a genuine cross-domain join key. An entity that appears in only one semantic model stays inline in that model's Table `## Semantic annotations`.

| KG field | dbt source | Transformation | Notes |
|---|---|---|---|
| `name` | `semantic_models[].entities[].name` | Title-case | |
| `kind` | Always `dimension` | | Entities are join-key dimensions |
| `access_modifier` | Always `public_access` | | |
| `expression_sql` | `entities[].expr` if present | As-is | |

### Filter ← semantic_model measure filter

| KG field | dbt source | Transformation | Notes |
|---|---|---|---|
| `name` | Derived: `<measure_name> scope` | Title-case | e.g. `Completed Orders Scope` |
| `mandatory` | Always `true` | | Static measure filters are always unconditional |
| `predicate_sql` | `measures[].filter` | As-is | Already a SQL WHERE fragment |
| `synonyms` | — | Empty | |

### Filter ← semantic_model model-level filter

| KG field | dbt source | Transformation | Notes |
|---|---|---|---|
| `name` | Derived: `<semantic_model_name> base filter` | Title-case | |
| `mandatory` | Always `true` | | Model-level filters apply to every query on the model |
| `predicate_sql` | `semantic_models[].where_filter` | As-is | |

### BusinessRule ← data_tests predicate

| KG field | dbt source | Transformation | Notes |
|---|---|---|---|
| `name` | Derived: `<model_name> — <column_name> <test_name>` | Title-case | e.g. `Dim Customer — Status accepted values` |
| `definition` | Test type + parameters rendered as SQL | `accepted_values`: `column IN ('a', 'b')`; `not_null`: `column IS NOT NULL`; `foreign_key`: `column IN (SELECT pk FROM ref_table)` | Custom SQL tests: use the `predicate` or `query` parameter directly |
| `consequence_if_violated` | — | `REQUIRES MANUAL AUTHORING` | Tests document what is checked; business impact is not encoded |

### BusinessRule ← snapshot SCD2 definition

| KG field | dbt source | Transformation | Notes |
|---|---|---|---|
| `name` | Derived: `<snapshot_name> — SCD2 versioning` | Title-case | |
| `definition` | `strategy`, `unique_key`, `updated_at` rendered as prose + SQL | e.g. `unique_key = entity_id; new version opened when updated_at changes` | |
| `consequence_if_violated` | — | `REQUIRES MANUAL AUTHORING` | |

### VerifiedQuery ← exposure

| KG field | dbt source | Transformation | Notes |
|---|---|---|---|
| `name` | `exposures[].name` | Title-case, strip underscores | |
| `onboarding_question` | Always `false` | | Cannot be derived; default to false |
| `verified_by` | `exposures[].owner.email` or `owner.name` | As-is | |
| `verified_at` | `manifest.json` → `metadata.generated_at` as proxy | ISO date | Use as lower bound; update manually when the exposure is formally reviewed |
| `status` | Always `Active` unless model dependency is `Deprecated` | | |
| `question` | — | `REQUIRES MANUAL AUTHORING` | `exposures[].description` is close but is a consumer description, not a business question |
| `sql` | — | `REQUIRES MANUAL AUTHORING` | The exposure `depends_on` shows source models, not the consumer's SQL |

---

## Section 4 — Edge mapping

### Hyperlink edges

| KG edge kind | dbt source relationship | How to detect | Cardinality | Notes |
|---|---|---|---|---|
| `contain` | Domain → model / measure / filter / rule | All non-staging model nodes belong to their domain by folder or project | 1:N | |
| `joinedTo` | Model → model (direct lineage) | `manifest.json` `depends_on.nodes` — one hop only | N:M | Transitive deps not imported as `joinedTo`; only direct `ref()` calls |
| `joinedTo` | Model → source | `depends_on.nodes` referencing a source node | N:M | |
| `calculate` | Table → Measure | Semantic model `model:` field links a measure to its source model | 1:N | |
| `calculate` | Table → Attribute | Semantic model `model:` field links a dimension to its source model | 1:N | |
| `apply` | BusinessRule → Table | The model where a `data_tests` rule is defined | N:M | |
| `relatedTo` | Exposure → dependent models | `exposures[].depends_on.nodes` beyond the direct `VerifiedQuery → Table` | N:M | Use for models that are depended on but not the primary source |
| `implement` | Not derivable | No dbt concept for "this model implements a business concept" | — | Requires manual authoring after Vocabulary layer is authored |
| `disambiguate` | Not derivable | No dbt concept | — | Manual |

### Reified edges (Relationship pages)

| KG edge kind | dbt source situation | `reason` derivability | `consequence` derivability |
|---|---|---|---|
| `mandatory` | A semantic model filter that applies to a table unconditionally | Partially: from measure `filter` or `description` | `REQUIRES MANUAL AUTHORING` |
| `requires` | A measure whose `filter` scopes its population | Partially: from measure `filter` expression | `REQUIRES MANUAL AUTHORING` |
| `guards` | Not derivable automatically | `REQUIRES MANUAL AUTHORING` | `REQUIRES MANUAL AUTHORING` |
| `overrides` | Not derivable automatically | `REQUIRES MANUAL AUTHORING` | `REQUIRES MANUAL AUTHORING` |
| `demonstrates` | Not derivable automatically | `REQUIRES MANUAL AUTHORING` | `REQUIRES MANUAL AUTHORING` |

Stub Relationship pages are created for `mandatory` and `requires` edges where the dependency can be detected structurally. `reason` is pre-populated from the filter expression; `consequence` is left as `REQUIRES MANUAL AUTHORING` for a domain expert to complete.

---

## Section 5 — Extraction protocol

### System metadata channel (automated)

| What to extract | Where it lives | How to read it |
|---|---|---|
| All model nodes, columns, descriptions, tests, tags, config, meta | `manifest.json` → `nodes` dict | Parse `node_type: model`; filter out staging |
| Source definitions | `manifest.json` → `sources` dict | Parse `node_type: source` |
| Seed definitions | `manifest.json` → `sources` or `seeds` dict | Parse `node_type: seed` |
| Snapshot definitions | `manifest.json` → `nodes` dict | Parse `node_type: snapshot` |
| Semantic model definitions (measures, dimensions, entities, filters) | `manifest.json` → `semantic_models` dict | Available from dbt Core 1.6+ |
| Metric definitions (fallback) | `manifest.json` → `metrics` dict | Used when `semantic_models:` not present |
| Exposure definitions | `manifest.json` → `exposures` dict | Parse `node_type: exposure` |
| Resolved lineage (all `ref()` and `source()` calls) | `manifest.json` → each node's `depends_on.nodes` | Direct; no SQL parsing needed |
| Compiled SQL | `manifest.json` → `compiled_code` per node | Available after `dbt compile` |
| Actual column types (fallback) | `catalog.json` → `nodes` dict | Use when YAML `data_type:` absent |
| Test definitions | `manifest.json` → `nodes` dict, `node_type: test` | Each test carries `attached_node`, `test_metadata.name`, `test_metadata.kwargs` |

**System access method:** Run `dbt compile --select state:modified+` for incremental extraction, or `dbt docs generate` for a full snapshot. Both produce `manifest.json` in the project's `target/` directory. No API credentials or network access to the dbt Cloud API are required for local projects; dbt Cloud API access is optional for metadata about runs and job state.

**Incremental detection:** Compare `manifest.json` → `metadata.generated_at` timestamp against the last successful import timestamp. For node-level granularity, compare per-node `checksum.checksum` values stored in the importer state from the previous run. Only nodes with changed checksums need re-import.

### Project documentation channel (manual)

| What to author | Where to find source material |
|---|---|
| `Subject` nodes and `implement ->` edges | Domain documentation, data dictionaries, business glossaries maintained alongside the dbt project |
| `Disambiguation` nodes and `always_ask` text | Column descriptions flagged during import as potentially ambiguous; domain expert review |
| `consequence` on all reified edges | Domain expert who understands the business impact of query correctness failures |
| `question` on `VerifiedQuery` nodes | Analytics engineers or analysts who own the exposure |
| `sql` on `VerifiedQuery` nodes | The consumer system's query (e.g. BI tool SQL, notebook cell) |
| `table_kind` overrides | When folder/prefix heuristic is incorrect for a specific model |

### What cannot be extracted automatically

| KG field | Node type | Reason |
|---|---|---|
| `always_ask` | `Disambiguation` | Depends on project-specific usage patterns; no dbt encoding |
| `consequence` on reified edges | `Relationship` | Tests check correctness but do not encode business impact |
| `question` | `VerifiedQuery` | `description:` on an exposure is a consumer note, not a business question |
| `sql` | `VerifiedQuery` | dbt does not know the consumer's query |
| `business_definition` | `Subject` | No cross-project stable concept — all Vocabulary nodes are manual |
| `table_kind` | `Table` | When model name has no recognized prefix and `meta.table_kind` is absent |
| `reason` on `guards` / `demonstrates` / `overrides` | `Relationship` | No dbt structural signal for these edge kinds |

---

## Section 6 — Import procedure

See [`dbt-import.md`](dbt-import.md) for the full step-by-step import procedure: manifest parsing, staging exclusion, table kind routing, semantic model extraction, test-to-BusinessRule conversion, edge construction, validation, and version comment format.

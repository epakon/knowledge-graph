# SAP HANA — Import Procedure

> **Draft** — conceptual design; specific query patterns should be verified against the live system before implementation.

This document describes the import procedure for the SAP HANA connector. It covers how HANA system metadata is discovered, extracted, transformed into Knowledge Graph nodes and edges, and written to content storage and the graph DB.

The goal is **not** to import the full HANA object universe — it is to extract the semantically relevant subset (Calculation Views in scope, physical tables in the Data Warehouse and Data Mart layers, and their dependency relationships) that directly populates KG node types. Extraction is bounded by a curated seed list and a schema-scoped dependency walk.

---

## Conceptual architecture

```
SAP HANA system
    │
    │  1. Seed — known Calculation Views and tables in scope
    │  2. Discover — dependency walk via SYS.OBJECT_DEPENDENCIES
    │  3. Extract — view/table metadata from SYS.* catalog tables
    │  4. Transform — map to KG node types and field mappings
    │
    ▼
Extracted records (JSON)
    │
    │  5. Validate — schema, uniqueness, required fields
    │  6. Import — content storage pages + graph DB nodes/edges
    │
    ▼
Knowledge Graph (Confluence + Graph DB)
```

Steps 1–4 are HANA-specific. Steps 5–6 follow the generic import procedure in [`connectors.md`](../connectors.md) §Section 6.

---

## Step 1 — Seed: known objects in scope

Extraction starts from a curated seed list of Calculation Views and physical tables that are known to carry business meaning for the domains in scope. The seed list is the primary editorial control: objects not reachable from the seed are not imported.

**Seed selection criteria:**
- Calculation Views in `_SYS_BIC` with a package prefix matching an in-scope domain.
- Query and model views (typically `CVQ_` and `CVM_` prefixes) carry the most business meaning and are the natural entry points.
- Physical tables in the Data Warehouse / Core layer that are direct data sources for in-scope Calculation Views.
- `SYS.VIEWS.IS_VALID = 'TRUE'` — inactive or broken views are excluded.

Dimension views (typically `CVD_` prefix) are added automatically during the dependency walk (Step 2) when referenced by a seed object — they do not need to be listed explicitly unless they are a primary entry point.

> **Configuration note:** the actual prefixes and schema suffixes depend on the naming convention of the specific HANA implementation. Document the convention used and update the routing rules in Step 4 accordingly. See [`sap-hana-technical.md`](sap-hana-technical.md) §Section 1.

---

## Step 2 — Discover: dependency walk

Starting from each seed object, the importer walks the object dependency graph to discover related objects.

### Dependency source

The primary source is `SYS.OBJECT_DEPENDENCIES`, which tracks all SQL-level dependencies across the HANA catalog:

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
WHERE DEPENDENCY_TYPE = 1  -- standard SQL dependency
```

When the HANA system uses `_SYS_BI.BIMC_SYNONYM_MAPPING` (available on systems with XS Classic or XSA runtime), join it to resolve synonym-to-Calculation View mappings that may not appear in `SYS.OBJECT_DEPENDENCIES`. On HDI-only systems, use `SYS.SYNONYMS` alone.

### Walk direction

The import procedure walks **upstream** (from a Calculation View toward its source tables) to discover the physical data foundations of each seed view. The [Lineage Explorer addon](addons/sap-hana-lineage-explorer.md) supports both directions for ad-hoc navigation.

### Walk rules

| Object type encountered | Action |
|---|---|
| `VIEW` in `_SYS_BIC` (Calculation View) | Follow — may expose additional `Table`, `Measure`, or `Filter` nodes |
| `TABLE` in the Data Warehouse / Core layer | Record as `Table` node candidate; stop — do not walk upstream further |
| `VIEW` in the Data Mart / Semantic layer (SQL view, not CV) | Record as `Table` node candidate; follow one hop upstream |
| `SYNONYM` | Resolve to the underlying object via `SYS.SYNONYMS`; follow |
| `FUNCTION` | Record as `BusinessRule` candidate; do not follow |
| `PROCEDURE` | Skip — procedural logic is not imported automatically |
| `TABLE` in the Landing or Staging layer | Skip — transient or unprocessed data |
| Objects with `IS_VALID = 'FALSE'` | Skip — inactive objects |

Walk depth is bounded to **4 hops** from each seed. Objects at depth > 4 are recorded in the discovery log but not imported unless they appear in the seed list.

---

## Step 3 — Extract: catalog metadata

For each discovered object, the importer reads the following catalog tables.

### 3a. View metadata — `SYS.VIEWS`

| Column | Content | Used for |
|---|---|---|
| `SCHEMA_NAME` | Schema (e.g. `_SYS_BIC`) | Domain inference |
| `VIEW_NAME` | Full view name (e.g. `acme.payments.views/CVQ_FEES`) | Node identity, prefix parsing |
| `VIEW_TYPE` | `CALC` for Calculation Views, `ROW`/`COLUMN` for SQL views | Filter: import `CALC` and `COLUMN` views |
| `COMMENTS` | Free-text description | `description` field |
| `IS_VALID` | View activation state | `status` field |

### 3b. Table metadata — `SYS.TABLES`

| Column | Content | Used for |
|---|---|---|
| `SCHEMA_NAME` | Schema | Domain inference |
| `TABLE_NAME` | Table name | Node identity |
| `TABLE_TYPE` | `COLUMN` / `ROW` | Filter: import column-store tables |
| `COMMENTS` | Free-text description | `description` field |

### 3c. Column metadata — `SYS.VIEW_COLUMNS`, `SYS.TABLE_COLUMNS`

| Column | Content | Used for |
|---|---|---|
| `COLUMN_NAME` | Column name | `Measure` / `Attribute` node name |
| `DATA_TYPE_NAME` | Data type | Type hints for `kind` |
| `COMMENTS` | Free-text | `synonyms` candidate |
| `IS_NULLABLE` | Nullability | Filter detection heuristic |

HANA does not structurally distinguish measure columns from attribute columns in the catalog. This distinction must be inferred from column naming conventions (e.g. `_AMOUNT`, `_COUNT`, `_SUM` suffixes suggest measures) and verified by a domain expert.

### 3d. Synonym resolution — `SYS.SYNONYMS`

| Column | Content | Used for |
|---|---|---|
| `SCHEMA_NAME` | Synonym schema | |
| `SYNONYM_NAME` | Synonym name | Source node in lineage |
| `OBJECT_SCHEMA` | Target schema | |
| `OBJECT_NAME` | Target object name | Transparent resolution |

---

## Step 4 — Transform: map to KG node types

Transformation applies the field mapping tables from [`sap-hana-technical.md`](sap-hana-technical.md) §Section 3.

### Object-type-to-node-type routing

The routing rules below use the common `CVQ_` / `CVM_` / `CVD_` prefix convention. Update these rules to match the actual naming convention of the implementation.

| Signal | Target KG node type |
|---|---|
| `_SYS_BIC` view with query/fact prefix (e.g. `CVQ_` / `CVM_`) | `Table` (`table_kind: fact`) |
| `_SYS_BIC` view with dimension prefix (e.g. `CVD_`) | `Table` (`table_kind: dimension`) |
| DWH / Core table with fact-like name (e.g. `*_HEADER`, `*_ITEM`, `*_TRANSACTION`) | `Table` (`table_kind: fact`) — requires domain expert confirmation |
| DWH / Core table with dimension-like name (e.g. `*_MASTER`, `*_CONFIG`, `*_REF`, `*_DIM`) | `Table` (`table_kind: dimension`) — requires domain expert confirmation |
| Column with aggregation-hint suffix (e.g. `_AMOUNT`, `_COUNT`, `_SUM`) on a fact view | `Measure` candidate — requires domain expert confirmation |
| Column on a dimension view or dimension table | `Attribute` candidate |
| CV input parameter with no default value | `Filter` (`mandatory: true`) |
| CV input parameter with a default value | `Filter` (`mandatory: false`) |
| Column containing CASE/IF logic in view definition | `BusinessRule` candidate — requires domain expert review |
| Same column name with different semantics in multiple schemas | `Disambiguation` candidate — requires manual authoring |

### Domain inference

The KG `Domain` node is inferred from the package prefix or schema name:

| Source | Domain inference rule |
|---|---|
| `_SYS_BIC` view | Strip the organisation prefix from the package path; take the first meaningful segment; title-case (e.g. `acme.payments.views` → `Payments`) |
| Application schema | Strip the layer suffix if one exists; split on word boundaries; title-case (e.g. `PAYMENTS_DWH` → `Payments`) |

When inference is ambiguous, flag as `REQUIRES MANUAL AUTHORING` and log for domain expert resolution.

### Edge candidates

| Discovered relation | KG edge kind |
|---|---|
| CV depends on another CV (`VIEW → VIEW` in `OBJECT_DEPENDENCIES`) | `joinedTo` candidate |
| CV depends on a physical table (`TABLE → VIEW`) | `calculate` (table feeds CV measure/attribute) |
| Synonym resolves to a CV (`SYNONYM → VIEW`) | `relatedTo` (after synonym transparent resolution) |
| CV mandatory parameter | `mandatory` reified edge candidate (stub Reification page) |
| CV in package → Domain node | `contain` |

Reified edge candidates are created as stub Reification pages with `REQUIRES MANUAL AUTHORING` for `reason` and `consequence`. They are not considered verified until a domain expert completes both fields.

---

## Step 5 — Validate

Before writing to content storage:

1. **Required fields** — all fields required in [`spec/schema.yaml`](../../../spec/schema.yaml) are present or explicitly set to `REQUIRES MANUAL AUTHORING`.
2. **Enum values** — `table_kind`, `kind`, `status`, `access_modifier` within allowed sets.
3. **Name uniqueness** — checked against the KG node index; conflicts logged and skipped.
4. **Measure vs. attribute ambiguity** — if a column cannot be classified by naming convention alone, flag as `REQUIRES MANUAL REVIEW` and import as `Attribute` by default.
5. **Seed reachability** — every imported object must be traceable to a seed entry via the discovery walk; orphaned objects are rejected.

Validation failures are written to an import log, not silently dropped.

---

## Step 6 — Import

Follows the generic procedure in [`connectors.md`](../connectors.md) §Section 6:

1. **Node creation** — content storage pages with template sections; graph DB upsert via `MERGE (label, name)`.
2. **Edge creation** — link statements per [`spec/link-format.md`](../../../spec/link-format.md); stub Reification pages for reified edges.
3. **Duplicate handling** — existing `Active` node: skip and log; existing `Deprecated` node: create new, link with `relatedTo`.
4. **Post-import audit** — audit rules from [`spec/schema.yaml`](../../../spec/schema.yaml).
5. **Version comment** on every created or updated page:

```
v1 | <YYYY-MM-DD> | sap-hana-import
Summary: Imported from SAP HANA — <SCHEMA_NAME>/<OBJECT_NAME>
Changed: Initial import
Reason: Extracted from <Calculation View / physical table>
Breaking: no
```

---

## Incremental extraction

Primary signal: `SYS.VIEWS.LAST_DDL_TIME` and `SYS.TABLES.LAST_DDL_TIME` compared against the last extraction timestamp stored in the importer state.

Secondary signal: HDI container deployment log (tracked in CI/CD pipeline, external to HANA). A new deployment of an HDI container triggers re-extraction of all views in the affected package.

Objects whose source schema changed but that are not in the current seed list are logged as candidates for seed expansion, not automatically imported.

---

## What cannot be extracted automatically

| KG field | Node type | Reason |
|---|---|---|
| `business_definition` | `Subject` | No annotation framework; requires data model docs or domain expert |
| `always_ask` | `Disambiguation` | Project-specific usage context |
| `consequence` on reified edges | `Reification` | Business impact not encoded in HANA metadata |
| `question` | `VerifiedQuery` | Natural-language question requires human formulation |
| `table_kind` | `Table` (ambiguous naming) | No `@Analytics.dataCategory` equivalent |
| `kind` | `Measure` | HANA does not annotate measure vs. attribute columns structurally |
| `definition_sql` | `BusinessRule` | Graphical CV calculation nodes do not expose SQL text |
| `reason` on `demonstrates` | `Reification` | No structural signal |

---

## Seed list

> Maintained manually per deployment. Add objects here to expand extraction scope.

| Calculation View / Table | Schema | Domain | Rationale |
|---|---|---|---|
| *(add seed objects per domain onboarding)* | | | |

# SAP HANA — Technical Layer

> **Draft** — this document is work in progress and has not been formally reviewed.

This document is the entry point for the SAP HANA connector. It covers the **Domain layer** — technical elements extracted from the HANA database: Calculation Views, physical tables, synonyms, and the dependency relationships between them.

SAP HANA is a database platform, not an ERP or business process system. It has no built-in business process concepts, no standard annotation framework (such as CDS VDM annotations in S/4HANA), and no lifecycle contracts. **The Vocabulary layer (Subject and Disambiguation nodes) is therefore not populated by this connector** — those nodes require manual authoring by domain experts and are outside the extraction scope. See [`connectors.md`](../connectors.md) for the rationale on optional business layers.

## Documents in this layer

| Document | Purpose |
|---|---|
| **This file** — `sap-hana-technical.md` | Source system overview, node type mapping, field mapping tables, edge mapping, extraction protocol. |
| [`sap-hana-import.md`](sap-hana-import.md) | Import procedure — how HANA metadata is discovered (schema-scoped seed + dependency walk via `SYS.OBJECT_DEPENDENCIES`), extracted, transformed into KG node types, validated, and written to content storage and graph DB. |

## Addons

| Addon | Purpose |
|---|---|
| [`addons/sap-hana-lineage-explorer.md`](addons/sap-hana-lineage-explorer.md) | Lineage Explorer — read-only, ephemeral map of the HANA object dependency graph. No human enrichment; used by data engineers to navigate object dependencies and by the import procedure as the discovery index. |

---

## Section 1 — Source system overview

SAP HANA is an in-memory column-store database. It is used both as an operational database and as an analytics and semantic layer platform. It hosts:

- **Calculation Views** — columnar, multi-node analytic models published in the `_SYS_BIC` schema. These are the primary semantic layer: they define measures, dimension attributes, mandatory parameters, and join logic. Built and deployed via SAP HANA Development (HDI containers or XS Classic packages).
- **Physical tables** — column-store tables in application schemas, serving as the data foundation.
- **Synonyms** — cross-schema pointers exposing tables from one schema into another, and mapping `_SYS_BIC` Calculation View paths to user-friendly names.
- **SQL views and procedures** — supplementary logic, less semantically rich than Calculation Views.

### Schema naming conventions

HANA implementations typically use schema names or suffixes to identify which data warehouse layer a schema belongs to. The table below maps common data warehouse layers to their KG relevance. Implementations may use different naming conventions or no convention at all — document the actual mapping during connector configuration.

| Data warehouse layer | Typical role | KG relevance |
|---|---|---|
| **Landing / Raw** | Data as received from the source system, unmodified. Often a replication target (e.g. SLT, CDC). | Low — no transformations or business semantics yet |
| **Staging** | Intermediate processing area: cleansing, deduplication, type casting. Transient; may be truncated after each load. | Low — ephemeral, not a stable reference |
| **Data Warehouse / Core** | Integrated, historised, and conformed data. The persistent foundation of the analytical platform. Owned by data engineers. | High — primary `Table` nodes for physical data sources |
| **Data Mart / Semantic** | Subject-area views and Calculation Views optimised for consumption. Exposes measures, attributes, and parameters to end users. | High — primary `Table`, `Measure`, `Filter` nodes |
| **Reporting / Presentation** | Pre-aggregated or denormalised outputs ready for direct BI tool consumption. May coexist with the semantic layer or replace it. | Medium — import if not already covered by the semantic layer |

When a HANA system receives replicated tables from SAP ECC or S/4HANA, those tables typically live in a dedicated landing schema. Their KG relevance depends on whether they are directly consumed by Calculation Views in scope.

### Object naming conventions (Calculation Views)

Calculation Views in `_SYS_BIC` follow a `<Package>/<Prefix><Name>` pattern. Common prefixes:

| Object prefix | Meaning | KG `table_kind` |
|---|---|---|
| `CVQ_` | Query view — top-level consumable, typically a fact model with measures | `fact` |
| `CVM_` | Model view — intermediate analytic model, may expose measures | `fact` |
| `CVD_` | Dimension view — lookup / master data | `dimension` |

Prefixes are a convention, not a HANA standard — an implementation may use different prefixes or none. Document the actual convention during connector configuration and update the routing rules in the import procedure accordingly.

Package hierarchy maps to domain: the first meaningful path segment of the package name is used as the KG `Domain` node name (e.g. `acme.payments.views` → `Payments`).

### Technical surfaces for extraction

| Surface | Content | Used for |
|---|---|---|
| `SYS.VIEWS` | All views including Calculation Views in `_SYS_BIC` | Node discovery, `table_kind`, description |
| `SYS.TABLES` | Physical tables in application schemas | `Table` nodes |
| `SYS.OBJECT_DEPENDENCIES` | All object-to-object dependencies | Dependency walk for lineage explorer and edge extraction |
| `SYS.SYNONYMS` | Synonym definitions | Cross-schema pointer resolution |
| `SYS.VIEW_COLUMNS` | Column metadata for views | `Attribute` and `Measure` candidates |
| `SYS.TABLE_COLUMNS` | Column metadata for tables | `Attribute` candidates |
| `SYS.PROCEDURE_PARAMETERS` | Procedure input/output parameters | `Filter` candidates |
| `_SYS_BI.BIMC_SYNONYM_MAPPING` | Maps synonym names to their `_SYS_BIC` qualified paths | Synonym resolution for Calculation View lineage (present when XS Classic or XSA is deployed) |

> `_SYS_BI.BIMC_SYNONYM_MAPPING` is only available in HANA systems with XS Classic or XSA runtime. On HDI-only systems, synonym resolution is done via `SYS.SYNONYMS` alone.

### Versioning and lifecycle

HANA has no lifecycle contract system. Stability is inferred from:
- Schema naming convention (implementation-specific — see above).
- Calculation View activation state: inactive views (`SYS.VIEWS.IS_VALID = 'FALSE'`) are treated as `Deprecated`.
- HDI container version: tracked in the HDI container's deployment history, not in HANA system tables.

---

## Section 2 — Node type mapping

| KG node type | HANA source | Key mapping notes |
|---|---|---|
| `Subject` | not applicable | No built-in business concepts. All `Subject` nodes require manual authoring by domain experts. |
| `Domain` | Package hierarchy in `_SYS_BIC`; schema name | First meaningful package path segment or schema name prefix (strip suffix; title-case) |
| `Table` | Calculation View in `_SYS_BIC`; physical table in application schema | CV as `fact` or `dimension` by name prefix; physical table by naming heuristic or domain expert |
| `Measure` | Calculation View output column with aggregation function (SUM, COUNT, AVG) | Requires manual confirmation — HANA does not annotate measures structurally |
| `Attribute` | Calculation View dimension output column; characteristic column in physical table | Columns used as GROUP BY dimensions, not aggregated |
| `Filter` | Calculation View mandatory input parameter; variable with no default | `mandatory = true` when parameter has no default value |
| `VerifiedQuery` | SQL query against a `_SYS_BIC` path, reviewed and approved by a domain expert | Manual authoring only; cannot be auto-generated |
| `BusinessRule` | Calculated column containing CASE/IF logic; union node encoding a classification | Often requires manual review to determine if rule is significant |
| `Disambiguation` | Column name with different semantics across schemas or views | Manual authoring only |
| `Relationship` | Dependency between two nodes with a stated business reason and consequence | Reified only when `reason` + `consequence` can be authored by a domain expert |

`Subject` and `Disambiguation` nodes may be added manually by domain experts as the KG matures. The connector does not block their creation — it simply cannot auto-generate them.

---

## Section 3 — Field mapping tables

### Table ← Calculation View (`_SYS_BIC`)

| KG field | Source field / path | Transformation | Notes |
|---|---|---|---|
| `name` | `VIEW_NAME` in `_SYS_BIC`, strip package prefix up to `/` | Strip known CV prefix (`CVQ_`, `CVM_`, `CVD_`); title-case remainder | e.g. `acme.payments.views/CVQ_FEES` → `Fees` |
| `table_kind` | Object name prefix | `CVQ_` / `CVM_` → `fact`; `CVD_` → `dimension` | Default `fact` if prefix unrecognised; flag for domain expert review |
| `source` | `_SYS_BIC/<package>/<VIEW_NAME>` | Concatenate schema and view name | Used by agents to construct `FROM` clause |
| `description` | `SYS.VIEWS.COMMENTS` | As-is if present | `REQUIRES MANUAL AUTHORING` if NULL |
| `status` | `SYS.VIEWS.IS_VALID` | `TRUE` → `Active`; `FALSE` → `Deprecated` | |

### Table ← Physical Table

| KG field | Source field / path | Transformation | Notes |
|---|---|---|---|
| `name` | `SYS.TABLES.TABLE_NAME` | Title-case, replace underscores with spaces | |
| `table_kind` | Heuristic: table name suffix or domain expert | Common: `*_HEADER` / `*_ITEM` → `fact`; `*_MASTER` / `*_CONFIG` / `*_REF` → `dimension` | Always requires domain expert confirmation |
| `source` | `<SCHEMA_NAME>.<TABLE_NAME>` | Direct | |
| `description` | `SYS.TABLES.COMMENTS` | As-is if present | `REQUIRES MANUAL AUTHORING` if NULL |
| `status` | Data warehouse layer of the schema | Data Mart / Core → `Active`; Staging / Landing → flag for review | Implementation-specific |

### Measure ← Calculation View Output Column

| KG field | Source field / path | Transformation | Notes |
|---|---|---|---|
| `name` | `SYS.VIEW_COLUMNS.COLUMN_NAME` | Title-case, replace underscores | |
| `kind` | Column aggregation type | Aggregated (SUM/COUNT/AVG) → `aggregate expression` | HANA does not annotate this; requires inspection of CV definition |
| `synonyms` | `SYS.VIEW_COLUMNS.COMMENTS` | As-is if present | |
| `status` | Inherited from parent view | Same as parent `Table` status | |
| `definition_sql` | Extracted from CV calculation node | Aggregation expression if readable | `REQUIRES MANUAL AUTHORING` if CV is graphical and expression is not exposed as SQL |

### Attribute ← Calculation View Dimension Column

| KG field | Source field / path | Transformation | Notes |
|---|---|---|---|
| `name` | `SYS.VIEW_COLUMNS.COLUMN_NAME` | Title-case | |
| `kind` | Column role in CV | Dimension pass-through → `dimension`; derived CASE column → `derived` | |
| `access_modifier` | Layer of the source schema | Data Mart / Semantic layer → `public_access`; Core / DWH layer → `restricted_access` | Implementation-specific |
| `expression_sql` | CASE/CAST expression in CV definition | As-is if readable | Omit if direct column pass-through |

### Filter ← Calculation View Input Parameter

| KG field | Source field / path | Transformation | Notes |
|---|---|---|---|
| `name` | Parameter name in CV definition | Strip common prefixes (e.g. `IP_`, `P_`) if present; title-case | |
| `mandatory` | Parameter has no default value | `true` if no default; `false` otherwise | Check CV definition |
| `predicate_sql` | Parameter binding expression | e.g. `WHERE COLUMN = :PARAM_NAME` | Manual construction from parameter name and bound column |
| `synonyms` | Parameter label in CV metadata | As-is if present | |

### VerifiedQuery ← Domain Expert Authored

| KG field | Source | Notes |
|---|---|---|
| `name` | Descriptive name provided by domain expert | |
| `onboarding_question` | Manual flag | `false` by default |
| `verified_by` | Domain expert name | |
| `verified_at` | Date of verification | ISO date |
| `question` | Natural-language business question | Cannot be auto-generated |
| `sql` | SQL query against a `_SYS_BIC` source path | Must be valid and tested against live system |

### BusinessRule ← Calculated Column / CASE Logic

| KG field | Source | Transformation | Notes |
|---|---|---|---|
| `name` | Descriptive rule name | Manual; derive from column name and CASE condition | |
| `definition` | CASE/IF expression from CV definition | Exact SQL expression | `REQUIRES MANUAL AUTHORING` if CV is graphical |
| `consequence_if_violated` | Business impact | One sentence; requires domain expert | Always manual |

---

## Section 4 — Edge mapping

### Hyperlink edges

| KG edge kind | HANA source relationship | How to detect | Cardinality | Notes |
|---|---|---|---|---|
| `joinedTo` | CV join node between two data sources; SQL view joining two tables | `SYS.OBJECT_DEPENDENCIES` where both source and target are TABLE or VIEW | N:M | Join key stored as `on` property — requires inspection of join definition |
| `calculate` | CV exposes a measure or attribute sourced from a physical table | Walk `OBJECT_DEPENDENCIES`: TABLE → VIEW path | 1:N | |
| `relatedTo` | Synonym bridging two schemas; conceptual relationship without structural join | `SYNONYM → VIEW` edges after synonym resolution | N:M | |
| `contain` | Package prefix → Domain; Schema → Domain | Package path match to `Domain` node | 1:N | |
| `apply` | A business rule governs a specific table or measure | Manual authoring only — no structural signal in HANA | N:M | |
| `implement` | A Calculation View implements a concept in the Vocabulary layer | Manual authoring — links to a manually-authored `Subject` node | N:M | |
| `disambiguate` | A subject has a context-dependent meaning | Manual authoring only — no structural signal in HANA | 1:N | |

### Reified edges (require Relationship page)

| KG edge kind | HANA situation | `reason` + `consequence` |
|---|---|---|
| `mandatory` | A CV input parameter with no default that must be supplied for the query to execute | `reason`: derive from parameter name and bound column; `consequence`: manual authoring — query fails or returns full-scan result without it |
| `requires` | A measure's aggregation scope depends on a filter parameter | `reason`: from CV parameter binding; `consequence`: incorrect aggregation if omitted — always manual |
| `guards` | A filter restricts valid rows for a measure | Manual authoring — no structural signal |
| `demonstrates` | A verified query exemplifies a business rule | Manual authoring |

---

## Section 5 — Extraction protocol

### System metadata channel

What can be read programmatically from the live HANA system:

| What to read | Source | Produces |
|---|---|---|
| All Calculation Views in scope | `SYS.VIEWS WHERE SCHEMA_NAME = '_SYS_BIC'` filtered by package | `Table` node candidates |
| Physical tables in Data Warehouse / Core schemas | `SYS.TABLES` filtered by schema | `Table` node candidates |
| View and table column metadata | `SYS.VIEW_COLUMNS`, `SYS.TABLE_COLUMNS` | `Measure` and `Attribute` candidates |
| Object dependency graph | `SYS.OBJECT_DEPENDENCIES` | Edge candidates (`joinedTo`, `calculate`, `relatedTo`) |
| Synonym definitions | `SYS.SYNONYMS` | Cross-schema pointer resolution |
| CV activation state | `SYS.VIEWS.IS_VALID` | `status` field |
| Object comments | `SYS.VIEWS.COMMENTS`, `SYS.TABLES.COMMENTS` | `description` field |

**What cannot be extracted automatically — always requires domain expert authoring:**
- `business_definition` for any `Subject` node (no annotation framework).
- `always_ask` for any `Disambiguation` node.
- `consequence` for all reified edges.
- `question` for `VerifiedQuery` nodes.
- `table_kind` for physical tables without a clear naming pattern.
- Which CV output columns are measures vs. attributes (HANA does not structurally distinguish them in `SYS.VIEW_COLUMNS`).
- Business rules encoded as graphical CV calculation nodes (expressions not exposed as SQL text).

### Project documentation channel

All Vocabulary-layer knowledge comes from project documentation, not from HANA system tables:
- Data model design documents and BRDs — source of `Subject` business definitions.
- Domain expert review sessions — source of `VerifiedQuery` and reified edge `reason`/`consequence`.

These must be explicitly planned as manual authoring tasks before KG nodes are considered complete.

### Scope filtering

Extraction is scoped by schema and package to avoid importing the full HANA object universe. The specific include/exclude rules depend on the schema naming convention of the implementation and must be configured per deployment. General principles:

- Include: `_SYS_BIC` views matching the in-scope package prefixes.
- Include: Data Warehouse / Core and Data Mart schemas confirmed to feed in-scope Calculation Views.
- Exclude: HANA system schemas (`_SYS_*`, `SYS`, `SAP_*`, `HANA_*`).
- Exclude: Landing and Staging schemas — transient or unprocessed data.
- Exclude: user personal schemas and system-generated runtime schemas.

### Incremental detection

Primary signal: `SYS.VIEWS.LAST_DDL_TIME` and `SYS.TABLES.LAST_DDL_TIME` compared against the last extraction timestamp stored in the importer state. Secondary signal: HDI container deployment log (external to HANA — tracked in the CI/CD pipeline).

---

## Section 6 — Import procedure

See [`sap-hana-import.md`](sap-hana-import.md) for the full step-by-step import procedure: seed selection, dependency walk via `SYS.OBJECT_DEPENDENCIES`, object-type-to-KG-node routing, validation, and version comment format.

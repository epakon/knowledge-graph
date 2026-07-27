# SAP S/4HANA — Import Procedure

> **Draft** — conceptual design; implementation details (specific function modules, class names, API paths) are placeholders to be filled once verified against a live system.

This document describes the conceptual design of the import procedure for [`sap-s4hana-technical.md`](sap-s4hana-technical.md). It covers how SAP S/4HANA system metadata is discovered, extracted, and transformed into Knowledge Graph nodes and edges.

The goal is **not** to replicate the S/4HANA metadata graph in full — it is to extract the semantically rich subset (CDS annotations, VDM structure, dependency relationships) that directly populates KG node types. The extraction is bounded by what already has embedded business meaning in the source system.

---

## Conceptual architecture

```
S/4HANA system
    │
    │  1. Seed — known VDM roots
    │  2. Discover — dependency walk via where-used index
    │  3. Extract — DDL source + annotations from DD* tables
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

Steps 1–4 are S/4HANA-specific. Steps 5–6 follow the generic import procedure defined in [`connectors.md`](../connectors.md) §Section 6.

---

## Step 1 — Seed: known VDM roots

Extraction does not scan the entire DDIC. It starts from a curated seed list of VDM interface views (`I_<Entity>`) and consumption views (`C_<Entity>`) that are known to carry business meaning for the domains in scope (FI, FI-AR, MM, SD, CO).

The seed list is maintained manually in this document (see [Seed list](#seed-list)). It is the primary editorial control point: objects not reachable from the seed are not imported.

**Seed selection criteria:**
- VDM lifecycle contract `#CLOUD_PUBLIC` or `#RESTRICTED_REUSE` — these are SAP-stable surfaces
- Application component within scope (FI, FI-AR, MM, SD, CO)
- `@Analytics.dataCategory` annotation present — confirms analytic intent

DDIC transparent tables (e.g. `BKPF`, `BSEG`) are added to the seed only as fallback when no CDS view covers the same entity. They are not walked via the where-used index — their field semantics come from data elements (`DD04L`/`DD04T`), not from CDS annotations.

---

## Step 2 — Discover: dependency walk

Starting from each seed object, the importer walks the CDS dependency graph to discover related objects that should also be imported (associations, extend views, parameter-bound views).

### Where-used index

The S/4HANA where-used index for CDS objects provides the inverse dependency direction: given a CDS entity, which other entities reference it? This is the primary discovery interface.

> **Source table:** `DDLS_RIS_INDEX` — the CDS-specific reverse-index that covers both SQL-view-backed and HANA-native activation paths. Unlike the classic ABAP cross-reference (`WBCROSSGT`), it is not limited to objects with generated SQL views. Exact column names and available `DEPTYPE` values should be verified on a live system; see [`addons/sap-s4hana-lineage-explorer.md`](addons/sap-s4hana-lineage-explorer.md) for the full schema notes.

### Reification to the Knowledge Graph's own graph DB

The where-used index in S/4HANA is itself a dependency graph. Once extracted, it can optionally be projected into the Knowledge Graph's graph DB as a subgraph of CDS-to-CDS `RELATED_TO` or `joinedTo` edges. This is **not required for the initial import** — the graph DB's primary purpose here is duplicate prevention and traversal of KG nodes. However, projecting the S/4HANA dependency structure into the graph DB enables:

- Impact analysis: which KG `Table` nodes are affected if a CDS base view changes?
- Discovery completeness: are all CDS views reachable from the seed actually imported?
- Lineage tracing: follow `implement ->` edges from a `Subject` back through the CDS dependency chain to the physical DDIC table

If projected, CDS-to-CDS dependencies are stored as `RELATED_TO` relationships between `Table` nodes, with an additional property `s4_dependency_type` (e.g. `association`, `extend`, `parameter_binding`).

### Walk rules

| Dependency type | Action |
|---|---|
| CDS association (`ASSOCIATION TO`) | Follow — the associated view may expose a related `Table` node |
| Extend view (`EXTEND VIEW`) | Follow — extensions may add semantically important elements |
| CDS composition (`COMPOSITION OF`) | Follow — child entity in a RAP business object |
| Parameter binding | Record as candidate `Filter` node; do not recursively follow |
| Text view (`_Text` suffix) | Follow — provides `@EndUserText` content for synonyms |
| Private / internal views (`@VDM.private: true`) | Stop — do not import |
| SAP internal contract only (`#SAP_INTERNAL_USE_ONLY`) | Stop — mark as `Deprecated` if already in KG |

Walk depth is bounded to **3 hops** from each seed to avoid importing the entire VDM layer. Objects at depth > 3 are recorded in the discovery log but not imported unless they appear in the seed list.

---

## Step 3 — Extract: DDL sources and annotations

For each discovered CDS object, the importer reads:

### 3a. DDL source text — `DDDDLSRC`

| Column | Content | Used for |
|---|---|---|
| `DDLNAME` | CDS entity name | Object identity |
| `DDLSRC` | Full DDL source text (CLOB) | Annotation parsing, element names, CASE expressions |
| `DDLTYPE` | Object type (view, abstract entity, etc.) | Filter: import views only |

The DDL source is parsed to extract:
- All `@`-prefixed annotations at view and element level
- Element names and their expressions (direct columns, CASE, CAST, aggregates)
- `PARAMETERS` clause (→ Filter nodes)
- `ASSOCIATION` declarations (→ edge candidates)

### 3b. Structured annotation tables

Parsing raw DDL text is fragile. Where S/4HANA provides structured annotation storage (separate `DDANNOT`-family tables or equivalent), those are preferred over DDL text parsing. The exact table names are subject to verification.

### 3c. Data element texts — `DD04L` / `DD04T`

For elements without `@EndUserText` annotations (common in DDIC-backed views), the data element provides the fallback label:

| Column | Content | Used for |
|---|---|---|
| `ROLLNAME` | Data element name | Join key |
| `DDTEXT` | Short description | `name` fallback |
| `REPTEXT` | Column header text | `synonyms` candidate |

### 3d. Table / view catalog — `DD02L` / `DD02T`

Used for DDIC transparent table fallback (seed objects without a CDS view):

| Column | Content | Used for |
|---|---|---|
| `TABNAME` | Table/view name | Identity |
| `TABKAT` | Table category | `table_kind` heuristic |
| `DDTEXT` | Short description | `description` |

### 3e. ADT REST API (preferred for annotations)

Where the system has ADT (ABAP Development Tools) services enabled, the REST endpoint for DDL source and annotations is more reliable than direct table access — it applies activation-time resolution and handles include structures. Direct table reads via RFC are the fallback when ADT is unavailable.

```
GET /sap/bc/adt/ddic/ddl/sources/<DDLNAME>/source/main
```

---

## Step 4 — Transform: map to KG node types

Transformation applies the field mapping tables defined in [`sap-s4hana-technical.md`](sap-s4hana-technical.md) §Field mapping tables. This step produces a list of typed records ready for import.

### Annotation-to-node-type routing

| Annotation or signal | Target KG node type |
|---|---|
| `@Analytics.dataCategory: #FACT` or `#CUBE` | `Table` (fact) |
| `@Analytics.dataCategory: #DIMENSION` or `#TEXT` | `Table` (dimension) |
| `@Semantics.amount.*` or `@Semantics.quantity.*` on element | `Measure` |
| `@Analytics.dimension: true` on element | `Attribute` |
| CDS `PARAMETERS` without default | `Filter` (mandatory) |
| `@ObjectModel.filter.selectionType: #MANDATORY` | `Filter` (mandatory) |
| `@EndUserText.label` on view with no analytic annotation | `Subject` candidate — requires manual review |
| CDS `CASE` expression on an element | `BusinessRule` candidate |
| VDM lifecycle `#SAP_INTERNAL_USE_ONLY` | `status: Deprecated` |
| `@VDM.viewType: #CONSUMPTION` | `Table` — flag as consumption layer |
| `@VDM.viewType: #BASIC` or `#COMPOSITE` | `Table` — flag as interface layer |

### Disambiguation detection heuristics

A `Disambiguation` node is created when:
- The same element name appears in multiple views with different `@Semantics.*` annotations
- A date field (suffix `Date`) is present without `@Semantics.businessDate.*` clarifying its role
- An amount field is present without an associated `@Semantics.amount.currencyCode` reference

These are flagged as `REQUIRES MANUAL AUTHORING` — the `always_ask` text cannot be derived.

### Edge candidates from dependency walk

| Source | Discovered relation | KG edge kind |
|---|---|---|
| CDS `ASSOCIATION TO <Target>` | View references another view | `joinedTo` (candidate) |
| `C_<Entity>` over `I_<Entity>` | Consumption implements interface | `implement` |
| Element derived from association path | Source table calculates measure/attribute | `calculate` |
| `PARAMETERS` binding on view | Filter is mandatory for table | `mandatory` (reified candidate) |

Reified edge candidates (`mandatory`, `requires`) are created as stub Reification pages with `REQUIRES MANUAL AUTHORING` on `reason` and `consequence`. They are not considered verified until a domain expert completes both fields.

---

## Step 5 — Validate

Before writing to the content storage, each extracted record is validated:

1. **Required fields** — all fields marked required in [`spec/schema.yaml`](../../../spec/schema.yaml) are present or explicitly set to `REQUIRES MANUAL AUTHORING`
2. **Enum values** — `table_kind`, `kind`, `status`, `access_modifier` within allowed sets
3. **Name uniqueness** — checked against the KG node index (graph DB if available, else the JSON node index); conflicts are logged and skipped
4. **Annotation consistency** — a CDS element must not simultaneously carry `@Semantics.amount` and `@Analytics.dimension`; flag if found
5. **Seed reachability** — every imported object must be traceable to a seed entry via the discovery walk; orphaned objects are rejected

Validation failures are written to an import log, not silently dropped.

---

## Step 6 — Import

Follows the generic procedure in [`connectors.md`](../connectors.md) §Section 6:

1. **Node creation** — content storage pages with template sections; graph DB upsert via `MERGE (label, name)`
2. **Edge creation** — link statements per [`spec/link-format.md`](../../../spec/link-format.md); stub Reification pages for reified edges
3. **Duplicate handling** — existing `Active` node: skip and log; existing `Deprecated` node: create new, link with `relatedTo`
4. **Post-import audit** — audit rules from [`spec/schema.yaml`](../../../spec/schema.yaml)
5. **Version comment** on every created or updated page:

```
v1 | <YYYY-MM-DD> | sap-s4hana-import
Summary: Imported from SAP S/4HANA <release> — <source object name>
Changed: Initial import
Reason: Extracted from <CDS view / DDIC table / BW query>
Breaking: no
```

---

## Incremental extraction

The primary change signal is the ABAP transport system (SE09/SE10). A released transport containing CDS sources, DDIC changes, or Customizing entries triggers re-extraction of affected objects only.

Secondary signal: `DDDDLSRC` activation timestamp — compare against the last extraction timestamp stored in the importer state.

Objects touched by a transport but not in the current seed list are logged as candidates for seed expansion, not automatically imported.

---

## What cannot be extracted automatically

The following always require domain expert authoring after import:

| KG field | Node type | Reason |
|---|---|---|
| `always_ask` | `Disambiguation` | Depends on project-specific usage context |
| `consequence` on reified edges | `Reification` | Business impact not encoded in CDS metadata |
| `question` | `VerifiedQuery` | Natural-language question requires human formulation |
| `table_kind` | `Table` (DDIC fallback) | No `@Analytics.dataCategory` — heuristic only |
| `business_definition` | `Subject` | `@EndUserText` is technical; business meaning requires BRD |
| `reason` on `demonstrates` edges | `Reification` | Cannot be derived from structural metadata |

---

## Seed list

> Maintained manually. Add objects here to expand extraction scope.

| CDS view / DDIC table | Domain | Rationale |
|---|---|---|
| `I_AccountingDocument` | FI | Core FI document header |
| `I_AccountingDocumentItem` | FI | Core FI document line items |
| `C_CustomerOpenItemSrv` | FI-AR | Open receivables consumption view |
| `I_GLAccountLineItem` | FI | G/L line items |
| `I_PurchaseOrder` | MM | Purchase order header |
| `I_GoodsMovement` | MM | Goods movement |
| `I_MaterialStock` | MM | Stock overview |
| `I_SalesOrder` | SD | Sales order header |
| `C_BillingDocumentSrv` | SD | Billing document consumption view |
| `I_CostCenter` | CO | Cost center master |
| `I_InternalOrder` | CO | Internal order master |
| `BKPF` | FI | Accounting document header — DDIC fallback |
| `BSEG` | FI | Accounting document line items — DDIC fallback |

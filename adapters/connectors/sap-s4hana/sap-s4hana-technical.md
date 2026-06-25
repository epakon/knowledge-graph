# SAP S/4HANA — Technical Layer

> **Draft** — this document is work in progress and has not been formally reviewed.

This document covers the **Domain layer**: technical elements that implement S/4HANA business concepts in a specific version of the system — which CDS view holds the data, which field encodes the amount, which parameter is mandatory, which SQL expression computes the measure. The business meaning of each node lives in the Vocabulary layer; follow `implement <-` edges or see [`sap-s4hana-business.md`](sap-s4hana-business.md).

Technical objects — CDS views, DDIC tables, Fiori applications, transactions — are **concrete anchors**: they are where the business knowledge can be verified and tested.

---

## Node type mapping

| KG node type | SAP S/4HANA source | Technical anchor |
|---|---|---|
| `Subject` | Canonical business concept | VDM interface view (`I_<Entity>`); ABAP data element `@EndUserText.label` |
| `Domain` | SAP application area or functional module | Application component hierarchy (e.g. `FI`, `FI-AR`, `MM`, `SD`, `CO`) |
| `Table` | Business data entity used in queries | CDS view (preferred: `C_` consumption or `I_` interface); DDIC table as fallback |
| `Measure` | KPI or computed amount with a specific business definition | CDS element with `@Semantics.amount` / `@Semantics.quantity`; BW key figure; calculated key figure |
| `Attribute` | Dimension or characteristic used to slice measures | CDS element with `@Analytics.dimension`; ABAP characteristic |
| `Filter` | Selection condition required to scope a query correctly | CDS `PARAMETERS` clause; `@ObjectModel.filter.selectionType`; Fiori app mandatory selection field |
| `VerifiedQuery` | A business question with a validated SQL answer | BW query; OData `$filter/$select` from SAP API Hub; ABAP report with documented output |
| `BusinessRule` | A rule that governs how data must be filtered, signed, or aggregated | CDS `CASE` expression; RAP behavior definition validation; Customizing configuration value |
| `Disambiguation` | A term or field with context-dependent meaning | ABAP domain with overlapping values; dual-role business partner field; date field with multiple interpretations |
| `Relationship` | A dependency between two nodes that carries a business reason and consequence | CDS association with documented business rationale; reified only when reason + consequence can be stated |

---

## Domain layer — business processes

### Finance (FI)

**`Table` nodes**
- Accounting Document — a record of a complete business transaction. CDS views `I_AccountingDocument` (header) / `I_AccountingDocumentItem` (line items); DDIC tables `BKPF` / `BSEG`; Fiori app *Display Journal Entry* (F1369).
- Customer Open Item — an unpaid receivable line. CDS view `C_CustomerOpenItemSrv`; transaction `FBL5N`.
- Supplier Open Item — an unpaid payable line. CDS view `C_SupplierOpenItemSrv`; transaction `FBL1N`.
- G/L Account Line Item — a general ledger posting. CDS view `I_GLAccountLineItem`; transaction `FBL3N`.

**`Measure` nodes**
- Net Amount in Company Code Currency — `AmountInCompanyCodeCurrency` on accounting document item CDS views; sign applied via `DebitCreditCode`.
- Open Receivables — `SUM(signed AmountInCompanyCodeCurrency)` from `C_CustomerOpenItemSrv`.
- Days Sales Outstanding (DSO) — `(Open Receivables / Revenue) * days_in_period`; cross-domain, depends on `C_BillingDocumentSrv`.

**`Filter` nodes** (mandatory)
- Company Code — CDS parameter `P_CompanyCode`; `@ObjectModel.filter.selectionType: #MANDATORY`. Without it, amounts from different legal entities and currencies are mixed.
- Fiscal Year — CDS parameter `P_FiscalYear`. Without it, queries span multiple years.
- Posting Period — `FiscalYearPeriod` in CDS views; required for period-specific balance queries.

**`BusinessRule` nodes**
- Exclude Statistical Postings — `StatisticalIndicator = 'X'` or posting keys `09`/`19` are non-cash entries. Filter: `StatisticalIndicator = ' ' AND PostingKey NOT IN ('09','19')`.
- Open Item Definition — `ClearingDocument IS NULL`. Items with a clearing document have been paid or reversed.
- Debit/Credit Sign Convention — `DebitCreditCode = 'S'` is debit (positive); `'H'` is credit. Amounts stored unsigned; sign must be applied in aggregation.

---

### Accounts Receivable (FI-AR)

**`Table` nodes**
- Customer Master — business partner data in the customer role. CDS view `I_Customer`; transaction `XD03`.

**`Attribute` nodes**
- Dunning Level — `MAHNS` on customer open item views. Indicates how many payment reminders have been sent.

---

### Materials Management (MM)

**`Table` nodes**
- Purchase Order — CDS view `I_PurchaseOrder`; transaction `ME23N`; Fiori app *Manage Purchase Orders*.
- Goods Movement — CDS view `I_GoodsMovement`; transaction `MIGO`.
- Material Stock — CDS view `I_MaterialStock`; transaction `MMBE`.

**`BusinessRule` nodes**
- Three-Way Match — `PurchaseOrder`, `GoodsReceipt`, and `Invoice` must all match within tolerance before payment. Enforced by invoice verification (transaction `MIRO`).
- Moving Average vs. Standard Price — `PriceControl = 'V'` uses moving average; `'S'` uses standard price. Filter by `PriceControl` before aggregating costs.

---

### Sales and Distribution (SD)

**`Table` nodes**
- Sales Order — CDS view `I_SalesOrder`; transaction `VA03`; Fiori app *Sales Order*.
- Billing Document — CDS view `C_BillingDocumentSrv`; transaction `VF03`.

**`BusinessRule` nodes**
- Billing Status — `OverallBillingStatus = 'C'` means fully billed. Partial billing must not be counted as recognized revenue.
- Credit Block — `OverallDeliveryStatus = 'B'` means blocked. Do not count as committed revenue.

---

### Controlling (CO)

**`Table` nodes**
- Cost Center — CDS view `I_CostCenter`; transaction `KS03`.
- Internal Order — CDS view `I_InternalOrder`; transaction `KO03`.

**`BusinessRule` nodes**
- Primary vs. Secondary Costs — primary cost elements (`CElemCategory = '1'`) originate from FI; secondary (`'4'`, `'5'`) are CO-internal allocations. Filter by `CElemCategory` to avoid double-counting in cross-functional reports.

---

## Field mapping tables

### Subject ← VDM Interface View

| KG field | Source | Notes |
|---|---|---|
| `name` | CDS view name, strip `I_` prefix, title-case | e.g. `I_CompanyCode` → `Company Code` |
| `business_definition` | `@EndUserText.label` + `@EndUserText.quickInfo`; fallback to ABAP data element short text | Mark `REQUIRES MANUAL AUTHORING` if neither exists |
| `scope` | SAP application component assignment | Map component code to KG domain name |

### Domain ← SAP Application Component

| KG field | Source | Notes |
|---|---|---|
| `name` | Application component code (e.g. `FI-AR`) | Expand to full name if readable |
| `owner` | Responsible team from SAP Solution Manager | `REQUIRES MANUAL AUTHORING` if unavailable |

### Table ← CDS View

| KG field | Source | Notes |
|---|---|---|
| `name` | CDS view name, strip prefix, title-case | |
| `table_kind` | `@Analytics.dataCategory`: `#FACT`/`#CUBE` → `fact`; `#DIMENSION`/`#TEXT` → `dimension` | Default to `dimension` if annotation absent |
| `source` | CDS activation path: `_SYS_BIC/<package>/<view-name>` | Agents use this to construct SQL |
| `description` | `@EndUserText.label` | |
| `status` | `@VDM.lifecycle.contract.type`: `#CLOUD_PUBLIC`/`#RESTRICTED_REUSE` → `Active`; `#SAP_INTERNAL_USE_ONLY` → `Deprecated` | Custom `Z*`/`Y*` views default to `Active` |

### Table ← DDIC Transparent Table (fallback)

| KG field | Source | Notes |
|---|---|---|
| `name` | Table name (e.g. `BKPF`) | Uppercase as-is |
| `table_kind` | Heuristic from table category (`DD02L.TABKAT`) | Requires domain expert review |
| `source` | `<ABAP_schema>.<table_name>` | |
| `description` | Short text from `DD02T` | |

### Measure ← CDS Element or BW Key Figure

| KG field | Source | Notes |
|---|---|---|
| `name` | CDS element name or BW technical name, title-case | e.g. `NetAmountInTransactionCurrency` → `Net Amount in Transaction Currency` |
| `kind` | `@Semantics.amount`/`@Semantics.quantity` → `aggregate expression`; `@Analytics.indicator: #CALCULATED` → `derived formula` | |
| `synonyms` | `@EndUserText.label` in available languages | |
| `status` | Same mapping as Table `status` | |
| `definition_sql` | CDS element expression; BW formula translated to SQL | Flag `REQUIRES MANUAL REVIEW` if approximated |

### Attribute ← CDS Dimension Element

| KG field | Source | Notes |
|---|---|---|
| `name` | CDS element with `@Analytics.dimension`, title-case | |
| `kind` | `@Semantics.businessDate.*` → `time_dimension`; `@Semantics.organizational.*` → `dimension`; `@Semantics.amount` → `fact` | |
| `access_modifier` | `@ObjectModel.readOnly: true` or `#SAP_INTERNAL_USE_ONLY` → `private_access`; otherwise `public_access` | |
| `expression_sql` | CDS CASE/CAST expression | Omit if direct column reference |

### Filter ← CDS Parameter or Mandatory Selection Condition

| KG field | Source | Notes |
|---|---|---|
| `name` | CDS parameter name, strip `P_`, title-case | |
| `mandatory` | Parameter has no default value and query fails without it; or `@ObjectModel.filter.selectionType: #MANDATORY` | Also check Fiori app manifest for mandatory selection fields |
| `predicate_sql` | CDS parameter binding translated to SQL WHERE fragment | e.g. `CompanyCode = :P_CompanyCode` → `WHERE CompanyCode = '<value>'` |
| `synonyms` | `@EndUserText.label` | |

### VerifiedQuery ← BW Query / OData / ABAP Report

| KG field | Source | Notes |
|---|---|---|
| `name` | Descriptive name from BW query description, SAP API Hub, or report header | |
| `onboarding_question` | Manual flag | `false` by default |
| `verified_by` | Transport owner or report author | |
| `verified_at` | Transport request date or last activation date | ISO date |
| `question` | SAP API Hub description or BW query description | Manual curation required — do not auto-generate |
| `sql` | OData `$filter/$select` or BW MDX translated to SQL | Must be valid SQL against `_SYS_BIC` path |

### BusinessRule ← CDS Expression / RAP Behavior / Customizing

| KG field | Source | Notes |
|---|---|---|
| `name` | Rule name from behavior definition, CDS derivation, or Customizing key | Descriptive; no abbreviations |
| `definition` | ABAP `DETERMINATION`/`VALIDATION` body; CDS `CASE`/`IIF`; Customizing value | Exact column names and filter expressions — no prose |
| `consequence_if_violated` | Business impact from SAP Note, ABAP dump text, or domain expert | One sentence; quantify if possible — requires manual authoring for most rules |

### Disambiguation ← Ambiguous Field or Concept

| KG field | Source | Notes |
|---|---|---|
| `name` | Field name or concept pair, e.g. `Posting Date vs. Document Date` | |
| `always_ask` | Manually authored clarifying question | Cannot be auto-generated; domain expert required |

---

## Edge mapping

### Hyperlink edges

| KG edge kind | SAP source relationship | Technical anchor | Cardinality |
|---|---|---|---|
| `relatedTo` | Conceptual relationship without a binding dependency | CDS association without foreign key; BW navigation attribute | N:M |
| `joinedTo` | Structural join between two entities | CDS `ASSOCIATION ... TO ... ON`; DDIC foreign key | N:M |
| `calculate` | A table is the source of a measure or attribute | CDS element derived from that table's fields | 1:N |
| `implement` | A domain entity implements a business concept | `C_<Entity>` implements `I_<Entity>`; BW query implements a key figure | N:M |
| `apply` | A business rule governs a table or measure | RAP `VALIDATION` scoped to a CDS entity; Customizing rule in IMG | N:M |
| `contain` | An application area contains data entities | Application component hierarchy | 1:N |
| `disambiguate` | A subject has a context-dependent meaning | ABAP domain with overlapping fixed values | 1:N |

### Reified edges (require Relationship page)

| KG edge kind | SAP business situation | `reason` + `consequence` |
|---|---|---|
| `mandatory` | A filter that must be applied for a query to be legally, organizationally, or semantically valid | `reason`: derive from `@EndUserText.quickInfo` or parameter description; `consequence`: manual authoring |
| `requires` | A measure's calculation depends on a filter to scope its population correctly | `reason`: from CDS expression or parameter description; `consequence`: incorrect aggregation if omitted |
| `guards` | A filter restricts the valid rows for a measure | Manual authoring — cannot be derived from metadata |
| `overrides` | A business rule or Customizing setting overrides a default field value | `reason`: from RAP behavior definition comment; `consequence`: manual authoring |
| `demonstrates` | A verified query provides a working example of a business rule in action | Manual authoring |

For `guards` and `demonstrates`, stub Relationship pages with `REQUIRES MANUAL AUTHORING` placeholders are created on import. A domain expert must complete these before the edge is considered verified.

---

## Extraction protocol

| What to extract | Where it lives | How to read it |
|---|---|---|
| Business concept labels and definitions | CDS annotations `@EndUserText.label`, `@EndUserText.quickInfo` | ADT REST API: `GET /sap/bc/adt/ddic/ddl/sources/<view>/source/main`; or ABAP export report |
| Data entity structure and semantics | CDS VDM views (`I_`, `C_`); `@Analytics.dataCategory`, `@Semantics.*` | OData `$metadata` EDMX; ADT annotation extraction |
| Mandatory selection conditions | CDS `PARAMETERS` without default; `@ObjectModel.filter.selectionType: #MANDATORY` | OData metadata; Fiori app manifest `selectionFields` |
| Business rules and derivations | RAP behavior definitions; CDS `CASE` expressions; Customizing values | ADT REST for behavior definitions; Customizing table reads via RFC |
| Verified query examples | BW queries; OData services in SAP API Hub; standard ABAP reports | BW query designer export; SAP API Hub documentation |
| Fallback table structures | DDIC transparent tables when no CDS view exists | RFC `DDIF_FIELDINFO_GET`; `DD_TABL_GET` |

**What cannot be extracted automatically** — always requires domain expert authoring:
- `always_ask` text for Disambiguation nodes.
- `consequence` for most reified edges.
- `question` text for VerifiedQuery nodes.
- `table_kind` for DDIC tables without `@Analytics.dataCategory`.
- Business rules held in ABAP procedural code rather than declarative CDS expressions.

**Incremental detection:** use the ABAP transport system (SE09/SE10) as the change signal. A released transport containing CDS sources, DDIC changes, or Customizing entries triggers re-extraction of affected nodes.

---

## Import procedure

1. **Validation** — required fields present per [`spec/schema.yaml`](../../spec/schema.yaml); enum values within allowed set; name uniqueness checked against node index.
2. **Node creation** — page title `<NodeType>: <name>`; container path per [`spec/space-structure.md`](../../spec/space-structure.md); template sections populated per [`spec/page-templates.md`](../../spec/page-templates.md).
3. **Edge creation** — link statements following [`spec/link-format.md`](../../spec/link-format.md); stub Relationship pages created with `REQUIRES MANUAL AUTHORING` for edges that cannot be auto-derived.
4. **Duplicate handling** — existing `Active` node: skip and log; existing `Deprecated` node: create new and link with `relatedTo`.
5. **Post-import audit** — run audit rules from [`spec/schema.yaml`](../../spec/schema.yaml).
6. **Version comment** — structured comment following [`spec/versioning.md`](../../spec/versioning.md) on every create or update:

```
v1 | <YYYY-MM-DD> | sap-s4hana-adapter
Summary: Imported from SAP S/4HANA <release> — <source object name>
Changed: Initial import
Reason: Extracted from <CDS view / DDIC table / BW query>
Breaking: no
```

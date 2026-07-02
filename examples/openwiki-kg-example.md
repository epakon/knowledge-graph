# OpenWiki → KG Draft: Table Node Example

> This file shows what an OpenWiki-generated KG draft node looks like when the agent is prompted to follow `spec/page-templates.md` and `spec/schema.yaml` directly. It is based on a real dbt reporting view over SAP FI-CA tables DFKKOP and DFKKKO, with company-specific details removed.
>
> **How to read this file:** Fields the LLM can extract from source code are filled in. Fields that require domain expert knowledge are left as `<!-- TODO -->` placeholders. A reviewer's job is to fill in the TODOs and change the status comment to `active`.

---

# Table: fact_document_items_with_clearing

**Type:** Table
**TableKind:** fact
**Domain:** [Domain: FinancialAccounting](../domain)
**Source:** dbt model — `models/reporting/fact_document_items_with_clearing.sql` (materialized as view)

## Description
Reporting view that combines original FI-CA document items (DFKKOP) with their corresponding clearing counterparts. For every document item that carries a non-empty `ClearingDocumentNumber`, a synthetic mirror row is produced by joining to the document header (DFKKKO) of the clearing document and negating all monetary amounts. The original item row is also included with amounts as-is. The union of both sets is then enriched with customer dimension attributes via a left join on `BusinessPartner`. The result enables clearing-aware financial analysis: summing amounts over any clearing pair yields a net-zero balance.

## Fields

### Physical columns
| Column | PK | Type | Description |
|--------|----|------|-------------|
| DocumentNumber | | string | Document number (DFKKOP.OPBEL). For cleared rows sourced from the clearing document header; for original rows sourced from the fact item. |
| BusinessPartner | | string | Business partner identifier (DFKKOP.GPART); join key to customer dimension. |
| DocumentType | | string | Document type code (DFKKOP.BLART). For cleared rows sourced from the clearing document header. |
| DocumentDate | | date | Document date (DFKKOP.BLDAT). For cleared rows taken from the clearing document header. |
| PostingDate | | date | Posting date (DFKKOP.BUDAT). For cleared rows taken from the clearing document header. |
| TransactionCurrency | | string | ISO currency code for transaction amounts (DFKKOP.WAERS). |
| ClearingDocumentNumber | | string | Document number of the clearing document (DFKKOP.AUGBL); empty string for original (non-cleared) rows. |
| ClearingDate | | date | Date on which the item was cleared (DFKKOP.AUGDT). |
| ClearingStatus | | string | Clearing status indicator (DFKKOP.AUGST). |
| NetDueDate | | date | Net due date of the item (DFKKOP.FAEDN). |
| ItemText | | string | Item text (DFKKOP.OPTXT). NULL for cleared mirror rows. |
| CompanyCode | | string | Company code (DFKKOP.BUKRS). |
| LocalCurrency | | string | Local (company code) currency code. |
| ReferenceDocument | | string | Reference document number (DFKKOP.XBLNR). For cleared rows taken from the clearing header. |
| ContractAccount | | string | Contract account identifier (DFKKOP.VKONT). |
| AmountInTransactionCurrency | | decimal | Amount in transaction currency (DFKKOP.BETRW). Negated (×−1) for cleared mirror rows. |
| AmountInLocalCurrency | | decimal | Amount in local currency (DFKKOP.BETRH). Negated (×−1) for cleared mirror rows. |
| ClearedDoc | | string | For cleared mirror rows: the `DocumentNumber` of the original document that was cleared. Empty string for original rows. |
| FiscalYearPeriod | | string | Fiscal year-period in `YYYY0MM` format. For cleared rows derived from the clearing document header's date; for original rows taken from the fact item. |
| customer_PartnerNumber | | string | Business partner number from the customer dimension (SAP BP tables). Join key used to link to this row. |
| customer_Name | | string | Customer name. |
| customer_CountryCode | | string | Country code. |
| customer_CountryName | | string | Country name. |
| customer_Region | | string | Region. |
| customer_ValidFrom | | date | Customer record validity start date. |
| customer_ValidTo | | date | Customer record validity end date. |

### Semantic annotations
| Column | Kind | Synonym(s) | Notes | Calculated |
|--------|------|-----------|-------|------------|
| DocumentNumber | dimension | Doc number | Identifies the document (DFKKOP.OPBEL). | — |
| ClearingDocumentNumber | dimension | Clearing doc | Empty string for original rows; used as join key in cleared CTE (DFKKOP.AUGBL). | — |
| BusinessPartner | dimension | BP, Partner | Join key to customer dimension (DFKKOP.GPART). | — |
| ContractAccount | dimension | Contract acct | Contract account (DFKKOP.VKONT). | — |
| DocumentType | dimension | Doc type | — | — |
| CompanyCode | dimension | CoCd | — | — |
| ClearingStatus | dimension | Clearing status | — | — |
| ClearedDoc | dimension | Cleared document | For cleared rows: the original document that was offset. Empty string for original rows. | — |
| FiscalYearPeriod | time_dimension | Fiscal period | Format YYYY0MM. For cleared rows computed from clearing document header's date. | — |
| DocumentDate | time_dimension | Doc date | For cleared rows: from clearing document header (DFKKOP.BLDAT). | — |
| PostingDate | time_dimension | Posting date | For cleared rows: from clearing document header (DFKKOP.BUDAT). | — |
| ClearingDate | time_dimension | Clearing date | Date the item was cleared (DFKKOP.AUGDT). | — |
| NetDueDate | time_dimension | Due date | — | — |
| AmountInTransactionCurrency | fact | Amount (txn ccy) | Negated for cleared mirror rows (DFKKOP.BETRW). | — |
| AmountInLocalCurrency | fact | Amount (local ccy) | Negated for cleared mirror rows (DFKKOP.BETRH). | — |

## Relationships
<!-- TODO: Add Relationship pages once mandatory filters and business rules are identified by a domain expert. -->

## Joins
- [Table: fact_document_item_base](../fact_document_item_base) — Table: fact_document_items_with_clearing joinedTo -> Table: fact_document_item_base on ClearingDocumentNumber = header_DocumentNumber (cleared CTE only; INNER JOIN)
- [Table: dim_document_header](../dim_document_header) — Table: fact_document_items_with_clearing joinedTo -> Table: dim_document_header on ClearingDocumentNumber = header_DocumentNumber (INNER JOIN, cleared CTE only; rows with empty ClearingDocumentNumber excluded by WHERE filter)
- [Table: dim_customer](../dim_customer) — Table: fact_document_items_with_clearing joinedTo -> Table: dim_customer on BusinessPartner = customer_PartnerNumber (LEFT JOIN on final union)

## Caveats
- **Dual-row pattern:** Every cleared document item appears twice — once as the original row (amounts as-is, `ClearedDoc` = `''`) and once as the clearing mirror row (amounts negated, `ClearedDoc` = original `DocumentNumber`). Summing amounts without filtering will double-count unless this pattern is explicitly accounted for.
- **Amount sign convention:** All amount columns are multiplied by −1 for cleared mirror rows. The original row retains the sign from the source fact table.
- **WHERE filter on cleared CTE:** Only fact rows where `ClearingDocumentNumber <> ''` are included in the clearing mirror CTE. Rows with an empty clearing document number still appear via the original items CTE.
- **FiscalYearPeriod derivation for cleared rows:** Computed from the *clearing document header's* date, not from the original fact item.
- **NULL columns in cleared mirror rows:** `ItemText`, `ReferenceDocument`, and other header-derived fields are explicitly set to NULL for clearing mirror rows — see Physical columns table.
- **Customer join is LEFT:** Unmatched `BusinessPartner` values produce NULL for all `customer_*` columns; such rows are still included.
- **Materialization:** This is a dbt `view` — no physical table is written; the full SQL executes at query time.

## Links
<!-- TODO: Add Attribute and Measure links once promoted columns and measures are defined for this domain. -->

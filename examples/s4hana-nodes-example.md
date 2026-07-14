# SAP S/4HANA Node Examples — FI-AR Domain

> Illustrative examples showing how SAP S/4HANA source objects map to Knowledge Graph pages.  
> Domain: **FI-AR** (Accounts Receivable)  
> Source system: SAP S/4HANA 2023, CDS VDM layer

These examples follow the page templates in `spec/page-templates.md` and the field mappings in `adapters/connectors/sap-s4hana/sap-s4hana-technical.md`.

---

## Subject: Customer

**Extracted from:** VDM interface view `I_BusinessPartner` + `I_Customer`  
**`@EndUserText.label`:** "Business Partner — Customer role"

---

# Subject: Customer

**Type:** Subject  
**Scope:** global

## Business Definition

A party that has a commercial relationship with the organization in the role of a buyer of goods or services. In SAP S/4HANA, a customer is a Business Partner assigned the BP role `FLCU00` (FI Customer) or `FLCU01` (Customer with Order Management).

## Citations

- SAP Help: Business Partner Concept in S/4HANA — `I_BusinessPartner` VDM documentation
- SAP Note 2265093 — Business Partner migration from classic customer master

## Links

- [Subject: Customer implement -> Domain: FI-AR]
- [Subject: Customer disambiguate -> Disambiguation: Customer vs. Sold-to Party]
- [Subject: Customer relatedTo -> Subject: Business Partner]

---

## Domain: FI-AR

**Extracted from:** SAP application component `FI-AR`

---

# Domain: FI-AR

**Type:** Domain  
**Owner:** Accounts Receivable Team

## Tables

- [Table: Customer Open Item]
- [Table: Accounting Document Header]
- [Table: Accounting Document Item]
- [Table: Customer Master]

## Measures

- [Measure: Open Receivables Amount]
- [Measure: Days Sales Outstanding]
- [Measure: Overdue Receivables Amount]

## Filters

- [Filter: Company Code]
- [Filter: Fiscal Year]
- [Filter: Customer Account]
- [Filter: Posting Date Range]

## Verified Queries

- [VerifiedQuery: Open Receivables by Customer and Company Code]

## Rules

- [BusinessRule: Exclude Statistical Postings from Receivables]
- [BusinessRule: Overdue Threshold — 30 Days Net]

## Disambiguations

- [Disambiguation: Customer vs. Sold-to Party]
- [Disambiguation: Posting Date vs. Document Date]

---

## Table: Customer Open Item

**Extracted from:** CDS view `C_CustomerOpenItemSrv` / underlying `I_OpenItemFiDocItem`  
**`@Analytics.dataCategory`:** `#FACT`  
**Source path:** `_SYS_BIC/sap.s4.beh.finance.v1/C_CUSTOMEROPENITEMSRV`

---

# Table: Customer Open Item

**Type:** Table  
**TableKind:** fact  
**Domain:** FI-AR  
**Source:** `_SYS_BIC/sap.s4.beh.finance.v1/C_CUSTOMEROPENITEMSRV`

## Description

Fact view exposing open (unpaid) Accounts Receivable line items. Each row represents one open posting line from a customer billing document or manual journal entry that has not yet been cleared.

## Physical Columns

| Column | Data Element | Key | Notes |
|---|---|---|---|
| `CompanyCode` | `BUKRS` | PK | ABAP type `CHAR(4)` |
| `AccountingDocument` | `BELNR_D` | PK | |
| `FiscalYear` | `GJAHR` | PK | |
| `AccountingDocumentItem` | `BUZEI` | PK | 3-digit line item number |
| `Customer` | `KUNNR` | | 10-char customer account |
| `NetDueDate` | `FAEDT` | | |
| `AmountInTransactionCurrency` | `WRBTR` | | Signed; credit = negative |
| `TransactionCurrency` | `WAERS` | | ISO 4217 |
| `AmountInCompanyCodeCurrency` | `DMBTR` | | |
| `CompanyCodeCurrency` | `HWAER` | | |
| `PostingDate` | `BUDAT` | | |
| `DocumentDate` | `BLDAT` | | |
| `DebitCreditCode` | `SHKZG` | | `S` = debit, `H` = credit |
| `ClearingDocument` | `AUGBL` | | Non-null = cleared item |
| `ClearingDate` | `AUGDT` | | |

## Semantic Annotations

| Column | Kind | Synonyms | Calculated links |
|---|---|---|---|
| `AmountInTransactionCurrency` | fact | Transaction Amount, Invoice Amount | [Measure: Open Receivables Amount calculate <- Table: Customer Open Item] |
| `NetDueDate` | time_dimension | Due Date, Payment Due Date | |
| `Customer` | dimension | Customer Account, Debtor | |
| `CompanyCode` | dimension | Company, Legal Entity | [Disambiguation: Customer vs. Sold-to Party] |
| `DebitCreditCode` | dimension | DC Indicator | [BusinessRule: Exclude Statistical Postings from Receivables apply <- Table: Customer Open Item] |

## Joins

- [Table: Customer Open Item joinedTo -> Table: Accounting Document Header] on `CompanyCode = CompanyCode AND AccountingDocument = AccountingDocument AND FiscalYear = FiscalYear`
- [Table: Customer Open Item joinedTo -> Table: Customer Master] on `Customer = Customer`

## Links

- [Domain: FI-AR contain -> Table: Customer Open Item]
- [Filter: Company Code mandatory -> Table: Customer Open Item]
- [Filter: Fiscal Year mandatory -> Table: Customer Open Item]

## Caveats

- `AmountInTransactionCurrency` is signed (debit positive, credit negative). Always apply `DebitCreditCode` filter or SUM with sign to avoid netting errors.
- Cleared items (`ClearingDocument IS NOT NULL`) represent paid invoices. The Open Item view filters these out by default via the CDS view's built-in `WHERE` clause — do not add a manual `ClearingDocument IS NULL` unless querying the full document flow view.

---

## Measure: Open Receivables Amount

**Extracted from:** CDS element `AmountInCompanyCodeCurrency` in `C_CustomerOpenItemSrv` with `@Semantics.amount.currencyCode: 'CompanyCodeCurrency'`

---

# Measure: Open Receivables Amount

**Type:** Measure  
**Domain:** FI-AR  
**Kind:** aggregate expression  
**Synonyms:** AR Balance, Outstanding Receivables, Open AR  
**Status:** Active

## Definition

```sql
SUM(
  CASE DebitCreditCode
    WHEN 'S' THEN  AmountInCompanyCodeCurrency
    WHEN 'H' THEN -AmountInCompanyCodeCurrency
    ELSE 0
  END
)
FROM _SYS_BIC."sap.s4.beh.finance.v1/C_CUSTOMEROPENITEMSRV"
```

Currency: `CompanyCodeCurrency` (local currency of the company code).

## Reifications

- [Reification: Open Receivables Amount requires Company Code Filter]
- [Reification: Open Receivables Amount requires Fiscal Year Filter]

## Links

- [Table: Customer Open Item calculate -> Measure: Open Receivables Amount]
- [Domain: FI-AR contain -> Measure: Open Receivables Amount]
- [Subject: Customer implement -> Measure: Open Receivables Amount]

---

## Measure: Days Sales Outstanding

**Extracted from:** Derived formula — no direct CDS element; derived from `Open Receivables Amount` and `Revenue` (cross-domain)

---

# Measure: Days Sales Outstanding

**Type:** Measure  
**Domain:** FI-AR  
**Kind:** derived formula  
**Synonyms:** DSO, Debtor Days, Collection Period  
**Status:** Active

## Definition

```sql
(Open Receivables Amount / Revenue) * <days_in_period>
```

Where:
- `Open Receivables Amount` = `SUM(signed AmountInCompanyCodeCurrency)` from `C_CustomerOpenItemSrv`
- `Revenue` = net billing amount from `C_BillingDocumentSrv` (FI-SD domain)
- `days_in_period` = calendar days in the reporting period

Full SQL:

```sql
(
  SUM(CASE oi.DebitCreditCode WHEN 'S' THEN oi.AmountInCompanyCodeCurrency
      ELSE -oi.AmountInCompanyCodeCurrency END)
  /
  NULLIF(SUM(bi.NetAmountInCompanyCodeCurrency), 0)
) * <days_in_period>
FROM _SYS_BIC."sap.s4.beh.finance.v1/C_CUSTOMEROPENITEMSRV" oi
JOIN _SYS_BIC."sap.s4.beh.sd.v1/C_BILLINGDOCUMENTSRV" bi
  ON oi.CompanyCode = bi.CompanyCode
```

> Cross-domain measure. `Revenue` belongs to domain `FI-SD`. See [Subject: Revenue] for definition.

## Reifications

- [Reification: Days Sales Outstanding requires Company Code Filter]
- [Reification: Days Sales Outstanding requires Fiscal Year Filter]

## Links

- [Domain: FI-AR contain -> Measure: Days Sales Outstanding]
- [Measure: Days Sales Outstanding relatedTo -> Measure: Open Receivables Amount]

---

## Filter: Company Code

**Extracted from:** CDS parameter `P_CompanyCode` on `C_CustomerOpenItemSrv`  
**`@ObjectModel.filter.selectionType`:** `#SINGLE`  
**Default value:** none

---

# Filter: Company Code

**Type:** Filter  
**Domain:** FI-AR  
**Mandatory:** true  
**Synonyms:** Company, BUKRS, Legal Entity, Buchungskreis

## Predicate

```sql
WHERE CompanyCode = '<P_CompanyCode>'
```

## Reifications

- [Reification: Company Code Filter mandatory for Customer Open Item]
- [Reification: Open Receivables Amount requires Company Code Filter]

## Links

- [Domain: FI-AR contain -> Filter: Company Code]
- [Subject: Company Code implement -> Filter: Company Code]

---

## BusinessRule: Exclude Statistical Postings from Receivables

**Extracted from:** CDS filter in `I_OpenItemFiDocItem`: `PostingKey NOT IN ('09', '19')` and `StatisticalIndicator = ' '`

---

# Rule: Exclude Statistical Postings from Receivables

**Type:** BusinessRule  
**Domain:** FI-AR

## Definition

```sql
StatisticalIndicator = ' '
AND PostingKey NOT IN ('09', '19')
```

`StatisticalIndicator = 'X'` marks a posting as statistical only (used for reporting, does not affect balance). Posting keys 09 and 19 are internal SAP statistical entries.

## Consequence if Violated

Including statistical postings inflates the Open Receivables Amount by non-cash items, overstating AR balance by up to 15% in environments that use cross-company intercompany netting.

## Links

- [Rule: Exclude Statistical Postings from Receivables apply -> Table: Customer Open Item]
- [Rule: Exclude Statistical Postings from Receivables apply -> Measure: Open Receivables Amount]
- [Domain: FI-AR contain -> Rule: Exclude Statistical Postings from Receivables]

---

## Disambiguation: Customer vs. Sold-to Party

**Source:** Manual authoring — identified during extraction of `I_Customer` and `I_SalesOrder` where both reference a party with different roles

---

# Disambiguation: Customer vs. Sold-to Party

**Type:** Disambiguation  
**Domain:** FI-AR

## Always Ask

"Do you mean the **customer** in the accounting sense (the party that owes money, identified by `Customer` / `KUNNR` in AR), or the **sold-to party** (the party that placed the sales order, identified by `SoldToParty` / `KUNNR` in SD)? These are the same KUNNR field but filtered by business partner role — the answer changes which CDS view to query."

## Links

- [Subject: Customer disambiguate -> Disambiguation: Customer vs. Sold-to Party]

---

## VerifiedQuery: Open Receivables by Customer and Company Code

**Source:** OData query from SAP API Business Hub — `API_CUSTOMER_OPEN_ITEMS_SRV`  
**Verified by:** AR Team Lead  
**Verified at:** 2024-11-15

---

# VerifiedQuery: Open Receivables by Customer and Company Code

**Type:** VerifiedQuery  
**Domain:** FI-AR  
**Onboarding question:** true  
**Verified by:** AR Team Lead  
**Verified at:** 2024-11-15  
**Status:** Active

## Question

"What is the total open receivables amount per customer for a given company code and fiscal year?"

## SQL

```sql
SELECT
    CompanyCode,
    Customer,
    SUM(
        CASE DebitCreditCode
            WHEN 'S' THEN  AmountInCompanyCodeCurrency
            WHEN 'H' THEN -AmountInCompanyCodeCurrency
            ELSE 0
        END
    )                          AS OpenReceivablesAmount,
    CompanyCodeCurrency
FROM
    _SYS_BIC."sap.s4.beh.finance.v1/C_CUSTOMEROPENITEMSRV"
WHERE
    CompanyCode  = '<P_CompanyCode>'   -- mandatory
    AND FiscalYear = '<P_FiscalYear>'  -- mandatory
    AND StatisticalIndicator = ' '     -- Rule: Exclude Statistical Postings
    AND PostingKey NOT IN ('09', '19') -- Rule: Exclude Statistical Postings
GROUP BY
    CompanyCode,
    Customer,
    CompanyCodeCurrency
ORDER BY
    OpenReceivablesAmount DESC
```

## Reifications

- [Reification: Open Receivables by Customer demonstrates Exclude Statistical Postings Rule]

## Links

- [Domain: FI-AR contain -> VerifiedQuery: Open Receivables by Customer and Company Code]
- [Measure: Open Receivables Amount implement -> VerifiedQuery: Open Receivables by Customer and Company Code]

---

## Reification pages (reified edges)

### Reification: Company Code Filter mandatory for Customer Open Item

# Reification: Company Code Filter mandatory for Customer Open Item

**Type:** Reification  
**Kind:** mandatory  
**From:** Filter: Company Code  
**To:** Table: Customer Open Item

## Reason

`C_CustomerOpenItemSrv` does not apply a default company code filter in its CDS `WHERE` clause. Without an explicit `CompanyCode` parameter, the query returns data across all company codes in the system, crossing legal entity boundaries and potentially returning data the querying user is not authorized to see.

## Consequence

Omitting the company code filter returns data from all company codes, violating data segregation requirements and producing meaningless cross-entity totals.

---

### Reification: Open Receivables Amount requires Company Code Filter

# Reification: Open Receivables Amount requires Company Code Filter

**Type:** Reification  
**Kind:** requires  
**From:** Measure: Open Receivables Amount  
**To:** Filter: Company Code

## Reason

`Open Receivables Amount` is defined in company code currency (`AmountInCompanyCodeCurrency`). Aggregating across company codes mixes currencies, producing mathematically incorrect totals unless explicit multi-currency conversion is applied.

## Consequence

Aggregating Open Receivables Amount across company codes without a currency conversion layer produces a sum of amounts in different currencies, which is economically meaningless and will fail audit validation.

# SAP S/4HANA — Business Layer

> **Draft** — this document is work in progress and has not been formally reviewed.

SAP S/4HANA is a suite of integrated business processes — finance, procurement, manufacturing, sales, and more. Each process produces and consumes business knowledge: what a company code is, how revenue is recognized, which filters must be applied before an amount is meaningful, what distinguishes an open item from a cleared one.

This document covers the **Vocabulary layer**: business process concepts that are stable, domain-agnostic, and independent of the data model. These are the *what* and *why* — what a concept means, why a term is ambiguous. Each concept is bridged to its technical implementation in the Domain layer via `implement ->` edges; for the field-level mappings, extraction protocol, and import procedure see [`sap-s4hana-technical.md`](sap-s4hana-technical.md).

> **Note on `Disambiguation` nodes:** The examples below include `Disambiguation` nodes at the Vocabulary layer. Whether disambiguation content belongs on a dedicated `Disambiguation` node or inline on the related `Subject` page has not been decided yet — it may be sufficient to describe an ambiguous term directly on the `Subject` node that owns the concept. The `Disambiguation` entries here are illustrative; treat them as candidates until the pattern is established from more real examples.

Two kinds of knowledge belong here:

- **SAP process concepts** — what the standard system defines: Company Code, Posting Period, Purchase Order. These are stable across implementations.
- **Project decisions** — how a specific company configured, extended, or constrained the system. These come from BRDs and functional specs and must be explicitly authored because they exist nowhere else. Typical examples:
  - *Organizational scope* — which company codes, controlling areas, or plant codes are in scope for a given process, and why.
  - *Document type conventions* — which SAP document types are used for which business events, and which must be excluded from reporting.
  - *Fiscal year and currency* — whether the implementation uses calendar year or a custom variant; which currency is the reporting standard.
  - *Classification models* — company-defined segmentations or partner categories built on top of SAP fields, often pre-computed by an external system.
  - *Canonical field decisions* — where two SAP fields appear to answer the same question but only one is correct for this implementation.

---

## Finance (FI)

Finance covers the recording and reporting of all monetary transactions across legal entities.

**`Subject` nodes — SAP process concepts**
- Company Code — the smallest organizational unit for which a complete, self-contained set of accounts can be drawn up.
- Fiscal Year Variant — defines the fiscal year structure (calendar vs. non-calendar, number of posting periods).
- Posting Period — a time window within a fiscal year during which transactions can be posted.
- Chart of Accounts — the structured list of G/L accounts used by a company code.
- Document Type — a two-character key that classifies the business event behind an accounting document (invoice, payment, reversal, adjustment). Controls number ranges and account types allowed.

**`Subject` nodes — project decisions**
- Reporting Currency *(project-specific)* — the BRD defines one currency as the standard for all financial reporting across company codes. All measures are converted to this currency at the time of posting using the exchange rate type defined in configuration. Queries that mix transaction currencies without applying this conversion produce incomparable totals.
- Reversal Document *(project-specific)* — the project defined which document types represent reversals of prior postings. Reversals must be excluded from most financial measures to avoid double-counting; the canonical exclusion list is a configuration decision documented in the BRD.

**`Disambiguation` nodes**
- Posting Date vs. Document Date — `PostingDate` is when the entry was recorded in the system; `DocumentDate` is the date on the original business document. Period-based reporting uses posting date; document-age analysis uses document date.
- Company Code vs. Legal Entity — the SAP accounting unit does not always map 1:1 to a legal entity in group consolidation. The mapping is defined in the organizational structure BRD.

---

## Contract Accounts Receivable (FI-CA)

FI-CA manages high-volume receivables for companies with large partner or customer bases. Unlike FI-AR, it operates on contract accounts rather than customer accounts and is commonly extended with company-specific classifications and document type conventions.

**`Subject` nodes — SAP process concepts**
- Contract Account — the central object holding payment terms, dunning procedure, and correspondence settings for a business partner's contractual relationship.
- Open Item — an unpaid line item on a contract account that has not yet been cleared.
- Clearing — the process of matching a payment or credit against one or more open items.
- Write-Off — a receivable determined to be uncollectable and removed from the open AR balance.

**`Subject` nodes — project decisions**
- Write-Off *(project-specific)* — in this implementation, a write-off is recorded when a document is cleared using a `BO` clearing document type. Two structurally different questions use this concept — the total value written off and the timeliness of the write-off event — and they require different SQL filters. The choice of `BO` as the write-off marker is a configuration decision documented in the BRD.
- High-Value Partner *(project-specific)* — an accommodation partner classified as high-value based on `PARTNER_SEGMENTATION = 'High-value'`. The `IS_KEY_ACCOUNT` field exists in the system but is unreliable across segments and must not be used. The canonical field was established during the partner segmentation project.
- PBB Payment Model *(project-specific)* — the company's classification of partner payment arrangements (`BT Net`, `BT Gross`, `VCC`, `AGENCY`, and variants), pre-computed by an internal data pipeline from three raw SAP fields. Queries must always use the pre-computed `PAYMENT_MODEL` attribute — manual re-derivation produces inconsistent results.

**`Disambiguation` nodes**
- Bad Debt — may mean receivables already written off (cleared via BO, gone from open AR) or open aged receivables expected to be uncollectable (still active, used as a provision proxy). The two populations are distinct and must not be mixed.
- Write-Off vs. BO Entry — `ClearDocType = 'BO'` identifies the value written off; `CADocumentType = 'BO'` identifies the timing of the write-off event. These differ by orders of magnitude in EUR for typical periods.

---

## Accounts Receivable (FI-AR)

A sub-process of Finance focused on amounts owed by customers.

**`Subject` nodes**
- Customer — a business partner in the buyer role. Conceptually distinct from a sold-to party in order management.
- Net Due Date — the date by which a customer invoice must be paid, derived from payment terms.
- Dunning — the process of sending payment reminders for overdue items.

**`Disambiguation` nodes**
- Customer vs. Sold-to Party — the same business partner identifier appears in FI (the party that owes money) and in SD (the party that placed the order). Same field, different business meaning depending on process context.

---

## Materials Management (MM)

MM covers procurement, inventory management, and goods movements.

**`Subject` nodes**
- Purchase Order — a legally binding document committing to procure goods or services at agreed terms.
- Goods Receipt — the physical receipt of procured goods, triggering inventory increase and liability recognition.
- Material Stock — the quantity of a material available at a storage location at a point in time.
- Supplier — a business partner in the vendor role.

---

## Sales and Distribution (SD)

SD covers order-to-cash: sales orders, deliveries, and billing.

**`Subject` nodes**
- Sales Order — a customer request to deliver goods or services under agreed commercial terms.
- Billing Document — a customer invoice generated from a delivery or service, creating a receivable in FI.
- Revenue — net billing amount recognized in a period.

---

## Controlling (CO)

CO covers internal cost tracking and profitability analysis.

**`Subject` nodes**
- Cost Center — an organizational unit that accumulates costs for a defined area of responsibility.
- Cost Element — a classification of costs (salaries, depreciation, materials) that maps to a G/L account range.
- Internal Order — a temporary cost collector for a project, event, or investment measure.
- Profitability Segment — the combination of characteristics (product, customer, region) used to analyze margins.

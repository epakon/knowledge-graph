# Example: Node Pages

> Illustrative sample pages for all node types. All names and SQL are generic.
> Templates are in [spec/page-templates.md](../spec/page-templates.md).

---

## Subject: Revenue

```markdown
# Subject: Revenue

**Type:** Subject
**Scope:** global

## Business Definition
Revenue is the total monetary value received from customers for goods or services delivered,
before any deductions for returns, write-offs, or discounts.

## Links
- [Subject: Revenue implement -> Measure: GROSS_REVENUE](../Domain: Sales/measures/Measure: GROSS_REVENUE)
- [Subject: Revenue implement -> Measure: NET_REVENUE](../Domain: Finance/measures/Measure: NET_REVENUE)
- [Subject: Revenue implement -> Filter: EXCLUDE_RETURNS](path)
- [Subject: Revenue disambiguate -> Disambiguation: order-status](path)
```

---

## Domain: Sales

```markdown
# Domain: Sales

**Type:** Domain
**Owner:** Sales Analytics Team

## Tables
- [Table: ORDERS](tables/Table: ORDERS)
- [Table: ORDER_LINES](tables/Table: ORDER_LINES)

## Measures
- [Measure: GROSS_REVENUE](measures/Measure: GROSS_REVENUE)

## Filters
**Mandatory:**
- [Filter: ACTIVE_ORDERS](filters/Filter: ACTIVE_ORDERS)

**Optional:**
- [Filter: EXCLUDE_RETURNS](filters/Filter: EXCLUDE_RETURNS)

## Verified Queries
**Onboarding:**
- [VerifiedQuery: REVENUE_BY_REGION_MONTHLY](verified-queries/VerifiedQuery: REVENUE_BY_REGION_MONTHLY)

## Rules
- [Rule: exclude-cancelled-orders](rules/Rule: exclude-cancelled-orders)

## Disambiguations
- [Disambiguation: order-status](disambiguations/Disambiguation: order-status)
```

---

## Table: ORDERS

```markdown
# Table: ORDERS

**Type:** Table
**TableKind:** fact
**Domain:** [Domain: Sales](../domain)
**Source:** analytics.sales.orders

## Description
One row per customer order. Contains the order header — total amount, status, and timestamps.
Line items are in TABLE: ORDER_LINES.

## Fields

### Physical columns
| Column         | PK | Type      | Description                              |
|----------------|----|-----------|------------------------------------------|
| ORDER_ID       | ✓  | VARCHAR   | Unique order identifier                  |
| CUSTOMER_ID    |    | VARCHAR   | FK to customers table                    |
| ORDER_STATUS   |    | VARCHAR   | Current status of the order              |
| ORDER_DATE     |    | DATE      | Date the order was placed                |
| TOTAL_AMOUNT   |    | NUMERIC   | Order total before adjustments           |

### Semantic annotations
| Column        | Kind           | Synonym(s)       | Notes                   | Calculated                                          |
|---------------|----------------|------------------|-------------------------|-----------------------------------------------------|
| ORDER_STATUS  | dimension      | status, state    | See Disambiguation page | [Attribute: ORDER_STATUS](../attributes/Attribute: ORDER_STATUS) |
| ORDER_DATE    | time_dimension | date, order date |                         | —                                                   |
| TOTAL_AMOUNT  | fact           | revenue, amount  |                         | [Measure: GROSS_REVENUE](../measures/Measure: GROSS_REVENUE) |

## Reifications
- [Reification: ACTIVE_ORDERS mandatory -> ORDERS](../../reifications/Reification: ACTIVE_ORDERS mandatory ORDERS)

## Joins
- [Table: ORDERS joinedTo -> Table: ORDER_LINES on ORDERS.ORDER_ID = ORDER_LINES.ORDER_ID](path)

## Caveats
- TOTAL_AMOUNT is the pre-adjustment gross amount. Use Measure: GROSS_REVENUE for the correct filtered aggregate.
- ORDER_DATE is the order placement date, not the fulfillment date.
```

---

## Measure: GROSS_REVENUE

```markdown
# Measure: GROSS_REVENUE

**Type:** Measure
**Domain:** [Domain: Sales](../domain)
**Kind:** aggregate expression
**Synonyms:** total revenue, gross sales, revenue
**Status:** Active

## Definition
```sql
SUM(ORDERS.TOTAL_AMOUNT)
```

## Links
- [Table: ORDERS calculate <- Measure: GROSS_REVENUE](../tables/Table: ORDERS)
- [Measure: GROSS_REVENUE relatedTo -> Rule: exclude-cancelled-orders](../rules/Rule: exclude-cancelled-orders)
- [Measure: GROSS_REVENUE implement -> VerifiedQuery: REVENUE_BY_REGION_MONTHLY](../verified-queries/VerifiedQuery: REVENUE_BY_REGION_MONTHLY)
- [Subject: Revenue implement <- Measure: GROSS_REVENUE](path)
```

---

## Attribute: ORDER_STATUS

```markdown
# Attribute: ORDER_STATUS

**Type:** Attribute
**Domain:** [Domain: Sales](../domain)
**Kind:** dimension
**Synonyms:** status, order state
**access_modifier:** public_access

## Expression
```sql
CASE
  WHEN ORDER_STATUS IN ('confirmed', 'shipped', 'delivered') THEN 'active'
  WHEN ORDER_STATUS IN ('cancelled', 'returned') THEN 'inactive'
  ELSE 'pending'
END
```

## Business Definition
Normalized order lifecycle status. Maps raw system status codes to three business-level
categories: active, inactive, and pending.

## Links
- [Table: ORDERS calculate <- Attribute: ORDER_STATUS](../tables/Table: ORDERS)
- [Attribute: ORDER_STATUS relatedTo -> Filter: ACTIVE_ORDERS](../filters/Filter: ACTIVE_ORDERS)
- [Attribute: ORDER_STATUS relatedTo -> Disambiguation: order-status](../disambiguations/Disambiguation: order-status)
```

---

## Filter: ACTIVE_ORDERS

```markdown
# Filter: ACTIVE_ORDERS

**Type:** Filter
**Domain:** [Domain: Sales](../domain)
**Mandatory:** Yes
**Synonyms:** active, non-cancelled, valid orders
**Disambiguation:** [Disambiguation: order-status](../disambiguations/Disambiguation: order-status)

## Predicate
```sql
ORDER_STATUS IN ('confirmed', 'shipped', 'delivered')
```

## Reifications
- [Reification: ACTIVE_ORDERS mandatory -> ORDERS](../../reifications/Reification: ACTIVE_ORDERS mandatory ORDERS)

## Links
- [Subject: Revenue implement <- Filter: ACTIVE_ORDERS](path)
- [Filter: ACTIVE_ORDERS implement -> VerifiedQuery: REVENUE_BY_REGION_MONTHLY](path)
```

---

## VerifiedQuery: REVENUE_BY_REGION_MONTHLY

```markdown
# VerifiedQuery: REVENUE_BY_REGION_MONTHLY

**Type:** VerifiedQuery
**Domain:** [Domain: Sales](../domain)
**Onboarding question:** Yes
**Verified by:** jane.smith
**Verified at:** 2026-05-10
**Status:** Active

## Question
What is the gross revenue by region for each month?

## Reifications
- [Reification: REVENUE_BY_REGION_MONTHLY demonstrates -> exclude-cancelled-orders](../../reifications/Reification: REVENUE_BY_REGION_MONTHLY demonstrates exclude-cancelled-orders)

## Links
- [Measure: GROSS_REVENUE implement <- VerifiedQuery: REVENUE_BY_REGION_MONTHLY](path)
- [Filter: ACTIVE_ORDERS implement <- VerifiedQuery: REVENUE_BY_REGION_MONTHLY](path)

## SQL
```sql
SELECT
    c.region,
    DATE_TRUNC('month', o.order_date) AS month,
    SUM(o.total_amount) AS gross_revenue
FROM orders o
JOIN customers c ON o.customer_id = c.customer_id
WHERE o.order_status IN ('confirmed', 'shipped', 'delivered')   -- Filter: ACTIVE_ORDERS
GROUP BY 1, 2
ORDER BY 2, 1
```
```

---

## BusinessRule: exclude-cancelled-orders

```markdown
# Rule: exclude-cancelled-orders

**Type:** BusinessRule
**Domain:** [Domain: Sales](../domain)

## Definition
ORDER_STATUS NOT IN ('cancelled', 'returned')

## Consequence if Violated
Cancelled and returned orders inflate revenue by up to 12% in months with high return rates.

## Links
- [Rule: exclude-cancelled-orders apply -> Measure: GROSS_REVENUE](path)
- [Rule: exclude-cancelled-orders apply -> Table: ORDERS](path)
- [Subject: Revenue implement <- Rule: exclude-cancelled-orders](path)
- [Rule: exclude-cancelled-orders relatedTo -> Filter: ACTIVE_ORDERS](path)
```

---

## Disambiguation: order-status

```markdown
# Disambiguation: order-status

**Type:** Disambiguation
**Domain:** [Domain: Sales](../domain)

## Always Ask
> "Do you want to include only active orders (confirmed/shipped/delivered), or all orders including cancelled and returned ones?"

## Option A: Active orders only
→ [Filter: ACTIVE_ORDERS](../filters/Filter: ACTIVE_ORDERS)

## Option B: All orders including inactive
→ [Rule: exclude-cancelled-orders](../rules/Rule: exclude-cancelled-orders) (do not apply)

## Links
- [Subject: Revenue disambiguate <- Disambiguation: order-status](path)
```

---

## Reification: ACTIVE_ORDERS mandatory ORDERS

```markdown
# Reification: ACTIVE_ORDERS mandatory ORDERS

**Type:** Reification
**Kind:** mandatory
**From:** [Filter: ACTIVE_ORDERS](../filters/Filter: ACTIVE_ORDERS)
**To:** [Table: ORDERS](../tables/Table: ORDERS)

## Reason
ORDERS also holds cancelled and returned orders; every revenue figure is defined on valid orders only.

## Consequence if Ignored
Including cancelled and returned orders inflates reported revenue by up to 12% in high-return months.
```

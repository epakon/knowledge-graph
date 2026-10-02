# Example: Domain Layout

> Illustrative example of a complete domain structure. All names are generic.
> See [spec/space-structure.md](../spec/space-structure.md) for the canonical hierarchy.

---

## Scenario

A company has two domains: **Sales** and **Finance**. Both domains share the business concept of "Revenue", defined globally as a Subject. Each domain has its own Measure that computes revenue differently.

---

## Space hierarchy

```
Knowledge Graph (root)
│
├── vocabulary/
│   ├── subjects/
│   │   ├── Subject: Revenue          ← shared business definition
│   │   ├── Subject: Write-Off        ← shared business definition
│   │   └── Subject: Active Customer  ← shared business definition
│   └── policies/
│       └── Policy: Revenue counts valid orders only  ← shared business rule
│
├── Domain: Sales
│   ├── tables/
│   │   ├── Table: ORDERS
│   │   └── Table: ORDER_LINES
│   ├── measures/
│   │   └── Measure: GROSS_REVENUE
│   ├── attributes/
│   │   └── Attribute: ORDER_STATUS
│   ├── filters/
│   │   ├── Filter: ACTIVE_ORDERS
│   │   └── Filter: EXCLUDE_RETURNS
│   ├── verified-queries/
│   │   └── VerifiedQuery: REVENUE_BY_REGION_MONTHLY
│   ├── rules/
│   │   └── Rule: exclude-cancelled-orders
│   ├── reifications/
│   │   ├── Reification: GROSS_REVENUE requires ACTIVE_ORDERS
│   │   └── Reification: ACTIVE_ORDERS mandatory ORDERS
│   └── disambiguations/
│       └── Disambiguation: order-status
│
└── Domain: Finance
    ├── tables/
    │   └── Table: LEDGER_ENTRIES
    ├── measures/
    │   └── Measure: NET_REVENUE
    ├── filters/
    │   └── Filter: EXCLUDE_WRITE_OFFS
    ├── rules/
    │   └── Rule: write-off-threshold
    └── reifications/
        └── Reification: NET_REVENUE requires EXCLUDE_WRITE_OFFS
```

---

## Cross-domain linking

`Subject: Revenue` is defined once. Each domain's Measure page links up to it; the Subject page does not link down:

```
Measure: GROSS_REVENUE                (Sales)
  ## Links
  - [Measure: GROSS_REVENUE implement -> Subject: Revenue]
```

```
Measure: NET_REVENUE                  (Finance)
  ## Links
  - [Measure: NET_REVENUE implement -> Subject: Revenue]
```

The business definition of "Revenue" is written once on the Subject page and is accessible to agents working in both domains via semantic search. To list every implementation, search the edge index for `implement <- Subject: Revenue`.

A shared rule works the same way. `Policy: Revenue counts valid orders only` (`relatedTo -> Subject: Revenue`) states the rule once, and `Filter: ACTIVE_ORDERS` in Sales owns `implement -> Policy: Revenue counts valid orders only`. Finance has no Filter or Rule implementing it on `Table: LEDGER_ENTRIES`, even though `Measure: NET_REVENUE` implements `Subject: Revenue` — the `policy_coverage` audit reports exactly this gap.

---

## Adding a third domain

To add a **Logistics** domain:

1. Create root container: `Knowledge Graph: Logistics`
2. Create domain index: `Domain: Logistics`
3. Create type containers: `tables/`, `measures/`, `filters/`, etc.
4. For shared concepts (e.g. "Active Customer"), link to existing Subject pages — do not create new Subject pages.
5. Create domain-specific Reification pages for mandatory filters and requires edges in the new domain.

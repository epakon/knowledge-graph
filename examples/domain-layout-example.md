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
├── concepts/
│   └── subjects/
│       ├── Subject: Revenue          ← shared business definition
│       ├── Subject: Write-Off        ← shared business definition
│       └── Subject: Active Customer  ← shared business definition
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
│   ├── relationships/
│   │   ├── Relationship: GROSS_REVENUE requires ACTIVE_ORDERS
│   │   └── Relationship: ACTIVE_ORDERS mandatory ORDERS
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
    └── relationships/
        └── Relationship: NET_REVENUE requires EXCLUDE_WRITE_OFFS
```

---

## Cross-domain linking

`Subject: Revenue` is defined once and linked from both domains:

```
Subject: Revenue
  ## Links
  - [Subject: Revenue implement <- Measure: GROSS_REVENUE]   (back-ref from Sales domain)
  - [Subject: Revenue implement <- Measure: NET_REVENUE]     (back-ref from Finance domain)
```

Each Measure page links back to the Subject:

```
Measure: GROSS_REVENUE
  ## Links
  - [Subject: Revenue implement <- Measure: GROSS_REVENUE]
```

```
Measure: NET_REVENUE
  ## Links
  - [Subject: Revenue implement <- Measure: NET_REVENUE]
```

The business definition of "Revenue" is written once on the Subject page and is accessible to agents working in both domains via semantic search.

---

## Adding a third domain

To add a **Logistics** domain:

1. Create root container: `Knowledge Graph: Logistics`
2. Create domain index: `Domain: Logistics`
3. Create type containers: `tables/`, `measures/`, `filters/`, etc.
4. For shared concepts (e.g. "Active Customer"), link to existing Subject pages — do not create new Subject pages.
5. Create domain-specific Relationship pages for mandatory filters and requires edges in the new domain.

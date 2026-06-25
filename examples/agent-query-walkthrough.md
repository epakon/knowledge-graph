# Example: Agent Query Walkthrough

> End-to-end example showing how an agent uses the knowledge graph to answer a business question correctly. All domain-specific names and data values are illustrative.

This example demonstrates the full agent workflow:
1. Semantic search via Glean to find relevant knowledge graph pages
2. Mandatory filter discovery
3. Business rule application
4. Verified query retrieval
5. SQL generation and execution

---

## Setup

**Agent:** Cursor (or any LLM with Glean MCP configured)
**Prerequisites:** Glean MCP — not Atlassian MCP. The agent reads knowledge pages via Glean semantic search; it does not need direct Confluence access for read-only queries.

> **Note on Snowflake:** the same workflow runs inside a Snowflake Cortex Agent via the Glean MCP Connector (see [adapters/engine/confluence/confluence-adapter.md](../adapters/engine/confluence/confluence-adapter.md#snowflake-cortex-integration)). No separate setup is needed — the Glean MCP server is reused.

---

## Question

> "How do I query `<TABLE_NAME>` correctly? Show me `<metric>` for the last period."

---

## Step 1 — Glean search surfaces the knowledge graph

The agent searches Glean with a natural-language phrase:

```
"how to query <TABLE_NAME> correctly"
"mandatory filters for <TABLE_NAME>"
```

Glean returns the Table page, mandatory Filter pages, and related Relationship pages.

---

## Step 2 — Mandatory filters discovered

The Table page's `## Relationships` section links to Relationship pages of kind `mandatory`. The agent fetches each one and extracts the Filter predicate.

**Example Relationship page:** `Relationship: <FILTER_A> mandatory <TABLE_NAME>`

```
Reason:    <Filter A> excludes records of the wrong type — omitting it inflates counts and amounts materially.
Consequence: Queries without this filter return incorrect totals.
```

**Filter predicate:**
```sql
<column_a> = '<value>'
```

A second mandatory filter is found the same way:

```sql
<column_b> <> '<excluded_value>'
```

Both predicates **must** appear in every query on this table.

---

## Step 3 — Business rule applied

The agent's question includes a specific metric (e.g. "write-off amount"). Glean returns a Rule page:

**Rule page:** `Rule: <rule-name>`

```
Definition:       <column_c> = '<type_value>'
Consequence:      Using the wrong column for this metric produces results that differ by orders of magnitude.
```

The rule specifies which column and value to filter on for this metric — the agent uses the Rule definition, not a raw column reference.

---

## Step 4 — Verified query retrieved

Glean returns a VerifiedQuery page that matches the question pattern:

**VerifiedQuery:** `VerifiedQuery: <METRIC>_BY_PERIOD`

```
Verified by: <name>
Verified at: <YYYY-MM-DD>
Onboarding question: Yes
```

**Verified SQL:**
```sql
SELECT
    <time_column>,
    SUM(<amount_column>) AS metric_value,
    COUNT(*) AS record_count
FROM <schema>.<TABLE_NAME>
WHERE <column_a> = '<value>'          -- mandatory filter A
  AND <column_b> <> '<excluded_value>'  -- mandatory filter B
  AND <column_c> = '<type_value>'       -- business rule
GROUP BY ALL
ORDER BY <time_column>
```

---

## Step 5 — SQL adapted for the specific question ("last period")

The agent adapts the verified SQL to answer the specific question — last period only:

```sql
WITH last_period AS (
    SELECT MAX(<time_column>) AS max_period
    FROM <schema>.<TABLE_NAME>
    WHERE <column_a> = '<value>'
      AND <column_b> <> '<excluded_value>'
      AND <column_c> = '<type_value>'
)
SELECT
    t.<time_column>,
    SUM(t.<amount_column>) AS metric_value,
    COUNT(*) AS record_count
FROM <schema>.<TABLE_NAME> t
JOIN last_period lp ON t.<time_column> = lp.max_period
WHERE t.<column_a> = '<value>'
  AND t.<column_b> <> '<excluded_value>'
  AND t.<column_c> = '<type_value>'
GROUP BY t.<time_column>
```

---

## Key observations

**What the knowledge graph prevented:**
- Using the wrong column for the metric (the Rule page explicitly warns this produces results off by orders of magnitude)
- Omitting mandatory filters (the Relationship pages state the consequence: materially distorted counts and amounts)
- Writing SQL from scratch when a verified reference already exists

**What the agent cited:**
- Table page (schema and joins)
- Relationship pages (mandatory filter reason + consequence)
- Rule page (exact filter expression)
- VerifiedQuery page (reference SQL pattern)

Every claim in the SQL is grounded in a specific knowledge graph page — no business logic was fabricated.

---

## Snowflake agent variant

The identical workflow runs in a Snowflake Cortex Agent with Glean as an MCP Connector:
- The agent calls Glean via the Cortex Agents MCP Connector (no separate Cortex Search index needed)
- Glean returns the same knowledge graph pages
- The agent generates and executes SQL inside Snowflake directly
- No code change is needed — the knowledge base is the same, only the execution runtime differs

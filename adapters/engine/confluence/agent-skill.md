# Agent Skill — Confluence

> Operational guide for agents working with the Knowledge Graph on Confluence.
> Read the [Confluence Adapter](confluence-adapter.md) and [SPEC.md](../../SPEC.md) before starting.

> **Prerequisites:** The Atlassian MCP server must be configured and authenticated before any workflow in this skill can execute. All read and write operations go through MCP tools (`confluence_search`, `confluence_get_page`, `confluence_create_page`, `confluence_update_page`, `confluence_get_version_history`, `confluence_get_child_pages`). Without an active Atlassian MCP connection, no workflow is possible.

---

## Supported intents

This table maps natural-language user requests to the workflow that handles them. Use it to identify the correct workflow before starting.

| What the user says | What the agent does | Workflow |
|---|---|---|
| "What does `<term>` mean?" | Search → Subject or Disambiguation page → answer from Business Definition | A — Read |
| "What is the `<measure>` formula?" | Search → Measure page → return Definition + `## Links` (Table sources) | A — Read |
| "Which filters are mandatory for table T?" | Search → Reification pages where kind=`mandatory` and To=T | A — Read |
| "Show lineage for measure X" | Fetch Measure page → follow Reification + Related links to Filters, Rules, Tables | D — Navigate |
| "What depends on filter F?" | Search for pages referencing `Filter: F` → traverse downstream | D — Navigate |
| "Show me verified SQL for question Q" | Search → VerifiedQuery page matching Q → return SQL | A — Read |
| "What are the onboarding questions for domain D?" | Search VerifiedQuery pages with `Onboarding question: Yes` in domain D | A — Read |
| "Update `<measure>` — change `<field>` to `<value>`" | Fetch page → confirm change → update with version comment | C — Update |
| "Add a new `<node type>` for `<name>`" | Draft page using template → confirm → create → update parent | B — Write |
| "Add this SQL as a verified query" | Create VerifiedQuery page → link from Measure + Domain pages | B — Write |
| "Create a new domain for `<Domain>`" | Create domain index + scaffold all type container pages | B — Write |
| "What changed in `<page>` last month?" | Fetch version history → parse version comments → summarize | E — Version history |
| "Roll back `<page>` to before `<date>`" | Fetch version history → retrieve target version → apply via Workflow C | E → C |

---

## Prerequisites

Before starting any workflow:

1. Read `SPEC.md` and `spec/logical-layer.md` if not already done in this session.
2. Know the space key and parent page ID for the target domain (from your instance reference table — never fabricate a page ID).
3. For write operations: always fetch the current page version number immediately before calling `confluence_update_page`.

---

## Choosing: MCP tool calls vs. Knowledge Graph API

| Situation | Use |
|---|---|
| Single page — read, create, or update | MCP tool calls directly |
| 2–5 pages — targeted updates | MCP tool calls directly |
| 6+ pages — same structural change across many pages | Knowledge Graph API |
| Regex-based HTML surgery (changing link format, moving metadata) | Knowledge Graph API |
| Bulk rename / edge renaming across the graph | Knowledge Graph API |
| Debugging Confluence storage-format HTML | Knowledge Graph API |

See [graph-api.md](graph-api.md) for Knowledge Graph API patterns.

---

## Workflow A — Read: look up a definition, rule, or query

**Trigger:** user asks what something means, how a measure is computed, which filters are mandatory, what a verified query contains, or any "what is / how does / show me" question.

### Steps

1. **Search** for the concept name:
   ```
   title ~ "<concept>" AND space = "<SPACE_KEY>"
   ```

2. **Resolve the node type.** From search results, identify which page is relevant:
   - Concept meaning → **Subject** page first.
   - Specific filter/measure/rule → that node's page directly.
   - Verified SQL → **VerifiedQuery** page.

3. **Fetch the page** with `confluence_get_page`.

4. **Follow links** if needed. If the page references Reification pages, Disambiguation pages, or Subjects, fetch those too for a complete answer.

5. **Answer** using retrieved page content. Cite which pages you retrieved. Never fabricate definitions — every claim must come from a retrieved page.

---

## Workflow B — Write: add a new page

**Trigger:** user asks to add a new measure, filter, rule, verified query, subject, relationship, or disambiguation page — single or batch.

### Steps

1. **Identify all pages to create** from the user's description.

   **Before adding any `Reification:` page, check the intended Kind against `spec/schema.yaml`'s `reified_edge_kinds` list** — no other kind qualifies, however strong the "why it matters" narrative behind the edge (see [spec/logical-layer.md §2.2](../../spec/logical-layer.md#22-reified-edge-kinds-reification-pages--typed-relationships-with-properties) for why). If a caveat doesn't fit one of those kinds but still needs its own Reason/Consequence, put it on a `BusinessRule` page instead — check for an existing one first.

2. **Present a creation plan and ask for confirmation BEFORE creating anything.**
   Show a compact table — node type, count, names only. No page body content.

   | Node type | Count | Names |
   |-----------|-------|-------|
   | Subject   | 1     | Write-Off |
   | Measure   | 2     | REVENUE, GROSS_MARGIN |
   | **Total** | **3** | |

   Ask: "Proceed with all N pages?" — do not create anything until confirmed.

3. **Check for duplicates.** For each page in the confirmed list:
   ```
   title = "<NodeType>: <Name>" AND space = "<SPACE_KEY>"
   ```
   Skip existing pages (switch to Workflow C). Report skipped pages.

4. **Select the correct template** from [spec/page-templates.md](../../spec/page-templates.md).

5. **Draft and create each page** with `confluence_create_page`:
   - `spaceKey`: your space key
   - `title`: `<NodeType>: <Name>`
   - `body`: content following the template, with proper `<ac:link-body>` links
   - `parentId`: correct parent ID from your instance reference table

6. **After each page is created**, update the parent page to add a link to the new page:
   - Fetch the parent, add the link under the correct section.
   - Native version comment: `Summary: Added link to <new page>. Changed: <section>. Reason: New page created. Breaking: no`

---

## Workflow C — Update: edit an existing page

**Trigger:** user asks to change a definition, formula, predicate, rule, or any other field.

**Decision point:** if this is a structural change affecting 6+ pages (e.g. link format migration, header field removal, edge renaming), use the Knowledge Graph API instead (see [graph-api.md](graph-api.md)).

### Steps for targeted updates (≤ 5 pages)

1. **Resolve the page.** Search by title, fetch with `confluence_get_page`. Note `version.number`.

2. **Identify the change.** Confirm with user if not clear:
   - Which field or section changes?
   - New value?
   - Breaking change?

3. **Apply the change** to the page body. Keep all other fields and content unchanged.

4. **Confirm with the user** before saving:
   ```
   Field:    <field name>
   Before:   <old value>
   After:    <new value>
   Breaking: yes | no
   ```
   Ask: "Apply this update?" — do not update until confirmed.

5. **Update the page** with `confluence_update_page`:
   - `version`: current version + 1
   - `body`: updated content
   - Version comment in the required format (see [spec/versioning.md](../../spec/versioning.md))

6. **If breaking:** check Reification pages referencing this node — update `## Consequence if Ignored` if needed.

### Steps for bulk updates (6+ pages)

1. Describe the structural change, enumerate affected pages, ask for confirmation.
2. Use the Knowledge Graph API — see [graph-api.md](graph-api.md) for patterns.
3. Run on one page first. Show before/after diff. Confirm before running on all.
4. Execute the full batch only after confirmation.

---

## Workflow D — Navigate: lineage and graph traversal

**Trigger:** "what depends on X", "which rules apply to table Y", "show me all verified queries for measure Z", "what is the lineage of column C".

### Steps

1. **Resolve the starting node.** Search and fetch the page.

2. **Determine traversal direction:**
   - **Downstream** (what uses X): search for pages that reference this page:
     ```
     text ~ "<NodeType>: <Name>" AND space = "<SPACE_KEY>"
     ```
   - **Upstream** (what X depends on): follow `## Links` and `## Reifications` links inside the page.

3. **Fetch linked pages** for each relevant hop. Do not traverse the full graph — stop at the depth that answers the question.

4. **Present the lineage** as a structured list showing node type, name, and edge kind:
   ```
   Measure: REVENUE
     --[mandatory]--> Reification: ACTIVE_CUSTOMERS mandatory -> ORDERS
       --[to]-->      Filter: ACTIVE_CUSTOMERS
     --[requires]--> Reification: REVENUE requires ACTIVE_CUSTOMERS
       --[to]-->      Filter: ACTIVE_CUSTOMERS
     --[implement <-]- VerifiedQuery: REVENUE_BY_REGION
     --[implement <-]- Subject: Revenue
   ```

5. If the user wants to go deeper on any node, fetch that page and continue.

---

## Workflow E — Version history: what changed and when

**Trigger:** "what changed in X", "show history of rule Y", "who updated filter Z".

### Steps

1. **Resolve the page** by title search.

2. **Fetch version history** with `confluence_get_version_history`:
   - `contentId`: resolved page ID
   - `limit`: 10 (or as requested)

3. **Parse the version comments.** Confluence returns version number, date, and author natively per entry; the comment text itself follows:
   `Summary: ... | Changed: ... | Reason: ... | Breaking: yes/no`

4. **Present as a table**: Version, Date, Author, Summary, Breaking. Highlight breaking changes.

5. To restore a prior version: retrieve that version's content and walk through Workflow C — do not silently overwrite.

---

## Constraints (always apply)

- **Never fabricate a page ID.** Always resolve via `confluence_search` or from the instance reference table.
- **Always fetch the current version number** before calling `confluence_update_page`.
- **Every `confluence_update_page` call must set the native `versionComment`** in the required format — never write it into the page body.
- **Prose belongs only on Subject and Disambiguation pages.** All other pages use structured fields and predicate/definition blocks.
- **When creating a page, also update its parent page** to add a link to the new page.
- **No drafts.** `confluence_create_page` publishes immediately. Show a summary and wait for user confirmation before creating any pages.
- **Do not dump page HTML bodies into chat.** Show field values only, not raw storage-format HTML.

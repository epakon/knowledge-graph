# Agent Skill — Core

> Engine-neutral operational guide for agents working with the Knowledge Graph.
> Read [SPEC.md](../../SPEC.md) and the backend skill for your content storage before starting:
> [Confluence](confluence/agent-skill.md) · [Markdown](markdown/agent-skill.md).

This document defines the workflows once. Each backend skill maps the operations below to its own tooling and adds the steps and constraints only that backend needs. How to read the graph when answering a business question (start at the Measure, follow `mandatory`/`requires`, and so on) is defined in [SPEC.md §8](../../SPEC.md#8-agent-integration) and is not repeated here.

---

## Backend operations

Workflows refer to these operations by name. Every backend skill maps each of them.

| Operation | Purpose |
|---|---|
| **Find by name** | Locate a node by `<NodeType>: <Name>` or by name alone |
| **Find references** | Locate every node whose content links to a given node (downstream traversal) |
| **Read** | Retrieve a node's current content (and version, if the backend tracks one) |
| **Create** | Publish a new node at its target location |
| **Update** | Replace a node's content with a new version |
| **Index update** | Add a link to a new node in the index that lists it |
| **Record version** | Attach the structured version comment ([spec/versioning.md](../../spec/versioning.md)) to a create or update, outside the node content |
| **History** | List prior versions of a node with date, author and version comment |
| **Restore** | Retrieve the content of a prior version |

---

## Supported intents

This table maps natural-language user requests to the workflow that handles them. Use it to identify the correct workflow before starting.

| What the user says | What the agent does | Workflow |
|---|---|---|
| "What does `<term>` mean?" | Find → Subject or Disambiguation node → answer from Business Definition | A — Read |
| "What is the `<measure>` formula?" | Find → Measure node → return Definition + `## Links` (Table sources) | A — Read |
| "Which filters are mandatory for table T?" | Find Reification nodes where kind=`mandatory` and To=T | A — Read |
| "Show me verified SQL for question Q" | Find → VerifiedQuery node matching Q → return SQL | A — Read |
| "What are the onboarding questions for domain D?" | Find VerifiedQuery nodes in domain D with the onboarding-question flag set | A — Read |
| "Add a new `<node type>` for `<name>`" | Draft node using template → confirm → create → index update | B — Write |
| "Add this SQL as a verified query" | Create VerifiedQuery node → link from Measure + Domain nodes | B — Write |
| "Add an agent for `<purpose>`" | Run the `uses`-overlap check → draft Agent node → confirm → create under `ai/` | B — Write |
| "Record the existing agent `<name>`" | Draft Agent node from the deployed agent as it is → run the `uses`-overlap check, report findings without blocking → confirm → create under `ai/` | B — Write |
| "Create a new domain for `<Domain>`" | Create domain index + scaffold all type containers | B — Write |
| "Update `<measure>` — change `<field>` to `<value>`" | Read → confirm change → update with version comment | C — Update |
| "Show lineage for measure X" | Read Measure node → follow `## Reifications` and `## Links` to Filters, Rules, Tables | D — Navigate |
| "What depends on filter F?" | Find references to `Filter: F` → traverse downstream | D — Navigate |
| "Which agents use `<node>`?" | Read the node → the `uses <-` back-references in `## Links` | D — Navigate |
| "What changed in `<node>` last month?" | History → parse version comments → summarize | E — Version history |
| "Roll back `<node>` to before `<date>`" | History → restore target version → apply via Workflow C | E → C |

---

## Prerequisites

Before starting any workflow:

1. Read `SPEC.md` and `spec/logical-layer.md` if not already done in this session.
2. Meet the backend skill's prerequisites (access, location of the knowledge graph).
3. For write operations: **Read** the node immediately before changing it — never overwrite blindly.

---

## Choosing: direct operations vs. Knowledge Graph API

| Situation | Use |
|---|---|
| 1–5 nodes — read, create, or targeted update | Direct backend operations |
| 6+ nodes — same structural change across many nodes | Knowledge Graph API |
| Bulk rename / edge renaming across the graph | Knowledge Graph API |

Backend skills list further cases that call for the API. See the backend's `graph-api.md` for patterns.

---

## Workflow A — Read: look up a definition, rule, or query

**Trigger:** user asks what something means, how a measure is computed, which filters are mandatory, what a verified query contains, or any "what is / how does / show me" question.

### Steps

1. **Find by name** for the concept.

2. **Resolve the node type.** From results, identify which node is relevant:
   - Concept meaning → **Subject** node first.
   - Specific filter/measure/rule → that node directly.
   - Verified SQL → **VerifiedQuery** node.
   - Which agent answers a question, or what an agent reads → **Agent** node, then its `uses` links.

3. **Read** the node.

4. **Follow links** if needed. If the node references Reification, Disambiguation or Subject nodes in `## Reifications` or `## Links`, read those too for a complete answer.

5. **Answer** using retrieved content. Cite which nodes you retrieved. Never fabricate definitions — every claim must come from a retrieved node.

---

## Workflow B — Write: add a new node

**Trigger:** user asks to add a new concept, subject, process, table, measure, attribute, filter, rule, verified query, reification, disambiguation, or agent — single or batch.

### Steps

1. **Identify all nodes to create** from the user's description.

   **Before adding any `Reification:` node, check the intended Kind against `spec/schema.yaml`'s `reified_edge_kinds` list** — no other kind qualifies, however strong the "why it matters" narrative behind the edge (see [spec/logical-layer.md §2.2](../../spec/logical-layer.md#22-reified-edge-kinds-reification-pages--typed-relationships-with-properties) for why). If a caveat doesn't fit one of those kinds but still needs its own Reason/Consequence, put it on a `BusinessRule` node instead — check for an existing one first.

2. **Present a creation plan and ask for confirmation BEFORE creating anything.**
   Show a compact table — node type, count, names only. No node content.

   | Node type | Count | Names |
   |-----------|-------|-------|
   | Subject   | 1     | Write-Off |
   | Measure   | 2     | REVENUE, GROSS_MARGIN |
   | **Total** | **3** | |

   Ask: "Proceed with all N nodes?" — do not create anything until confirmed.

3. **Check for duplicates.** **Find by name** for each node in the confirmed list. Skip existing nodes (switch to Workflow C). Report skipped nodes.

4. **Select the correct template** from [spec/page-templates.md](../../spec/page-templates.md).

5. **Draft and create each node** with **Create** and **Record version**.

6. **Index update** for each new node, under the correct section. Version comment: `Summary: Added link to <new node>. Changed: <section>. Reason: New node created. Breaking: no`

### Additional steps for Agent nodes

Agent nodes follow [spec/consumption-layer.md](../../spec/consumption-layer.md). Before step 4:

1. **Every agent gets an Agent node** (consumption-layer §1), including agents already deployed. For a new agent, do not build one that duplicates an existing agent or reads the wrong nodes. For an existing agent, record it as it is, even if it overlaps or is wrong: the node is what makes its impact visible.
2. **Run the overlap check** (§8.3). List the nodes the agent will `uses`, then compare that set with the `uses` set of every `Active` Agent node (Jaccard similarity). For each high-overlap pair, walk through the reasonable-overlap test (§8.2) with the user. A true duplicate means extending the existing agent instead; otherwise record the answer in the new node's `## Differentiation` section. When recording an existing agent, the check never blocks: record the findings and hand them to review (§8.2, §8.4).
3. **Keep the node to the agent's own behavior** (§5, §5.1): purpose, the response and orchestration instructions that differ for this agent, and sample questions that have no VerifiedQuery yet. Rules, synonyms and join caveats go on the nodes the agent uses. No vendor syntax (§2.1); record the deployed artifact in the knowledge base's repository README (§7). When recording an existing agent, do not copy rules its deployed instructions restate; link the nodes via `uses`, and report any place where the deployed text contradicts a node as a finding for review.
4. **Place it under `ai/`** and add the `uses <-` back-reference to the `## Links` of every target node in the same batch.

---

## Workflow C — Update: edit an existing node

**Trigger:** user asks to change a definition, formula, predicate, rule, or any other field.

**Decision point:** if this is a structural change affecting 6+ nodes (e.g. link format migration, header field removal, edge renaming), use the Knowledge Graph API instead.

### Steps for targeted updates (≤ 5 nodes)

1. **Resolve the node.** **Find by name**, then **Read** it.

2. **Identify the change.** Confirm with user if not clear:
   - Which field or section changes?
   - New value?
   - Breaking change?

3. **Apply the change** to the content. Keep all other fields and content unchanged.

4. **Confirm with the user** before saving:
   ```
   Field:    <field name>
   Before:   <old value>
   After:    <new value>
   Breaking: yes | no
   ```
   Ask: "Apply this update?" — do not save until confirmed.

5. **Update** the node with **Record version**.

6. **If breaking:** find Reification nodes referencing this node — update `## Consequence if Ignored` if needed. Also list the Agent nodes that use this node (the `uses <-` back-references in its `## Links`) in the change summary, so their owners see the breaking change.

### Steps for bulk updates (6+ nodes)

1. Describe the structural change, enumerate affected nodes, ask for confirmation.
2. Use the Knowledge Graph API.
3. Run on one node first. Show before/after diff. Confirm before running on all.
4. Execute the full batch only after confirmation.

---

## Workflow D — Navigate: lineage and graph traversal

**Trigger:** "what depends on X", "which rules apply to table Y", "show me all verified queries for measure Z", "what is the lineage of column C".

### Steps

1. **Resolve the starting node.** **Find by name**, then **Read** it.

2. **Determine traversal direction:**
   - **Downstream** (what uses X): **Find references** to X.
   - **Upstream** (what X depends on): follow `## Links` and `## Reifications` links inside the node.

3. **Read linked nodes** for each relevant hop. Do not traverse the full graph — stop at the depth that answers the question.

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

5. If the user wants to go deeper on any node, read it and continue.

---

## Workflow E — Version history: what changed and when

**Trigger:** "what changed in X", "show history of rule Y", "who updated filter Z".

### Steps

1. **Resolve the node** with **Find by name**.

2. **History** for the node (last 10 versions, or as requested).

3. **Parse the version comments.** The backend returns version, date and author natively per entry; the comment text itself follows:
   `Summary: ... | Changed: ... | Reason: ... | Breaking: yes/no`

4. **Present as a table**: Version, Date, Author, Summary, Breaking. Highlight breaking changes.

5. To roll back: **Restore** that version's content and walk through Workflow C — do not silently overwrite.

---

## Constraints (always apply)

- **Never fabricate a node identifier** (page ID, path). Always resolve it via **Find by name** or the backend's naming convention.
- **Read before every write.**
- **The version comment goes through Record version** — never into the node content.
- **Prose belongs only on Subject and Disambiguation nodes.** All other nodes use structured fields and predicate/definition blocks.
- **When creating a node, also run Index update.**
- **Confirm before writing.** Show a plan, draft or diff and wait for user confirmation before creating or updating any node.
- **Do not dump raw node content into chat.** Show field values only.

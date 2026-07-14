# Agent Skill — Markdown

> Operational guide for agents working with the Knowledge Graph on Markdown files.
> Read the [Markdown Adapter](markdown-adapter.md) and [SPEC.md](../../SPEC.md) before starting.

> **Prerequisites:** The agent must have read/write access to the knowledge graph directory tree in the repository. All read and write operations use direct file tools — no MCP server is required for this adapter.

---

## Supported intents

| What the user says | What the agent does | Workflow |
|---|---|---|
| "What does `<term>` mean?" | Search → Subject or Disambiguation file → answer from Business Definition | A — Read |
| "What is the `<measure>` formula?" | Search → Measure file → return Definition + `## Links` (Table sources) | A — Read |
| "Which filters are mandatory for table T?" | Search Reification files where kind=`mandatory` and To=T | A — Read |
| "Show lineage for measure X" | Fetch Measure file → follow Reification + Related links to Filters, Rules, Tables | D — Navigate |
| "What depends on filter F?" | Search for files referencing `Filter: F` → traverse downstream | D — Navigate |
| "Show me verified SQL for question Q" | Search → VerifiedQuery file matching Q → return SQL | A — Read |
| "What are the onboarding questions for domain D?" | Search VerifiedQuery files with `onboarding_question: true` in domain D | A — Read |
| "Update `<measure>` — change `<field>` to `<value>`" | Read file → confirm change → write file → commit with version comment | C — Update |
| "Add a new `<node type>` for `<name>`" | Draft file using template → confirm → write file → update domain index → commit | B — Write |
| "Add this SQL as a verified query" | Create VerifiedQuery file → add link from Measure + Domain files → commit | B — Write |
| "Create a new domain for `<Domain>`" | Create domain directory scaffold + domain index file | B — Write |
| "What changed in `<file>` last month?" | `git log` on the file → parse version comments from commit messages → summarize | E — Version history |
| "Roll back `<file>` to before `<date>`" | `git log` → find target commit → retrieve that version → apply via Workflow C | E → C |

---

## Prerequisites

Before starting any workflow:

1. Read `SPEC.md` and `spec/logical-layer.md` if not already done in this session.
2. Know the root path of the knowledge graph directory tree in the repository.
3. For write operations: always read the current file content before modifying — never overwrite blindly.

---

## Choosing: direct file operations vs. Knowledge Graph API

| Situation | Use |
|---|---|
| Single file — read, create, or update | Direct file read/write |
| 2–5 files — targeted updates | Direct file read/write |
| 6+ files — same structural change across many files | Knowledge Graph API |
| Regex-based frontmatter or link surgery across the tree | Knowledge Graph API |
| Bulk rename / edge renaming across the graph | Knowledge Graph API |
| Snapshot generation or graph DB index export | Knowledge Graph API |

See [graph-api.md](graph-api.md) for Knowledge Graph API patterns.

---

## Workflow A — Read: look up a definition, rule, or query

**Trigger:** user asks what something means, how a measure is computed, which filters are mandatory, what a verified query contains, or any "what is / how does / show me" question.

### Steps

1. **Search** for the concept name using ripgrep:
   ```bash
   rg "name: <concept>" <kg-root>/ --include="*.md" -l
   # or by title
   rg "^# (Measure|Subject|Filter): <concept>" <kg-root>/ --include="*.md" -l
   ```

2. **Resolve the node type.** From results, identify which file is relevant:
   - Concept meaning → **Subject** file first.
   - Specific filter/measure/rule → that node's file directly.
   - Verified SQL → **VerifiedQuery** file.

3. **Read the file.**

4. **Follow links** if needed. If the file references Reification files, Disambiguation files, or Subjects in `## Reifications` or `## Links`, read those too for a complete answer.

5. **Answer** using retrieved file content. Cite which files you retrieved. Never fabricate definitions — every claim must come from a retrieved file.

---

## Workflow B — Write: add a new file

**Trigger:** user asks to add a new measure, filter, rule, verified query, subject, relationship, or disambiguation — single or batch.

### Steps

1. **Identify all files to create** from the user's description.

   **Before adding any `Reification:` file, check the intended Kind against `spec/schema.yaml`'s `reified_edge_kinds` list** — no other kind qualifies, however strong the "why it matters" narrative behind the edge (see [spec/logical-layer.md §2.2](../../spec/logical-layer.md#22-reified-edge-kinds-reification-pages--typed-relationships-with-properties) for why). If a caveat doesn't fit one of those kinds but still needs its own Reason/Consequence, put it on a `BusinessRule` file instead — check for an existing one first.

2. **Present a creation plan and ask for confirmation BEFORE creating anything.**
   Show a compact table — node type, count, names only. No file content.

   | Node type | Count | Names |
   |-----------|-------|-------|
   | Subject   | 1     | Write-Off |
   | Measure   | 2     | REVENUE, GROSS_MARGIN |
   | **Total** | **3** | |

   Ask: "Proceed with all N files?" — do not create anything until confirmed.

3. **Check for duplicates.** For each file in the confirmed list:
   ```bash
   rg "^name: <Name>$" <kg-root>/ --include="*.md" -l
   ```
   Skip existing files (switch to Workflow C). Report skipped files.

4. **Select the correct template** from [spec/page-templates.md](../../spec/page-templates.md). Convert the template from Confluence storage format to Markdown: header paragraph fields become YAML frontmatter; `<ac:link-body>` links become `[edge statement](relative/path.md)`.

5. **Determine the target path** using the directory structure from [markdown-adapter.md](markdown-adapter.md).

6. **Draft and write each file.** Show the draft to the user before writing.

7. **After each file is created**, update the domain index file (`domain-<name>.md`) to add a link to the new file under the correct section. Commit message: `Summary: Added link to <new file>. Changed: <section>. Reason: New file created. Breaking: no`

8. **Commit** all new files and domain index updates together with a structured version comment.

---

## Workflow C — Update: edit an existing file

**Trigger:** user asks to change a definition, formula, predicate, rule, or any other field.

**Decision point:** if this is a structural change affecting 6+ files (e.g. link format migration, frontmatter field rename, edge renaming), use the Knowledge Graph API instead (see [graph-api.md](graph-api.md)).

### Steps for targeted updates (≤ 5 files)

1. **Resolve the file.** Search by name, read the current content.

2. **Identify the change.** Confirm with user if not clear:
   - Which frontmatter field or section changes?
   - New value?
   - Breaking change?

3. **Apply the change** to the file. Keep all other fields and content unchanged.

4. **Confirm with the user** before saving:
   ```
   Field:    <field name>
   Before:   <old value>
   After:    <new value>
   Breaking: yes | no
   ```
   Ask: "Apply this update?" — do not write until confirmed.

5. **Write the updated file** and commit with a structured version comment (see [spec/versioning.md](../../spec/versioning.md)).

6. **If breaking:** find Reification files referencing this node — update `## Consequence if Ignored` if needed.

### Steps for bulk updates (6+ files)

1. Describe the structural change, enumerate affected files, ask for confirmation.
2. Use the Knowledge Graph API — see [graph-api.md](graph-api.md) for patterns.
3. Run on one file first. Show before/after diff. Confirm before running on all.
4. Execute the full batch only after confirmation.

---

## Workflow D — Navigate: lineage and graph traversal

**Trigger:** "what depends on X", "which rules apply to table Y", "show me all verified queries for measure Z", "what is the lineage of column C".

### Steps

1. **Resolve the starting file.** Search and read it.

2. **Determine traversal direction:**
   - **Downstream** (what uses X): search for files that reference this node:
     ```bash
     rg "<NodeType>: <Name>" <kg-root>/ --include="*.md" -l
     ```
   - **Upstream** (what X depends on): follow `## Links` and `## Reifications` links inside the file.

3. **Read linked files** for each relevant hop. Do not traverse the full graph — stop at the depth that answers the question.

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

5. If the user wants to go deeper on any node, read that file and continue.

---

## Workflow E — Version history: what changed and when

**Trigger:** "what changed in X", "show history of rule Y", "who updated filter Z".

### Steps

1. **Resolve the file path** by name search.

2. **Fetch git log** for the file:
   ```bash
   git log --follow --format="%H %as %an%n%B" -- <path/to/file.md>
   ```

3. **Parse the version comments** from commit message bodies. Git returns the hash, date, and author natively per entry; the message body itself follows:
   `Summary: ... | Changed: ... | Reason: ... | Breaking: yes/no`

4. **Present as a table**: Version, Date, Author, Summary, Breaking. Highlight breaking changes.

5. To restore a prior version: retrieve that version's content with `git show <hash>:<path>` and walk through Workflow C — do not silently overwrite.

---

## Constraints (always apply)

- **Never fabricate a node path.** Always resolve via search or by deriving the path from the `name` field using the naming convention.
- **Always read the current file content** before calling any write operation.
- **Every commit must set the commit message body** to the structured summary in the required format — never write it into the file content.
- **Prose belongs only on Subject and Disambiguation files.** All other files use structured frontmatter and predicate/definition blocks.
- **When creating a file, also update its domain index** to add a link to the new file.
- **No silent writes.** Show a draft or diff and wait for user confirmation before writing any file.
- **Do not dump raw file content into chat** if it is large — show field values only, not the full file body.

# Knowledge Graph API

> Programmatic interface for graph operations on the knowledge base — bulk updates, structural migrations, spec-compliance auditing, and local mirror generation.
> The Knowledge Graph API sits between the agent (MCP tool calls) and the backend (Confluence REST API today, graph database in the future).
> Use it instead of MCP tool calls when a change affects 6+ pages or requires programmatic operations outside the LLM context.

> **Naming note:** "Knowledge Graph API" refers to our interface for graph operations. It is distinct from the Confluence REST API (Atlassian's vendor HTTP API, which the Knowledge Graph API uses internally as its current transport layer) and from Confluence MCP (the agent-facing tool interface).

> **This is a guide/contract, not the implementation.** It describes what operations the API must expose and why, independent of any one codebase. The current reference implementation is a private package (not in this repo, since this spec repo stays generic/scrubbed) — see its own README for exact function signatures, file layout, CLI flags, and setup steps. If this document and that implementation ever disagree on *behavior*, the implementation's README is authoritative for "how it actually works today"; this document is authoritative for "what it's supposed to do and why."

## Why the Knowledge Graph API instead of MCP

Two reasons to prefer the Knowledge Graph API over MCP tool calls for bulk work:

1. **Token cost.** Each MCP tool call (`confluence_get_page`, `confluence_update_page`) consumes LLM context tokens — for the page body, the response, and the conversation turn. Updating 50 pages via MCP means 100+ tool calls inside a single agent session, which is expensive and slow. The Knowledge Graph API makes the same 100 HTTP calls outside the LLM entirely — zero token cost per page.

2. **Backend independence.** The Knowledge Graph API defines graph operations (get/update/delete a page, crawl the tree, search) as an interface. Today the transport layer is the Confluence REST API. When Confluence is eventually replaced or supplemented by a dedicated graph database, only the transport layer changes — the calling convention and the operations themselves stay the same. This is the migration path described in [spec/data-model.md §7](../../spec/data-model.md#7-graph-database-migration-notes).

---

## Setup contract

Credentials must be read from environment variables — never hardcoded in source. At minimum: an account identity, an API token, and the backend's base URL; optionally a default space/scope. The client should fail fast, at construction time, with a clear error naming which variable is missing — never fail deep inside an HTTP call with an opaque 401.

```
Client(identity?, token?, base_url?, scope?)
  reads any argument not passed explicitly from environment variables
  raises immediately if a required value is missing from both
```

---

## Core operations

Every higher-level operation (download, audit, migrate) is built from this minimal primitive set — an implementation's client object should expose exactly these, no more, no less:

```
get_page(page_id) -> { id, title, body, version: { number, ... } }
    Fetch one page's content and current version.
    Call this immediately before any update — the version number must
    be exact (current + 1) or the write is rejected (409 Conflict).

get_children(page_id) -> [ { id, title, ... }, ... ]
    Direct children only. Recursing over the subtree is the caller's job
    (see Crawling below) — this stays a flat, paginated primitive.

search(query) -> [ { id, title, body, ... }, ... ]
    Pattern search, body included to avoid a second round-trip per hit.
    Pattern matching (e.g. Confluence CQL `~`) is "contains," not exact —
    callers must still filter results themselves (e.g. exact-prefix check).

update_page(page_id, title, version, body, message?) -> page
    version must be current + 1. `message` becomes the page's *native*
    version comment (visible in its version history) — the version-history
    mechanism per spec/versioning.md. Never duplicate it as a text block
    inside the page body.

delete_page(page_id) -> None
    Permanent. No dry-run at this layer — the caller must confirm before
    invoking this, and must have already removed any stray references to
    the page being deleted (a dangling link is worse than a slow deletion).
```

## Crawling

Two distinct traversal shapes are needed on top of `get_children`, and using the wrong one for a task silently produces incomplete results:

```
collect_typed_nodes(root_ids, type_prefixes) -> [ { id, title }, ... ]
    BFS/DFS from root(s); keep only pages whose title matches a known
    node-type prefix ("Subject:", "Table:", ...). Correct for migrations,
    which only ever touch typed node pages.

collect_all_pages(root_id) -> [ { id, title, path }, ... ]
    BFS/DFS from root; keep every descendant — container/folder pages and
    index pages included, not just typed nodes. Required for anything that
    claims to check or mirror "the whole database": a spec-compliance
    audit or a full local-mirror download built on collect_typed_nodes
    instead will silently skip container/index pages, producing a false
    "0 findings" or an incomplete mirror with no error at all. (This is a
    real bug that was found and fixed once already.)
```

## Storage-layer helpers

Whatever the backend's native content format (Confluence storage-format XHTML, Markdown, etc.), transforms need a small set of shared building blocks so each one doesn't reinvent the same low-level parsing:

```
link(target_title, label?) -> a typed link to another page, optionally with
    a clickable label distinct from the target's own title
ensure_section(body, heading, new_items) -> body with new_items inserted
    under <heading>, creating the section if it doesn't exist yet
rename_section(body, old_heading, new_heading) -> body with the heading
    renamed in place; section contents untouched
remove_header_field(body, field_name) -> body with that metadata field
    stripped (e.g. once its content has moved into a section link)
```

Centralizing these is what keeps transforms short and focused on migration logic instead of on content-format munging — every ad-hoc script duplicating its own copy of these same helpers was exactly the problem a shared library is meant to fix.

## Transform convention

A transform is a pure function:

```
transform(page) -> (new_body, changed: bool)
```

Transforms don't talk to the backend directly — a shared runner applies them. The runner alone is responsible for: defaulting to dry-run (compute and report the diff, write nothing), re-fetching the page's current version immediately before any real write (never reuse a version number captured earlier in a batch), and reporting per-page success/no-change/error so one bad page in a large batch doesn't silently swallow the rest.

Transforms that must run in a specific order relative to another (e.g. a section-rename before a section-insert on the same page, to avoid creating a duplicate heading) should compose the earlier transform explicitly rather than relying on registration or execution order.

## Auditing

An audit is a read-only pass — one check function per spec-compliance rule:

```
check(page) -> [ Finding(page_id, title, category, detail), ... ]
```

Each check independently inspects a page and yields zero or more findings; it never modifies anything. The set of checks is not a fixed list — it must be kept in sync with the spec, growing whenever the spec changes in a way that could make existing live pages non-compliant. A check should validate against the spec's actual source of truth wherever a machine-readable one exists (e.g. a schema's enum of valid values), rather than hardcoding its own copy of that list — otherwise the check silently drifts out of sync the next time the spec's list changes, exactly the kind of duplication risk called out in [spec/data-model.md §2.2](../../spec/data-model.md#22-reified-edge-kinds-relationship-pages--typed-relationships-with-properties).

## Dry-run / apply pattern

Every write-capable operation must default to dry-run (print what would change, write nothing) and require an explicit, separate flag to actually write. Verify against a single representative page before running over a full page set. This applies uniformly to migrations and to destructive operations like page deletion.

---

## Back-reference injection rules

The three constraints from [spec/link-format.md](../../spec/link-format.md#back-reference-constraints) apply equally to MCP writes and Knowledge Graph API calls:

| Constraint | Symptom in raw content | Fix |
|---|---|---|
| No symmetric duplicates | Same target appears in `## Links` twice — once with `->` and once with `<-` | Before injecting `<-`, scan existing labels for `-> TargetName` with the same kind |
| Edge kind must match | The back-reference label uses a different verb than the owning page's label | Derive the `<-` label by replacing `->` with `<-` in the owning label — never infer the kind from node types |
| No `implement` between Subjects | `Subject: X implement <- Subject: Y` in `## Links` | Replace the edge kind with `relatedTo` |

---

For exact function signatures, file layout, CLI flags, dependencies, and implementation-specific pitfalls (regex/encoding edge cases, etc.), see the implementation package's own README.

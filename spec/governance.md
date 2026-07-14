# Governance

> Part of the [Knowledge Graph Specification](../SPEC.md).

> **Draft** — this document is work in progress and has not been formally reviewed.

---

## 1. What this covers

> Companion to [versioning.md](versioning.md), which covers how a change to a page is communicated. This document covers a gap versioning does not: how the graph finds out that a page *needs* to change.

`versioning.md` guarantees that when someone edits a `BusinessRule`, `Filter`, or `Measure` page, the change is communicated: a structured version comment, breaking-change propagation to dependent pages, notification of downstream consumers.

That mechanism only fires when a human already knows a KG page needs editing. It has no answer for the opposite failure: **the source system the page describes changes, and nobody tells the page.** The page sits there, unversioned, unedited, and wrong — and every agent that reads it keeps repeating the stale rule with full confidence, because nothing marked it as suspect.

This document defines who is responsible for catching that, and what should trigger a check.

---

## 2. The failure mode this addresses

A `BusinessRule` in a receivables domain instructed agents to always wrap an amount column in `ABS()` as a defensive default. The source semantic model later reversed that convention: sign is intentional, and applying `ABS()` inflates the resulting KPI. The source model changed. The KG rule did not. Nothing in the spec connects the two, so nothing surfaced the contradiction — an agent using the KG today would still generate SQL that double-counts.

This is not a versioning failure. `versioning.md`'s `Breaking: yes/no` field works fine for the page that changes. The gap is that **no page changed**, because no process pointed anyone at the rule that needed reviewing.

---

## 3. Ownership

| Role | Responsibility |
|---|---|
| **Domain owner** (`Domain.owner` property) | Accountable for the correctness of all `BusinessRule`, `Filter`, and `Measure` pages inside their domain. Not expected to personally verify every change — expected to ensure the review trigger below actually happens. |
| **Source model author** (whoever changes the upstream semantic model, e.g. a dbt model or equivalent) | Responsible for flagging, in their own PR description, whether the change affects a convention that a downstream KG rule might depend on (sign handling, aggregation logic, filter semantics, join cardinality). This is a cheap check at the point of most context — the author just changed the logic and knows why. |
| **KG reviewer** (whoever approves the source PR, or a designated KG steward if the two are decoupled) | Confirms the flag in the PR description was actually checked against the KG before merge, or explicitly waives it. |

This mirrors `versioning.md`'s existing breaking-change propagation step (dependent Reification pages, downstream notification, VerifiedQuery review) — it just adds the missing *inbound* direction: source system → KG, not just KG page → KG page.

---

## 3a. Change procedure: self-serve vs. review-required

`versioning.md` defines *how* a change is recorded (comment format, breaking-change propagation). This section defines *who must sign off before it's applied* — the gap between "I know what to change" and "it's now live in the graph."

Every change to the graph falls into exactly one of two tracks. The criteria below are deliberately general — evaluate a change against them rather than against a fixed list of node types, since that list is expected to grow (see §3b).

### Self-serve

A change may be applied directly by its author, with no second approver, when **all** of the following hold:

- **Non-breaking** per `versioning.md`'s breaking-change table (description/synonym edits, new `## Links` edge, verified-by/date updates).
- **Scoped to a single domain** the author owns (per `Domain.owner`) or a conceptual-layer page they authored per `conceptual-layer.md` §5.
- **Does not change a property that other pages' correctness depends on** — i.e. not `definition_sql`, `predicate_sql`, `expression_sql`, `mandatory`, `rule_modality`, or a `Reification` page's `reason`/`consequence`.

Self-serve changes still require a version comment (per `versioning.md`) — "no review required" is not "no record."

### Review-required

A change requires a second approver (the domain owner if the author isn't the owner, or a designated KG steward if the author is the domain owner) before or immediately after it is applied, when **any** of the following hold:

- **Breaking**, per `versioning.md`'s table — SQL/predicate changes, mandatory filter added/removed, `BusinessRule` definition changed, status set to `Deprecated`, node renamed.
- **Crosses domains** — a `relatedTo`, `exactMatch`/`closeMatch`, or conceptual-layer edge that a different domain's pages depend on.
- **Creates or removes a node type or edge kind** — always a spec change (`schema.yaml` + CHANGELOG), never a page-level decision regardless of how small it looks.
- **Triggered by §4's mechanical check** — a source-system change that surfaced a potentially stale KG page always goes through review; the whole point of §4 is that the author (the source-model changer) is often not the KG domain owner and may not have full context to self-certify.

Review-required changes follow: propose (draft the change, cite the trigger — §4 finding, new requirement, correction) → domain owner or steward confirms → apply with version comment → propagate per `versioning.md`'s breaking-change propagation steps.

### Why a binary split, not a graduated one

A three-tier model (self-serve / lightweight review / full review) was considered and rejected for now: the two properties that actually matter — *can this silently make an agent's SQL wrong* (breaking) and *does it affect anyone outside the author's own accountable scope* (cross-domain, cross-node-type) — both already exist as binary tests elsewhere in the spec (`versioning.md`'s breaking table; `Domain.owner`). Introducing a third tier would mean inventing a new test with no existing binary signal behind it. Revisit if a real case appears where treating something as fully review-required is too heavy but self-serve is too loose — but adjudicate that case's own criteria on its merits.

---

## 3b. Extending this split

Both lists in §3a are meant to grow. When a new node type, edge kind, or property is added to the spec, classify it against the same two questions rather than leaving it unclassified:

1. Is a bad value on this property something another page's correctness silently depends on (→ review-required), or is it purely descriptive/navigational (→ self-serve)?
2. Does this property or edge ever cross a domain boundary or a node-type boundary (→ review-required), or is it always contained within one author's accountable scope (→ self-serve)?

Record the classification in the CHANGELOG entry that introduces the property — e.g. `BusinessRule.rule_modality` changes which fields are required (`consequence_if_violated` becomes optional when `necessity`) but does not itself change SQL construction, so an edit to it alone is self-serve; but if it is edited *alongside* `consequence_if_violated` or `definition`, the combined change is judged by those fields' review-required status, not by `rule_modality` in isolation.

---

## 4. Review trigger

A source-system change should prompt a check against the KG when it touches any of:

- Sign, rounding, or unit conventions on a column already referenced by a `BusinessRule` or `Attribute.expression_sql`
- Aggregation logic (SUM vs. AVG, inclusion/exclusion filters) behind a `Measure.definition_sql`
- A filter predicate's meaning (e.g. what a status code means, which values are terminal vs. in-progress) behind a `Filter.predicate_sql`
- Join cardinality behind a `Table joinedTo Table` edge, if it changes from 1:1 to 1:many or introduces fan-out risk

**Mechanical check.** Before merging a source-system PR that touches one of the above, search the KG for pages whose `definition`, `definition_sql`, `expression_sql`, or `predicate_sql` reference the changed column or table. If a match exists, the PR description must state one of:
- "Checked — no downstream KG rule affected."
- "Checked — `Rule: <name>` / `Measure: <name>` updated in the same change or a linked follow-up."
- "Checked — `Rule: <name>` flagged for review, ticket: `<link>`."

This is a manual grep-and-confirm step, not tooling that needs to be built before this process can start. Automating the search (e.g. a CI check that greps upstream model diffs against the KG node index for referenced column/table names) is a natural follow-up, not a prerequisite.

---

## 5. Change log

This is a log of changes to **graph objects** (node and Reification pages) — who changed what, when, and why. It is a different axis from [`CHANGELOG.md`](../CHANGELOG.md), which tracks changes to the **spec itself** (node types, edge kinds, schema fields). Do not conflate the two: adding `rule_modality` to `BusinessRule` is a `CHANGELOG.md` entry; a domain owner editing one `Rule:` page's definition next month is a governance change-log entry. The former changes what a page *can* contain; the latter changes what one page *does* contain.

There is no separate log to maintain by hand. Every content storage adapter is already required to provide **version history** as a capability (`engine.md`'s content storage capability table) and every change is already required to carry a structured version comment (`versioning.md`: `Summary`/`Changed`/`Reason`/`Breaking`). The change log is this history, read back:

- **Per-page log** — the backend's native version history for that page (Confluence page history, git log for the file), which already carries author, timestamp, and the structured comment. Nothing new to build.
- **Graph-wide log** — a derived view across all pages, produced the same way the node/edge indexes are: an engine adapter capability that walks content storage and aggregates each page's latest version metadata (author, date, `Summary`/`Reason`/`Breaking`) into one feed, keyed by page. This is a projection, exactly like the Graph DB's structural projection (`engine.md` §"What does 'structural projection' mean?") — it doesn't store anything the backend doesn't already have, it just makes "what changed across the graph in the last N days" answerable without opening every page.

The graph-wide log is not yet a required capability — no adapter currently implements it. Treat it as the natural next addition to `engine.md`'s content storage capability table (alongside Hierarchical organisation, Cross-document links, Version history, Full-text/semantic search, Programmatic read/write) once a concrete consumer needs it (e.g. a weekly digest of what changed per domain, or feeding §6's staleness checks with "which rules were touched most recently").

---

## 6. Monitoring

Where §4 catches staleness triggered by an *external* event (the source system changed), monitoring catches problems that accumulate *internally* — structural decay that no single edit introduces but that builds up over many independent changes. This is a graph-quality check, run against the graph, not against any one page.

The mechanism already exists and does not need to be invented: every content storage adapter provides an auditing capability — a read-only pass that checks pages against spec-compliance rules and reports findings (see `graph-api.md` §"Auditing" for any given adapter). `logical-layer.md` §6's audit rules (`no_duplicate_nodes`, `no_duplicate_edges`, `reified_beats_hyperlink`, `back_refs_excluded`, `relationship_pages_flattened`) already use this mechanism. Monitoring means running that same auditing capability on a recurring schedule instead of only after a snapshot regeneration, plus adding a few checks that need history across pages instead of just one page's structure:

| Check | What it catches | Basis |
|---|---|---|
| **Duplicate nodes** | Two pages for the same real-world thing (e.g. two connectors independently creating a `Subject: Customer`) | Already an audit rule (`logical-layer.md` §6, `no_duplicate_nodes`). Monitoring makes it a recurring check, not just a pre-creation guard. |
| **Orphaned `Reification` pages** | A reified edge whose `from`/`to` page was renamed or deleted, leaving a dangling reference | New check: confirm every `Reification` page's `from`/`to` still resolves to a live node. |
| **Domains missing `owner`** | A domain with no accountable owner, so §3's ownership model has no one to page | New check: `Domain.owner` is optional in schema — flag any `Domain` node where it's unset. |
| **Stale rules** | A `BusinessRule` whose linked `Table`/`Attribute` has been edited more recently than the rule itself, suggesting the rule may not have been re-certified | New check, using §5's change history: the mechanical, always-on complement to §4's PR-time check — §4 catches it at the moment the source changes; this catches it if §4 was skipped or the connection wasn't obvious at review time. |
| **Deprecated nodes still referenced** | A `Measure`/`VerifiedQuery` with `status: Deprecated` that still has live inbound edges, meaning agents may still be traversing into it | New check: cross-reference `status` against inbound edge count. |

None of these require a new schema field — every input already exists in the node index, edge index, or §5's change history. What's missing is running existing checks on a schedule and adding a few more of the same kind, not building new infrastructure. The risk this guards against is real: `graph-api.md` §"Crawling" documents an incident where an audit silently produced a false "0 findings" result because it scanned an incomplete set of pages — the exact failure mode monitoring exists to prevent is a check that's defined but not actually running, or running against incomplete input, with nothing surfacing the gap. Scheduling and the cross-page checks (stale rules, orphaned Reifications) are proposed additions, not yet implemented by any adapter.

---

## 7. What this does not require

- No new node type or property. A `reviewed_at` or `superseded_by` field on `BusinessRule` was considered and rejected: a timestamp only helps if someone already knows to set it; it doesn't create the trigger that gets it set. This gap is procedural, not structural — no schema addition fixes it.
- No schema version bump. This is a process document, not a schema change.
- No mandatory tooling investment up front. The check in §4 is a manual step added to existing PR review; §5's graph-wide log and §6's monitoring checks are proposed engine adapter capabilities, not requirements — automation is optional future work, and the per-page mechanisms they build on (version history, structured comments, audit rules) already work today without it.

---

## 8. Open questions

- Should the "search the KG" step in §4 be scoped to the domain(s) the source model feeds, or the whole graph? Scoping to domain keeps the check cheap but risks missing cross-domain references (an `Attribute` reused via `relatedTo` from another domain).
- Should a stale rule that's caught late (like the `ABS()` case) get a distinct marker beyond a normal edit + version comment — e.g. calling it out explicitly as a correction in the version comment's `Reason` field, so anyone auditing history can see it was a re-certification fix rather than a routine wording change? Current recommendation: yes, use `Reason: source convention changed upstream, rule was stale` — no new field needed, this fits the existing `versioning.md` comment format.
- Who runs §6's monitoring checks, and how often? This document defines *what* to check but not the operational owner (KG steward? scheduled CI job?) or cadence (daily? on every snapshot regeneration?) — deferred until a concrete engine adapter implements the graph-wide log the checks depend on.

---

## Provenance

This document was drafted after finding a live instance of the failure mode described in §2: a `BusinessRule` page describing a sign convention had gone stale relative to a change in its upstream source model, with no process step that would have caught it. Fixing that specific instance is a separate, out-of-scope follow-up from this design document.

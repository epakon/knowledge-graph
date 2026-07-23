# Consumption Layer

> Part of the [Knowledge Graph Specification](../SPEC.md).
> For the conceptual layer (Concept, Subject, Process) see [conceptual-layer.md](conceptual-layer.md).
> For the logical layer (Table, Measure, Attribute, Filter, BusinessRule, VerifiedQuery, Disambiguation) see [logical-layer.md](logical-layer.md).

> **Draft** — this document is work in progress and has not been formally reviewed.

---

## 1. Purpose

The consumption layer holds knowledge about **who reads the graph and how they should behave when they do** — as opposed to the conceptual layer (what the business means) or the logical layer (how it's implemented in SQL). It is the only layer whose nodes are not, themselves, business or technical facts about the data warehouse — they are facts about a consumer of the graph.

**The stability test.** A node belongs in this layer if it would still be meaningful after swapping the underlying delivery technology — an LLM agent replaced by a different agent framework, a Snowflake Cortex Agent replaced by a Claude/Cursor Skill, this year's orchestration model replaced by next year's. "This agent answers questions about order and revenue data and must always disclose which semantic view it used" survives that swap. "Set `orchestration.budget.seconds` to 120" does not — that is a Cortex-specific deployment detail and belongs in a consumer adapter, never on the node itself.

| Property | Value |
|---|---|
| **Path** | `ai/` — deliberately broader than "agents": `Agent` is this layer's only node type today, but the folder name shouldn't have to change if a second consumption-layer node type is added later (see §6) |
| **Scope** | Global — an agent is not owned by a single domain, even when most of its edges point into one (mirrors `Domain`'s own global scope) |
| **Authored by** | Whoever owns the agent/skill's behavior — typically the same data engineers who own the domains it reads, not a separate "AI team" by default |
| **Changes when** | The agent's purpose, behavior, or routing logic changes — or when a node it depends on changes in a way that affects correctness (see §5) |

**Why "Consumption" and not "AI".** The layer is named for the durable relationship it captures — *something reads specific graph elements and needs guidance to read them correctly* — not for the technology currently delivering that reading. `Agent` is the only node type in this layer today because AI agents are the graph's primary consumer; the name is chosen so that a future non-agent consumer (a scheduled reconciliation job, for instance) would not require renaming the layer to be added to it. This is a deliberate choice, not a hedge: see §6 for why no such second node type exists yet.

**What does NOT belong here.** Anything that is a fact about the business (→ conceptual layer) or a fact about the SQL implementation (→ logical layer) does not move here just because an agent happens to read it. This layer holds only the agent's own behavior — its purpose, its response/orchestration rules, its sample questions — plus the edges pointing at what it reads. It never duplicates the content of what it reads; see §5.

---

## 2. Node Type Schema

### 2.1 Agent

An `Agent` is a defined AI consumption surface — a Snowflake Cortex Agent, a Claude/Cursor Agent Skill, an MCP tool, or any future equivalent — represented once, in vendor-neutral form, regardless of how many downstream systems render it.

| Property | Type | Required | Description |
|---|---|---|---|
| `purpose` | string | Yes | One sentence — what this agent answers questions about. No SQL, no vendor syntax. |
| `response_instructions` | string | No | Vendor-neutral rules for how the agent should format or present answers (e.g. always disclose which node was used, never substitute an inaccessible view). |
| `orchestration_instructions` | string | No | Vendor-neutral rules for choosing between multiple tools/views and routing a question to the right one. |
| `sample_questions` | list[string] | No | Representative questions this agent is expected to answer. |
| `status` | enum (`Active`, `Deprecated`) | Yes | `Deprecated` means this agent has been merged into or superseded by another — see §8. A consumer adapter must treat `Deprecated` as "do not render or keep deployed," not merely a note. |

Everything vendor-specific — Cortex's `orchestration.budget.seconds`, a Claude Skill's `allowed-tools`, Snowflake's requirement that `skills: []` stay empty — is a rendering detail owned by the relevant consumer adapter (§7), never a property on this node. If a property only makes sense for one vendor's syntax, it fails this layer's stability test (§1) and does not belong here.

### Uniqueness constraint

```cypher
CREATE CONSTRAINT FOR (n:Agent) REQUIRE n.name IS UNIQUE;
```

---

## 3. Edge Type Schema

### 3.1 `uses` — the only edge kind in this layer

| Kind | Reification type | Valid source labels | Valid target labels | Notes |
|---|---|---|---|---|
| `uses` | `USES` | Agent | Table, Measure, Attribute, Filter, BusinessRule, VerifiedQuery, Subject, Domain | Owning side: `Agent` page. Plain hyperlink — not reified. |

**Why plain hyperlink, not reified.** `uses` is navigational/traceability, the same tier as `contain` or `calculate` — it answers "which agents depend on this node," not "what breaks if this dependency is violated." There is no reason/consequence story to tell for the fact that an agent reads a table, so promoting it to a Reification page would produce an empty page (see [logical-layer.md §3](logical-layer.md#why-this-list-is-closed-for-now) for the general rule).

**Why a dedicated kind and not `relatedTo` or the existing `consumes`.** `conceptual-layer.md` §3.1 already defines `consumes` for `Process -> Subject` ("the process requires this Subject's data as an input to operate"). `uses` is semantically similar but deliberately a separate kind: it needs to be free to grow its own properties (e.g. a future `usage_frequency` or `criticality` field scoped to agent traceability) without those properties leaking onto the unrelated `Process -> Subject` relationship, and without forcing every future `Agent -> Table` edge through the generic, property-less `relatedTo`.

**Back-reference.** Every node an `Agent` uses carries the back-reference in its own `## Links`:

```
Owning (on Agent page):    [Agent: SALES_ASSISTANT uses -> Table: ORDERS]
Back-ref (on Table page):  [Agent: SALES_ASSISTANT uses <- Table: ORDERS]
```

This is what makes the layer's core promise real: from any `Table`, `Measure`, `Filter`, or `Subject` page, a human or agent can see every `Agent` that depends on it, with no separate registry to query.

---

## 4. Space structure

```
ai/
└── Agent: <Name>
```

Global, at the KG root — parallel to `vocabulary/`, not nested under any domain, since an `Agent` commonly crosses domain boundaries (a sales agent may read tables owned by an orders domain and a `Subject` bridged from a finance domain).

**Note on naming.** The folder is `ai/`, not `agents/` — broader than the current single node type, so that a second consumption-layer node type would not require a folder rename. This mirrors §1's reasoning for calling the *layer* "Consumption" rather than "AI," with one intentional difference: the folder itself uses "ai" rather than "consumption," which is a narrower, more technology-flavored word than the layer name. That's a deliberate author choice, not an inconsistency to be resolved — the layer's normative name (and its stability test) is what governs classification decisions in §6; the folder name is cosmetic.

---

## 5. Relationship to existing content — never a second authored copy

An `Agent` node must never restate the content of what it uses. If an agent needs to explain a join caveat, a synonym, or a calculation rule to answer questions correctly, that explanation belongs on the `BusinessRule`, `Attribute.synonyms`, or `Measure` page it already has — or should have — and the `Agent` page links to it via `uses`. The only content that legitimately lives on the `Agent` page itself is the agent's own behavior: how it should respond, how it should route between tools, what it's for.

This is the same one-authored-source discipline the rest of the spec already enforces (`logical-layer.md`'s promotion criteria, `governance.md`'s staleness checks) — it just applies it to a new class of reader. Concretely: if a Cortex Agent's `tool_spec.description` currently hand-types "Partner = Hotel_id" as a synonym, the fix is not to also write it on the `Agent` node — it's to put it on `Attribute.synonyms` (if not already there) and have the rendering step (§7) pull it in via the `uses` edge.

Because `Agent` participates in ordinary graph edges, it automatically inherits mechanisms that already exist for this exact problem, with no bespoke addition:

- **Breaking-change propagation** (`versioning.md`) — a `Breaking: yes` change to a `Table` or `BusinessRule` an `Agent` uses is already in that node's fan-out once the `uses` edge exists.
- **Staleness monitoring** (`governance.md` §6) — the existing "stale rule" check (a dependency edited more recently than the page that depends on it) applies to `Agent` pages the same way it applies to any other dependent page.
- **Review-required classification** (`governance.md` §3a) — an edit to `Agent.response_instructions` or `orchestration_instructions` that changes what SQL gets generated is review-required, the same tier as editing a `BusinessRule.definition`.

---

## 6. Why only one node type today

`space-structure.md`'s "What does NOT live in the knowledge base" section excludes dashboard and report definitions — deliberately, and that exclusion still stands. The distinction: a dashboard is a static, pre-built, human-reviewed artifact; it does not autonomously interpret a `Subject`'s meaning or generate SQL at runtime the way an `Agent` does. An `Agent`'s behavior can silently produce a wrong answer if it reads the wrong node or misapplies a rule — that is precisely the correctness-propagation problem §5 hooks into `versioning.md`/`governance.md` for. A dashboard's correctness was already settled at build time by a human; there is no equivalent live failure mode to guard against, so there is nothing a graph node would add.

If a second consumption-layer node type is ever proposed (a scheduled job, a non-agent API consumer with the same live-interpretation risk), classify it against `governance.md` §3b's two questions before adding it — this layer is named `Consumption`, not `AI`, precisely so that classification can happen without a rename.

---

## 7. Rendering — consumer adapters

Compiling an `Agent` node plus its `uses` targets into a specific vendor's format (Cortex Agent DDL, a Claude/Cursor `SKILL.md`, an MCP tool definition) is the job of a **consumer adapter** — a new adapter category alongside the existing `adapters/engine/` (content storage) and `adapters/connectors/` (source systems): `adapters/consumers/`. Each consumer adapter documents its own mapping from `Agent` + `uses` targets to that vendor's syntax, the same four-document contract shape the engine adapters already use. No consumer adapter exists yet in this repository; this section reserves the category for when one is authored.

A consumer adapter **must** check `Agent.status` before rendering or deploying anything: `Deprecated` means do not render, and if a previously-deployed artifact exists for this agent, decommission or redirect it. This is what makes §8's merge procedure actually reach the user instead of stopping at the graph page.

---

## 8. Overlap and deduplication

Agent sprawl — many overlapping agents built independently, with no shared visibility — is a known failure mode for this layer specifically: duplication is easy when nothing surfaces what already exists before someone builds the next thing. This section defines a KG-native check for it: cheap, non-blocking, and deliberately tolerant of overlap that has a real reason behind it.

### 8.1 What gets measured

Every `Agent` already has a `uses` edge set. Two `Agent` nodes with a high-overlap `uses` set — measured as the Jaccard similarity of their target nodes, computed from the edge index, no embeddings or new infrastructure required — are candidates for functional duplication. This is deliberately a **structural**, not semantic, signal: it will miss two agents that overlap in intent but were built against differently-named nodes, and it will flag two agents that happen to read the same tables for unrelated reasons. Both are acceptable false-negative/false-positive rates for a cheap, always-on check whose output is a human review prompt, not a verdict.

### 8.2 The reasonable-overlap test

A high `uses`-overlap pair is **not** a duplicate if the two `Agent` pages differ in at least one of:

- **`response_instructions`** — different audience or presentation (e.g. an executive-summary agent vs. an analyst deep-dive agent reading the same tables)
- **`orchestration_instructions`** — different question-routing behavior over the same data
- **Scope shape** — one is a deliberate broader "front door" over several narrower agents (an intended pattern, not a smell)

If none of these differ — same `uses` set, same audience, same routing, different name only — treat it as a true duplicate and merge (§8.4).

### 8.3 Where the check runs

- **At creation** — the duplicate-check step every new page already goes through (`agent-skill.md` Workflow B) is extended for `Agent` specifically: run the `uses`-overlap check against existing `Active` agents before creating a new one. Non-blocking — if the author has a real answer against §8.2, they proceed, but must record it once on the new page as a `## Differentiation` section (see the page template) so the next overlap review doesn't re-raise a question that was already answered.
- **On a schedule** — two agents built independently by different teams can converge over time as their `uses` sets grow, with neither author aware of the other. Add `agent_overlap_review` (see `schema.yaml`) to `governance.md` §6's recurring monitoring rather than treating this as a creation-time-only check.

### 8.4 Merge procedure

When a review concludes two `Agent` pages should collapse into one:

1. Set the losing page's `status` to `Deprecated`.
2. Add `[Agent: <Name> relatedTo -> Agent: <surviving>]` on the deprecated page, documenting the merge target.
3. Set the version comment per `versioning.md`: `Breaking: yes` (status changed to `Deprecated` is already breaking per that spec's table) — this triggers the existing propagation steps, and §7's consumer-adapter requirement takes it from there.

### 8.5 Why not embedding-based similarity yet

A semantic check (e.g. embedding the `purpose`/`sample_questions` text and comparing similarity) would catch overlap the structural `uses` check misses — two agents with genuinely similar intent but little table overlap. That is deliberately left to whatever front-door/orchestration tooling sits above this graph, if any: it depends on an embedding capability this spec does not otherwise require, and reuse-before-build applies to governance tooling too — start with the free structural check here, and treat semantic dedup as an optional addition layered on top, not a reason to delay this one.

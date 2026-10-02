# Feedback Connector

> **Draft** — this document is work in progress and has not been formally reviewed.

This connector reads the work of agents that already use the Knowledge Graph and turns it into **proposals** for the graph. It evaluates existing pages against what happened in agent sessions and proposes new pages where sessions show a gap.

It differs from the other connectors in two ways:

- **Its source is the graph's own consumers**, not a business system. A session is evidence, never a source of truth.
- **It never writes to the graph.** Every output is a proposal in the backend's review mechanism (a git merge request, a Confluence draft) and is always review-required under [`governance.md`](../../../spec/governance.md) §3a.

It adds no node types, edge kinds or properties. Evidence for a proposal goes into the `Reason` field of the version comment ([`versioning.md`](../../../spec/versioning.md)).

---

## Section 1 — Source system overview

The source is the session log of any `Agent` ([`consumption-layer.md`](../../../spec/consumption-layer.md)) that reads the graph: the user's question, the KG pages the agent read, the SQL it generated, and how the user reacted.

Each session produces one of three findings:

| Finding | Signal in the session | Outcome |
|---|---|---|
| **Confirm** | The agent followed the pages and the user accepted the answer. | No proposal. Confirmations are attached as supporting evidence when the same page is reviewed for another reason. |
| **Contradict** | The user corrected an answer that followed a page, or SQL built from a page failed. | A proposal to change that page, routed to its domain owner as a staleness review ([`governance.md`](../../../spec/governance.md) §4). |
| **Gap** | The agent found no page and had to infer a rule, join or meaning, or had to ask the user a clarifying question. | A proposal to create the missing page or edge. |

**Lifecycle.** Sessions carry no lifecycle signal. One session is weak evidence; the connector groups findings by target page, and the reviewer decides how much evidence is enough.

---

## Section 2 — Node type mapping

| KG node type | Session evidence | Key mapping notes |
|---|---|---|
| `Subject` | A business term asked about repeatedly with no matching Subject | Gap proposal only; a domain expert writes the definition |
| `Policy` | A user states a business rule while correcting the agent ("returns never count as revenue") | `statement` drafted from the user's words, without SQL |
| `Filter` / `BusinessRule` | A predicate the user made the agent add | Drafted from the accepted SQL; linked to the Policy it applies, if any |
| `Measure` | A user corrects a formula behind a Measure | Contradict proposal on the existing page |
| `Table` / `Attribute` | A user corrects a column meaning or a caveat | Contradict proposal on the existing page |
| `VerifiedQuery` | SQL the user accepted for a recurring question | Candidate only; `verified_by` and `verified_at` are set by the human reviewer |
| `Disambiguation` | The agent had to ask which meaning the user intended | `always_ask` drafted from the clarifying question |
| `Reification` | An answer was rejected because a filter was missing | Proposed `mandatory` or `requires` edge |
| `Domain` | not applicable | Domain structure is an ownership decision, not session evidence |
| `Agent` | not applicable | Agent pages are authored from the agent's configuration |
| `Concept` / `Process` | not applicable | Too abstract to infer from single sessions |

---

## Section 3 — Field mapping tables

The connector fills existing fields only.

| KG field | Session source | Transformation | Notes |
|---|---|---|---|
| `name` | Term or rule from the question or correction | Follow the type's naming convention | Checked against the node index before proposing |
| `statement` (`Policy`) | User's correction | Rephrase as one business-language rule | No table or column names |
| `predicate_sql`, `definition`, `definition_sql` | SQL the user accepted | Extract the relevant fragment | Never from SQL the user rejected |
| `question`, `sql` (`VerifiedQuery`) | Accepted question and SQL | As-is, with result values removed | |
| `always_ask` (`Disambiguation`) | Agent's clarifying question | Generalize away from the single session | |
| `reason`, `consequence` (`Reification`) | User's correction and the wrong answer | One sentence each | Reviewer confirms the consequence |
| `verified_by`, `verified_at` | — | Never set | Human verification only |
| Version comment `Reason` | Session references | Session id, date, quoted correction | Strip personal data and result values |

---

## Section 4 — Edge mapping

| KG edge kind | Session evidence | How to detect | Notes |
|---|---|---|---|
| `uses` | Pages the agent actually read | Tool calls in the session | Proposes missing `Agent uses ->` edges; flags declared edges never used |
| `implement` | A filter or rule applied because of a stated business rule | Correction text names the rule | Proposed on the logical page (`implement -> Policy/Subject`) |
| `disambiguate` | A clarifying question resolved a term | Agent asked, user chose | Proposed on the Disambiguation page |
| `joinedTo` | A join the agent had to infer | Join in accepted SQL with no matching edge | |
| `mandatory`, `requires` (reified) | An answer rejected for a missing filter | Correction adds a predicate | `reason` and `consequence` per Section 3 |
| `relatedTo` | not derivable | | Requires manual authoring |

---

## Section 5 — Extraction protocol

**System metadata channel.** Session logs exported by each agent runtime. Read per session: the question, the KG pages read, the SQL generated, errors, and the user's next turn (accepted, corrected, or abandoned).

**Project documentation channel.** Not applicable. The human step is the domain owner's review of each proposal.

**Incremental detection.** Process sessions closed since the last run; group new findings with open proposals on the same page instead of opening another.

**What cannot be extracted.** Whether a correction is right. A user can be wrong, and an accepted answer can still be wrong. This judgment stays with the reviewer.

---

## Section 6 — Import procedure

The connector never writes directly to the graph.

1. **Validation** — each proposal passes the [`schema.yaml`](../../../spec/schema.yaml) required-field and enum checks.
2. **Node creation** — new pages are drafted with the standard templates and placed in their normal container path, inside a proposal.
3. **Edge creation** — edges follow [`link-format.md`](../../../spec/link-format.md); cross-layer edges are placed on the logical page.
4. **Duplicate handling** — if the node already exists, the proposal becomes an edit to it; findings from several sessions on one page merge into one proposal.
5. **Post-import audit** — the audit rules run on the proposal branch or draft before it is offered for review.
6. **Version comment** — the reviewer applies the change with a version comment whose `Reason` cites the sessions.

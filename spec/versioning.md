# Versioning

> Part of the [Knowledge Graph Specification](../SPEC.md).

---

## Overview

The backend (wiki, git, etc.) versions every page update automatically — no manual changelog is needed for individual page changes. Version history lives entirely in the **backend's native versioning mechanism**: the Confluence page version comment (the `version.message` field on an update) or a git commit message for the Markdown adapter. It is **not** a text block written into the page body — page content should stay focused on the node's definition, not accumulate a growing changelog that distracts from it.

The structured format below is the required shape for that native comment/message, so agents and humans can parse *what* changed, *why*, and *whether it is breaking* from version history without re-reading the whole page.

---

## Version comment format

Every page update must set the backend's native version comment to:

```
Summary: <one sentence describing the change>
Changed: <section or field name>
Reason: <why the change was made>
Breaking: yes | no
```

Do **not** prefix this with `v<n> | <date> | <author>` — the backend already records the version number, timestamp, and author natively (Confluence: version number/`when`/`by`; git: commit hash/date/author). Repeating them inside the message is redundant.

### Where it lives, per backend

| Backend | Mechanism | API/field |
|---|---|---|
| Confluence | Native page version comment | `update_page(..., message=...)` → `version.message`, shown in the page's version history UI |
| Markdown (git) | Commit message | `git commit -m "<Summary/Changed/Reason/Breaking block>"` |

See [`adapters/engine/confluence/graph-api.md`](../adapters/engine/confluence/graph-api.md) and [`adapters/engine/markdown/graph-api.md`](../adapters/engine/markdown/graph-api.md) for the exact call shape.

### Fields

| Field | Description |
|---|---|
| `Summary` | One sentence describing what was changed. |
| `Changed` | The section or field that was modified (e.g. `## Definition`, `Mandatory`, `## SQL`). |
| `Reason` | Why the change was made (e.g. business rule changed, bug fix, clarification). |
| `Breaking` | `yes` if the change alters SQL construction or query interpretation; `no` otherwise. |

### Example

Confluence version comment (passed as `message` to `update_page`, not written into the page body):

```
Summary: Corrected predicate to exclude pending documents.
Changed: ## Predicate
Reason: Pending documents were being included in calculations incorrectly.
Breaking: yes
```

---

## What counts as a breaking change

A change is **breaking** if it could cause an agent or SQL query that was correct before to become incorrect after the change:

| Change type | Breaking? |
|---|---|
| Node renamed | Yes — all links to the old name break |
| SQL definition changed | Yes — queries built from the old definition are wrong |
| Filter predicate changed | Yes |
| Mandatory filter added or removed | Yes |
| BusinessRule definition changed | Yes |
| Description / synonym updated | No |
| New `## Links` edge added | No |
| Verified by / date updated | No |
| Status changed to Deprecated | Treat as breaking — agents should stop using the node |

---

## Breaking change propagation

When a change is marked `Breaking: yes`:

1. **Update dependent Relationship pages** — check `## Consequence if Ignored` for accuracy.
2. **Notify downstream consumers** — annotate the version comment with `BREAKING CHANGE:` prefix if the backend supports full-text search indexing of version comments.
3. **Review VerifiedQuery pages** that reference the changed node — their SQL may need updating.

---

## Schema versioning (this specification)

This specification uses [Semantic Versioning](https://semver.org):

| Increment | When to use |
|---|---|
| `MAJOR` | Breaking change to the spec: node type renamed or removed, edge kind removed, folder path changed, required template field removed |
| `MINOR` | Additive change: new node type, new edge kind, new template section, new optional field |
| `PATCH` | Clarification, wording fix, non-breaking convention update |

The current version of the specification is tracked in [CHANGELOG.md](../CHANGELOG.md).

---

## Rollback procedure

To restore a prior version of a page:

1. Retrieve the version history for the page (via backend API or UI).
2. Identify the version to restore by its version number, date, and summary.
3. Retrieve that version's content.
4. Apply it as a new update (do not silently overwrite the current version).
5. Set the native version comment: `Summary: Rolled back to v<k>. Changed: all. Reason: <why>. Breaking: yes/no`

---

## Agent versioning requirements

Agents that update pages via API must:

- **Always fetch the current version number** immediately before updating — never use a cached version number.
- **Always set the backend's native version comment** in the required format — never write it into the page body.
- **Increment the version number** by exactly 1 from the fetched value.
- **Mark breaking changes** as `Breaking: yes` and check dependent pages.

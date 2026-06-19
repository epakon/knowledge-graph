# Versioning

> Part of the [Knowledge Graph Specification](../SPEC.md).

---

## Overview

The backend (wiki, git, etc.) versions every page update automatically — no manual changelog is needed for individual page changes. The version comment format below provides structured metadata that agents and humans can parse to understand what changed, why, and whether it is breaking.

---

## Version comment format

Every page update must include a structured version comment:

```
v<n> | <YYYY-MM-DD> | <author>
Summary: <one sentence describing the change>
Changed: <section or field name>
Reason: <why the change was made>
Breaking: yes | no
```

### Fields

| Field | Description |
|---|---|
| `v<n>` | Version number (integer, incremented by 1 per update). |
| `<YYYY-MM-DD>` | Date of the update. |
| `<author>` | Name or identifier of the person or agent making the change. |
| `Summary` | One sentence describing what was changed. |
| `Changed` | The section or field that was modified (e.g. `## Definition`, `Mandatory`, `## SQL`). |
| `Reason` | Why the change was made (e.g. business rule changed, bug fix, clarification). |
| `Breaking` | `yes` if the change alters SQL construction or query interpretation; `no` otherwise. |

### Example

```
v7 | 2026-06-15 | jane.smith
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
| New `## Related` edge added | No |
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
5. Include a version comment: `v<n+1> | <date> | <author> | Summary: Rolled back to v<k>. Changed: all. Reason: <why>. Breaking: yes/no`

---

## Agent versioning requirements

Agents that update pages via API must:

- **Always fetch the current version number** immediately before updating — never use a cached version number.
- **Always include a version comment** in the required format.
- **Increment the version number** by exactly 1 from the fetched value.
- **Mark breaking changes** as `Breaking: yes` and check dependent pages.

# Agent Skill — Confluence

> Confluence tooling for the [core agent skill](../agent-skill.md), which defines the intents, workflows and common constraints.
> Read the [Confluence Adapter](confluence-adapter.md) and [SPEC.md](../../../SPEC.md) before starting.

> **Prerequisites:** The Atlassian MCP server must be configured and authenticated before any workflow can execute. All read and write operations go through MCP tools (`confluence_search`, `confluence_get_page`, `confluence_create_page`, `confluence_update_page`, `confluence_get_version_history`, `confluence_get_child_pages`). Without an active Atlassian MCP connection, no workflow is possible.

---

## Prerequisites

In addition to the [core prerequisites](../agent-skill.md#prerequisites):

1. Know the space key and parent page ID for the target domain (from your instance reference table — never fabricate a page ID).
2. For write operations: fetch the current page version number immediately before calling `confluence_update_page`.

---

## Backend operations

| Operation | Confluence |
|---|---|
| **Find by name** | `confluence_search` with CQL `title = "<NodeType>: <Name>" AND space = "<SPACE_KEY>"`; for fuzzy lookup `title ~ "<concept>"` |
| **Find references** | `confluence_search` with CQL `text ~ "<NodeType>: <Name>" AND space = "<SPACE_KEY>"` |
| **Read** | `confluence_get_page`; note `version.number` |
| **Create** | `confluence_create_page` with `spaceKey`, `title` = `<NodeType>: <Name>`, `body` following the template with `<ac:link-body>` links, `parentId` from your instance reference table |
| **Update** | `confluence_update_page` with `version` = current + 1 and the updated `body` |
| **Index update** | Fetch the **parent page**, add the link under the correct section, update it |
| **Record version** | The native `versionComment` of the create or update call |
| **History** | `confluence_get_version_history` with `contentId` and `limit` |
| **Restore** | Fetch the target version's content from the history |

The onboarding-question flag is the header field `Onboarding question: Yes`.

---

## When to use the Knowledge Graph API

In addition to the [core thresholds](../agent-skill.md#choosing-direct-operations-vs-knowledge-graph-api):

| Situation | Use |
|---|---|
| Regex-based HTML surgery (changing link format, moving metadata) | Knowledge Graph API |
| Debugging Confluence storage-format HTML | Knowledge Graph API |

See [graph-api.md](graph-api.md) for Knowledge Graph API patterns.

---

## Constraints (Confluence)

In addition to the [core constraints](../agent-skill.md#constraints-always-apply):

- **Never fabricate a page ID.** Resolve it via `confluence_search` or the instance reference table.
- **Always fetch the current version number** before calling `confluence_update_page`.
- **No drafts.** `confluence_create_page` publishes immediately, so the core confirmation step is the only safeguard.
- **Do not dump page HTML bodies into chat.** Show field values only, not raw storage-format HTML.

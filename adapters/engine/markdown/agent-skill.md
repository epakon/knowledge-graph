# Agent Skill — Markdown

> Markdown tooling for the [core agent skill](../agent-skill.md), which defines the intents, workflows and common constraints.
> Read the [Markdown Adapter](markdown-adapter.md) and [SPEC.md](../../../SPEC.md) before starting.

> **Prerequisites:** The agent must have read/write access to the knowledge graph directory tree in the repository. All read and write operations use direct file tools and git — no MCP server is required for this adapter.

---

## Prerequisites

In addition to the [core prerequisites](../agent-skill.md#prerequisites):

1. Know the root path of the knowledge graph directory tree (`<kg-root>`) in the repository.

---

## Backend operations

| Operation | Markdown |
|---|---|
| **Find by name** | `rg "^name: <Name>$" <kg-root>/ --glob "*.md" -l`; by title `rg "^# <NodeType>: <Name>" <kg-root>/ --glob "*.md" -l` |
| **Find references** | `rg "<NodeType>: <Name>" <kg-root>/ --glob "*.md" -l` |
| **Read** | Read the file |
| **Create** | Write the file at the path given by the directory structure in [markdown-adapter.md](markdown-adapter.md) |
| **Update** | Write the updated file |
| **Index update** | Add the link to the domain index file (`domain-<name>.md`). Global nodes (Concept, Subject and Process under `vocabulary/`, Agent under `ai/`) have no domain index; skip it for them |
| **Record version** | Commit the change; the commit message body is the version comment. Commit new files and their index updates together |
| **History** | `git log --follow --format="%H %as %an%n%B" -- <path/to/file.md>` |
| **Restore** | `git show <hash>:<path>` |

When using a template from [spec/page-templates.md](../../../spec/page-templates.md), convert it from Confluence storage format to Markdown: header paragraph fields become YAML frontmatter; `<ac:link-body>` links become `[edge statement](relative/path.md)`. The onboarding-question flag is the frontmatter field `onboarding_question: true`.

---

## When to use the Knowledge Graph API

In addition to the [core thresholds](../agent-skill.md#choosing-direct-operations-vs-knowledge-graph-api):

| Situation | Use |
|---|---|
| Regex-based frontmatter or link surgery across the tree | Knowledge Graph API |
| Snapshot generation or graph DB index export | Knowledge Graph API |

See [graph-api.md](graph-api.md) for Knowledge Graph API patterns.

---

## Constraints (Markdown)

In addition to the [core constraints](../agent-skill.md#constraints-always-apply):

- **Never fabricate a node path.** Resolve it via search or derive it from the `name` field using the naming convention.
- **Never write the version comment into the file.** It belongs in the commit message body only.

# Bulk Update Scripts

> Python patterns for bulk updates to Knowledge Graph pages via the Confluence REST API.
> Use these instead of MCP tool calls when a change affects 6+ pages or requires regex-based HTML surgery.

---

## Setup

Each script requires Confluence credentials. Use environment variables — never hardcode tokens in source files:

```python
import os
from requests.auth import HTTPBasicAuth

EMAIL    = os.environ["CONFLUENCE_USER"]   # e.g. user@example.com
TOKEN    = os.environ["CONFLUENCE_TOKEN"]  # Atlassian personal access token
BASE_URL = os.environ["CONFLUENCE_BASE_URL"]  # e.g. https://yourinstance.atlassian.net
AUTH     = HTTPBasicAuth(EMAIL, TOKEN)
```

---

## Shared REST helpers

All scripts share the same helper structure. Copy these from any existing script as the base for a new one.

### `get_page(page_id)`

Fetches a single page with body and version. Always call this before updating — you need the current `version.number`.

```python
def get_page(page_id):
    url = f"{BASE_URL}/wiki/rest/api/content/{page_id}"
    r = requests.get(url, auth=AUTH, params={"expand": "body.storage,version"})
    r.raise_for_status()
    return r.json()
```

### `get_children(page_id, limit=200)`

Paginates through all child pages of a parent. `limit=200` is the Confluence maximum.

```python
def get_children(page_id, limit=200):
    results = []
    start = 0
    while True:
        url = f"{BASE_URL}/wiki/rest/api/content/{page_id}/child/page"
        r = requests.get(url, auth=AUTH, params={"limit": limit, "start": start, "expand": "version"})
        r.raise_for_status()
        data = r.json()
        results.extend(data["results"])
        if start + len(data["results"]) >= data["size"]:
            break
        start += len(data["results"])
    return results
```

### `collect_kg_pages(root_ids)`

BFS from root page IDs, filters to pages whose title starts with a known node-type prefix.

```python
KG_NODE_PREFIXES = [
    "Subject:", "Domain:", "Table:", "Measure:", "Attribute:",
    "Filter:", "VerifiedQuery:", "Rule:", "Relationship:", "Disambiguation:"
]

def collect_kg_pages(root_ids):
    pages = []
    queue = list(root_ids)
    visited = set()
    while queue:
        pid = queue.pop(0)
        if pid in visited:
            continue
        visited.add(pid)
        children = get_children(pid)
        for child in children:
            queue.append(child["id"])
            if any(child["title"].startswith(p) for p in KG_NODE_PREFIXES):
                pages.append({"id": child["id"], "title": child["title"]})
    return pages
```

### `update_page(page_id, title, version, body, comment)`

PUT to the Confluence REST API. Always pass `version + 1`.

```python
def update_page(page_id, title, version, body, comment):
    url = f"{BASE_URL}/wiki/rest/api/content/{page_id}"
    payload = {
        "version": {"number": version + 1, "message": comment},
        "title": title,
        "type": "page",
        "body": {"storage": {"value": body, "representation": "storage"}}
    }
    r = requests.put(url, auth=AUTH, json=payload)
    r.raise_for_status()
    return r.json()
```

### `search_pages(cql, limit=50)`

CQL search with body included — avoids a second GET per page.

```python
def search_pages(cql, limit=50):
    url = f"{BASE_URL}/wiki/rest/api/content/search"
    r = requests.get(url, auth=AUTH, params={"cql": cql, "limit": limit, "expand": "body.storage,version"})
    r.raise_for_status()
    return r.json()["results"]
```

> CQL `~` is "contains", not exact match. Always add a `.startswith("Rule:")` guard on the results to exclude false positives.

---

## Dry-run / testing pattern

Every script should support a dry-run mode that prints diffs without posting:

```python
TEST_PAGE_ID = "<one representative page ID>"
DRY_RUN = True   # set False to actually update

pages = [{"id": TEST_PAGE_ID, "title": "..."}]   # override full list for testing

for p in pages:
    page = get_page(p["id"])
    old_body = page["body"]["storage"]["value"]
    new_body, changed = transform_body(old_body)
    if changed:
        print(f"--- {p['title']} ---")
        print("BEFORE:", old_body[:300])
        print("AFTER: ", new_body[:300])
        if not DRY_RUN:
            update_page(p["id"], p["title"], page["version"]["number"], new_body, VERSION_COMMENT)

print(f"Changed: {sum(1 for _ in changed_pages)} pages")
```

Only set `DRY_RUN = False` and restore the full page list after confirming the output is correct.

---

## Script: download pages

**Purpose:** Download all knowledge graph pages as Markdown files with YAML frontmatter.

**Output structure:**
```
<output_dir>/
├── Subject/
│   └── Subject_Write-Off.md
├── Measure/
│   └── Measure_REVENUE.md
├── Filter/
│   └── Filter_ACTIVE_CUSTOMERS.md
└── ... (one folder per node type)
```

Each file has YAML frontmatter (`id`, `title`, `type`, `url`, `version`, `last_modified`) followed by the page body converted from storage-format HTML to readable Markdown.

**CLI flags:**
```bash
python kg_download_pages.py                          # download all pages
python kg_download_pages.py --dry-run                # list pages only, no files written
python kg_download_pages.py --page <page_id>         # single page
python kg_download_pages.py --output /tmp/kg         # custom output directory
```

---

## Script: fix link labels

**Purpose:** Migrate `## Related` links from a bare-link + trailing-text format to `<ac:link-body>` clickable-label format.

**Old format (broken):**
```html
<ac:link><ri:page ri:content-title="Filter: X" .../></ac:link> &mdash; Subject: A implement &rarr; Filter: X
```

**New format (correct):**
```html
<ac:link>
  <ri:page ri:content-title="Filter: X" ri:space-key="<SPACE_KEY>"/>
  <ac:link-body>Subject: A implement -> Filter: X</ac:link-body>
</ac:link>
```

**Key function:**
```python
def html_to_text(s):
    """Normalize HTML arrow/dash entities to ASCII for use in <ac:link-body>."""
    s = s.replace("&rarr;", "->").replace("&larr;", "<-")
    s = s.replace("&#x2192;", "->").replace("&#x2190;", "<-")
    s = s.replace("\u2192", "->").replace("\u2190", "<-")
    s = s.replace("&mdash;", "-").replace("&#x2014;", "-")
    return s.strip()
```

**CLI flags:**
```bash
python kg_fix_link_labels.py                         # run on all KG pages
python kg_fix_link_labels.py --dry-run               # preview only
python kg_fix_link_labels.py --page <page_id>        # single page
```

---

## Script: fix headers

**Purpose:** Remove metadata header fields made redundant when their relationships were moved to `## Related` as typed edge links.

**Pattern for removing a header field:**
```python
import re

def remove_header_field(body, field_name):
    pattern = re.compile(
        r'<br\s*/>\s*<strong>' + re.escape(field_name) + r':</strong>\s*'
        r'<ac:link><ri:page[^/]*/></ac:link>',
        re.DOTALL
    )
    new_body, n = pattern.subn("", body)
    return new_body, n > 0
```

---

## Script: inject back-references into Relationship pages

**Purpose:** Ensure every node page that participates in a Relationship has a link to that Relationship page in its `## Relationships` section.

**Pattern:**

```python
def li(space_key, page_title, label):
    """Build a <li> with an <ac:link-body> for a Relationship page link."""
    return (
        f'<li><ac:link>'
        f'<ri:page ri:space-key="{space_key}" ri:content-title="{page_title}" />'
        f'<ac:link-body>{label}</ac:link-body>'
        f'</ac:link></li>'
    )

def ensure_relationships_section(body, new_items_html):
    """Insert items into existing ## Relationships, or create the section."""
    if "<h2>Relationships</h2>" in body:
        return body.replace("</ul>", new_items_html + "</ul>", 1), True
    else:
        return body + f"<h2>Relationships</h2><ul>{new_items_html}</ul>", True
```

---

## Back-reference injection rules

The three constraints from [spec/link-format.md](../../spec/link-format.md#back-reference-constraints) apply equally to MCP writes and Python scripts. Watch for these symptoms in raw HTML:

| Constraint | Symptom in raw HTML | Fix |
|---|---|---|
| No symmetric duplicates | Same target title appears in `## Related` twice — once with `->` and once with `<-` | Before injecting `<-`, scan existing `<ac:link-body>` text for `-> TargetName` with the same kind |
| Edge kind must match | `<ac:link-body>` on target page uses a different verb than the owning page | Derive the `<-` label by replacing `->` with `<-` in the owning label |
| No `implement` between Subjects | `Subject: X implement <- Subject: Y` in `## Related` | Replace the edge kind with `relatedTo` |

---

## Common pitfalls

| Pitfall | Symptom | Fix |
|---|---|---|
| `changed` flag not set inside a `lambda` passed to `re.sub` | Pages skipped despite needing update | Use `def repl(...): nonlocal changed` instead of lambda |
| Unicode `→` in regex doesn't match HTML entity `&rarr;` | Pattern silently misses pages | Use `(?:→\|&rarr;\|&#x2192;)` or normalize with `html_to_text()` first |
| `&rarr;` inside `<ac:plain-text-body>` | Arrow renders as literal `&rarr;` text | Replace tag with `<ac:link-body>` and use ASCII `->` |
| Posting version N when Confluence is already at N | 409 Conflict error | Always fetch current version immediately before the update |
| CQL `title ~ "Rule:"` returns non-Rule pages | Updating wrong pages | Add `.startswith("Rule:")` guard on results |
| BFS `get_children` only returns direct children | Deep pages missed | The BFS loop handles recursion — call `get_children` on each child too |
| `re.sub` with `re.DOTALL` missing across `<br/>` boundaries | Field not removed | Add `re.DOTALL` flag and verify the pattern spans the `<br />` correctly |

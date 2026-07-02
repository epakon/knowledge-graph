# Knowledge Graph API — Markdown

> Programmatic interface for graph operations on the Markdown knowledge base — bulk updates, structural migrations, snapshot generation, and graph DB import preparation.
> Use it instead of direct file operations when a change affects 6+ files or requires programmatic operations outside the LLM context.

This is the Markdown-adapter counterpart of [`adapters/engine/confluence/graph-api.md`](../confluence/graph-api.md). The operations and JSON index formats are identical — only the transport layer differs (filesystem instead of Confluence REST API).

---

## Setup

All scripts operate on a local directory tree. No credentials are needed — only the root path of the knowledge graph directory:

```python
import os
from pathlib import Path

KG_ROOT = Path(os.environ.get("KG_ROOT", "knowledge-graph"))
```

Dependencies:

```bash
pip install pyyaml
```

---

## Core operations

These are the fundamental Knowledge Graph API operations for the Markdown backend. All higher-level operations are built on top of these.

### `get_page(path)`

Reads a single `.md` file. Returns parsed frontmatter and raw body.

```python
import yaml

def get_page(path: Path) -> dict:
    text = path.read_text(encoding="utf-8")
    if text.startswith("---"):
        _, fm, body = text.split("---", 2)
        meta = yaml.safe_load(fm)
    else:
        meta, body = {}, text
    return {"path": str(path), "meta": meta, "body": body.strip()}
```

### `write_page(path, meta, body, comment)`

Writes a `.md` file with YAML frontmatter. Always call `get_page` first to preserve existing content — never overwrite blindly.

```python
def write_page(path: Path, meta: dict, body: str, comment: str) -> None:
    fm = yaml.dump(meta, allow_unicode=True, sort_keys=False).strip()
    content = f"---\n{fm}\n---\n\n{body.strip()}\n"
    path.write_text(content, encoding="utf-8")
    # Commit is the caller's responsibility — do not auto-commit inside this function.
```

> Committing is always done explicitly by the caller after all writes for a logical change are complete — not file-by-file.

### `get_children(directory)`

Lists all `.md` files directly under a directory (non-recursive).

```python
def get_children(directory: Path) -> list[Path]:
    return sorted(directory.glob("*.md"))
```

### `collect_kg_pages(root)`

BFS walk of the entire KG directory tree, filtered to files that have a recognised `type` frontmatter field.

```python
KG_NODE_TYPES = {
    "Subject", "Domain", "Table", "Measure", "Attribute",
    "Filter", "VerifiedQuery", "BusinessRule", "Relationship", "Disambiguation"
}

def collect_kg_pages(root: Path) -> list[dict]:
    pages = []
    for md_file in sorted(root.rglob("*.md")):
        page = get_page(md_file)
        if page["meta"].get("type") in KG_NODE_TYPES:
            pages.append(page)
    return pages
```

### `search_pages(root, pattern)`

Searches all KG pages for a regex pattern in body or frontmatter. Returns matching pages with full content — avoids a second read per file.

```python
import re

def search_pages(root: Path, pattern: str) -> list[dict]:
    rx = re.compile(pattern)
    results = []
    for md_file in root.rglob("*.md"):
        text = md_file.read_text(encoding="utf-8")
        if rx.search(text):
            results.append(get_page(md_file))
    return results
```

### `name_to_path(root, node_type, name)`

Derives the canonical file path for a node from its type and name. Use this to resolve link targets without filesystem search.

```python
def name_to_path(root: Path, node_type: str, name: str) -> Path:
    TYPE_DIRS = {
        "Subject": "vocabulary/subjects",
        "Table": "{domain}/tables",
        "Measure": "{domain}/measures",
        "Attribute": "{domain}/attributes",
        "Filter": "{domain}/filters",
        "VerifiedQuery": "{domain}/verified-queries",
        "BusinessRule": "{domain}/rules",
        "Disambiguation": "{domain}/disambiguations",
        "Relationship": "{domain}/relationships",
    }
    slug = name.lower().replace(" ", "-").replace("_", "-").replace(":", "")
    filename = f"{node_type.lower()}-{slug}.md"
    subdir = TYPE_DIRS.get(node_type, "")
    return root / subdir / filename
```

> For domain-scoped types the `{domain}` placeholder must be substituted with the actual domain slug before use.

---

## Dry-run / testing pattern

Every script should support a dry-run mode that prints diffs without writing files:

```python
TEST_FILE = KG_ROOT / "sales/measures/measure-revenue.md"
DRY_RUN = True   # set False to actually write

files = [TEST_FILE]  # override full list for testing

for path in files:
    page = get_page(path)
    old_body = page["body"]
    new_body, changed = transform_body(old_body)
    if changed:
        print(f"--- {path} ---")
        print("BEFORE:", old_body[:300])
        print("AFTER: ", new_body[:300])
        if not DRY_RUN:
            write_page(path, page["meta"], new_body, VERSION_COMMENT)

print(f"Changed: {sum(1 for _ in changed_files)} files")
```

Only set `DRY_RUN = False` and restore the full file list after confirming the output is correct.

---

## Operation: scan and validate

**Purpose:** Walk the entire KG directory tree and report files that are missing required frontmatter fields, have unresolvable links, or violate naming conventions.

```python
REQUIRED_FIELDS = {"type", "name"}

def validate_kg(root: Path) -> list[str]:
    errors = []
    for page in collect_kg_pages(root):
        path = page["path"]
        meta = page["meta"]
        for field in REQUIRED_FIELDS:
            if field not in meta:
                errors.append(f"{path}: missing required field '{field}'")
        # Check for unicode arrows in link text
        if re.search(r'\[.*?[→←].*?\]', page["body"]):
            errors.append(f"{path}: unicode arrow in link text — use ASCII -> or <-")
    return errors
```

**CLI flags:**
```bash
python kg_validate.py                          # validate entire tree
python kg_validate.py --root /path/to/kg       # custom root
python kg_validate.py --file <path>            # single file
```

---

## Operation: download / export to Markdown

**Purpose:** Export a snapshot of the KG as standalone Markdown files (e.g. for publishing or review). The Markdown adapter already stores files natively — this operation is mainly useful when the source is Confluence and you want a Markdown mirror.

If using Confluence as the primary backend and Markdown as a secondary mirror, use the Confluence `graph-api.md` download operation (`kg_download_pages.py`) to produce files conforming to this adapter's frontmatter and link conventions.

---

## Operation: fix link paths

**Purpose:** Migrate link hrefs after files are moved or renamed — e.g. when a domain is restructured.

**Key function:**

```python
def rewrite_links(body: str, old_prefix: str, new_prefix: str) -> tuple[str, bool]:
    """Replace all occurrences of old_prefix in Markdown link hrefs with new_prefix."""
    pattern = re.compile(r'(\[.*?\])\(' + re.escape(old_prefix) + r'(.*?)\)')
    new_body, n = pattern.subn(lambda m: f"{m.group(1)}({new_prefix}{m.group(2)})", body)
    return new_body, n > 0
```

**CLI flags:**
```bash
python kg_fix_links.py --old-prefix ../old-domain/ --new-prefix ../new-domain/
python kg_fix_links.py --dry-run
```

---

## Operation: inject back-references

**Purpose:** Ensure every node file that is the target of an edge has the corresponding `<-` back-reference link in its `## Links` section.

**Key function:**

```python
def ensure_back_reference(body: str, back_ref_line: str) -> tuple[str, bool]:
    """Insert a back-reference line into ## Links if not already present."""
    if back_ref_line in body:
        return body, False
    if "## Links" in body:
        return body.replace("## Links\n", f"## Links\n{back_ref_line}\n", 1), True
    return body + f"\n## Links\n{back_ref_line}\n", True
```

Back-reference injection rules (same as [spec/link-format.md](../../spec/link-format.md#back-reference-constraints)):

| Constraint | Symptom | Fix |
|---|---|---|
| No symmetric duplicates | Same target appears in `## Links` with both `->` and `<-` | Before injecting `<-`, check existing `## Links` for `-> TargetName` with same kind |
| Edge kind must match | `<-` line uses a different verb than the forward edge | Derive the `<-` label by replacing `->` with `<-` in the owning label |
| No `implement` between Subjects | `Subject: X implement <- Subject: Y` in `## Links` | Replace edge kind with `relatedTo` |

---

## Common pitfalls

| Pitfall | Symptom | Fix |
|---|---|---|
| `write_page` called before `get_page` | Existing content overwritten | Always read the current file before writing |
| Relative path computed from wrong base | Broken links | Always compute relative paths from the file's own directory, not from `KG_ROOT` |
| Unicode arrows `→` in link text | Edge parser misses the edge | Normalize with `html_to_text()` or enforce ASCII-only at write time |
| `rglob("*.md")` includes non-KG files | Wrong files modified | Filter by `type` frontmatter field; add `validate_kg()` to CI |
| Missing `name` field in frontmatter | `name_to_path()` cannot resolve | Enforce `name` as a required frontmatter field in `validate_kg()` |
| Committing after every file write | Noisy git history | Batch all writes for one logical change into a single commit |

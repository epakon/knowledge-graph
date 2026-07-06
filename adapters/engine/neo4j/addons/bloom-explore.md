# Neo4j — Bloom Explore

> **Addon** — optional capability for the [Neo4j adapter](../neo4j-adapter.md). The adapter works without it; agents query Neo4j directly via the official driver regardless of which (if any) exploration tool is configured.

Bloom Explore gives humans a no-code, visual way to browse the same Neo4j instance the Knowledge Graph agents query with Cypher — searching by node/relationship pattern, expanding neighbors, and following paths interactively. It is a convenience layer for people; it has no effect on graph semantics, the import procedure, or agent behavior.

---

## Why this is an addon, not part of the core adapter

Exploration tooling is interchangeable by design:

- The [Neo4j adapter](../neo4j-adapter.md) only requires Cypher + the official driver. No exploration tool is required to import, query, or sync the graph.
- Several tools can visualize the same Neo4j instance equally well — Bloom Explore, NeoDash, yWorks Data Explorer, SemSpect, Graphlytic, or a custom build on the Neo4j Visualization Library (NVL). Choosing one, several, or none does not change the graph, the constraints, or the import scripts.
- Keeping this as an addon means a deployment can swap or drop the exploration layer without touching [`neo4j-adapter.md`](../neo4j-adapter.md).

---

## What Bloom Explore is

Bloom (surfaced as **Explore** inside the Aura Console) is Neo4j's visual graph exploration product — near-natural-language search, click-to-expand neighbors, styling by label/property, no Cypher required.

| | Bloom Basic (free) | Bloom Full Access (Enterprise) |
|---|---|---|
| Near-natural-language search | Yes | Yes |
| Graph exploration | Yes | Yes |
| Scene saving | No | Yes |
| Perspective sharing / role-based authorization | No | Yes |

Community Edition instances get **Bloom Basic** for free by registering the instance as a self-managed deployment in the Aura Console — no Enterprise license or plugin required for this path.

---

## Setup

1. **Create a free Aura account** at [console.neo4j.io](https://console.neo4j.io) (no paid Aura instance required).
2. In the console, go to **Instances → Self-managed → Add Deployment → Unmonitored** (monitoring via Fleet Manager is a separate, unrelated capability — not needed just for Explore).
3. Provide:
   - Connection URI (`bolt://<host>:7687` or `bolt+s://<host>:7687` if TLS-terminated)
   - Username / password
   - Database name (`neo4j` by default)
4. Click **Connect**, then open **Explore** from the console against the registered instance.

## Network requirements

The console tools connect **directly from your browser** to your Neo4j instance — Aura's servers do not proxy your graph data. Your browser needs outbound access to:

| Port | Purpose |
|---|---|
| 443 | Aura Console UI (hosted, global) |
| 7687 | Bolt — the actual query/exploration traffic to your instance |
| 7474 | HTTP fallback, used by Browser/Query |

If your Neo4j instance sits behind a firewall or private network, either expose the Bolt port to wherever the browser runs, or connect over a VPN into that network.

## Known limitations for Community Edition

- No scene saving or perspective sharing (Enterprise-only).
- No remote monitoring/metrics beyond what Fleet Manager exposes for CE (unrelated to Explore itself).
- Neo4j documents self-managed CE compatibility with Aura Console tooling as **best-effort**, without a formal support SLA.

---

## Alternatives

Any of these can replace or supplement Bloom Explore against the same Neo4j instance, and could be documented as their own addon if adopted:

| Tool | Notes |
|---|---|
| [NeoDash](https://neo4j.com/labs/neodash/) | Open-source, low-code dashboard builder (Neo4j Labs) |
| [yWorks Data Explorer for Neo4j](https://www.yworks.com/products/data-explorer-for-neo4j) | Free, no-code exploration with automatic layouts and graph algorithms |
| [SemSpect](https://www.semspect.de/) | Exploration via visual aggregation, aimed at large/dense graphs |
| [Graphlytic](https://graphlytic.com/databases/neo4j) | Free for single user; click-driven exploration, saved query templates |

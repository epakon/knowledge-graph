# Adapters

An adapter bridges the Knowledge Graph specification and a real external system. Adapters fall into two types by role: **engine adapters** implement the infrastructure the graph runs on; **connectors** extract business knowledge from source systems into the graph. Any adapter may also have **addons**.

All adapter documents live under `adapters/`.

---

## Engine adapters

Engine adapters implement the storage and traversal infrastructure the Knowledge Graph runs on. They do not add knowledge — they provide the surface on which knowledge is authored, queried, and navigated.

Full definitions, capability requirements, adapter contracts, and the list of existing adapters: [`engine/engine.md`](engine/engine.md).

---

## Connectors

Connectors extract business knowledge from source systems — SAP S/4HANA, Salesforce, Workday, or any system that encodes business concepts, data models, calculation rules, and mandatory conditions. A connector translates the source system's implicit knowledge into the typed nodes and edges of the Knowledge Graph.

Because knowledge extraction is outside the scope of the core specification, connectors need their own contract document. The contract (six mandatory sections every connector must implement) is defined in [`connectors/connectors.md`](connectors/connectors.md).

---

## Addons

An addon is an optional, self-contained capability document attached to an adapter or connector. It extends the adapter with a feature that is useful alongside it but not required for the core function — different tooling, different audience, or a different data lifecycle.

**Rules:**
- An addon lives in an `addons/` subfolder of its adapter.
- It is listed in the adapter's entry-point document, clearly marked as an addon.
- It does not affect KG node/edge semantics — adding or removing an addon is non-breaking.
- An addon may reuse tooling from another adapter (e.g. the visualization pipeline) without being part of it.

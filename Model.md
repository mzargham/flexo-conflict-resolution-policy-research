# Model

A model is a structured representation of a system — its elements, relationships, and constraints — built to support reasoning about that system.

## Two Senses

### Systems Engineering Sense

A model captures the *what* and *how* of a system design: parts, connections, behaviors, requirements, and the constraints that bind them. In [[SysML v2]], a model is a tree of elements (packages, parts, connections, requirements, constraints, etc.) linked by ownership and typed relationships.

Key properties:
- **Compositional** — elements nest and reference each other
- **Typed** — elements conform to metamodel types (the modeling language's grammar)
- **Constrained** — the model carries explicit constraints (requirements, parametric relations) that restrict valid configurations

### Flexo Storage Sense

In [[Flexo MMS]], a model at a given commit is a **named graph of [[RDF]] triples** in the [[quadstore]]. Each triple is an atomic assertion about a model element (its type, name, ownership, relationship to other elements).

- The **state of a model** at commit *c* = the set of triples in the snapshot's model graph for *c*
- A **commit** transforms one model state into another via a [[SPARQL]] Update (`INSERT`/`DELETE`)
- The snapshot may or may not be materialized (see [[Quadstore#Snapshot Lifecycle]])

## Why Both Senses Matter

The SE sense defines *what the model means* — its structure carries engineering intent. The storage sense defines *how the model is represented and versioned* — as a set of triples in a graph database.

Conflict resolution operates at the intersection: it must manipulate the storage representation (triples) while respecting the engineering semantics (types, constraints, intent). A resolution that is valid at the triple level may be invalid at the model level — this is the core challenge the [[Flexo Conflict Resolution Mapping|formalism]] must address. The [[Key Insight]] develops why this dual nature forces a separation between commits (change descriptions) and states (model snapshots).

---
← [[Flexo MMS]] · [[RDF]] · [[Quadstore]] · [[SysML v2]] · [[Key Insight]]

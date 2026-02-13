# RDF

Resource Description Framework — a [W3C standard](https://www.w3.org/TR/rdf11-concepts/) data model where information is expressed as subject-predicate-object triples.

## Triple Structure

An RDF triple is the atomic assertion:

| Component | What it is | What it can be |
|-----------|-----------|----------------|
| **Subject** | The resource being described | IRI or blank node |
| **Predicate** | The property or relationship | IRI |
| **Object** | The value or related resource | IRI, blank node, or literal |

A triple asserts a single fact: *"subject has property predicate with value object."* For example, a triple might assert that a particular model element has a name, or that one element owns another.

## Named Graphs

A **named graph** is a set of triples identified by an IRI (the graph name). Adding the graph name to a triple produces a **quad**: `(subject, predicate, object, graph)` — the atomic unit of storage in a [[quadstore]] ([Architecture](https://openmbee.atlassian.net/wiki/spaces/OPENMBEE/pages/320765953/Flexo-MMS+Architecture)).

Named graphs partition the quadstore into addressable, scopeable sets of triples. Flexo uses this partitioning for:
- **Isolation** — separate model data per snapshot, so each commit's model state lives in its own named graph
- **Scoping** — target [[SPARQL]] queries and updates to specific graphs rather than the entire store

## Named Graph Categories in Flexo

Per project, the [[quadstore]] contains three categories of named graphs ([Architecture](https://openmbee.atlassian.net/wiki/spaces/OPENMBEE/pages/320765953/Flexo-MMS+Architecture)):

| Category | Contents |
|----------|----------|
| **Metadata graph** | Versioning information: commit history (the DAG), deltas, and refs (branches, tags, locks) |
| **Model graph (snapshot)** | The entire model data for a given commit. A Snapshot object in the Metadata graph associates a Ref to its materialized Model graph |
| **System graphs** | Flexo's own state: orgs, repos, collections, access policies, cluster config — stored alongside user data |

The Flexo MMS Ontology spans these graphs — classes and their relationships travel both within and across named graphs ([Architecture](https://openmbee.atlassian.net/wiki/spaces/OPENMBEE/pages/320765953/Flexo-MMS+Architecture)).

## In Flexo MMS

- All model content and all system state are stored as RDF ([flexo-mms-layer1-service](https://github.com/Open-MBEE/flexo-mms-layer1-service))
- Models are handled as RDF graphs, exposed via [[SPARQL]] and [[Graph Store Protocol]] endpoints ([Architecture](https://openmbee.atlassian.net/wiki/spaces/OPENMBEE/pages/320765953/Flexo-MMS+Architecture))
- Not limited to SysML — Flexo can store anything expressed as RDF ([openmbee.org](https://www.openmbee.org/flexo.html))

## Toward Formalizing State

The RDF data model gives us a precise notion of "state" in Flexo:

- The **state of a model** at a given commit = the set of triples in its snapshot's model graph
- A **commit** = a [[SPARQL]] Update (`INSERT`/`DELETE`) applied to the parent snapshot's model graph, producing a new model graph
- The **commit DAG** tracks lineage; each node implies a model graph (materialized or recoverable)

These are distinct objects — the commit is the *change description*, the model graph is the *resulting state* — and the [[Key Insight]] explains why maintaining this distinction is necessary for conflict resolution in structured models.

→ [[Flexo Conflict Resolution Mapping]]

## References

- [Flexo-MMS Architecture](https://openmbee.atlassian.net/wiki/spaces/OPENMBEE/pages/320765953/Flexo-MMS+Architecture) — OpenMBEE Confluence
- [flexo-mms-layer1-service](https://github.com/Open-MBEE/flexo-mms-layer1-service) — GitHub
- [Flexo MMS and SysMLv2](https://www.openmbee.org/flexo.html) — openmbee.org

---
← [[Flexo MMS]] · [[Flexo Organization Level]]

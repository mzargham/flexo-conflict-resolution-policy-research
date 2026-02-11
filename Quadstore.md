# Quadstore

A database that stores [[RDF]] quads — triples (subject-predicate-object) plus a fourth component, the named graph.

## The Quad

A quad extends an RDF triple with the graph it belongs to:

```
(subject, predicate, object, graph)
```

A quadstore is a set of quads. Equivalently, it can be viewed as a mapping from graph IRIs to sets of triples — each graph IRI names a partition of the store that can be addressed and queried independently.

## What Lives in the Quadstore

Flexo MMS stores **all** content — both system and user — in one quadstore. There is no separate metadata database ([flexo-mms-layer1-service](https://github.com/Open-MBEE/flexo-mms-layer1-service)). The contents fall into three categories of named graphs (see also [[RDF#Named Graph Categories in Flexo]]):

| Category | Contents | Scope |
| -------- | -------- | ----- |
| **System graphs** | Cluster config, org/repo/collection definitions, access policies, user records | Deployment-wide |
| **Metadata graphs** | Commit DAG (parent pointers), ref → commit mappings, snapshot → model graph associations, deltas | Per project |
| **Model graphs (snapshots)** | Complete set of triples representing the model at a specific commit | Per commit (when materialized) |

A Snapshot object in the Metadata graph associates a Ref to its materialized Model graph ([Architecture](https://openmbee.atlassian.net/wiki/spaces/OPENMBEE/pages/320765953/Flexo-MMS+Architecture)).

## Snapshot Lifecycle

A snapshot is a materialized model graph for a specific commit — the full set of triples representing the model's state at that point in history ([Commits, Refs and Snapshots](https://openmbee.atlassian.net/wiki/spaces/OPENMBEE/pages/613613569)).

| Condition | Snapshot status |
| --------- | --------------- |
| HEAD of an active branch | Always materialized — ready for query and update |
| Historical commit with a lock | Materialized on demand when the lock is created |
| Historical commit, no lock | Not materialized — recoverable by replaying commits from a materialized ancestor |
| Branch or lock deleted | Associated snapshot is deallocated |
| Branch and lock point to same commit | Share the same snapshot (resource conservation) |

This is a resource management mechanism: storing a complete model graph for every commit in history would exhaust the [[quadstore]]'s storage capacity or degrade query performance ([Commits, Refs and Snapshots](https://openmbee.atlassian.net/wiki/spaces/OPENMBEE/pages/613613569)).

## Commit as Mutation

Creating a new commit is a state transition on the quadstore ([Commits, Refs and Snapshots](https://openmbee.atlassian.net/wiki/spaces/OPENMBEE/pages/613613569)):

1. The parent commit must be materialized as a snapshot (its model graph must exist in the quadstore)
2. A [[SPARQL]] Update (`INSERT`/`DELETE`) is applied to the parent's model graph
3. The resulting model graph becomes the new commit's snapshot
4. The branch ref advances to point at the new commit

The SPARQL Update **mutates** the dataset it is applied against — the parent's snapshot graph is transformed into the child's. This is why the parent must be materialized first.

**Conflict scenario**: when two contributors independently create commits with the same parent, both intend to mutate the same parent snapshot. Reconciling these concurrent mutations is the problem addressed by the [[Flexo Conflict Resolution Mapping|conflict resolution formalism]].

## Reference Implementation

[[Apache Jena Fuseki]], binding on port 3030, with SPARQL Query, Update, and [[Graph Store Protocol]] endpoints ([flexo-mms-layer1-service](https://github.com/Open-MBEE/flexo-mms-layer1-service)).

## References

- [Flexo-MMS Architecture](https://openmbee.atlassian.net/wiki/spaces/OPENMBEE/pages/320765953/Flexo-MMS+Architecture) — OpenMBEE Confluence
- [Commits, Refs and Snapshots](https://openmbee.atlassian.net/wiki/spaces/OPENMBEE/pages/613613569) — OpenMBEE Confluence
- [flexo-mms-layer1-service](https://github.com/Open-MBEE/flexo-mms-layer1-service) — GitHub

---
← [[RDF]] · [[SPARQL]] · [[Flexo Organization Level]]

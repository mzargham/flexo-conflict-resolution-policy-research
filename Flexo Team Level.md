# Flexo Team Level

**Concern:** Engineering team developing and maintaining specific model assets.

## Version Control Primitives

| Concept | Flexo Entity | Purpose |
|---------|-------------|---------|
| Baseline | Tag | Immutable reference to a commit (milestone, release) |
| Working copy | Branch | Mutable ref that automatically gets adjusted to point at the latest commit when an update is made ([Commits, Refs and Snapshots](https://openmbee.atlassian.net/wiki/spaces/OPENMBEE/pages/613613569)) |
| Commit | Commit | Atomic change to model state; forms a DAG with parent pointers |
| Query access | Lock | Users can create namespaced locks to "lock" access to a particular commit's data (via its corresponding snapshot), and then subsequently delete locks to "release" access ([Commits, Refs and Snapshots](https://openmbee.atlassian.net/wiki/spaces/OPENMBEE/pages/613613569)) |

## Commit DAG Semantics

Each commit points only to its parent, with the root commit pointing to a reserved constant `rdf:nil`. Refs (which includes locks and branches) connect snapshots to commits ([Commits, Refs and Snapshots](https://openmbee.atlassian.net/wiki/spaces/OPENMBEE/pages/613613569)).

Branching enables parallel development streams (e.g., `develop`, `feature-X`, `release-v2`). When two commits share the same parent, this divergence manifests different states of the model.

## Snapshot Management

As long as a branch exists, it will always have a snapshot materialized in the database ready to be queried and updated. For historical commits, teams must create locks to materialize snapshots for querying ([Commits, Refs and Snapshots](https://openmbee.atlassian.net/wiki/spaces/OPENMBEE/pages/613613569)).

This is a resource management mechanism: storing a complete copy of a project's model for each commit can quickly impact query performance or exhaust the storage capacity of the underlying [[quadstore]]. Flexo MMS conserves resources when possible — e.g., a branch and a lock pointing to the same commit share the same snapshot.

When multiple clients need stable access to the same historical commit, each creates its own namespaced lock. The snapshot is materialized when the first lock is created and remains materialized as long as any lock references that commit. Deleting a lock decrements the effective reference count; the snapshot becomes eligible for eviction only when no locks (and no branches) reference it. This reference-counting behavior makes locks a coordination primitive: clients can independently acquire and release access to historical state without explicit coordination with each other.

## Team Workflow Example

1. Create feature branch from `develop`
2. Make commits against feature branch
3. When ready, merge to `develop` (this is where [[Flexo Conflict Resolution Mapping|conflict resolution]] applies)
4. Tag releases for baselines
5. Lock specific commits when external tools need stable query access

## References

- [Commits, Refs and Snapshots](https://openmbee.atlassian.net/wiki/spaces/OPENMBEE/pages/613613569) — OpenMBEE Confluence
- [Flexo-MMS Architecture](https://openmbee.atlassian.net/wiki/spaces/OPENMBEE/pages/320765953/Flexo-MMS+Architecture) — OpenMBEE Confluence

---
← [[Flexo MMS]]

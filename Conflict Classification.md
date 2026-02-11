# Conflict Classification

How conflicts are currently documented in Flexo MMS, what kinds of conflicts exist, and what the [[Flexo Conflict Resolution Mapping|formalism]] must account for.

## Current State in Flexo

Conflict detection in the existing codebase is minimal:

### What Exists

1. **Optimistic concurrency conflicts** (Layer 1): ETag mismatch (412) or transaction mutex contention (409) when two clients try to update the same branch simultaneously. This is a *write-write race* on a single branch, not a merge conflict.

2. **Guarded patch precondition failures** (Layer 1): The `guardedPatch()` function allows WHERE-clause preconditions on SPARQL UPDATEs. If the assumed state no longer holds, the patch fails (412). This is a *conditional write* mechanism — closer to optimistic locking than conflict classification.

3. **Merge conflict identity list** (SysML v2 API): The merge endpoint returns 409 with an array of `DataIdentity` objects — UUIDs of conflicting elements. No conflict type, no context, no description of *what* conflicted or *how*.

### What Does Not Exist

- No conflict **taxonomy** (types of conflicts are not classified)
- No conflict **description** (only element identity, not the nature of the conflict)
- No conflict **severity** or **resolution guidance**
- No distinction between conflicts that can be resolved mechanically and those requiring judgment
- The merge endpoint itself is **stub code** — conflict detection logic is not implemented

## Preliminary Conflict Taxonomy

The following taxonomy is not drawn from Flexo documentation — it is our working classification, constructed from the data model ([[RDF]], [[Quadstore]], [[Model]]) and the [[Verification and Validation]] boundary. It is sufficient to explore the conflict resolution problem but is not intended as a prescription. We expect a fit-for-purpose taxonomy to emerge from the work on merge conflict identification and resolution itself.

Conflicts are organized by the level at which they manifest:

### Level 1: Syntactic Conflicts

Concurrent modifications that touch the **same triples**.

| Type | Description | Example |
|------|-------------|---------|
| **Update-update** | Same triple modified to different values in both branches | Both branches change an element's name |
| **Delete-update** | One branch deletes a triple that the other modifies | Branch A removes an element; Branch B renames it |
| **Insert-insert** | Both branches insert triples that are structurally incompatible | Both add a different value for a single-valued property |

These are detectable by comparing the two [[Diff and Delta|diffs]] from the common ancestor: any triple appearing in both the delete-set or insert-set of both diffs is a syntactic conflict.

**Resolution**: Often mechanically resolvable — "last writer wins," "source wins," "target wins," or union. Falls in [[Verification and Validation|verification]] scope.

### Level 2: Structural Conflicts

The merged result violates **metamodel or schema constraints**, even if no individual triple conflicts exist.

| Type | Description | Example |
|------|-------------|---------|
| **Cardinality violation** | Merged state violates multiplicity constraints | Two branches each add a different owner to an element with max cardinality 1 |
| **Type constraint violation** | Merged state violates typing rules | Branch A changes an element's type; Branch B adds a relationship valid only for the original type |
| **Dangling reference** | Merged state contains a reference to a deleted element | Branch A deletes element X; Branch B adds a relationship targeting X |
| **Containment cycle** | Merged ownership creates a cycle in the containment tree | Branch A moves package P into Q; Branch B moves Q into P |

These require checking the merged triple-set against the metamodel — they are not visible from the diffs alone.

**Resolution**: Some are mechanically resolvable (e.g., dangling references can be flagged and removed). Others require judgment. Straddles the V&V boundary.

### Level 3: Semantic Conflicts

The merged result is syntactically and structurally valid but violates **domain constraints or engineering intent**.

| Type | Description | Example |
|------|-------------|---------|
| **Invariant violation** | A constraint expression (e.g., parametric equation, requirement satisfaction) no longer holds | Two branches independently adjust parameters that together violate a system constraint |
| **Behavioral inconsistency** | Merged behavior model is internally contradictory | Branch A modifies a state machine transition; Branch B modifies the guard condition — the merged behavior may be unreachable or contradictory |
| **Requirement conflict** | Merged state satisfies contradictory requirements | Branch A satisfies requirement R1 by design choice D; Branch B satisfies R2 by a choice incompatible with D |

These require evaluating the model's *meaning*, not just its structure.

**Resolution**: Cannot be automated in general. Falls in [[Verification and Validation|validation]] scope. The best a policy can do is detect that a constraint check fails and surface it for human review.

## Logical vs Behavioral Constraints

A key distinction for the formalism:

**Logical constraints** are declarative predicates over model state:
- "Every port must have a type"
- "Mass of subsystems must not exceed mass budget"
- "All requirements must be satisfied by at least one design element"

These can be expressed as [[SPARQL]] queries or constraint rules and checked mechanically. They are the natural candidates for automated verification in [[Continuous Integration|CI]] and [[Policy]]-based resolution.

**Behavioral constraints** involve the dynamic semantics of the model:
- "This state machine must be deterministic"
- "This sequence of interactions must terminate"
- "This control loop must be stable under parameter variation"

These may require simulation, formal analysis, or expert judgment. They are harder to express as RDF-level predicates and harder to check automatically.

The formalism's constraint set should distinguish between these categories, because they have different:
- **Computability** — logical constraints are generally decidable; behavioral constraints may not be
- **Expressibility** — logical constraints map to SPARQL or SHACL; behavioral constraints may require external tools
- **Resolution authority** — logical constraint violations may be auto-resolvable; behavioral issues require human review

## Global Rules and Their Assertability

For the constrained optimization approach, constraints fall into two categories:

**Assertable constraints** — can be expressed as predicates over model state and checked by the system:
- Metamodel conformance (typing, cardinality, containment)
- Explicit constraint expressions in the model (parametric equations, requirement text patterns)
- SPARQL-expressible invariants over the graph

**Non-assertable constraints** — known to matter but not expressible as computable predicates:
- "The design must be manufacturable"
- "The interface is intuitive"
- "The architecture supports future extensibility"

The formalism should handle assertable constraints as hard or soft constraints in the optimization. Non-assertable constraints map to validation concerns — the system flags relevant changes but cannot evaluate them.

## Open Questions

- **Constraint language**: What language should constraints be expressed in? SPARQL ASK queries? SHACL shapes? OWL axioms? A combination? The choice affects what Level 2 and Level 3 conflicts are detectable.
- **Conflict granularity**: Should conflicts be classified at the triple level, element level, or relationship level? The Layer 1 diff operates on triples; the SysML v2 API operates on elements. The right level for conflict resolution may be different from either.
- **Conflict interaction**: Can two individually resolvable conflicts compose into an unresolvable situation? (E.g., resolving conflict A one way creates a new structural conflict with the resolution of conflict B.)
- **Constraint priority**: When constraints conflict with each other (e.g., satisfying constraint C1 requires violating C2), how should the optimization trade off? This is where the "constrained" in constrained optimization becomes interesting.
- **Behavioral constraint detection**: Can any behavioral constraints be lifted to structural checks? (E.g., state machine determinism can be checked structurally; control loop stability cannot.)
- **Partial resolution**: Can a merge be partially resolved — some conflicts automated, others deferred — while maintaining model consistency? What are the invariants that must hold between partial resolution and final human review?

## References

- [flexo-mms-layer1-service](https://github.com/Open-MBEE/flexo-mms-layer1-service) — `GuardedPatch.kt`, `Conditions.kt`, `ModelCommit.kt`
- [flexo-mms-sysmlv2 OpenAPI](https://github.com/Open-MBEE/flexo-mms-sysmlv2) — merge 409 response schema

---
← [[Model]] · [[Merge]] · [[Diff and Delta]] · [[Policy]] · [[Verification and Validation]] · [[Flexo Conflict Resolution Mapping]]

# Flexo Conflict Resolution Mapping

Maps the conflict resolution formalism to [[Flexo MMS]] operations. To be developed.

Conflict resolution is framed as a [[Policy]] that respects the [[Verification and Validation]] boundary — automating deterministic structural checks while deferring judgment-sensitive conflicts to human reviewers.

The [[Merge]] endpoint (`POST /projects/{projectId}/branches/{targetBranchId}/merge`) is where the resolution policy would be invoked. The formalism will define policies as functions over [[Model]] state that identify [[Conflict Classification|conflicts]] and produce resolutions, operating on the [[Diff and Delta|diff/delta]] representation.

The [[Predicate Compliance Oracle]] provides the constraint evaluation — mapping model state to per-predicate compliance scores. In the optimal control framing, these scores ground the dual variables (Lagrange multipliers) that encode the shadow price of each constraint.

---
← [[Flexo MMS]] · [[Model]] · [[Diff and Delta]] · [[Merge]] · [[Conflict Classification]] · [[Predicate Compliance Oracle]] · [[Policy]]

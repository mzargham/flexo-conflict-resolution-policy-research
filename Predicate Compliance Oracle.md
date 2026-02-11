# Predicate Compliance Oracle

An abstraction over the heterogeneous mechanisms that evaluate whether a [[Model]]'s predicates are compliant. The oracle accepts the full state of a Flexo MMS model and returns a compliance score for each predicate.

## Interface

**Input**: The complete model state — the set of [[RDF]] triples in the snapshot's named graph at a given commit (see [[Quadstore]])

**Output**: A score for each predicate in the model, representing its degree of compliance with applicable constraints

```
oracle : ModelState → (Predicate → Score)
```

The oracle is a function from model state to a scoring over predicates. It does not prescribe *how* the score is computed — only that one is produced.

## Implementation Heterogeneity

The oracle is deliberately abstract because different predicates require fundamentally different evaluation mechanisms:

| Predicate type | Check mechanism | Example |
|----------------|-----------------|---------|
| Schema conformance | SHACL validation, OWL reasoning | "Every port must have exactly one type" |
| Referential integrity | [[SPARQL]] ASK query | "All relationship targets must exist" |
| Parametric constraint | Computation engine / external solver | "Subsystem mass ≤ budget allocation" |
| Requirement satisfaction | External tool or human review | "Design satisfies requirement R-42" |
| Behavioral correctness | Simulation, formal analysis, expert judgment | "Control loop is stable under perturbation" |

In practice, the oracle is a **composite** — a dispatch over predicate types to the appropriate evaluation backend (database lookup, API call, CI check, human reviewer). The abstraction collapses this heterogeneity into a uniform interface.

## Connection to Dual Variables

In the optimal control framing of [[Flexo Conflict Resolution Mapping|conflict resolution]], the model state is the primary variable and the predicates are the constraints. The oracle provides the **constraint evaluation**.

Each predicate's compliance score maps to a **dual variable** (Lagrange multiplier) in the optimization:

- The dual variable represents the *sensitivity of the resolution objective to that predicate's compliance* — the shadow price of relaxing or tightening the constraint
- A high dual value means the constraint is binding — small changes in compliance have large effects on the resolution outcome
- A zero dual value means the constraint is slack — it is satisfied with margin and does not influence the optimal resolution

The oracle makes the dual variables concrete: they are not abstract mathematical objects but correspond to real evaluations performed by real systems (or people). The heterogeneity of the oracle is why the formalism needs the abstraction — the optimization framework treats all constraints uniformly, even though their evaluation mechanisms are radically different.

## Scoring Semantics

The score need not be binary. A continuous score supports:

- **Hard constraints** (score ∈ {0, 1}) — pass/fail, as in schema conformance checks. These correspond to inequality constraints in the optimization.
- **Soft constraints** (score ∈ [0, 1]) — degree of compliance, as in parametric constraints that may be partially satisfied. These correspond to penalty terms or soft constraints.
- **Unbounded scores** (score ∈ ℝ) — distance from compliance, useful for constraints where "how far off" matters for prioritizing resolution effort.

The choice of scoring semantics per predicate type is itself a [[Policy]] decision.

## V&V Boundary

The oracle spans the [[Verification and Validation]] boundary:

- **Verification-scope predicates** produce scores that are deterministic, reproducible, and automatable. Their dual variables are computable.
- **Validation-scope predicates** produce scores that depend on judgment. Their dual variables are estimated or assigned by human reviewers — the formalism accommodates them but cannot compute them autonomously.

The [[Conflict Classification]] taxonomy maps onto oracle behavior: Level 1 and Level 2 conflicts involve predicates with verification-scope oracles; Level 3 conflicts involve predicates whose oracles require human input.

---
← [[Model]] · [[Conflict Classification]] · [[Verification and Validation]] · [[Policy]] · [[Flexo Conflict Resolution Mapping]]

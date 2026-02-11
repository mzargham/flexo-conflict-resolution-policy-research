# Conflict Resolution Problem Statement

## 1 Narrative Framing

### The Setting

[[Flexo MMS]] is a version control system for structured data. It stores models as [[RDF]] graphs in a [[Quadstore|quadstore]], exposes [[SPARQL]] and [[Graph Store Protocol]] endpoints for query and update, and versions model state through a commit history of SPARQL UPDATE patches. Branches, snapshots, and diffs are native operations. In principle, Flexo provides for models what Git provides for source code: a content-addressable, history-preserving, branch-and-merge workflow.

In practice, the "merge" part is missing.

### Two Senses of Model

The word "[[Model|model]]" carries two meanings in this context, and the tension between them is the core of the problem.

In the *systems engineering sense*, a model is a structured representation of a system — its parts, connections, behaviors, requirements, and the constraints that bind them. A [[SysML v2]] model, for instance, is a tree of typed elements linked by ownership and reference relationships, carrying explicit constraints (parametric equations, requirement satisfaction assertions) that restrict valid configurations. The model encodes engineering intent.

In the *storage sense*, a model at a given commit is a set of RDF triples in a named graph. Each triple is an atomic assertion — an element's type, its name, its containership, its relationship to another element. A commit transforms one triple-set into another via a SPARQL UPDATE. The model, from this vantage, is a flat collection of facts.

Conflict resolution must operate at the intersection. It manipulates the storage representation — inserting, deleting, and modifying triples — while respecting the engineering semantics that give those triples meaning. A resolution that is valid at the triple level (no duplicate triples, no syntactic errors) may be invalid at the model level (broken type constraints, dangling references, violated parametric equations). This dual nature is what makes the problem harder than textual merge.

### What Exists Today

Flexo's Layer 1 service supports branching, committing, and [[Diff and Delta|diffing]]. The diff operation computes the set-theoretic symmetric difference between two model snapshots — triples present in one but absent from the other — and stores the result as insert and delete graphs. Each commit records its delta as a SPARQL UPDATE literal, enabling snapshot reconstruction by replaying patches forward from a materialized ancestor.

The SysML v2 service defines a [[Merge|merge endpoint]] per the OMG specification: `POST /projects/{projectId}/branches/{targetBranchId}/merge`. It accepts source commit identifiers, optional conflict resolutions from the client, and returns either a merge commit (201) or a list of conflicting element identities (409).

But the implementation is scaffolding. The actual merge logic — finding a common ancestor, computing a three-way diff, detecting conflicts, applying resolutions, creating a merge commit — is not yet built. The commit model uses a single-parent linked list, so merge commits (which require multiple parents) cannot yet be represented. There is no conflict taxonomy. There is no resolution logic. There is no policy framework governing how conflicts should be handled.

### The Gap

The absence is not merely an unfinished feature. It is an unsolved design problem. To see why, consider what a merge must do:

1. Find the common ancestor of two divergent branch heads.
2. Compute two diffs: each branch's changes relative to the ancestor.
3. Identify conflicts — places where both branches modified the same content.
4. Resolve or surface those conflicts.
5. Produce a merged model state and commit it.

Steps 1 and 2 are algorithmic. Step 5 is mechanical once the merged state is known. The hard problems are in steps 3 and 4, and they are hard because "conflict" is not a single thing.

### A Spectrum of Conflicts

[[Conflict Classification|Conflicts]] manifest at three levels, each with different detection mechanisms and different resolution characteristics.

*Syntactic conflicts* are concurrent modifications to the same triples. Both branches changed an element's name to different values; one branch deleted a triple the other modified; both inserted incompatible values for a single-valued property. These are detectable by comparing the two diffs from the common ancestor — any triple appearing in the modification sets of both diffs is a syntactic conflict. Resolution is often mechanical: last-writer-wins, source-wins, union.

*Structural conflicts* arise when the merged result violates metamodel or schema constraints, even though no individual triple conflicts. Two branches each add a different owner to an element with a maximum cardinality of one. One branch deletes an element that the other references. One branch moves package P into Q while the other moves Q into P. These are invisible in the diffs alone — they appear only when the merged triple-set is checked against the metamodel.

*Semantic conflicts* occur when the merged model is syntactically and structurally valid but violates domain constraints or engineering intent. Two branches independently adjust parameters that together violate a system constraint. A merged state machine has unreachable states. Contradictory requirements are simultaneously satisfied by incompatible design choices. These require evaluating what the model *means*, not just what triples it contains.

This spectrum maps onto the [[Verification and Validation]] boundary. Syntactic and most structural conflicts fall within *verification* scope: they can be checked by computable predicates over model state. Semantic conflicts — and some structural ones — fall within *validation* scope: they require judgment, and the best an automated system can do is detect that a check has failed and surface it for human review.

### The Constraint Evaluation Problem

For any resolution strategy to work, there must be a way to evaluate whether a candidate merged state satisfies the constraints that matter. But constraints are heterogeneous. Schema conformance is checked by SHACL validation. Referential integrity is checked by SPARQL ASK queries. Parametric constraints require computation engines or external solvers. Requirement satisfaction may require human review.

The [[Predicate Compliance Oracle]] is an abstraction that collapses this heterogeneity. It is a function from model state to a per-predicate compliance score — a uniform interface over the diverse mechanisms that evaluate constraints. In practice, the oracle is a composite: a dispatch over predicate types to the appropriate evaluation backend, whether that is a database query, a CI check, or a human reviewer. The abstraction lets the formalism treat all constraints uniformly while acknowledging that their evaluation mechanisms are radically different.

### The Approach

We frame conflict resolution as a *constrained optimization problem*. The model state is the primary variable. The engineering constraints — metamodel conformance, referential integrity, parametric bounds, requirement satisfaction — are the constraints. The objective captures the desiderata of a good resolution: fidelity to the changes on both branches, minimal unnecessary modification, preference for the simpler resolution when multiple candidates exist.

In this framing, the Predicate Compliance Oracle provides the constraint evaluation, and the *dual variables* (Lagrange multipliers) of the optimization acquire a concrete interpretation. Each dual variable is a *shadow price*: it measures the sensitivity of the resolution objective to a given constraint's compliance. A high shadow price means the constraint is binding — relaxing it would significantly improve the resolution. A zero shadow price means the constraint is slack — it is satisfied with margin and does not influence the outcome. The oracle makes these dual variables real: they correspond to actual evaluations performed by actual systems or people, not to abstract mathematical quantities.

This is why the formalism is useful even though no one will implement a literal optimizer. The shadow prices tell you *which constraints matter most for this particular merge*, and that information is valuable regardless of whether the resolution is computed by an algorithm, chosen from a menu, or decided by an engineer.

### Resolution as Policy

There is no single correct resolution strategy. The right approach depends on the type of model content (structural, behavioral, parametric), the organizational context (safety-critical versus exploratory), the team's workflow conventions, and the specific constraints in play. This is why the formalism describes not a single resolution algorithm but a *family* of resolution [[Policy|policies]].

A policy, in the Flexo governance sense, is a locally-imposed, tailorable rule — a constraint asserted within a scope (organization, repository, team) that can be adjusted to circumstances. Policies are composable: multiple policies may apply simultaneously, and their interaction must be well-defined. They sit between global [[Rules and Norms|rules]] (protocol specifications, data format standards) and informal norms (workflow conventions, review expectations).

The connection to the [[Flexo MMS#Engineering Operations Levels|operational levels]] is direct. Organizations set merge policies — which constraint checks are required, which conflicts block merging, which are resolved automatically. Teams execute within those policies, tailoring parameters to their workflow. Individual contributors exercise judgment where the policy defers — reviewing flagged conflicts, evaluating validation-scope concerns, deciding between resolutions that the automated system cannot rank.

[[Continuous Integration]] is the enforcement mechanism. CI workflows give policies their teeth: they run the verification-scope checks, invoke the relevant oracles, and gate merges on the results. The formalism defines what those checks are and how their results compose; CI executes them.

### What Follows

The remainder of this document makes the framing precise. §2 states the formal problem: the state variable, dynamics, objective, and constraints of the optimal control formulation. §3 establishes notation, drawing on a generalized dynamical systems framework. §4 develops the constrained optimization and Lagrange duality, connecting dual variables to predicate compliance scores and shadow prices. §5 defines the family of conflict resolution policies in terms of admissible actions and valid model states. §6 gives concrete examples — heuristic policies like last-writer-wins, source-wins, and union-with-constraint-check — and frames them in the governance structure: policy-making at the organizational level, judgment by qualified individuals on teams.

---

## 2 Formal Problem Description

*To be provided by the user — the optimal control formulation: state variable, dynamics, cost functional, constraint structure.*

---

## 3 Notation

*To be provided by the user — notation conventions from the generalized dynamical systems framework: state spaces, action sets, transition maps.*

---

## 4 Constrained Optimization and Lagrange Duality

*Collaborative: assistant drafts the standard optimization/duality framing; user validates the mapping from dual variables to predicate compliance scores and shadow prices.*

---

## 5 Family of Conflict Resolution Policies

*Collaborative: assistant drafts the policy family structure (admissible actions × valid states); user validates consistency with the GDS framework and the optimal control formulation.*

---

## 6 Heuristic Policy Examples

*Assistant-led: concrete merge strategies as example policies, with governance framing.*

---
← [[Flexo MMS]] · [[Flexo Conflict Resolution Mapping]]

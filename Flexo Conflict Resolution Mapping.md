# Flexo Conflict Resolution Mapping

This document maps the conflict resolution formalism to concrete [[Flexo MMS]] operations. It is a condensed overview — the core idea, key equations, and the bridge from abstract concepts to Flexo's architecture. The full mathematical treatment, including derivations, notation, duality theory, and worked policy examples, lives in the [[Conflict Resolution Problem Statement]].

---

## The Problem in Brief

[[Flexo MMS]] provides branching, committing, and [[Diff and Delta|diffing]] for [[RDF]] graph models. The [[Merge|merge endpoint]] is defined in the [[SysML v2]] API specification, but the implementation is scaffolding — the actual logic for finding a common ancestor, computing a three-way diff, detecting conflicts, and applying resolutions is not yet built.

Merging models is harder than merging text because models carry constraints. A [[Model|model]] is not just a collection of triples — it is a structured representation of a system with typed elements, ownership relationships, parametric equations, and behavioral properties. A merge that produces a syntactically valid triple-set may still violate the engineering semantics that give those triples meaning: broken type constraints, dangling references, violated parametric bounds.

[[Conflict Classification|Conflicts]] manifest at three levels. *Syntactic conflicts* are concurrent modifications to the same triples — detectable by comparing diffs. *Structural conflicts* arise when the merged result violates schema or metamodel constraints, even though no individual triple conflicts — detectable only by checking the merged state against the metamodel. *Semantic conflicts* occur when the merged model violates domain constraints or engineering intent — requiring evaluation of what the model *means*. This spectrum maps onto the [[Verification and Validation]] boundary: syntactic and structural conflicts are verification-scope (computable), while semantic conflicts are validation-scope (requiring judgment).

---

## The Core Idea

The formalism frames conflict resolution as a **constrained optimization problem**. A model state $X$ is the set of elements, relationships, and attributes at a given commit. Commits $u$ and $v$ are control inputs — descriptions of intended state change. The state transition function $f$ applies a commit to a state:

$$X^+ = f(X, u)$$

When two contributors produce commits $u$ and $v$ from a common ancestor $X$, we assess their interaction by applying each on top of the other — both orderings — and evaluating constraints on the results. Any constraint violated in either ordering is a conflict:

$$\mathcal{C}_{\text{conflict}} = \{c_i \in \mathcal{C} \mid c_i(f(f(X,u),v)) > 0 \;\lor\; c_i(f(f(X,v),u)) > 0\}$$

When conflicts exist, the resolution policy synthesizes a commit $w^*$ by minimizing an **intent loss** — how far the resolution deviates from the ideal composition of both commits — subject to all constraints being satisfied:

$$\min_{w} \; L_{\text{intent}}(w;\, u, v) \quad \text{subject to} \quad C(f(X, w)) \leq \mathbf{0}$$

The **Lagrangian** associates a multiplier $\mu_i \geq 0$ with each constraint:

$$\mathcal{L}(w, \mu) = L_{\text{intent}}(w;\, u, v) + \mu^\top C(f(X, w))$$

At the optimum, the multipliers $\mu^*$ become **shadow prices**: each $\mu^*_i$ measures how much constraint $c_i$ influenced the resolution. A high shadow price means the constraint is binding — it actively shaped the outcome. A zero shadow price means the constraint is slack — satisfied with margin. These shadow prices trace through a satisfaction relation to **requirements**, so every deviation from the ideal merge is attributable to a specific stakeholder concern.

The key insight: shadow prices are actionable even without implementing a literal optimizer. They tell you *which constraints matter most for this particular merge* — information that is valuable whether the resolution is computed by an algorithm, selected from a menu of heuristics, or decided by an engineer.

---

## Resolution Policies

A resolution policy is not a fully automated algorithm. It is a **human-machine protocol** — the machine evaluates constraints, computes violations, and presents structured information; the human exercises judgment where the machine cannot. The formalism defines a family of policies, parameterized by which constraints are checked, how aggressively the system attempts automated resolution, and where the boundary between machine and human falls.

| Policy | Checks | Produces | Governance Context |
| --- | --- | --- | --- |
| **Last-writer-wins** | None | Accepted commit (no validation) | Exploratory, low-stakes |
| **Source/target-wins** | Optional | Priority-resolved commit | Asymmetric workflows (main is authoritative) |
| **Union-with-constraint-check** | $\mathcal{C}_{\text{active}}$ | Merged commit or conflict report | Teams wanting safe auto-merge with escalation |
| **Constraint-aware synthesis** | $\mathcal{C}_{\text{active}}$ | Resolution + shadow prices + traceability | High-stakes, constraint-rich models |
| **Escalation-only** | $\mathcal{C}_{\text{active}}$ | Conflict report (no resolution) | Safety-critical, every conflict requires human review |

The progression from top to bottom is a progression in **information**: from no constraint evaluation, to pass/fail checks, to full shadow prices and traceability. Each step requires more infrastructure and produces more insight. The right policy depends on how much information the organization needs and what it is willing to invest.

Policies compose across [[Flexo MMS#Engineering Operations Levels|operational levels]]. The organization sets $\mathcal{C}_{\text{active}}$ and the automation boundary. The team selects workflow parameters. The individual exercises judgment on validation-scope conflicts. [[Continuous Integration]] enforces the machine portion.

---

## Mapping to Flexo Operations

The table below maps formalism concepts to their concrete counterparts in the Flexo architecture.

| Formalism | Flexo Operation | Layer | Status |
| --- | --- | --- | --- |
| Model state $X$ | Named graph (snapshot) in [[Quadstore\|quadstore]] | 0 — Quadstore | Exists |
| Commit $u$ | [[Diff and Delta\|SPARQL UPDATE patch]] | 1 — Layer 1 | Exists |
| State transition $f(X, u)$ | `POST .../branches/{branchId}/update` | 1 — Layer 1 | Exists |
| Common ancestor (LCA) | Commit DAG traversal | 1 — Layer 1 | **Not yet implemented** (single-parent linked list) |
| Three-way diff | Two diffs from ancestor to each branch head | 1 — Layer 1 | **Not yet implemented** (two-way diff exists) |
| Constraint evaluation $C(X)$ | [[Predicate Compliance Oracle]] dispatch: [[SPARQL\|SHACL]], SPARQL ASK, external solvers, human review | Cross-layer | **Not yet implemented** |
| Resolution policy $g$ | [[Merge]] endpoint: `POST /projects/{projectId}/branches/{targetBranchId}/merge` | 2 — [[SysML v2]] API | Endpoint exists (scaffolding); logic not implemented |
| Shadow prices $\mu^*$ | Merge response payload (conflict report with constraint attribution) | 2 — SysML v2 API | **Not yet implemented** |
| CI enforcement | [[Continuous Integration\|CI]] pipeline gates merge on $\mathcal{C}_{\text{active}}$ | External | **Not yet implemented** |
| Governance scoping | [[Flexo MMS#Engineering Operations Levels\|Operational levels]]: org sets constraints, team sets parameters, individual exercises judgment | Cross-layer | Governance model defined; enforcement not implemented |

### What Exists

The storage and versioning infrastructure is in place. Model state lives in named graphs. Commits are SPARQL UPDATE patches. Branches, snapshots, and two-way diffs work. The merge endpoint exists in the SysML v2 API with the correct signature per the OMG specification. Optimistic concurrency control (ETags, transaction mutexes) prevents lost updates on single branches.

### What Needs to Be Built

1. **Commit DAG**: Extend the commit model from a single-parent linked list to a DAG supporting multiple parents, enabling merge commits and LCA computation.

2. **Three-way diff**: Compute two diffs from the common ancestor — one to each branch head — and correlate them to identify conflicting and non-conflicting changes.

3. **Predicate Compliance Oracle**: A dispatch layer that evaluates heterogeneous constraints — SHACL shapes for schema, SPARQL ASK for referential integrity, external solvers for parametric bounds, human review for behavioral properties — and returns a uniform compliance vector.

4. **Policy engine**: Logic behind the merge endpoint that implements the resolution policy family — from pass-through (no conflicts) to constraint-aware synthesis (full optimization).

5. **CI integration**: Pipeline hooks that invoke the oracle, gate merges on $\mathcal{C}_{\text{active}}$, and report shadow prices to reviewers.

---

## Full Formalism

The [[Conflict Resolution Problem Statement]] develops the mathematics in full:

- **§2** states the formal problem: model state, commits, constraint evaluation, the concurrent modification problem, and the constrained optimization.
- **§3** establishes notation from a generalized dynamical systems framework.
- **§4** develops the Lagrange duality interpretation: domain vs. optimization constraints, complementary slackness, sensitivity analysis, infeasibility handling, and the explainability payoff.
- **§5** defines the family of resolution policies as human-machine protocols with formal properties (transparency, validity, monotonicity) and governance scoping.
- **§6** gives five concrete policy examples with governance framing.
- **§7** identifies open problems: oracle implementation, delta composition, computational tractability, multi-party merges, federation.
- **§8** collects references across constrained optimization, dynamical systems, MDO, and software merging.

This document maps the *what* and *where*; the problem statement develops the *why* and *how*.

---
← [[Flexo MMS]] · [[Model]] · [[Diff and Delta]] · [[Merge]] · [[Conflict Classification]] · [[Predicate Compliance Oracle]] · [[Policy]]

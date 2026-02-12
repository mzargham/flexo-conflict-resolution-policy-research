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

### Requirements and Constraints

In [[SysML v2]], requirements and constraints are ontologically distinct. A *requirement* is a stakeholder need — "the system shall…" — evaluated by judgment. A *constraint* is a computable predicate over model state — something the system can check mechanically.

| Aspect | Requirement | Constraint |
| ------ | ----------- | ---------- |
| Nature | Stakeholder need | Computable predicate |
| Expresses | *What* must be achieved | *How* we verify achievement |
| Evaluation | Judgment | Computation |
| Role in V&V | What we [[Verification and Validation\|validate]] | How we [[Verification and Validation\|verify]] |

Let $\mathcal{R} = \{r_1, r_2, \ldots, r_n\}$ be the set of requirements. Let $\mathcal{C} = \{c_1, c_2, \ldots, c_m\}$ be the set of constraints.

The **satisfaction relation** $S \subseteq \mathcal{C} \times \mathcal{R}$ captures how constraints provide evidence for requirements. We write $c \vdash r$ to mean that constraint $c$ demonstrates (partially or fully) that requirement $r$ is satisfied. The relation is many-to-many: a constraint may demonstrate multiple requirements, and a requirement may be demonstrated by multiple constraints.

The formalism operates at the constraint level — these are what the system can evaluate. But it traces back to the requirement level — these are what stakeholders care about.

### Model State and Commits

A **model state** $X \in \mathcal{X}$ is a structured representation of the [[Model|model]] at a given point in its history. Formally:

$$X = (E, R, A)$$

where $E$ is the set of elements, $R$ is the set of typed relationships between elements, and $A$ assigns attribute values to elements. In [[Flexo MMS]], this corresponds to the set of [[RDF]] triples in a snapshot's named graph.

A **commit** $u \in \mathcal{U}$ is a description of intended state change: elements to add or remove, relationships to add or remove, attribute values to modify. In Flexo, a commit is a [[Diff and Delta|SPARQL UPDATE patch]] applied to the model graph.

The **state transition function** $f$ applies a commit to a state:

$$X^+ = f(X, u)$$

yielding a new state $X^+$. This is the control-theoretic framing: the model state is the state variable, and commits are control inputs.

### Composition of Commits

Given two commits $u$ and $v$, their **composition** $u \oplus v$ denotes the intended union of their changes — what the merged result *should* be if both sets of changes could coexist without interference.

Composition is **not commutative in general**. Applying $u$ first and then $v$ may yield a different state than applying $v$ first and then $u$:

$$f(f(X, u), v) \neq f(f(X, v), u) \quad \text{in general}$$

Commutativity holds only when the commits modify disjoint portions of the state — when there is no coupling between the changes. This non-commutativity is the structural reason that conflict detection must evaluate *both* orderings, and it is what makes the problem fundamentally different from set union.

### Constraint Evaluation

Each constraint $c_i : \mathcal{X} \to \mathbb{R}$ maps a model state to a real number, with the convention:

$$c_i(X) \leq 0 \implies \text{constraint } c_i \text{ is satisfied}$$

A positive value indicates violation; its magnitude indicates severity. The **constraint evaluation operator** $C : \mathcal{X} \to \mathbb{R}^m$ collects all constraint evaluations into a vector:

$$C(X) = \begin{pmatrix} c_1(X) \\ \vdots \\ c_m(X) \end{pmatrix}$$

A state is valid when $C(X) \leq \mathbf{0}$ (componentwise). The operator $C$ is the formal counterpart of the [[Predicate Compliance Oracle]] — the uniform interface over heterogeneous constraint evaluation mechanisms.

Constraints partition by scope:

| Category                         | Description                | Example                                        |
| -------------------------------- | -------------------------- | ---------------------------------------------- |
| **Local** ($\mathcal{C}_L$)      | Single element             | mass $\geq$ 0                                  |
| **Relational** ($\mathcal{C}_R$) | Between connected elements | port types must match                          |
| **Aggregate** ($\mathcal{C}_A$)  | Over collections           | total mass $\leq$ budget                       |
| **Coupling** ($\mathcal{C}_K$)   | Across design disciplines  | thermal dissipation $\leq$ structural capacity |
| **Behavioral** ($\mathcal{C}_B$) | Dynamic semantics of model elements | state machine must be deterministic            |

**Coupling constraints** are the primary source of machine identifiable merge conflicts. They span decisions made by different contributors — and they are why individually valid commits can be jointly invalid.

This classification is by scope. A separate distinction — between constraints that define the domain (pass/fail predicates like schema conformance) and constraints that admit degrees of violation (continuous-valued predicates like budget slack) — is developed in §4, where it determines which constraints carry shadow prices.

Behavioral constraints deserve explicit comment. They are included because the models under version control are not static artifacts — they describe systems with dynamic behavior, and the engineers working with them need behavioral properties to participate in conflict resolution. Some behavioral constraints can be evaluated computationally: simulation pipelines that check stability, reachability, or termination and return numerical results. Others require human judgment — an engineer runs an experiment, performs analysis, or exercises domain expertise, and records the outcome.

In both cases, the result enters the model state as a stored value, making the satisfaction status available for computation by the [[Predicate Compliance Oracle]]. The human or simulation pipeline acts as an oracle in the literal sense: it evaluates a predicate the system cannot evaluate internally and deposits the answer where the system can use it. Once recorded, a behavioral constraint's satisfaction value is operationally identical to any other constraint value in the formalism.

The critical caveat is **staleness**. A stored behavioral evaluation is valid only relative to the model state against which it was produced. When any upstream model element that the evaluation depended on changes — a parameter is modified, a component is replaced, an interface is restructured — the stored result is **defunct** and must be re-evaluated before it can be trusted. Tracking this dependency and invalidation is itself a constraint management problem, but for the purposes of this formalism, the key point is simpler: behavioral constraints participate in the constraint evaluation operator $C$ on the same terms as all other constraints, with the understanding that their values may require re-evaluation when the model state changes beneath them.

### The Concurrent Modification Problem

Two contributors pull from a common ancestor state $X$ where all constraints are satisfied: $C(X) \leq \mathbf{0}$. Contributor A produces commit $u$; contributor B produces commit $v$. Each is individually valid:

$$C(f(X, u)) \leq \mathbf{0} \qquad C(f(X, v)) \leq \mathbf{0}$$

We seek a reconciled state $X^+$ that satisfies all constraints. To find it, we first assess the damage. Define the **cross-application states** — what happens when we apply each commit on top of the other:

$$X_{uv} = f(f(X, u), v) \qquad X_{vu} = f(f(X, v), u)$$

and evaluate constraints on both:

$$\lambda_{uv} = C(X_{uv}) \qquad \lambda_{vu} = C(X_{vu})$$

These **constraint violation vectors** reveal where the commits interact. A constraint conflict exists whenever either ordering produces a violation:

$$\mathcal{C}_{\text{conflict}} = \{c_i \in \mathcal{C} \mid (\lambda_{uv})_i > 0 \;\lor\; (\lambda_{vu})_i > 0\}$$

The **impacted requirements** are those demonstrated by conflicting constraints:

$$\mathcal{R}_{\text{impacted}} = \{r_j \in \mathcal{R} \mid \exists\, c_i \in \mathcal{C}_{\text{conflict}} : c_i \vdash r_j\}$$

Since $u$ and $v$ are individually valid, any violations in $\lambda_{uv}$ or $\lambda_{vu}$ arise from *interactions* between the commits — neither commit is defective on its own. These interactions can trigger violations at any constraint scope: a relational constraint when both commits connect to the same port, an aggregate constraint when both consume remaining slack in a shared budget, a behavioral constraint when both modify inputs to the same simulation. The common thread is shared dependencies in the model state, not any single constraint category.

### Resolution Policy

The **resolution [[Policy|policy]]** $g$ takes the ancestor state, both violation vectors, and both commits, and produces a resolution commit:

$$w = g(X, \lambda_{uv}, \lambda_{vu}, u, v)$$

such that $X^+ = f(X, w)$ and $C(X^+) \leq \mathbf{0}$.

The policy operates in two regimes:

| Condition                                      | Regime           | Behavior                                                                                                                    |
| ---------------------------------------------- | ---------------- | --------------------------------------------------------------------------------------------------------------------------- |
| $\mathcal{C}_{\text{conflict}} = \emptyset$    | **Pass-through** | $w := u \oplus v$ satisfying $C(f(X, u \oplus v)\le 0$. No conflict; commits compose without modification.                  |
| $\mathcal{C}_{\text{conflict}} \neq \emptyset$ | **Synthesis**    | $w := \hbox{argmin}_w\, L(w;u,v)$ s.t. $C(f(X,w))\le0$. The policy synthesizes a resolution that satisfies all constraints. |

Two properties hold by construction:

- **Transparency**: When no conflict exists, the policy does not modify the commits. The result is exactly the composition $u \oplus v$.
- **Validity**: In both regimes, the resulting state satisfies all constraints: $C(f(X, w)) \leq \mathbf{0}$.

### Synthesis via Constrained Optimization

When $\mathcal{C}_{\text{conflict}} \neq \emptyset$, the policy synthesizes $w^*$ by solving a constrained optimization problem. The **primal problem** is:

$$\min_{w \in \mathcal{U}} \; L_{\text{intent}}(w;\, u, v) \quad \text{subject to} \quad C(f(X, w)) \leq \mathbf{0}$$

The **intent loss** $L_{\text{intent}}$ measures how far the resolution deviates from the ideal composition:

$$L_{\text{intent}}(w;\, u, v) = d_X(w,\, u \oplus v) \;+\; \gamma \cdot \|w\|_{\text{complexity}}$$

The first term penalizes deviation from what the contributors intended. The distance $d_X$ is state-dependent: the same commit description can have different impact depending on the model state it is applied to, so measuring how far $w$ deviates from $u \oplus v$ requires knowledge of the ancestor state $X$ against which both are evaluated. The second term regularizes toward simpler resolutions. The weight $\gamma$ is a policy parameter that controls the tradeoff between the two terms.

The **Lagrangian** associates a multiplier $\mu_i \geq 0$ with each constraint:

$$\mathcal{L}(w, \mu) = L_{\text{intent}}(w;\, u, v) \;+\; \mu^\top C(f(X, w))$$

At the optimum, the **dual variables** $\mu^*$ acquire a concrete interpretation as **shadow prices**. The shadow price $\mu^*_i$ measures the sensitivity of the resolution objective to constraint $c_i$:

- A high $\mu^*_i$ means constraint $c_i$ is *binding* — it is tight, and relaxing it would materially improve the resolution. This constraint actively shaped the outcome.
- A zero $\mu^*_i$ means constraint $c_i$ is *slack* — it is satisfied with margin and did not influence the resolution.

The shadow prices make explicit *which constraints matter for this particular merge*. This information is valuable whether the resolution is computed by an optimizer, selected from a menu of heuristics, or decided by a human engineer.

### Requirement Prices

The dual variables live at the constraint level. To trace their impact to stakeholder concerns, we aggregate them to the requirement level via the satisfaction relation. For each impacted requirement $r_j$:

$$\nu_j = \sum_{c_i \in \mathcal{C}(r_j)} \mu^*_i$$

where $\mathcal{C}(r_j) = \{c_i \mid c_i \vdash r_j\}$ is the set of constraints that demonstrate requirement $r_j$. The **requirement price** $\nu_j$ indicates how much requirement $r_j$ influenced the resolution overall.

The traceability chain is complete: every deviation of $w^*$ from $u \oplus v$ is attributable to a binding constraint ($\mu^*_i > 0$), which in turn demonstrates a requirement ($c_i \vdash r_j$), which traces to a stakeholder need. No unexplained modifications.

### Assumptions

The formulation rests on four essential assumptions:

1. **Individual validity.** Each commit, applied in isolation to the common ancestor, satisfies all constraints: $C(f(X, u)) \leq \mathbf{0}$ and $C(f(X, v)) \leq \mathbf{0}$. Contributors have locally validated their changes.

2. **Non-commutativity of composition.** $f(f(X,u), v) \neq f(f(X,v), u)$ in general. Commits compose order-dependently; commutativity holds only when they modify disjoint state. This is the structural reason both orderings must be evaluated.

3. **Feasibility.** The feasible set $\{w \in \mathcal{U} \mid C(f(X, w)) \leq \mathbf{0}\}$ is non-empty. When this fails — when no valid resolution exists — the policy escalates to human review rather than producing an invalid state.

4. **Conservative conflict detection.** If *either* application order violates constraints, the policy enters analysis mode. If one order is the valid and the other is invalid then the valid ordering is the default synthesis.

---

## 3 Notation

The notation draws on a generalized dynamical systems framework — state spaces, control inputs, transition functions — applied to model version control.

| Symbol | Meaning |
| ------ | ------- |
| $X$ | Model state (elements, relationships, attributes) |
| $u, v, w$ | Commits (control inputs) |
| $u \oplus v$ | Composition of commits (order-dependent; not commutative in general) |
| $f(X, u)$ | State transition: apply commit $u$ to state $X$ |
| $\mathcal{C}$, $c_i$ | Set of constraints; individual constraint ($\leq 0$ when satisfied) |
| $C(X)$ | Constraint evaluation operator (vector of all $c_i(X)$) |
| $\mathcal{R}$, $r_j$ | Set of requirements; individual requirement |
| $c \vdash r$ | Constraint $c$ demonstrates requirement $r$ |
| $\lambda$ | Constraint violation vector (from cross-application analysis) |
| $\mu^*$ | Shadow prices (Lagrange multipliers at the optimum) |
| $\nu$ | Requirement prices (aggregated from $\mu^*$ via satisfaction relation) |
| $g$ | Resolution policy |
| $\mathcal{C}_{\text{conflict}}$ | Set of conflicting constraints |
| $\mathcal{X}$ | Space of model states |
| $\mathcal{U}$ | Space of commits (control inputs) |
| $d_X$ | State-dependent distance between commits |
| $\gamma$ | Policy weight (tradeoff between intent preservation and resolution complexity) |
| $L_{\text{intent}}$ | Intent loss (objective measuring deviation from ideal composition) |
| $\theta$ | Policy configuration (parameters defining a specific resolution policy) |
| $\mathcal{G}$ | Family of resolution policies $\{g_\theta : \theta \in \Theta\}$ |
| $\mathcal{C}_{\text{active}}$ | Active constraint set (subset of $\mathcal{C}$ enforced by policy) |
| $\mathcal{A}(X)$ | Admissible action set (commits producing valid states from $X$) |
| $\mathcal{V}$ | Valid state set $\{X \in \mathcal{X} \mid C_{\text{active}}(X) \leq \mathbf{0}\}$ |

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

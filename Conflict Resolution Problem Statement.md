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

The remainder of this document makes the framing precise. §2 states the formal problem: the state variable, dynamics, objective, and constraints of the optimal control formulation. §3 establishes notation, drawing on a generalized dynamical systems framework. §4 develops the constrained optimization and Lagrange duality, connecting dual variables to predicate compliance scores and shadow prices. §5 defines the family of conflict resolution policies in terms of admissible actions and valid model states. §6 gives concrete examples — heuristic policies like last-writer-wins, source-wins, and union-with-constraint-check — and frames them in the governance structure: policy-making at the organizational level, judgment by qualified individuals on teams. §7 identifies open problems and future work. §8 collects references.

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

yielding a new state $X^+$. This is the control-theoretic framing: the model state is the state variable, and commits are control inputs (see [[Key Insight]] for why this separation is necessary for structured models).

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

---

## 4 Constrained Optimization and Lagrange Duality

§2 stated the optimization problem and introduced the Lagrangian and shadow prices. This section develops the interpretation: what the duality tells you in practice, how the abstract constraint functions connect to concrete evaluation mechanisms, and what happens when things go wrong.

### Domain Constraints and Optimization Constraints

The constraints in §2 were defined uniformly as $c_i : \mathcal{X} \to \mathbb{R}$ with $c_i(X) \leq 0$ meaning satisfied. In practice, constraints play two distinct roles, and the distinction determines which constraints carry shadow prices.

**Domain constraints** are hard pass/fail predicates — schema conformance, referential integrity, type correctness. A state that fails a domain constraint is not a valid model state at all: it is a malformed graph, a dangling reference, a type error. Domain constraints define the space $\mathcal{X}$ of well-formed model states and the space $\mathcal{U}$ of well-formed commits. They do not participate in the optimization as inequality constraints with multipliers — they define the space within which the optimization operates.

**Optimization constraints** are continuous-valued: parametric bounds, budget allocations, behavioral metrics. They admit degrees of violation — a mass budget can be exceeded by a little or a lot — and their constraint functions $c_i(X)$ return real-valued magnitudes. These are the constraints that enter the Lagrangian and carry meaningful shadow prices.

| Constraint role | Nature | Example | In the formalism |
| --------------- | ------ | ------- | ---------------- |
| **Domain** | Pass/fail; defines well-formedness | Referential integrity, schema conformance, type correctness | Constrains $\mathcal{X}$ and $\mathcal{U}$ (no multiplier) |
| **Optimization** | Continuous-valued; admits degrees of violation | Mass budget, thermal capacity, behavioral metric | $c_i(X) \leq 0$ with shadow price $\mu^*_i$ |

The reason for this split is mathematical. A shadow price $\mu^*_i$ measures *sensitivity* — the rate at which the resolution objective changes per unit of constraint relaxation. This requires the constraint function to be continuous: you need a meaningful notion of "slightly more" or "slightly less" violated. Binary pass/fail constraints have no such rate. They are either satisfied (the state is in the domain) or violated (the state is outside it entirely). There is no gradient to compute.

### Connecting to the Predicate Compliance Oracle

The [[Predicate Compliance Oracle]] implements constraint evaluation. Different predicate types map to different evaluation mechanisms and to different roles in the formalism:

| Predicate type | Evaluation mechanism | Role |
| -------------- | -------------------- | ---- |
| Schema conformance | [[SPARQL\|SHACL]] validation | Domain |
| Referential integrity | SPARQL ASK query | Domain |
| Parametric constraint | Computation engine / external solver | Optimization (continuous) |
| Behavioral correctness | Simulation pipeline / expert judgment | Optimization (continuous or judgment-estimated) |
| Requirement satisfaction | Human review | Optimization (judgment-estimated) |

Domain constraints are [[Verification and Validation|verification]]-scope: deterministic, automatable, and binary. Optimization constraints span both verification (parametric bounds computed mechanically) and validation (behavioral properties evaluated by simulation or human judgment). The oracle abstracts over this heterogeneity — it dispatches each predicate to the appropriate evaluation mechanism and returns a score that the formalism can use.

### Complementary Slackness

At the optimum, a fundamental relationship holds between each constraint and its shadow price:

$$\mu^*_i \cdot c_i(X^+) = 0 \quad \text{for each optimization constraint } c_i$$

This says: either the constraint is **slack** ($c_i(X^+) < 0$, satisfied with margin) and its multiplier is zero ($\mu^*_i = 0$, it did not influence the resolution), or the constraint is **binding** ($c_i(X^+) = 0$, tight) and its multiplier may be positive ($\mu^*_i > 0$, it actively shaped the outcome). Both cannot be nonzero simultaneously.

The consequence is **sparsity**. For any given merge, most constraints are slack — the resolution satisfies them with room to spare, and their shadow prices are zero. Only the binding constraints have nonzero $\mu^*_i$, and these are the constraints that *matter for this particular merge*. The shadow price vector $\mu^*$ is a sparse, targeted summary of which constraints shaped the resolution.

### Sensitivity and the Dual as Diagnostic

The shadow price $\mu^*_i$ has a precise sensitivity interpretation:

$$\mu^*_i = \frac{\partial L^*_{\text{intent}}}{\partial b_i}$$

where $b_i$ represents the right-hand side of constraint $c_i(X) \leq b_i$ (in §2 we set $b_i = 0$, but the sensitivity is defined with respect to perturbations). In words: *relaxing constraint $c_i$ by one unit improves the resolution objective by $\mu^*_i$*.

This is actionable information. A high shadow price on a coupling constraint might mean:

- The constraint is too tight for the current design — the engineers should revisit the constraint, not the merge
- Two teams are working at cross purposes — the conflict is not a merge artifact but a design disagreement that the merge has surfaced
- The constraint is correct but the resolution required significant compromise — the reviewer should inspect what was sacrificed

Combined with the requirement prices from §2 — $\nu_j = \sum_{c_i \in \mathcal{C}(r_j)} \mu^*_i$ — the sensitivity analysis traces from individual constraints to stakeholder requirements. An engineer reviewing a merge can see not just *that* a conflict was resolved, but *which requirements bore the cost*.

### Infeasibility

Assumption 3 (§2) states that a feasible resolution exists. When it fails — when no commit $w$ produces a state satisfying all active constraints — the primal problem is infeasible.

Infeasibility is not a failure of the formalism. It is information: the two commits *cannot coexist* under the current constraints, and the dual provides structured evidence for why. The dual problem, when the primal is infeasible, identifies a minimal set of constraints that cannot be simultaneously satisfied — an **infeasibility certificate**.

The policy response is **escalation**. The system reports:

- $\mathcal{C}_{\text{conflict}}$ — the conflicting constraints
- $\mathcal{R}_{\text{impacted}}$ — the impacted requirements
- The infeasibility certificate — which constraint subset is mutually unsatisfiable
- $\lambda_{uv}$ and $\lambda_{vu}$ — the violation vectors from both application orderings

A human then decides: relax a constraint (change the policy), reject one commit (preserve the other), or restructure the model (resolve the underlying design conflict). The formalism ensures this decision is made with full information about *why* no automated resolution exists.

### The Explainability Payoff

The duality machinery serves a single purpose: **auditability**. Every merge resolution produced by the formalism comes with a complete explanation:

- For each deviation of $w^*$ from $u \oplus v$: the binding constraint ($\mu^*_i > 0$) that forced the deviation, and the requirement ($c_i \vdash r_j$) it traces to
- For feasible resolutions: the shadow prices rank constraints by their influence on the outcome
- For infeasible cases: the certificate names the constraints that cannot be simultaneously satisfied

This is the connection to governance. Organizational [[Policy|policy]] sets which constraints are domain (hard, defining well-formedness) and which are optimization (continuous, admitting tradeoffs). The dual variables report the cost of those choices for each specific merge. If a policy decision — say, classifying a behavioral constraint as required rather than advisory — consistently produces high shadow prices or infeasibility, that is evidence the policy should be revisited. The formalism makes the consequences of policy choices visible and quantifiable.

---

## 5 Family of Conflict Resolution Policies

### Policies as Human-Machine Protocols

A resolution policy is not a fully automated algorithm. It is a **protocol** that allocates steps between machine computation and human judgment, with the machine augmenting human decision-making to discover a maximally intent-preserving feasible commit.

The machine's role is to evaluate domain constraints, compute constraint violation vectors, solve or approximate the optimization, identify $\mathcal{C}_{\text{conflict}}$ and $\mathcal{R}_{\text{impacted}}$, compute shadow prices, and present structured conflict reports. It operates within the verification scope — the domain of computable predicates.

The human's role is to evaluate validation-scope constraints (behavioral properties, requirement satisfaction), exercise judgment on flagged conflicts, choose between candidate resolutions that the machine cannot rank, and decide how to proceed when the problem is infeasible. Humans serve as [[Predicate Compliance Oracle|oracles]] in the sense developed in §2: they evaluate predicates the system cannot evaluate internally and deposit their judgments into the model state where the formalism can use them.

The formalism structures this interaction. It does not replace human judgment — it ensures that human judgment is exercised with full information about the constraint landscape, the cost of each binding constraint, and the requirements at stake.

### Policy Parameterization

A resolution policy $g$ is parameterized by a configuration $\theta$:

- $d_X$: the choice of state-dependent distance — how "deviation from intent" is measured
- $\gamma$: the complexity weight — the tradeoff between intent preservation and resolution simplicity
- $\mathcal{C}_{\text{active}} \subseteq \mathcal{C}$: the set of constraints that are enforced — which checks the organization requires
- **Oracle configuration**: which evaluation mechanisms are invoked (including human review steps), with what timeouts and fallbacks
- **Automation boundary**: which steps are machine-executed and which require human sign-off

Different configurations produce different policies. The **family** $\mathcal{G} = \{g_\theta : \theta \in \Theta\}$ is the set of all admissible resolution policies. There is no single correct policy — the right choice depends on the type of model content, the organizational context, the team's workflow conventions, and the specific constraints in play.

### Admissible Actions and Valid States

Given a configuration $\theta$, the **admissible action set** at state $X$ is:

$$\mathcal{A}(X) = \{w \in \mathcal{U} \mid C_{\text{active}}(f(X, w)) \leq \mathbf{0}\}$$

These are the commits that produce valid states under the active constraints. The **valid state set** is:

$$\mathcal{V} = \{X \in \mathcal{X} \mid C_{\text{active}}(X) \leq \mathbf{0}\}$$

A policy $g_\theta$ is **valid** if for every input $(X, u, v)$ with $X \in \mathcal{V}$, the output $w = g_\theta(X, \lambda_{uv}, \lambda_{vu}, u, v)$ satisfies $f(X, w) \in \mathcal{V}$. Valid policies never produce invalid states.

Note that different $\mathcal{C}_{\text{active}}$ produce different $\mathcal{V}$ — different notions of "valid." An organization that enforces coupling constraints has a stricter $\mathcal{V}$ (fewer valid states) than one that treats them as advisory.

### Policy Properties

Three properties characterize well-behaved policies:

- **Transparency.** When no conflict exists ($\mathcal{C}_{\text{conflict}} = \emptyset$), the policy does not modify the commits: $g_\theta(\ldots) = u \oplus v$. The pass-through regime is parameter-independent — it holds for every $\theta$.

- **Validity.** The policy always produces a valid state: $f(X, g_\theta(\ldots)) \in \mathcal{V}$ for all valid inputs.

- **Monotonicity.** If $\mathcal{C}_{\text{active}} \subseteq \mathcal{C}'_{\text{active}}$, then $\mathcal{A}'(X) \subseteq \mathcal{A}(X)$. More constraints mean fewer admissible resolutions. Tightening policy never expands options — it can only narrow them.

Monotonicity has a practical consequence: adding a constraint to the active set can turn a previously feasible merge into an infeasible one, but never the reverse. Organizations should be aware that each additional required check reduces the space of automated resolutions available.

### Composability and Governance Scoping

Policies compose across the [[Flexo MMS#Engineering Operations Levels|operational levels]] through a nesting structure. Each level narrows the family $\mathcal{G}$ by fixing parameters that lower levels cannot override:

| Level | What is set | Example |
| ----- | ----------- | ------- |
| **Organization** | $\mathcal{C}_{\text{active}}$, domain vs. optimization classification, automation boundary | "All coupling constraints are enforced; behavioral constraints require human oracle; automated synthesis is permitted for parametric conflicts" |
| **Team** | $\gamma$, $d_X$ choice, oracle timeouts, review workflow | "Prefer minimal-diff resolutions (low $\gamma$); run simulation oracle with 60s timeout; two-reviewer sign-off on coupling conflicts" |
| **Individual** | Judgment on validation-scope conflicts, oracle evaluations | Engineer reviews flagged $\mathcal{R}_{\text{impacted}}$, runs experiments, records behavioral constraint evaluations, decides between candidate resolutions |

This is the [[Policy|policy]] structure from the Flexo governance model: scoped, assertive, tailorable, composable. The organization restricts $\mathcal{C}_{\text{active}}$ and sets the automation boundary. The team selects parameters and review workflows within that. The individual exercises judgment on what remains. Each level narrows $\Theta$ further, and the composition is well-defined because the nesting respects scope.

[[Continuous Integration]] enforces the machine portion of the protocol. CI runs the verification-scope predicates in $\mathcal{C}_{\text{active}}$, gates merges on domain constraints, and reports shadow prices. Human steps — review, behavioral evaluation, judgment calls — are workflow steps that CI can *require* but not *perform*. The boundary between what CI executes and what it delegates to humans is itself a policy parameter, set at the organizational level.

---

## 6 Heuristic Policy Examples

The policies below are concrete instances of the family $\mathcal{G}$ from §5. Each is defined by its parameter configuration $\theta$, its behavior, and the governance context in which it applies. They are ordered from simplest to most sophisticated — and from least to most information produced.

### Last-Writer-Wins

The simplest possible policy: accept the most recent commit based on timestamp; discard the other.

- **Configuration**: $\mathcal{C}_{\text{active}} = \emptyset$. No constraints are checked. $\gamma$, $d_X$ are irrelevant.
- **Behavior**: $w = u$ or $w = v$ depending on which was committed later. No constraint evaluation, no conflict detection, no synthesis.
- **Governance context**: Individual-level discretion in low-stakes, exploratory work. Appropriate when the cost of an invalid state is low and contributors can fix problems manually.
- **Limitation**: No validity guarantee. The resulting state may violate any constraint — domain or optimization. No shadow prices, no conflict report, no traceability.

This is what most version control systems do by default for non-overlapping text changes. It works when the model has few constraints or when contributors are working on disjoint parts of the state.

### Source-Wins and Target-Wins

Priority policies: when conflicts exist, one branch's changes take precedence.

- **Configuration**: $\mathcal{C}_{\text{active}}$ may or may not be checked. A priority rule replaces the optimization: conflicts are resolved in favor of $u$ (source-wins) or $v$ (target-wins) without evaluating the intent loss.
- **Behavior**: $w = u \oplus v$ with conflicts resolved by dropping the lower-priority branch's conflicting changes.
- **Governance context**: Asymmetric workflows. Target-wins is natural when the main branch is authoritative and feature branches must conform. Source-wins applies when a feature branch has been reviewed and approved, and the target should absorb it wholesale.
- **Limitation**: Priority is blanket, not per-constraint. A source-wins policy drops *all* of the target's conflicting changes, even when some of them are more important than the source's. No shadow prices are produced because no optimization is solved.

### Union-with-Constraint-Check

Compose both commits; validate the result; escalate if invalid.

- **Configuration**: $\mathcal{C}_{\text{active}}$ is organization-defined. No optimization is performed ($\gamma$, $d_X$ unused). Escalation replaces synthesis.
- **Behavior**: Compute $w = u \oplus v$. Evaluate $C_{\text{active}}(f(X, w))$. If $\leq \mathbf{0}$, accept — this is the pass-through regime from §2. If any constraint is violated, report $\mathcal{C}_{\text{conflict}}$ and $\mathcal{R}_{\text{impacted}}$ and escalate to human review.
- **Governance context**: Teams that want automated merges where possible but refuse to produce invalid states. The organization defines $\mathcal{C}_{\text{active}}$; [[Continuous Integration|CI]] enforces the check; the team reviews escalated conflicts.
- **Tradeoff**: This policy produces conflict reports (which constraints failed, which requirements are impacted) but does not attempt resolution. It is a *diagnostic* policy — it tells you what's wrong without proposing a fix. The human receives the structured report and decides.

This is a natural starting point for organizations adopting constraint-aware merging. It requires only the [[Predicate Compliance Oracle]] — no optimizer — and it never produces an invalid state.

### Constraint-Aware Synthesis

The full optimization from §2: minimize intent loss subject to constraint satisfaction.

- **Configuration**: All parameters active — $\mathcal{C}_{\text{active}}$, $d_X$, $\gamma$, oracle configuration, automation boundary.
- **Behavior**: When $\mathcal{C}_{\text{conflict}} \neq \emptyset$, solve $w^* = \arg\min_{w} L_{\text{intent}}(w; u, v)$ subject to $C_{\text{active}}(f(X, w)) \leq \mathbf{0}$. Report shadow prices $\mu^*$, requirement prices $\nu$, and the full traceability chain.
- **Governance context**: High-stakes, constraint-rich models — safety-critical systems, regulated industries, large multi-team projects where the cost of an invalid merge is high and the cost of manual resolution is also high.
- **What it produces**: A candidate resolution $w^*$, a shadow price vector $\mu^*$ explaining which constraints shaped it, requirement prices $\nu$ tracing the impact to stakeholder concerns, and the full traceability chain from §2. A human reviewer receives the proposed resolution *with its explanation* and decides whether to accept, modify, or reject it.
- **Note**: This is the general case. All other policies in this section are degenerations or approximations — they arise from restricting $\theta$ (disabling the optimizer, emptying $\mathcal{C}_{\text{active}}$, replacing synthesis with escalation).

### Escalation-Only

Never auto-resolve conflicts; always defer to human judgment.

- **Configuration**: $\mathcal{C}_{\text{active}}$ is organization-defined. Synthesis is disabled — the automation boundary excludes resolution entirely.
- **Behavior**: If $\mathcal{C}_{\text{conflict}} \neq \emptyset$, reject the merge and report $(\mathcal{C}_{\text{conflict}}, \mathcal{R}_{\text{impacted}}, \lambda_{uv}, \lambda_{vu})$. No resolution commit is produced.
- **Governance context**: Safety-critical or regulatory environments where automated resolution is unacceptable — where the organization's policy is that every conflict requires human review, full stop.
- **What it produces**: The formalism still adds value even though it produces no resolution. The conflict report — which constraints are violated, which requirements are impacted, the violation magnitudes in both orderings — gives the human reviewer structured information rather than a raw diff. The reviewer knows *what* to look at and *why* it matters.

### The Governance Connection

These five policies are not merely technical options. They are **organizational decisions** about risk tolerance, automation scope, and the boundary between machine [[Verification and Validation|verification]] and human judgment.

The formalism makes these decisions explicit. Without it, the choice between last-writer-wins and escalation-only is implicit in code — buried in merge driver configuration, CI pipeline logic, and team conventions. With it, the choice is a parameter configuration $\theta$ with documented consequences: which constraints are checked, which shadow prices are produced, what information the human reviewer receives, and what validity guarantees hold.

The progression from last-writer-wins to constraint-aware synthesis is a progression in **information**: from no constraint evaluation, to pass/fail checks, to full shadow prices and traceability. Each step produces more information about the merge and requires more infrastructure (oracle evaluation, optimization, dual variable computation). The right policy for an organization depends on how much information it needs and what it is willing to invest in producing it.

---

## 7 Future Work

### Constraint Language and Oracle Implementation

The formalism defines $c_i : \mathcal{X} \to \mathbb{R}$ abstractly. Implementing the [[Predicate Compliance Oracle]] requires choosing concrete constraint languages: [[SPARQL]] ASK queries for referential integrity, SHACL shapes for schema conformance, OWL axioms for type reasoning, computation engines or external solvers for parametric constraints. The choice of language determines what the oracle can evaluate, at what computational cost, and with what granularity of violation reporting. A practical implementation will likely require a heterogeneous dispatch — different constraint types evaluated by different backends — and the design of that dispatch is itself a significant engineering problem.

### Three-Way Diff and Commit DAG

The formalism assumes the cross-application states $X_{uv}$ and $X_{vu}$ are computable. In [[Flexo MMS]], this requires three-way diff (ancestor vs. source, ancestor vs. target) and merge base identification in a commit directed acyclic graph (DAG). The current commit model uses a single-parent linked list; supporting [[Merge|merge commits]] — which require multiple parents — requires extending the data model to a DAG. The algorithm for finding a common ancestor and computing semantically meaningful three-way diffs over [[RDF]] graphs is not yet implemented.

### Delta Composition Semantics

Commit composition $u \oplus v$ is defined conceptually as "the intended union of changes." In Flexo, commits are [[Diff and Delta|SPARQL UPDATE patches]], and composing two arbitrary patches is non-trivial: order of operations matters, the net effect depends on which triples the patches share, and some higher-order operations (rename, move, retype) may be lost when reduced to triple-level insert/delete. Formalizing composition at the patch level — and determining when two patches commute — is an open problem that connects directly to the non-commutativity assumption in §2.

### Computational Tractability

The constrained optimization in §2 and §4 is stated abstractly. For practical use in interactive merge workflows, the feasible set must be characterizable, the intent loss $L_{\text{intent}}$ computable, and the solver efficient enough to return results within the latency expectations of a merge operation. Specific instantiations of $d_X$, $\gamma$, and the constraint functions will determine whether exact solutions, convex relaxations, or heuristic approximations are needed. The structure of the constraint functions — whether they are convex, separable, or have exploitable sparsity — will shape the choice of solver.

### Conflict Interaction and Partial Resolution

The formalism treats conflict resolution as a single optimization over all conflicting constraints simultaneously. In practice, resolving one conflict may create or exacerbate another — satisfying constraint $c_i$ may require violating $c_j$. Whether partial resolution is possible — some conflicts resolved automatically, others deferred to human review — while maintaining model consistency is an open question. The monotonicity property from §5 provides a starting point: removing a constraint from $\mathcal{C}_{\text{active}}$ can only expand the feasible set, never shrink it, so deferring a constraint is always safe in terms of feasibility.

### Multi-Party Merges

The formalism handles two commits $u$ and $v$ from a common ancestor. Extending to $n$-way merges — multiple branches converging simultaneously — requires generalizing the cross-application analysis (from two orderings to $n!$ permutations), the conflict detection (violations may appear only in specific orderings), and the optimization (the intent loss must account for $n$ contributors' intentions). The combinatorial explosion of orderings is the central challenge.

### Staleness and Dependency Tracking

§2 notes that stored behavioral constraint evaluations become defunct when upstream model elements change. Formalizing this dependency — which model elements a behavioral evaluation depends on, and when a change invalidates a stored result — is itself a constraint management problem. It connects to the broader question of incremental constraint evaluation: when the model state changes, which constraints need re-evaluation and which remain valid? An efficient invalidation mechanism would reduce the computational cost of the oracle and enable tighter integration with [[Continuous Integration]] workflows.

### Cross-Organization Federation

The formalism assumes a single repository with a single set of constraints and a single governance hierarchy. Federated model management — where different organizations maintain portions of a shared model — introduces coordination protocols beyond single-repository merge. Constraint evaluation may span organizational boundaries (a coupling constraint between components owned by different organizations), and the governance hierarchy (§5) must accommodate multiple organizations with potentially conflicting policies.

### Implementation Mapping

The [[Flexo Conflict Resolution Mapping]] connects the formalism developed here to concrete [[Flexo MMS]] operations — the merge endpoint, SPARQL infrastructure, [[Continuous Integration|CI]] pipelines, and the governance model. That document maps the *what* and *where*; the open problems above define what remains to be solved.

---

## 8 References

### Constrained Optimization and Duality

- Boyd, S. & Vandenberghe, L. *Convex Optimization*. Cambridge University Press, 2004. Chapters 5 (Duality) and 11 (Interior-point methods). Standard reference for Lagrange duality, shadow prices, complementary slackness, and KKT conditions. Freely available at [stanford.edu/~boyd/cvxbook](https://stanford.edu/~boyd/cvxbook/).

- Bertsekas, D.P. *Constrained Optimization and Lagrange Multiplier Methods*. Athena Scientific, 1996. Multiplier methods, sensitivity analysis, and the connection between dual variables and constraint prices.

- Nocedal, J. & Wright, S.J. *Numerical Optimization*. 2nd ed., Springer, 2006. Chapters 12–19 cover theory and algorithms for constrained optimization, including penalty methods, augmented Lagrangian, and sequential quadratic programming.

### Optimal Control and Dynamical Systems

- Bertsekas, D.P. *Dynamic Programming and Optimal Control*. 4th ed., Athena Scientific, 2017. The control-theoretic framing used throughout this document: state variables, control inputs, transition functions, cost functionals, and constraint handling in sequential decision problems.

- Zargham, M. & Shorish, J. "Generalized Dynamical Systems Part I: Foundations." Working paper, 2022. Available at [WU Vienna](https://research.wu.ac.at/ws/portalfiles/portal/23782375/Zargham_Shorish_GDS_Part_I_Foundations_2022.pdf). The GDS framework from which this document's notation and state-space formulation are drawn.

- Zhang, Z. et al. "On modeling blockchain-enabled economic networks as stochastic dynamical systems." *Applied Network Science* 5, 19 (2020). [doi:10.1007/s41109-020-0254-9](https://doi.org/10.1007/s41109-020-0254-9). Earlier application of the GDS framework to governance and policy in networked systems.

### Multidisciplinary Design Optimization

- Martins, J.R.R.A. & Lambe, A.B. "Multidisciplinary design optimization: A survey of architectures." *AIAA Journal* 51(9), 2049–2075, 2013. Survey of MDO architectures; the coupling constraints in §2 and §4 draw on this tradition of cross-discipline constraint management.

- Sobieszczanski-Sobieski, J. & Haftka, R.T. "Multidisciplinary aerospace design optimization: Survey of recent developments." *Structural Optimization* 14(1), 1–23, 1997. Foundational survey on optimization across engineering disciplines with coupled constraints.

### Software Merging and Version Control

- Mens, T. "A state-of-the-art survey on software merging." *IEEE Transactions on Software Engineering* 28(5), 449–462, 2002. Comprehensive survey of merge techniques (textual, syntactic, semantic); the conflict classification in §1 draws on this taxonomy.

- Westfechtel, B. "Structure-oriented merging of revisions of software documents." *Proceedings of the 3rd International Workshop on Software Configuration Management*, 68–79, 1991. Early work on structure-aware merging beyond line-level diff.

- Zhu, H. et al. "Conflict Resolution for Structured Merge via Version Space Algebra." *Proc. ACM Program. Lang.* 2 (OOPSLA), Article 166, 2018. [doi:10.1145/3276536](https://doi.org/10.1145/3276536). Formal framework for structured merge using version space algebra.

### Model Versioning

- Brosch, P. et al. "Towards Semantics-Aware Merge Support in Optimistic Model Versioning." In *Models in Software Engineering*, Springer, 2012. Semantic-aware merge for models, addressing the gap between textual and structural conflict detection.

- Cicchetti, A. et al. "Models in Conflict — Towards a Semantically Enhanced Version Control System for Models." In *Models in Software Engineering*, Springer, 2008. Model-level conflict detection beyond textual diff.

---
← [[Flexo MMS]] · [[Flexo Conflict Resolution Mapping]]

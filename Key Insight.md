# Key Insight

[[VCS|Version control systems]] conflate two distinct objects: the *record of change* (the patch, the diff, the delta) and the *resulting state* (the snapshot, the tree, the model graph). In text-based VCS this conflation is harmless — the merge algorithm operates on content alone, and if it produces output, that output *is* the merged state. For structured, constraint-laden [[Model|models]], the conflation breaks down. Whether a change is acceptable depends not only on what it changes but on the state it lands on, and evaluating that requires machinery that text merge does not need and cannot provide. This is why the formalism in the [[Conflict Resolution Problem Statement]] and the [[Flexo Conflict Resolution Mapping]] adopts a control-engineering convention: commits are *control inputs*, model states are *state variables*, and the state transition function connects them.

![[control-action-export.png]]

The diagram shows the state space of the conflict resolution problem. Green boxes are valid states ($C(X) \leq \mathbf{0}$); red boxes are constraint-violating states. From the common ancestor $X$, commits $u$ and $v$ each produce individually valid states — but the cross-application states $f(f(X,u),v)$ and $f(f(X,v),u)$ both violate constraints. The resolution commit $w$ (thick black arrow) produces a valid state $f(X,w)$ directly from the ancestor. The dashed arrows $u'$ and $v'$ are the commits that would carry each branch head to the resolution: $f(f(X,u), v') = f(X,w)$ and $f(f(X,v), u') = f(X,w)$. These two-hop paths are useful for checking intent preservation — comparing $v'$ against $v$ measures how much Bob's intent was modified to reach the resolution, and comparing $u'$ against $u$ measures the same for Alice. They provide an intuition source for  defining a distance metric $d_X(w; u,v)$.

---

## The VCS Convention

In Git, "commit" refers simultaneously to two things: the *diff* (what changed) and the *tree* (the resulting snapshot). (The [[VCS]] note develops a finer decomposition — action, artifact, and state — and explains why VCS collapses them.) `git show` prints the patch; `git checkout` materializes the tree. The commit object binds them together — the patch is derived from the tree difference, and the tree is derived by applying the patch to the parent. The two representations are duals of the same commit, and it rarely matters which one you think about.

[[Flexo MMS]] inherits this pattern. A commit is a [[Diff and Delta|SPARQL UPDATE patch]] *and* the named-graph snapshot it produces. The patch is procedural (a sequence of INSERT/DELETE operations); the snapshot is declarative (the set of [[RDF]] triples at that point in history). They are stored separately but understood as one thing: the commit.

This works because text [[Merge|merge]] is **stateless** in a specific sense: the merge algorithm takes three inputs (base, ours, theirs), operates on their content, and produces output. There is no need to evaluate the result against external predicates. If the three-way merge produces output without conflict markers, that output is accepted. The result is *self-certifying* — it needs no further validation.

---

## Why This Breaks for Structured Models

Three assumptions underpin the VCS convention. All three hold for text. None hold for models.

**1. Merge is a pure function of content.**

For text, the merge output depends only on the three input texts. For models, the *validity* of the output depends on constraints that span elements, subsystems, and disciplines — constraints that are not encoded in the diff itself. Two commits that each pass all checks individually can jointly violate a [[Conflict Classification|coupling constraint]]. Alice's motor upgrade satisfies the thermal budget on her branch; Bob's cooling redesign satisfies it on his; together, the higher-power motor overwhelms the redesigned cooling system. The merge algorithm cannot know this without evaluating the result against the constraint set.

**2. Application order doesn't matter (or matters only locally).**

For text, if two patches touch disjoint lines, order is irrelevant — the patches commute. For models, non-commutativity is the norm:

$$f(f(X, u), v) \neq f(f(X, v), u) \quad \text{in general}$$

This is not an edge case. It is the expected situation whenever commits touch elements connected by typed relationships or governed by shared constraints. The constraints that bind the result depend on the *full state*, not just the changed triples. A commit that is safe when applied to state $X$ may violate constraints when applied to $f(X, u)$, because $u$ changed something the commit's validity depends on.

**3. The result is self-certifying.**

For text, a "clean" merge (no conflict markers) is a valid merge. For models, a "clean" triple-level merge — one where no triple was modified by both branches — can still produce a state that violates schema constraints, referential integrity, parametric bounds, or behavioral properties. The result must be *evaluated* by the [[Predicate Compliance Oracle]], not just *produced* by the merge algorithm. This evaluation is what the [[Verification and Validation]] framework structures: computable predicates are checked mechanically; judgment-requiring predicates are surfaced for human review.

---

## Why CI Helps but Is Not Enough

The natural response to assumption 3 breaking is: add a [[Continuous Integration|CI]] pipeline. Run constraint checks on the merged state before accepting it. If a check fails, block the merge. This is correct — and it is part of the architecture the formalism defines. But it addresses only one of the three broken assumptions, and even there it provides less than what is needed.

CI patches the **self-certification gap**. A CI pipeline that evaluates $C(X_{\text{merged}}) \leq \mathbf{0}$ before accepting a merge ensures that invalid states are not silently accepted. This is valuable and necessary. But CI is a **binary gate**: it tells you *whether* the merged state violates constraints, not *why*, not *which commits are responsible*, and not *what to do about it*.

CI does not address **non-commutativity**. A pipeline that checks the merged state sees one candidate — whichever the merge algorithm produced. It does not evaluate both application orderings $f(f(X, u), v)$ and $f(f(X, v), u)$, so it cannot distinguish between a conflict that manifests in both orderings (a genuine coupling conflict) and one that manifests only in one (an ordering artifact). This distinction matters for resolution: the former requires modifying the commits; the latter may be resolvable by choosing the right application order.

CI does not address **resolution synthesis**. When a constraint check fails, CI blocks the merge — but it does not generate alternative resolutions, compute how much each candidate deviates from the contributors' intent, or identify which constraints are binding and which are slack. The engineer sees a red check and must figure out the rest manually. The formalism's contribution is precisely this structured information: shadow prices, intent loss, requirement attribution. Producing that information requires reasoning about commits *relative to* states — applying candidate commits to the current state, evaluating constraints on the results, and comparing alternatives. This is not something a pass/fail gate can do.

In short: CI is the enforcement mechanism for the verification-scope portion of conflict resolution, and the formalism depends on it. But CI operates on states. The commit/state separation is what gives CI the right states to check, and what makes CI's results interpretable as part of a resolution protocol rather than a bare rejection.

---

## The Separation

The formalism adopts a convention from control engineering and dynamical systems that keeps these concepts distinct:

- A **commit** $u \in \mathcal{U}$ is a *control input* — a description of intended state change. In Flexo, this is a SPARQL UPDATE patch.
- A **model state** $X \in \mathcal{X}$ is the *state variable* — the full configuration of the model at a point in history. In Flexo, this is the set of RDF triples in a snapshot's named graph.
- The **state transition function** $f : \mathcal{X} \times \mathcal{U} \to \mathcal{X}$ applies a commit to a state: $X^+ = f(X, u)$.

The critical consequence: **constraints are predicates on states, not on commits.** The constraint evaluation operator $C(X) \leq \mathbf{0}$ takes a model state and returns a compliance vector. A commit is not "valid" or "invalid" in isolation — it is valid *relative to the state it acts on*. The same commit $u$ may satisfy all constraints when applied to state $X_0$ and violate them when applied to $X_1 = f(X_0, v)$, because $v$ changed something that $u$'s validity depends on.

This separation is what makes the rest of the formalism expressible:

- **Conflict detection** becomes: evaluate $C$ on both cross-application states $f(f(X, u), v)$ and $f(f(X, v), u)$. Any constraint violated in either ordering is a conflict.
- **Resolution** becomes: find a commit $w$ such that $C(f(X, w)) \leq \mathbf{0}$ while minimizing deviation from the intent of $u$ and $v$ — a constrained optimization problem.
- **Shadow prices** become: the Lagrange multipliers $\mu^*$ measuring the sensitivity of the resolution to each constraint — which constraints shaped the outcome and why.
- **[[Policy]]** becomes: a parameterized human-machine protocol governing which constraints are active ($\mathcal{C}_{\text{active}}$), what automation is permitted, and where human judgment is required.

None of this is expressible if "commit" and "state" are the same object.

---

## Why This Matters

The separation is not a notational preference. It is the structural prerequisite for defining conflict resolution policies. Without it, you can detect *syntactic* conflicts — the same triple was modified by both branches — but you cannot detect *structural* or *semantic* conflicts, because those require evaluating the merged state against constraints that the diff alone does not carry. You cannot evaluate candidate resolutions, because there is no language for "apply this candidate commit to the current state and check constraints." And you cannot explain *why* a resolution deviates from the contributors' intent, because shadow prices are defined in terms of constraints on states, not in terms of patches on diffs.

The [[A Policy-based Approach to Model Lifecycle Management with Flexo|lifecycle diagrams]] adopt this convention throughout: $u$ and $v$ are commits, $X_0$ and $X_1$ are states, and the diagrams trace the flow from control input through state transition to constraint evaluation. The convention propagates from the mathematics through the API design — the merge endpoint receives commits and returns states (or conflict reports with constraint attribution) — to the governance framework, where policies are defined in terms of admissible states and the constraints that bound them.

---
← [[VCS]] · [[Flexo MMS]] · [[Model]] · [[Diff and Delta]] · [[Merge]] · [[Continuous Integration]] · [[Conflict Resolution Problem Statement]] · [[Flexo Conflict Resolution Mapping]] · [[Policy]]

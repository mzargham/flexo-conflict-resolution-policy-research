# Policy

A policy is a locally-imposed rule — a constraint asserted within a scope (organization, repository, team) that is not a universal property of the protocol or data standard.

## Properties

- **Scoped** — applies to a specific context, not global
- **Assertive** — treated as a rule within its scope, not merely suggestive
- **Tailorable** — can be adjusted to circumstances, unlike protocol specs
- **Composable** — multiple policies may apply simultaneously; their interaction must be well-defined

## Policies and V&V

A well-designed policy respects the [[Verification and Validation]] boundary:

- **Verification scope** (automate) — constraint checking, structural consistency enforcement, deterministic conflict resolution where the "right answer" is computable
- **Validation scope** (defer) — flag conflicts that require judgment, surface advisories to reviewers, block merges that touch validation-sensitive content

## Merge Conflict Resolution as Policy

The mathematical formalism being developed describes a *family* of merge conflict resolution policies. "Family" is key — there is no single universal resolution strategy. The right policy depends on:

- The type of model content (structural vs behavioral vs parametric)
- The organizational context (safety-critical vs exploratory)
- The team's workflow conventions
- The specific constraints that must hold

## Policy vs Rule vs Norm

- A *rule* (global): the [[SysML v2]] API spec — all compliant tools must conform
- A *norm* (conventional): "prefer rebase over merge" — teams may follow or not
- A *policy* (local rule): "merges to `main` in this repo must pass constraint set C" — enforced here, tailorable there

See [[Rules and Norms]] for the full spectrum.

## Forward

The formalism in [[Flexo Conflict Resolution Mapping]] defines policies as functions over model state that identify conflicts and produce resolutions, borrowing structure from optimal control theory.

---
← [[Verification and Validation]] · [[Rules and Norms]] · [[Continuous Integration]] · [[Flexo Conflict Resolution Mapping]] · [[Key Insight]]

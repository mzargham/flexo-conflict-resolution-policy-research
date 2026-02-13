# Continuous Integration

The practice of automatically verifying changes against a defined set of checks before they are accepted. CI operationalizes [[Verification and Validation|verification]] — the deterministically checkable portion of correctness.

## CI as Rule Enforcement

[[Rules and Norms|Rules]] are deterministic, mechanically checkable constraints. CI is the mechanism that enforces them. Norms — which require judgment — remain outside CI's scope.

A CI workflow is the runtime manifestation of a [[Policy]]: the policy defines *what* to check; CI defines *when* and *how*. When a policy is scoped to a repository or branch, CI is what gives it teeth.

## Across Operational Levels

| Level | CI mechanism |
|-------|-------------|
| [[Flexo Ecosystem Level\|Ecosystem]] | API conformance suites, interop test harnesses |
| [[Flexo Organization Level\|Organization]] | Branch protection rules, required status checks, access control gates |
| [[Flexo Team Level\|Team]] | Pre-merge constraint checks, required reviewers, merge policies |
| [[Flexo Individual Contributor Level\|Individual]] | Pre-commit hooks, commit format linting, local validation |

## In Flexo MMS

Two categories:

**Build/test CI** (exists today) — CircleCI pipelines and SonarCloud quality gates on [flexo-mms-layer1-service](https://github.com/Open-MBEE/flexo-mms-layer1-service). Standard software CI: compile, test, static analysis.

**Model-constraint CI** (forward) — [[SPARQL]] constraint queries run against model state before a merge is accepted. The merge endpoint (`POST /projects/{projectId}/branches/{targetBranchId}/merge`) is the natural gate. This is the policy-driven enforcement the [[Flexo Conflict Resolution Mapping|conflict resolution formalism]] will define — checking that a proposed merge preserves structural invariants (schema conformance, required relationships, type constraints) before it takes effect.

## CI and the V&V Boundary

CI must not silently resolve validation concerns. When a check touches something that requires judgment, the correct CI behavior is to block and surface an advisory — not to auto-resolve. A failing verification check produces a hard gate (merge blocked). A validation concern produces an advisory or flag (merge blocked pending human review, not pending a rerun).

This is the same principle from [[Verification and Validation]] and [[Policy]], applied to the workflow layer.

---
← [[Verification and Validation]] · [[Rules and Norms]] · [[Policy]] · [[Flexo Conflict Resolution Mapping]] · [[Flexo Team Level]] · [[Flexo Organization Level]] · [[Key Insight]]

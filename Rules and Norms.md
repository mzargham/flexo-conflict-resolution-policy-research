# Rules and Norms

How [[Verification and Validation]] concerns manifest as governance mechanisms across [[VCS|version control]] workflows.

## Rules

Deterministic, enforceable constraints. Compliance can be verified mechanically. Violations are blocked or rejected.

Examples: protocol specifications (what state transitions git commands produce), data format standards ([[RDF]] schema, [[SysML v2]] API), branch protection (require CI pass before merge), pre-commit hooks.

## Norms

Conventional practices that guide behavior. Compliance is evaluated by judgment (validated, not verified). Violations are noted in review, not blocked by automation.

Examples: branching strategies (GitFlow, trunk-based), "feature branches should be short-lived", review turnaround expectations, commit message style.

## The Spectrum Is Not Binary

Something can be a norm at one scope and a rule at another. A branching strategy is a norm across the ecosystem but may be enforced as a rule within an organization. This scope-dependent promotion is exactly what [[Policy]] captures.

## Across Operational Levels

| Level                                              | Rules (verifiable)                                              | Norms (require judgment)                                          |
| -------------------------------------------------- | --------------------------------------------------------------- | ----------------------------------------------------------------- |
| [[Flexo Ecosystem Level\|Ecosystem]]               | Protocol specs, data format standards, API conformance          | Best practices, interop conventions                               |
| [[Flexo Organization Level\|Organization]]         | Branch protection, access control, required checks              | Branching strategy, naming conventions                            |
| [[Flexo Team Level\|Team]]                         | Merge policies, required reviewers, pre-merge constraint checks | Workflow conventions, review expectations                         |
| [[Flexo Individual Contributor Level\|Individual]] | Pre-commit hooks, commit format enforcement                     | Engineering expertise applied to modeling, reviewing & validating |

The rules in this table are enforced at runtime by [[Continuous Integration]] workflows — CI is the mechanism that gives policies their teeth.

---
← [[Verification and Validation]] · [[Policy]] · [[Continuous Integration]] · [[Flexo Ecosystem Level]] · [[Flexo Organization Level]] · [[Flexo Team Level]] · [[Flexo Individual Contributor Level]]

# Verification and Validation

Core systems engineering distinction, scoped here to model version control.

## Verification

"Did we build it right?" — checking conformance against explicit, formal specifications. Deterministic: a predicate over state that returns true/false.

In model VCS context:
- Does the model conform to its metamodel schema?
- Are all required relationships present?
- Does the merge preserve type constraints?
- Do [[SPARQL]] constraint queries pass?

## Validation

"Did we build the right thing?" — evaluating fitness for purpose against stakeholder intent. Requires judgment; cannot be fully reduced to a predicate.

In model VCS context:
- Is this the right decomposition of the system?
- Does this interface definition serve the intended use case?
- Is the modeling approach sound for this analysis?

## Why the Separation Matters for Automation

Automated merge policies can handle verification — enforce constraint predicates, check structural consistency, reject malformed state transitions. But they must not silently override validation concerns. When a conflict touches something that requires judgment, the policy should surface it to humans rather than resolve it mechanically.

A [[Policy]]'s automated resolution scope is bounded to verification-like checks. Validation concerns produce advisories or blocks, not automated resolutions.

---
← [[Policy]] · [[Rules and Norms]] · [[Continuous Integration]]

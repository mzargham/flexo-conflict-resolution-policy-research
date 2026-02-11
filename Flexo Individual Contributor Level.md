# Flexo Individual Contributor Level

**Concern:** Engineer proposing, evaluating, or merging changes to model elements.

## API Surface ([[SysML v2]] PSM)

The [flexo-mms-sysmlv2](https://github.com/Open-MBEE/flexo-mms-sysmlv2) service implements the OMG SysML v2 API Services Specification.

| Operation | API Pattern | Use Case |
|-----------|-------------|----------|
| Read element | `GET /projects/{projectId}/commits/{commitId}/elements/{elementId}` | Review current state |
| Read elements | `GET /projects/{projectId}/commits/{commitId}/elements` | Browse model content |
| Create commit | `POST /projects/{projectId}/commits` | Propose changes |
| Get changes | `GET /projects/{projectId}/commits/{commitId}/changes` | Review what changed |
| Diff | `GET /projects/{projectId}/commits/{compareCommitId}/diff` | Compare two commits |
| Merge | `POST /projects/{projectId}/branches/{targetBranchId}/merge` | Integrate changes |
| Query | `GET /projects/{projectId}/query-results` | Ad-hoc model queries |

## Individual Contributor Workflow

1. **Pull** — Query elements from a branch to understand current state
2. **Edit** — Prepare changes (typically in a client tool that maps to API)
3. **Commit** — Submit changes via `POST /projects/{projectId}/commits`
4. **Review** — Use diff API to compare with target branch
5. **Merge** — Request merge to target branch

## Where Conflict Resolution Enters

When merging, if the target branch has advanced since the contributor's base commit, concurrent modifications must be reconciled. The diff/merge API surfaces this; the [[Flexo Conflict Resolution Mapping|conflict resolution formalism]] provides the constraint-aware resolution policy.

The merge endpoint (`POST /projects/{projectId}/branches/{targetBranchId}/merge`) is where the resolution policy would be invoked.

## References

- [flexo-mms-sysmlv2](https://github.com/Open-MBEE/flexo-mms-sysmlv2) — GitHub
- [Layer 1 OpenAPI spec](https://www.openmbee.org/flexo-mms-layer1-openapi/)

---
← [[Flexo MMS]]

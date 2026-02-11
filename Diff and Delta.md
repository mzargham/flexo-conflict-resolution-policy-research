# Diff and Delta

How Flexo MMS computes, represents, and stores differences between model states.

## Two Interfaces

Flexo exposes diff through two API layers, each with different semantics:

| Layer | Endpoint | Granularity | Format |
|-------|----------|-------------|--------|
| Layer 1 | `POST /orgs/{orgId}/repos/{repoId}/diff` | **Triple-level** | Insert graph + delete graph (named graphs of RDF triples) |
| [[SysML v2]] PSM | `GET /projects/{projectId}/commits/{compareCommitId}/diff?baseCommitId=` | **Element-level** | Array of `DataDifference` objects |

### Layer 1 Diff

The diff endpoint accepts two refs or commits (`srcRef`, `dstRef`) and computes the **set-theoretic symmetric difference** of their model graphs using a SPARQL UNION pattern:

- **Deletions** = triples in source graph but absent from destination graph
- **Insertions** = triples in destination graph but absent from source graph

The result is stored as two named graphs (insert graph and delete graph) identified by a SHA256 hash. Both refs must have materialized snapshots for the diff to be computed.

### SysML v2 Diff

The SysML v2 diff endpoint returns an array of `DataDifference` objects, each containing:

```
{
  "@type": "DataDifference",
  "baseData": DataVersion | null,
  "compareData": DataVersion | null
}
```

Interpretation:
- `baseData` null, `compareData` present → **element added**
- `baseData` present, `compareData` null → **element deleted**
- Both present → **element modified** (compare payloads)

This is an element-level view built on top of the triple-level diff.

**Implementation status**: The SysML v2 diff endpoint in `flexo-mms-sysmlv2` is currently auto-generated scaffolding — the actual computation would delegate to the Layer 1 service.

## Delta Storage (Commit Patches)

Each commit stores its delta as an RDF literal in the metadata graph:

- **Datatype**: `mms-datatype:SPARQL` (raw) or `mms-datatype:SPARQL-Gz` (gzip compressed)
- **Content**: A SPARQL UPDATE string representing the changes made in that commit
- **Compression**: Trie-based IRI prefix compression, with optional gzip for large patches
- **Size limit**: Patches exceeding `maximumLiteralSizeKib` store a placeholder instead

The delta is **procedural** — a sequence of SPARQL INSERT/DELETE operations — rather than a declarative set of added/removed triples. This distinction matters: the same net change can be expressed by different SPARQL UPDATE sequences.

## Snapshot Reconstruction from Deltas

When a snapshot is needed for a commit that is not materialized (see [[Quadstore#Snapshot Lifecycle]]):

1. Traverse commit ancestry to find the nearest ancestor with a materialized snapshot
2. Collect delta patches along the path
3. Decompress patches (supporting gzip via MiGz and raw SPARQL)
4. Apply patches sequentially to reconstruct the model graph

This is a resource-performance tradeoff: only ref-head snapshots are kept materialized; historical states are recoverable by replaying deltas.

## Open Questions

- **Triple-level vs element-level**: The Layer 1 diff is a flat set difference of triples. How should this be lifted to a meaningful element-level or property-level diff for conflict resolution? A single element modification may involve many triple changes.
- **Patch ordering**: Deltas are SPARQL UPDATE sequences. Is the order of operations within a patch semantically significant, or is the net effect (set of inserts and deletes) sufficient?
- **Patch vs diff**: A commit's stored delta is the forward patch from parent to child. The diff endpoint computes the symmetric difference between any two states. These are related but not identical — the patch is directional; the diff is symmetric. The formalism needs to clarify which representation it operates on.
- **Completeness**: Can all model changes be captured as triple-level INSERT/DELETE? Or are there higher-order operations (rename, move, retype) that are lost when reduced to triple changes?

## References

- [flexo-mms-layer1-service](https://github.com/Open-MBEE/flexo-mms-layer1-service) — `DiffCreate.kt`, `ModelCommit.kt`, `Model.kt`
- [flexo-mms-sysmlv2 OpenAPI](https://github.com/Open-MBEE/flexo-mms-sysmlv2) — `DataDifference` schema
- [Layer 1 OpenAPI spec](https://www.openmbee.org/flexo-mms-layer1-openapi/)

---
← [[Model]] · [[Merge]] · [[Quadstore]] · [[Flexo Conflict Resolution Mapping]]

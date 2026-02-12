# SPARQL

SPARQL Protocol and RDF Query Language — the [W3C standard](https://www.w3.org/TR/sparql11-overview/) query and update language for [[RDF]] data.

## In Flexo MMS

The Layer 1 service requires a **SPARQL 1.1 compliant** [[quadstore]] as a backend ([Layer 1 docs](https://flexo-mms-deployment-guide.readthedocs.io/en/latest/flexo-mms-layer1-service/index.html)). The reference stack uses [[Apache Jena Fuseki]], which exposes three HTTP endpoints ([flexo-mms-layer1-service](https://github.com/Open-MBEE/flexo-mms-layer1-service)):

| Endpoint | Fuseki path | Flexo env var |
|----------|-------------|---------------|
| SPARQL Query | `/ds/sparql` | `FLEXO_MMS_QUERY_URL` |
| SPARQL Update | `/ds/update` | `FLEXO_MMS_UPDATE_URL` |
| [[Graph Store Protocol]] | `/ds/data` | `FLEXO_MMS_GRAPH_STORE_PROTOCOL_URL` |

For production deployments, read and write operations can be separated to distinct backend nodes using `FLEXO_MMS_MASTER_QUERY_URL` for strict consistency reads ([Layer 1 docs](https://flexo-mms-deployment-guide.readthedocs.io/en/latest/flexo-mms-layer1-service/index.html)).

## Cross-Commit Queries

Named graphs enable queries scoped to a specific commit snapshot:

```sparql
SELECT ?element ?name
WHERE {
    GRAPH <urn:mms:snapshot:commit-abc123> {
        ?element a sysml:PartDefinition ;
                 sysml:name ?name .
    }
}
```

They also enable cross-commit comparisons — effectively querying the diff between two model states:

```sparql
# Elements present in the new commit but absent from the old
SELECT ?element
WHERE {
    GRAPH <urn:mms:snapshot:commit-new> {
        ?element a sysml:Element .
    }
    FILTER NOT EXISTS {
        GRAPH <urn:mms:snapshot:commit-old> {
            ?element a sysml:Element .
        }
    }
}
```

This pattern complements the [[Diff and Delta|diff service]] — the diff service computes and stores deltas as SPARQL UPDATE patches, while cross-commit queries allow ad-hoc comparison without pre-computed deltas.

---
← [[Flexo Organization Level]] · [[RDF]]

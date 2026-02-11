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

---
← [[Flexo Organization Level]] · [[RDF]]

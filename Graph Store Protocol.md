# Graph Store Protocol

An HTTP-based protocol ([W3C spec](https://www.w3.org/TR/sparql11-http-rdf-update/)) for managing named graphs in an [[RDF]] store via standard CRUD operations — `GET`, `PUT`, `POST`, `DELETE` on graph URIs.

## In Flexo MMS

Flexo exposes a GSP endpoint alongside its [[SPARQL]] query and update endpoints. This enables direct graph-level operations (create, replace, delete entire named graphs) without writing SPARQL ([Architecture](https://openmbee.atlassian.net/wiki/spaces/OPENMBEE/pages/320765953/Flexo-MMS+Architecture)).

- Fuseki path: `/ds/data`
- Env var: `FLEXO_MMS_GRAPH_STORE_PROTOCOL_URL`
- Used during cluster initialization to insert the `.trig` bootstrap data into the [[quadstore]] ([Layer 1 docs](https://flexo-mms-deployment-guide.readthedocs.io/en/latest/flexo-mms-layer1-service/index.html))

---
← [[Flexo MMS]] · [[SPARQL]]

# Standards Reference

Standards and specifications that Flexo MMS depends on or implements.

| Standard | Purpose | Maintained by | Link |
|----------|---------|---------------|------|
| [[RDF]] | Data model (subject-predicate-object triples) | W3C | [RDF 1.1 Concepts](https://www.w3.org/TR/rdf11-concepts/) |
| [[SPARQL]] | Query and update language for RDF | W3C | [SPARQL 1.1 Overview](https://www.w3.org/TR/sparql11-overview/) |
| [[Graph Store Protocol]] | HTTP CRUD for named graphs | W3C | [SPARQL 1.1 Graph Store HTTP Protocol](https://www.w3.org/TR/sparql11-http-rdf-update/) |
| [[Linked Data Platform]] | RESTful read/write of RDF resources | W3C | [LDP 1.0](https://www.w3.org/TR/ldp/) |
| [[GraphQL]] | Client-specified query shape for APIs | GraphQL Foundation (Linux Foundation) | [spec.graphql.org](https://spec.graphql.org/) |
| [[OSLC]] | Cross-tool lifecycle integration | OASIS | [open-services.net](https://open-services.net/specifications/) |
| [[SysML v2]] | Systems modeling language and API | OMG | [SysML v2 spec](https://www.omg.org/sysml/sysmlv2/), [API spec](https://www.omg.org/spec/SystemsModelingAPI/) |

## Implementation

| Component | Role | Link |
|-----------|------|------|
| [[Apache Jena Fuseki]] | Reference [[quadstore]] backend | [jena.apache.org](https://jena.apache.org/documentation/fuseki2/) |
| OpenAPI | API description for Layer 1 CRUD and [[SysML v2]] service | [openapis.org](https://www.openapis.org/) |
| Docker / Docker Compose | Containerized deployment | [docs.docker.com](https://docs.docker.com/compose/) |
| LDAP | Enterprise authentication (auth service) | [RFC 4511](https://datatracker.ietf.org/doc/html/rfc4511) |
| S3 API | Object storage for large RDF uploads (store service) | [AWS S3 API](https://docs.aws.amazon.com/AmazonS3/latest/API/) |
| JWT | Token-based authentication | [RFC 7519](https://datatracker.ietf.org/doc/html/rfc7519) |
| TriG | [[RDF]] named-graph serialization for bootstrap data | [W3C TriG](https://www.w3.org/TR/trig/) |

---
← [[Flexo MMS]]

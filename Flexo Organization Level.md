# Flexo Organization Level

**Concern:** IT/platform team maintaining Flexo for the enterprise.

## Flexo Hierarchy

```
Deployment (Cluster)
└── Org(s)                    ← Administrative boundary
    └── Repo(s)               ← Version-controlled container
        └── Collection(s)     ← Logical grouping of model content
            └── Branch(es)    ← Mutable ref to commit lineage
            └── Tag(s)        ← Immutable ref to specific commit
            └── Lock(s)       ← Snapshot materialization control
```

Flexo-MMS exposes CRUD endpoints for management of [[RDF]] graphs (creating Orgs, Repos, Branches, etc.) ([Architecture](https://openmbee.atlassian.net/wiki/spaces/OPENMBEE/pages/320765953/Flexo-MMS+Architecture)). See also the [Layer 1 OpenAPI spec](https://www.openmbee.org/flexo-mms-layer1-openapi/).

## Key Entities

| Entity         | Description                                                                                                                     |
| -------------- | ------------------------------------------------------------------------------------------------------------------------------- |
| **Org**        | Administrative container for access control and governance. Maps to organizational units (business units, programs, divisions). |
| **Repo**       | A version-controlled container for related model content.                                                                       |
| **Collection** | Logical partitioning within a repo (e.g., by subsystem, by discipline).                                                         |

Partitions delimit responsibility for model assets but do not imply an absence of inter-collection dependencies.

## Deployment Infrastructure

The Layer 1 service requires a [[SPARQL]] 1.1 compliant [[quadstore]] as a backend. The reference stack uses [[Apache Jena Fuseki]], which binds locally on port 3030 ([flexo-mms-layer1-service](https://github.com/Open-MBEE/flexo-mms-layer1-service)).

Complementary services include an [auth service](https://github.com/Open-MBEE/flexo-mms-auth-service) for LDAP-based authentication and a [store service](https://github.com/Open-MBEE/flexo-mms-store-service) for large RDF data uploads to S3-compliant storage.

Flexo MMS stores all of its state information in the quadstore alongside user data ([flexo-mms-layer1-service](https://github.com/Open-MBEE/flexo-mms-layer1-service)).

Deployment examples (Docker Compose and k8s) are available in the [flexo-mms-deployment](https://github.com/Open-MBEE/flexo-mms-deployment) repo.

## IT Operations Responsibilities

- Quadstore provisioning and backup
- Cluster initialization with deployment-specific configuration (see [deployment guide](https://flexo-mms-deployment-guide.readthedocs.io/en/latest/flexo-mms-layer1-service/index.html))
- Access control policy configuration
- Performance tuning (snapshot retention, query optimization, read/write endpoint separation)

## References

- [Flexo-MMS Architecture](https://openmbee.atlassian.net/wiki/spaces/OPENMBEE/pages/320765953/Flexo-MMS+Architecture) — OpenMBEE Confluence
- [flexo-mms-layer1-service](https://github.com/Open-MBEE/flexo-mms-layer1-service) — GitHub
- [Layer 1 Service docs](https://flexo-mms-deployment-guide.readthedocs.io/en/latest/flexo-mms-layer1-service/index.html) — ReadTheDocs
- [flexo-mms-deployment](https://github.com/Open-MBEE/flexo-mms-deployment) — GitHub
- [Layer 1 OpenAPI spec](https://www.openmbee.org/flexo-mms-layer1-openapi/)

---
← [[Flexo MMS]]

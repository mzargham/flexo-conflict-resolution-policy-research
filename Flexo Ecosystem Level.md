# Flexo Ecosystem Level

**Concern:** Organizations sharing model assets across enterprise boundaries.

## Standards-Based Interoperability

The guiding principle for Flexo modules is to adopt open, web-compatible standards for data exchange at every interface — both data format and exchange protocols. [[SPARQL]], [[Graph Store Protocol]], [[Linked Data Platform]], [[GraphQL]], [[OSLC]], [[SysML v2]], JSON HyperSchema, gRPC, and so on, are all web-friendly protocols with broad tooling support in their respective ecosystems ([Architecture](https://openmbee.atlassian.net/wiki/spaces/OPENMBEE/pages/320765953/Flexo-MMS+Architecture)).

The [[SysML v2]] API layer conforms to the OMG standard, enabling interoperability with any compliant tooling ([flexo-mms-sysmlv2](https://github.com/Open-MBEE/flexo-mms-sysmlv2)).

Models in Flexo are not limited to enterprise model development platforms — Flexo can store anything expressed as [[RDF]] ([openmbee.org](https://www.openmbee.org/flexo.html)).

## Operational Considerations

- Each organization typically runs its own Flexo deployment (no federated multi-tenant architecture in the current release)
- Asset sharing occurs via API exchange (export/import) or [[OSLC]] linking
- Standards compliance (OMG [[SysML v2]] API) enables tool interoperability across organizational boundaries

## Gap

The [[Flexo Conflict Resolution Mapping|conflict resolution formalism]] assumes a single repository; cross-organization federation would require additional coordination protocols.

## References

- [Flexo-MMS Architecture](https://openmbee.atlassian.net/wiki/spaces/OPENMBEE/pages/320765953/Flexo-MMS+Architecture) — OpenMBEE Confluence
- [Flexo MMS and SysMLv2](https://www.openmbee.org/flexo.html) — openmbee.org
- [flexo-mms-sysmlv2](https://github.com/Open-MBEE/flexo-mms-sysmlv2) — GitHub

---
← [[Flexo MMS]]

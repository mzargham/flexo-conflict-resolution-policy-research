# Conflict Resolution Policy Research for Flexo MMS

Research into formalizing merge conflict resolution for [Flexo MMS](https://github.com/Open-MBEE/flexo-mms-layer1-service), a graph-native version control system for structured data from [OpenMBEE](https://www.openmbee.org/).

## About

Flexo MMS stores models as RDF graphs and versions them through SPARQL Update patches. It supports branching and diffing but does not yet implement merge conflict resolution. This repository develops a formal problem statement for conflict resolution, framing it as a constrained optimization problem with Lagrange duality — defining a family of resolution policies parameterized by organizational context, constraint priorities, and the boundary between automated verification and human judgment.

## Usage

This repository is an [Obsidian](https://obsidian.md/) vault. To read the content with full wikilink navigation:

1. Clone the repository:
   ```sh
   git clone https://github.com/mzargham/flexo-conflict-resolution-policy-research.git
   ```
2. Open Obsidian and select **Open folder as vault**
3. Navigate to the cloned directory
4. When prompted, **enable community plugins** — the vault includes the [Diagram Popup](https://github.com/ChenPengyuan/obsidian-mermaid-popup) plugin, which lets you click any Mermaid diagram to open it in a draggable, zoomable popup

## Where to Start

| If you want to…                                                     | Start here                                                                              |
| ------------------------------------------------------------------- | --------------------------------------------------------------------------------------- |
| Get an overview of Flexo and navigate the vault                     | [Flexo MMS](Flexo%20MMS.md)                                                             |
| Understand the conflict resolution approach without the math        | [Flexo Conflict Resolution Mapping](Flexo%20Conflict%20Resolution%20Mapping.md)         |
| Read the full mathematical formalism                                | [Conflict Resolution Problem Statement](Conflict%20Resolution%20Problem%20Statement.md) |
| Understand Flexo's architecture and data model                      | [RDF](RDF.md) → [Quadstore](Quadstore.md) → [SPARQL](SPARQL.md)                         |
| Understand the governance and policy framework                      | [Verification and Validation](Verification%20and%20Validation.md) → [Policy](Policy.md) |
| See what models are, what merge looks like today and what's missing | [Model](Model.md) → [Merge](Merge.md)                                                     |
| See the lifecycle in action with sequence diagrams                  | [A Policy-based Approach to Model Lifecycle Management with Flexo](A%20Policy-based%20Approach%20to%20Model%20Lifecycle%20Management%20with%20Flexo.md) |

## Links

- [OpenMBEE](https://github.com/open-mbee) — open model-based engineering environment
- [Flexo MMS](https://github.com/Open-MBEE/flexo-mms-layer1-service) — core Layer 1 service
- [Flexo MMS deployment guide](https://flexo-mms-deployment-guide.readthedocs.io/en/latest/)
- [openmbee.org/flexo](https://www.openmbee.org/flexo.html)

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

The entry point is **Flexo MMS.md**.

## Links

- [OpenMBEE](https://github.com/open-mbee) — open model-based engineering environment
- [Flexo MMS](https://github.com/Open-MBEE/flexo-mms-layer1-service) — core Layer 1 service
- [Flexo MMS deployment guide](https://flexo-mms-deployment-guide.readthedocs.io/en/latest/)
- [openmbee.org/flexo](https://www.openmbee.org/flexo.html)

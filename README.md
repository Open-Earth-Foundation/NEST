# NEST: An Adaptive Intelligence Framework for Nested Energy System Transitions

NEST is a research initiative in frontier AI and energy-system modelling. It
studies nested AI architectures: systems in which AI agents coordinate
existing models that work at different scales, so that a national
energy-system model and a city building-stock model produce one coherent
picture of a transition instead of two separate ones. Large language models
orchestrate the exchange between models, and Active Inference provides the
probabilistic layer that lets the system hold an uncertain, updatable model
of the world it is reasoning about, across scales, rather than a set of
disconnected outputs.

A central concern is how cities are represented in energy and climate
transition modelling. Cities hold over half the world's population and most
of its emissions, yet they remain underrepresented in national and global
scenarios. Linking urban and national models so that each informs the other
is NEST's main area of focus and its first application.

The work runs in four stages. The first automates the coupling between an
existing national model and an existing urban building-stock model, using a
pair that has already been linked by hand as the baseline. The second adds
the Active Inference layer, compressing model outputs into a few
interpretable variables with explicit uncertainty. The third extends the
approach to further urban models and sectors, maintained by different
institutions and written in different languages, together with a shared
ontology of policies and actions. The fourth brings the results to planners,
city teams, utilities and financiers in pilot regions, to test whether they
support real decisions.

NEST is led by the Open Earth Foundation with the International Institute for
Applied Systems Analysis, the Active Inference Institute, Central European
University, Imperial College London, C40 Cities, ICLEI, and the Collaborative
for Systemic Climate Action. It is funded by the Decarbonization and Energy
Virtual Institute (DEVI) at Schmidt Sciences.

## What this repository is

This is NEST's public repository. It holds the material the consortium
considers ready for others to read, question and build on: design notes,
specifications, event material, and in time the software for joining models.

The consortium's day-to-day work happens in a private workspace, because most
of it is experimental and changes quickly. Things move here when their owner
judges them stable enough to be useful to others. Each document carries a
status line. "Draft" means exactly that: open for comment, not a commitment.

## Status

NEST started in 2026 and is at an early stage. The first experiments in
agent-driven model coupling run in late 2026. Software for joining models
will be published here as it becomes usable.

## Taking part

- Questions and ideas: open a Discussion.
- Corrections and additions: open a pull request. See CONTRIBUTING.md.
- Partners who maintain their own material here are listed in CODEOWNERS.

## Licence

Code is licensed under Apache-2.0 and documents under CC BY 4.0 unless a
folder states otherwise. See LICENSE and LICENSE-docs.

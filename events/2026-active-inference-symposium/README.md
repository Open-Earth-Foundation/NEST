Status: Draft

# NEST at the 6th Applied Active Inference Symposium, 12 to 13 November 2026

This folder is maintained by the Active Inference Institute (AII). It holds the material behind NEST's talk at the symposium, which is online.

The talk is a 20-minute pre-recorded presentation with three speakers in sequence: John H. Clippinger, Holly Grimm and Daniel Ari Friedman, all of the Active Inference Institute. Its title is "Coupling energy-system models with a seam you can read and an agent you can audit: Active Inference in the NEST programme". The speakers present work of the NEST consortium, led by Open Earth Foundation with IIASA, CEU, Imperial College London, C40, ICLEI and SCAC.

## What the folder holds

Three folders from the consortium's private workspace, copied at the commit named in each file. The copies are unchanged apart from the status line, the source line and some values from IIASA's model package and its guide. Those values are marked `[withheld]` and keep their units. Every document starts with a status line and a source line. The data files (schema and example files) carry no lines of their own, so their source commits are listed here.

| Folder | What it is | Source commit on the workspace main branch |
|---|---|---|
| `seam-record/` | The declared boundary between IIASA's national energy model (MESSAGEix) and its building-stock model (STURM) for Western Europe. The seam record is written once for the pair. Each model also has a Markov blanket written in GNN (Generalized Notation Notation) and a companion boundary file. The GNN file declares the structure: each variable's name, size, type and role (sensory, active or internal). The boundary file gives each exchanged variable its unit, convention, temporal and spatial support, mapping rule and owner. | seam record `bb3d5727`, the two GNN blankets `e56ee114`, MESSAGEix boundary file `6c14c736`, STURM boundary file `d0977625` |
| `agent-trace-contract/` | The contract that the orchestrator's exported agent trace must satisfy, with its hash policy (`tool-outcome/v1`) and the test vectors an independent reader uses to check the hashes. The `v0` folder is the earlier version with its examples. | README, `v0.1/schema.json` and the hash vectors `1d10d92c`. `v0/schema.json` and `v0/examples/invalid-records.jsonl` `5f691e84`. `v0/examples/coupling-loop-example.jsonl` `bd4243dd` |
| `calibration-protocol/` | The simulation-based calibration protocol for the Active Inference layer and the record of its first synthetic run, pre-registered before the run. | `e6923581` |

Full commit identifiers are in the source line of each document. The table shortens them to eight characters.

## What this material is and is not

It is a set of records and a protocol. It is not a software package that you can install or run. The software that joins the models stays in the private workspace until it is stable, as the repository README says.

The seam record describes the boundary between two models that IIASA built and maintains. It contains no IIASA model data and no model code. Some fields are marked as pending the owner's answer, and the record says so.

The GNN blankets declare structure, not values. The sizes in brackets (for example `d[2,13,type=float]`) are illustrative. The 13 periods are the model years of IIASA's public linkage runs, 2020–2100. The Western Europe cutout that the boundary files name for experiment 1 declares 12 model years, 2030–2110. The roles are the declaration. The units, conventions and sources are in the boundary files.

The trace contract describes what an exported agent trace must contain and how its hashes are computed. It contains no orchestrator code and no exported trace. An independent reader can check an export against it. Passing those checks verifies the record. It does not validate the energy-model computation behind it.

The calibration protocol and its first run use synthetic data from a known generating process. They show that the inference harness is calibrated when every parameter is known. They do not show calibrated inference on real model outputs. A later run with parameter uncertainty, recorded in the workspace, missed its calibration gates. That result is reported in the talk and is not in this folder.

Links inside the copied documents that point into the private workspace do not resolve here.

## Slides

The `slides/` folder holds the three speakers' slides as PDF files, in the order of the talk. Each file is an export of the slides used in the recording, made on 7 November 2026.

| File | Speaker | Part |
|---|---|---|
| `slides/1-clippinger-a-vision-for-nested-modeling.pdf` | John H. Clippinger | Title and "A Vision for Resolving Nested Multi-Scale Modeling", 9 slides |
| `slides/2-grimm-the-seam-and-the-audit.pdf` | Holly Grimm | "The seam and the audit", 13 slides |
| `slides/3-friedman-active-inference-at-the-model-boundary.pdf` | Daniel Ari Friedman | "Active Inference at the model boundary", 8 slides |

The slides show units, structure, counts and ratios. They show no values from IIASA's model package.

## To come

The link to the recording will be added after the symposium.

## Licence

Documents in this folder are licensed under CC BY 4.0. The schema and example files are licensed under Apache-2.0. See the repository's LICENSE and LICENSE-docs.

## Contact

Holly Grimm, holly@activeinference.institute

# ORBI Swiftlet Lab

Research and compatibility laboratory for studying the upstream [Swiftlet](https://github.com/leonickson1/Swiftlet) expert-streaming runtime and extracting portable design lessons for ORBI.

## Purpose

This repository is **not** the production ORBI runtime. It is the controlled upstream/audit workspace used to:

- freeze and track a known Swiftlet revision;
- audit container, cache, I/O, model-graph, correctness and GPU-runtime behavior;
- separate portable mechanisms from Apple/Metal-specific implementation;
- produce reproducible compatibility notes for Windows and Android;
- preserve upstream attribution and license obligations.

## Upstream freeze

- Repository: `leonickson1/Swiftlet`
- Branch: `main`
- Frozen commit: `909c04213c9deb369dac0679d0872512cf3ab32e`
- Frozen upstream commit date: 2026-09-14
- Upstream license: Apache-2.0

No model weights are stored in this repository.

## ORBI gates

- **OSM-00** — upstream freeze and provenance
- **OSM-01** — architecture portability audit
- **OSM-02** — qpack compatibility contract
- **OSM-03** — portable CPU reference feasibility
- **OSM-04** — expert streaming/cache portability
- **OSM-05+** — implementation work moves to `ingeniusvictor/orbi-streammoe`

## Rule

Upstream code is treated as a reference and compatibility target. ORBI-specific production implementation belongs in `orbi-streammoe`.

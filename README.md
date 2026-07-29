# Luimneach-V
Luimneach RISC-V core

Copyright (c) 2026 Weqaar Janjua

Luimneach (/ˈlɪmnəx/) is the Irish-language name of the city of Limerick.

This work is developed at the University of Limerick, Ireland, as part of a
Doctorate in Engineering (DEng) programme under the supervision
of Dr. Eoin O'Connell and the co-supervision of Dr. Mihai Penica.

Luimneach-V is open-source academic research, released under permissive
licences (see LICENSE for the Apache-2.0 terms covering hardware RTL and
software; documentation and the accompanying textbook are covered by
CC-BY-4.0). It is not a product of any commercial organisation.

## Licensing and Attribution

Luimneach-V is licensed under the Apache License 2.0 (see [`LICENSE`](LICENSE)
for code, [`LICENSE-DOCS`](LICENSE-DOCS) for documentation under CC-BY-4.0).

Per Apache 2.0 §4, redistributions must reproduce the contents of
[`NOTICE`](NOTICE) in any product or publication that incorporates this work.

## Project overview

*A small, in-order, non-speculative, cache-free RV64IMC soft-core architecture for deterministic embedded use and FPGA implementation on the Microchip PolarFire SoC FPGA Icicle Kit.*

The short form **LV** is used in code, figures, and abbreviated references; the full name is *Luimneach RISC-V Core*.

Luimneach-V is a small, open-source 64-bit RISC-V soft processor core and one of the research outputs of the DEng programme.

Its distinguishing property is that it occupies an under-served point in the open-core design space: a 64-bit core that is deterministic, cache-free, and simple, rather than a small RV32 controller or a large speculative application processor.

Three design goals, taken together not separately:

1. **64-bit, modern toolchain fit.** RV64IMC matches the register width and ABI of mainstream RISC-V toolchains and RTOS distributions, so standard 64-bit artefacts run on the core without translation.
2. **Deterministic, analysable timing.** The pipeline is strictly in-order, with no branch prediction, no reorder buffer, and no speculative load path. Caches are absent; memory is tightly coupled (TCM-style). Each instruction class has a documented cycle bound, which makes worst-case execution time tractable to analyse.
3. **Open, vendor-independent, teachable.** The RTL targets an open-source release and vendor-independent FPGA flows. The teaching plan links the core design to a textbook and twelve practical labs suitable for undergraduate or master's-level study.

The core also reserves two encodings in the `custom-0` opcode space for future application-specific acceleration; their semantics are deliberately left unspecified here and are the subject of a forthcoming companion paper.

## Status

**Doctoral MVP, in development (Year 3 of 5).**

The MVP scope is the smallest version of the core that supports the research hypothesis it was built to test. Anything beyond the MVP is explicitly out of doctoral scope and is documented in [`plans/ROADMAP.md`](plans/ROADMAP.md) as post-doctoral work.

## ISA profile

| Component | Choice |
|---|---|
| Base | **RV64I** |
| Multiply / divide | **M** |
| Compressed | **C** |
| CSR access | **Zicsr** |
| Instruction fence | **Zifencei** |
| Privilege levels | **M-mode + U-mode** (no S-mode, no MMU, no virtual memory) |
| PMP regions | **4** |
| Atomics, FPU, vector, hypervisor | **omitted** (out of doctoral scope) |
| Pipeline | **Single-cycle** for the doctoral MVP |
| Custom instructions | two reserved `custom-0` slots (semantics in a forthcoming companion paper) |
| Caches | **none** (TCM-style direct interface to tightly-coupled memory) |
| Branch prediction | **none** |

## Proposed Repository layout

```
luimneach-rv/
├── README.md                  this file
├── LICENSE                    Apache-2.0 (code, RTL, software)
├── LICENSE-DOCS               CC-BY-4.0 (documentation, textbook, labs)
├── NOTICE                     Apache-2.0 attribution notice
├── docs/
│   ├── book/                  the textbook (Designing Luimneach-V)
│   ├── architecture/          ISA + microarchitecture specifications
│   ├── data-sheet/            timing, resource utilisation, pinouts
│   ├── publications/          publication manuscripts and review copies
│   └── HLD/                   high-level design document
├── rtl/                       synthesisable RTL (Verilog / SystemVerilog)
├── verification/              unit, ISA-compliance, reference-iss, formal
├── labs/                      graded teaching labs (lab-01 to lab-12)
├── fpga/polarfire-icicle/     Libero project, constraints, bitstreams
├── software/                  toolchain, BSP, Zephyr port, examples
├── renode/                    Renode platform definition
├── ci/                        continuous integration scripts
└── plans/                     ROADMAP, PEDAGOGY-NOTES, PUBLICATION-PLAN
```

## Where to start

| If you are | Start here |
|---|---|
| Understanding why the core exists | [`docs/origins.md`](docs/origins.md) |
| Reading the design proposal | [`docs/rfc/0001-luimneach-v-design-proposal.md`](docs/rfc/0001-luimneach-v-design-proposal.md) |

## Author and supervision

- **Candidate** - Weqaar Janjua, `janjua.weqaar@ul.ie`
- **Supervisor** - Dr. Eoin O'Connell, `eoin.oconnell@ul.ie`
- **Co-Supervisor** - Dr. Mihai Penica, `Mihai.Penica@ul.ie`
- **Programme** - Doctorate in Engineering (DEng), University of Limerick

## How to cite

> Janjua, W. (forthcoming). *Luimneach-V, A Small Deterministic RV64IMC Soft Core.* Dept. of Electronic and Computer Engineering, University of Limerick.

Accompanying paper (TODO) - *A Small Deterministic RV64IMC Soft Core, Design, Verification, and FPGA Implementation.*

# Luimneach-V, Origins

> *Why this core exists, how we arrived at its shape, and what we set out to build with it.*

## What Luimneach-V is

Luimneach-V (LV) is a small, open-source 64-bit RISC-V soft processor core, designed and verified as part of a Doctorate in Engineering (DEng) programme at the University of Limerick, Ireland. It targets the Microchip PolarFire SoC FPGA Icicle Kit (MPFS250T-FCVG484EES). The core's distinguishing property is that its microarchitecture is *in-order, non-speculative, and cache-free by construction*, which makes its instruction timing fully deterministic and its worst-case execution time straightforward to analyse. These same choices make the core well-suited to deterministic, low-latency embedded workloads where predictable timing matters more than peak throughput, and small and simple enough to serve as a teaching platform. (A by-product of having no speculation and no caches is that the microarchitectural side-channels which rely on them have no mechanism to exploit, though that is a consequence of the design rather than its aim.)

The short form **LV** is used in code, figures, and abbreviated references. The full name is *Luimneach RISC-V Core*; *Luimneach* (/ˈlɪmnəx/) is the Irish-language name of the city of Limerick.

## Why a new core

There is no shortage of open RISC-V soft cores. PicoRV32, Ibex, VexRiscv, CVA6, Rocket and others all exist and all have legitimate use cases. The motivation for designing a new one inside a doctoral programme is *not* that the existing cores are inadequate as general-purpose engines. The motivation is more specific.

Most existing open RV cores are designed for one of two extremes. At the small end, cores like PicoRV32 are sized for embedded control work: minimal, single-cycle or short-pipeline, but general-purpose and RV32. At the larger end, cores like CVA6 are Linux-class application processors: out-of-order or deeply-pipelined, cached, MMU-equipped, and built to run a full OS at high IPC. Both ends solve their problem well.

The middle is sparser. There is a real and growing class of use cases that want **64-bit register width** (to match modern software ABIs without ABI-translation overhead), **fully deterministic, analysable instruction timing** (rather than the data-dependent timing of a speculative, cached core), and **a small, FPGA-friendly footprint**. The combination is unusual. Most cores that are small enough are RV32; most cores that are RV64 use speculation, caches, and out-of-order issue to chase IPC. Luimneach-V is positioned in this middle ground: RV64IMC, deterministic, cache-free, and FPGA-friendly.

## What we set out to build

The doctoral-MVP target for Luimneach-V is small on purpose:

- **ISA profile.** RV64I + M (multiply / divide) + C (compressed) + Zicsr (CSR access) + Zifencei (instruction-fetch fence) + simplified privileged spec (M-mode and U-mode only, no S-mode, no MMU, no virtual memory) + Physical Memory Protection (PMP) with 4 regions.
- **Reserved custom-instruction slots.** Two encodings in the `custom-0` opcode space are reserved by the LV ISA for application-specific microarchitectural acceleration. Their specific encodings and semantics are the subject of a forthcoming companion paper and are not specified in this version of the core; the slots exist in the decoder and are routed to a parameterised extension unit.
- **Pipeline.** Single-cycle for the MVP. A 3-stage in-order pipeline is the obvious post-doctoral extension; it is not needed to demonstrate the doctoral hypothesis.
- **What we deliberately exclude.** No atomics (A extension), no floating-point (F or D), no vector (V), no caches of any kind, no branch prediction of any kind, no speculation. Each of these exclusions is what keeps timing deterministic and the design simple, and is a deliberate scope choice rather than a missing feature.

## How we plan to verify it

Three layers of verification, applied at every commit:

- **Unit testbenches** in Verilator, one per RTL block (ALU, register file, decoder, CSR file, PMP, branch unit, integration).
- **ISA-compliance**, using the standard RISCOF framework against Spike as the reference instruction-set simulator. All of RV64IMC, Zicsr, Zifencei, M-mode and U-mode, and PMP must pass the `riscv-arch-test` suite.
- **Formal verification** for selected properties (decoder consistency, PMP enforcement, CSR access semantics) using SymbiYosys.

Periodic hardware verification on the **Microchip PolarFire SoC FPGA Icicle Kit (MPFS250T-FCVG484EES)**: every tagged milestone produces a synthesis report and a measured bitstream, plus a record of timing closure and resource utilisation. Day-to-day CI runs in simulation; hardware bring-up is a manual checkpoint at named tags.

A **Renode** platform definition gives us functional simulation without the FPGA flow, for driver development, integration work, and end-to-end smoke testing that does not depend on cycle-accurate timing. Renode is ISA-level, not microarchitectural, so it complements Verilator and Spike rather than replacing them.

The intended OS-bring-up target is **Zephyr RTOS**, with a board-support package and a "hello world" sample running on the Icicle Kit as the headline integration milestone.

## How it will be released

LV is open-source academic research. The hardware RTL, software, examples, and CI scripts are released under the Apache License 2.0; the design documentation and RFCs are released under the Creative Commons Attribution 4.0 International License (CC-BY-4.0). Both licences are permissive: derivatives, including commercial derivatives, are allowed; attribution to the author and the University of Limerick must be preserved per the project's `NOTICE` file.

Contributions are accepted under the Developer Certificate of Origin (DCO); contributors add a `Signed-off-by:` trailer to each commit.

## Beyond the core itself

**A doctoral publication.** The microarchitecture of LV is the subject of a forthcoming paper, working title *"A Small Deterministic RV64IMC Soft Core, Design, Verification, and FPGA Implementation"* (Janjua, in preparation). Target venues are CARRV (the RISC-V workshop at ISCA), FCCM, or DATE. The paper's headline contribution is the design and verification of a bounded-timing 64-bit soft core and its FPGA implementation, covering an unpipelined, in-order, cache-free RV64IMC datapath with documented instruction-class timing from RTL through compliance to a working bitstream.

## Status, June 2026

- **Design.** ISA scope locked. Microarchitecture choice committed (single-cycle, in-order, non-speculative, cache-free). Custom-instruction encodings reserved but not finalised.
- **Implementation.** Not started in this repository. Previous prototype RTL exists in the candidate's private working tree, predating the doctoral programme; it is firewalled out of the doctoral submission and the RTL in this repository will be authored from first principles under supervision, commit by commit.
- **Verification.** Not started. Harness design committed to in this document.
- **Hardware bring-up.** Not started. Target board confirmed (Icicle Kit, MPFS250T-FCVG484EES).
- **Paper.** *A Small Deterministic RV64IMC Soft Core, Design, Verification, and FPGA Implementation* (Janjua, in preparation); target venues are CARRV, FCCM, or DATE.

This document is the project's narrative origin record. The technical specifications follow in `docs/architecture/`; the roadmap and milestones in `plans/ROADMAP.md`. Each is rooted in the reasoning above.

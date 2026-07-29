# RFC-0001, Luimneach-V Design Proposal

**Status** - Draft v0.1
**Type** - Design proposal
**Author** - Weqaar Janjua, Doctorate in Engineering candidate, University of Limerick (`janjua.weqaar@ul.ie`)
**Supervisor** - Dr. Eoin O'Connell
**Co-Supervisor** - Dr. Mihai Penica
**Date** - June 2026

> *This RFC is the first technical commit of the Luimneach-V open-source repository. It states what the project is, why it is being undertaken, what is in and out of scope, and how it will progress. All later commits trace back to this document for design rationale.*

---

## 1. Summary

**Luimneach-V (LV)** is an open-source 64-bit RISC-V soft processor core developed at the University of Limerick, Ireland, as part of a Professional Doctorate in Engineering (DEng) programme. Its distinguishing property is that it occupies an under-served point in the open-core design space: a small, **in-order, non-speculative, and cache-free** RV64IMC core that is deterministic by construction, comfortably FPGA-hostable, and simple enough to serve as a teaching artefact. A by-product of the cache-free, non-speculative datapath is that microarchitectural side-channels which depend on speculation or shared caches have no mechanism present in the core, but this is a consequence of the design rather than its central aim.

The target hardware is the Microchip PolarFire SoC FPGA Icicle Kit (MPFS250T-FCVG484EES). The intended operating-system bring-up target is Zephyr RTOS. The doctoral evaluation will run the standard `riscv-arch-test` compliance suite under the RISCOF framework with Spike as the reference instruction-set simulator.

Luimneach-V is research-stage. There is no shipping silicon, no commercial product, and no released bitstream at the time of this RFC. The intended outputs of the project are an ISA specification, a microarchitecture specification, a verified single-cycle RTL implementation, an FPGA bring-up on the Icicle Kit, a Zephyr port, a peer-reviewed publication, a doctoral thesis chapter, and a fully open-source artefact set published under permissive licences.

The name *Luimneach* (/ˈlɪmnəx/) is the Irish-language name of the city of Limerick. The short form **LV** is used in code and figures; the full name is *Luimneach RISC-V Core*.

## 2. Motivation

Open-source RISC-V soft cores have proliferated over the last decade. PicoRV32, Ibex, VexRiscv, CVA6, Rocket, BOOM, and others all exist and all have legitimate use cases. The motivation for a new core inside a doctoral programme is not that the existing cores are inadequate as general-purpose engines. The motivation is that a particular point in the design space remains under-explored.

Most existing open RV cores cluster at two extremes:

- **Small embedded cores** (PicoRV32, Ibex, Mi-V) are typically RV32, general-purpose, single-cycle or short-pipeline. They are effective within their word width, but a 64-bit toolchain and ABI cannot run on them without translation.
- **Application-class cores** (CVA6, Rocket, BOOM) are RV64, deeply pipelined, cached, MMU-equipped, and built to run Linux at high IPC. They use speculation, branch prediction, and caches to chase throughput, which makes their instruction timing data-dependent and hard to reason about, and makes them large and complex to build, study, or teach from.

There is a middle ground that is sparsely populated: **64-bit, deterministic, cache-free, and FPGA-friendly**. The combination is unusual because most cores small enough to be deterministic are RV32, and most cores that are RV64 use speculation and caches to chase IPC.

The middle ground matters because of three converging pressures:

1. **64-bit ABIs are now common.** Modern software toolchains, RTOS distributions, and library ecosystems support RV64 profiles. A 64-bit core lets the same toolchain artefacts run on the soft core that run on commodity RISC-V hardware, without ABI translation.
2. **Deterministic, analysable timing is increasingly valuable.** Real-time control, safety-related embedded systems, and cycle-accurate teaching all benefit from a core whose instruction timing is bounded and independent of cache or speculation history. An unpipelined, cache-free, non-speculative datapath removes cache-miss, branch-misprediction, and pipeline-hazard variability. Each instruction class still requires a documented cycle bound, including multiply, divide, memory, trap, and interrupt behaviour. A secondary consequence, noted but not pursued as the project's aim, is that timing side-channels relying on caches or speculation have no mechanism to exploit.
3. **FPGA fabrics on cost-competitive boards (Microchip PolarFire SoC, Lattice ECP5, AMD/Xilinx Artix-7) have enough resources for a small RV64 core**, especially without caches and without a deep pipeline. An RV64IMC core with no caches, no branch predictor, and a single-cycle or 3-stage in-order datapath fits comfortably on these platforms.

Luimneach-V is the design point that occupies this middle ground.

## 3. Hypothesis

The project tests the following hypothesis:

> *A small, in-order, non-speculative, cache-free RV64IMC soft core, with reserved custom-instruction slots in the `custom-0` opcode space for application-specific acceleration, can be implemented with bounded per-instruction-class timing, fit comfortably on cost-competitive FPGA fabrics, pass the standard RISC-V compliance suite, and boot a real-time operating system, while remaining simple enough to serve as an open teaching platform.*

The hypothesis is evaluated through three independent measurements, FPGA resource utilisation and timing closure on the PolarFire SoC Icicle Kit; ISA compliance against the standard `riscv-arch-test` suite via RISCOF with Spike as the reference; and per-instruction-class cycle bounds together with a successful Zephyr RTOS bring-up on the Icicle Kit.

## 4. Scope

### 4.1 In scope (doctoral)

- ISA specification (RV64I + M + C + Zicsr + Zifencei + simplified M-mode and U-mode + PMP with 4 regions).
- Microarchitecture specification (single-cycle datapath; in-order; non-speculative; cache-free; tightly-coupled-memory interface).
- Reserved custom-instruction encodings in the `custom-0` opcode space (two slots; specific semantics to be specified in a forthcoming companion paper and a follow-on RFC).
- Synthesisable Verilog/SystemVerilog RTL for the single-cycle core.
- Unit testbenches (Verilator), ISA-compliance harness (RISCOF + Spike), formal verification (SymbiYosys) for selected properties.
- Renode platform definition for functional simulation.
- FPGA bring-up on the Microchip PolarFire SoC FPGA Icicle Kit.
- Zephyr RTOS port (board-support package, "hello world" sample).
- Microarchitectural characterisation: a deterministic-timing and worst-case-execution-time (WCET) analysability argument derived directly from the RTL.
- Open-source release of the full artefact set under Apache-2.0 (code) and CC-BY-4.0 (documentation).
- A peer-reviewed publication.
- A doctoral thesis chapter.
- A teaching textbook with twelve practical labs (chapters 1-5 and labs 1-6 are the doctoral MVP; chapters 6-12 and labs 7-12 are post-doctoral).

### 4.2 Out of scope (deferred to post-doctoral work)

- 3-stage in-order pipeline.
- Atomics (A extension).
- Floating-point (F and D extensions).
- Vector (V extension).
- Hypervisor (H extension).
- S-mode, MMU, virtual memory.
- Caches of any kind.
- Branch prediction of any kind.
- Speculative execution.
- Additional custom-instruction encodings beyond the two reserved slots.
- Multi-board FPGA support (Lattice ECP5, AMD/Xilinx Artix-7, etc.). The doctoral artefact targets one board; portability is a post-doctoral extension.
- Zephyr networking, sensor drivers, and advanced peripheral support beyond what "hello world" needs.
- Production silicon manufacture. The project targets FPGA, not ASIC.
- Commercial productisation.

### 4.3 Non-goals

Luimneach-V is **not** a general-purpose Linux-class application processor. Anyone wanting to run mainline Linux on a single open-source RV64 core should use CVA6 or Rocket. LV is deliberately scoped below that line.

Luimneach-V is **not** a high-IPC core. The microarchitectural choices that give it deterministic timing and simplicity (no speculation, no caches, no branch prediction) are the *same* choices that bound its peak IPC well below a comparable-area speculative core. This is the trade-off the project explores, not a limitation to be apologised for.

## 5. Architecture (interface level)

### 5.1 Block-level structure

The core consists of seven blocks:

| Block | Role |
|---|---|
| **Decoder** | Decodes RV64IMC + Zicsr + Zifencei instructions and the two reserved custom-instruction encodings. |
| **Register file** | 32 × 64-bit integer registers, two read ports, one write port, `x0` hardwired to zero. |
| **ALU** | 64-bit add, subtract, logical, shift, compare. |
| **Multiplier / divider** | M-extension support (MUL, DIV, REM and 32-bit variants). |
| **CSR file** | M-mode and U-mode CSRs (`mstatus`, `mtvec`, `mcause`, `mepc`, `mtval`, `mip`, `mie`, plus the PMP CSRs). |
| **PMP unit** | Four PMP regions, evaluated on every memory access. |
| **Memory interface** | Tightly-coupled-memory (TCM) interface for instruction fetch, data load, data store. No caches between core and TCM. |
| **Extension unit** | Reserved decode and dispatch path for the two `custom-0`-opcode custom instructions. Specific semantics in a forthcoming RFC. |

The unpipelined datapath handles one architectural instruction at a time through fetch, decode, execute, memory access, and writeback. There are no inter-instruction pipeline hazards, forwarding paths, branch prediction, or speculative paths. Multi-cycle arithmetic, memory, trap, and interrupt behaviour is controlled by an explicit state machine.

### 5.2 Reserved custom-instruction slots

Two encodings in the `custom-0` opcode space (RISC-V opcode `0001011`) are reserved by the LV ISA for application-specific microarchitectural acceleration. The slots are present in the decoder and routed to the parameterised extension unit; their specific encodings and semantics are deliberately *not* specified in this RFC.

The rationale for reserving but not specifying:

- The custom instructions are workload-specific. The workload that motivates them is the subject of a companion doctoral research artefact whose publication ordering requires that the workload paper appear first.
- The LV core stands on its own as a general in-order RV64IMC implementation. The reserved slots make future specialisation possible without re-spinning the core for each application.
- A forthcoming companion paper and a follow-on RFC (RFC-0002 or later) will specify the encodings and semantics in full, including formal verification properties and measured speedup against a baseline-ISA-only LV configuration.

Until the companion RFC is published, the extension unit returns an illegal-instruction trap for any `custom-0` opcode it does not recognise, allowing safe coexistence with software that does not use the custom instructions.

### 5.3 Memory model

LV implements a simplified subset of the RISC-V Weak Memory Ordering (RVWMO) model. Because the core is in-order and single-issue, most ordering properties hold trivially: every memory operation is performed in program order, with no out-of-order loads, no speculative loads, and no store-to-load forwarding through a buffer. The Zifencei extension provides the instruction-fetch fence needed for self-modifying code (relevant when firmware is loaded into instruction RAM at boot).

### 5.4 Reset and boot

On reset, the program counter is set to a configurable boot address (default `0x0001_0000`), all general-purpose registers are zero, all CSRs are at their architectural reset values, and all PMP regions are unlocked and inactive. The reset behaviour is fully synchronous to the core clock; there is no asynchronous reset distribution.

## 6. Microarchitectural properties

Because the datapath is unpipelined, in-order, non-speculative, and cache-free, several properties hold by construction rather than through added logic:

- **Deterministic timing.** Each instruction class has a documented cycle bound. There are no cache misses, branch mispredictions, or pipeline hazards; any operand-dependent latency, including division, must be bounded and reported explicitly.
- **Analysability.** The absence of speculative and cached state makes worst-case execution time (WCET) analysis straightforward, which is valuable for real-time and safety-related use, and for teaching where a student can trace exactly what each cycle does.
- **Small, inspectable state.** The complete architectural and microarchitectural state is small enough to be reasoned about directly from the RTL, without needing to model predictor or cache contents.

A consequence of the same design choices is that the microarchitectural side-channels which depend on speculative execution (the Spectre family) or on shared caches (cross-thread cache-timing) have no underlying mechanism present in the core, and are therefore structurally absent. This is a by-product of a design chosen for determinism and simplicity, not the project's central claim, and it is not the subject of the headline evaluation.

## 7. Verification methodology

Three independent verification layers, applied at every commit by CI:

1. **Unit testbenches** in Verilator, one per RTL block (decoder, ALU, regfile, CSR, PMP, branch unit, memory interface, top-level integration). Each unit testbench is self-contained, runs in seconds, and produces a pass/fail result.
2. **ISA-compliance** via the standard RISCOF framework. The reference is Spike (the official RISC-V instruction-set simulator). The compliance suite is `riscv-arch-test`. The required compliance set is the full RV64IMC profile, the Zicsr and Zifencei extensions, the M-mode and U-mode privileged behaviour, and PMP enforcement.
3. **Formal verification** of selected properties using SymbiYosys. Targeted properties include: (a) decoder consistency (every well-formed instruction is decoded to exactly one operation; every ill-formed instruction traps); (b) PMP enforcement (no memory access succeeds outside an active and permitted PMP region); (c) CSR access semantics (every CSR read or write that requires a privilege level traps if executed at a lower level).

Hardware verification is performed at named milestone tags only, not on every commit. Each tagged release produces:

- A Libero SoC synthesis report (resource utilisation, timing closure).
- A bitstream loaded onto the Icicle Kit (MPFS250T-FCVG484EES).
- A boot trace from the on-board UART, captured to a text log and committed to the release artefacts.
- A timing log demonstrating the core meets its target clock frequency.

For functional simulation outside the FPGA flow, a **Renode** platform definition is provided. Renode is an ISA-level functional simulator; it does not model the LV microarchitecture (the pipeline, the absence of speculation, the absence of caches) and therefore cannot be used to evaluate the microarchitectural timing properties of §6. It is used for OS bring-up, driver development, and end-to-end smoke testing.

## 8. Toolchain

| Stage | Tool | Role |
|---|---|---|
| RTL authoring | Verilog, SystemVerilog | Synthesisable RTL |
| Linting | Verilator (`--lint-only`) | Continuous-integration style checks |
| Unit simulation | Verilator | Per-block testbenches |
| ISA compliance | RISCOF + Spike + `riscv-arch-test` | Compliance suite |
| Formal verification | SymbiYosys | Selected property checks |
| Functional simulation | Renode | OS bring-up, driver development, smoke tests |
| FPGA synthesis | Microchip Libero SoC | Bitstream for the PolarFire SoC FPGA Icicle Kit |
| Compilers | `riscv64-unknown-elf-gcc` and `clang -target riscv64` | Cross-toolchain for slot firmware and bare-metal examples |
| OS | Zephyr RTOS | OS bring-up target |
| Documentation | Pandoc + xelatex | Markdown to PDF for ISA spec, RFCs, book chapters |

All tools are open-source or available free for non-commercial / educational use.

## 9. Roadmap

| Year | Deliverable |
|---|---|
| Y3 (current) | ISA specification draft. Microarchitecture specification draft. Synthesisable RTL for the ALU, decoder, regfile, CSR file, PMP. RISCOF + Spike compliance harness. First passing RV64I tests. |
| Y4 | RV64IM and RV64IMC compliance. PMP enforcement verified. Reserved-custom-slot decoder and dispatch path. Libero project, constraints, first bitstream. Boot of bare-metal "hello world" on the Icicle Kit. |
| Y4-Y5 | Zephyr SoC port. Zephyr "hello world" sample running on the Icicle Kit. Renode platform definition complete. |
| Y5 | Companion paper specifying the custom-instruction encodings (follow-on to the workload-class paper). Doctoral thesis chapter. Open-source release. Tech-transfer process initiated through the UL Research Office. |

## 10. Publication plan

**Headline paper** - *"A Small Deterministic RV64IMC Soft Core, Design, Verification, and FPGA Implementation"* (Janjua, in preparation).

**Target venues**, in submission-preference order:

1. **CARRV** (RISC-V workshop, co-located with ISCA), strongest single fit for a custom RV64 soft core with an FPGA implementation and verification story.
2. **FCCM** (IEEE International Symposium on Field-Programmable Custom Computing Machines), excellent venue for FPGA-implemented custom cores.
3. **DATE** (Design, Automation and Test in Europe).
4. **IEEE CAL** (Computer Architecture Letters), if a focused 4-page letter is the right form factor.

Final venue choice depends on whether the paper emphasises the microarchitecture (CARRV), the FPGA implementation (FCCM), or a focused design-and-verification letter (CAL).

A companion paper specifying the reserved custom-instruction slots is planned as a follow-on, after the publication of the workload-class paper that motivates them.

## 11. Licensing

Luimneach-V is released under a two-licence model:

- **Apache-2.0** for hardware (RTL, constraints, bitstreams), software (BSP, Zephyr port, examples, CI), and Renode model code.
- **CC-BY-4.0** for prose content (RFCs, design documentation, figures, textbook chapters, lab content).

SPDX identifiers are present in every source file. Contributions are accepted under the Developer Certificate of Origin (DCO); contributors must add a `Signed-off-by:` line to each commit.

Attribution is enforced through an Apache-2.0 `NOTICE` file (see `NOTICE` in the repository root) which downstream redistributors are required to reproduce per Apache-2.0 §4.

The licence choice has been notified to the University of Limerick Research Office in accordance with UL's standard doctoral IP-disclosure procedure.

## 12. Repository and branch convention

The Luimneach-V repository is hosted at `github.com/weqaar/Luimneach-V` (private during early development; public release planned at first stable tag). LV is University of Limerick doctoral research; it is not the product of any commercial organisation.

| Branch | Purpose |
|---|---|
| `main` | Stable. Direct push prohibited. All changes via merge from `dev/*` branches with review. |
| `dev/weqaarjanjua/*` | Per-topic working branches. |

Commit messages follow Linux-kernel convention: `subsystem: short imperative summary` (max 60 chars), blank line, body explaining *why*, `Signed-off-by:` trailer.

## 13. Related work and references

The open RISC-V soft cores adjacent to Luimneach-V include the following projects.

- **PicoRV32** (Wolf, 2015 onwards), ISC-licensed, RV32IMC, single-cycle or 2-stage, no privileged spec; the canonical small open RV32 core.
- **Ibex** (lowRISC, 2018 onwards), Apache-2.0, RV32IMC with M and U privilege modes; widely used as the small core in OpenTitan and similar projects.
- **VexRiscv** (Papon, 2017 onwards), MIT, SpinalHDL-based, configurable RV32 with optional caches.
- **CVA6 / Ariane** (OpenHW Group / ETH Zürich, 2017 onwards), Apache-2.0, RV64GC, six-stage in-order, Linux-capable, cached.
- **Rocket** (UCB / SiFive, 2012 onwards), permissive, Chisel-based, RV64GC, cached.
- **Microchip Mi-V** family, vendor IP free for use on Microchip FPGAs, RV32IMC and RV32IMA variants, closed RTL.

LV's position relative to these is smaller than CVA6/Rocket, larger than PicoRV32/Ibex because it is RV64, and narrower in feature set than any of them because it omits caches, speculation, and branch prediction. It is distinguished less by any single feature than by the combination of a 64-bit ISA, bounded per-instruction-class timing on a cost-competitive FPGA, and a form simple enough to teach from.

## 14. Acknowledgements

This project is conducted at the University of Limerick, Ireland, under the supervision of Dr. Eoin O'Connell and the co-supervision of Dr. Mihai Penica, within the Professional Doctorate in Engineering (DEng) programme.

LV is released under permissive open-source licences as described in §11. No commercial organisation owns or controls its intellectual property.

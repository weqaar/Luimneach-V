# Luimneach-V ISA Specification

**Status** - Draft doctoral MVP.
This document is the authoritative instruction-set contract for the Luimneach-V (LV) core. The RTL, the RISCOF compliance harness, and the Spike reference configuration are all derived from this document. Where this document and the RTL disagree, this document is correct and the RTL is a bug.

LV follows the ratified RISC-V unprivileged and privileged specifications. This document does not restate those specifications; it records exactly which parts LV implements, and pins down the implementation-defined choices LV makes.

## 1. Profile summary

| Property | Value |
|---|---|
| Register width (XLEN) | 64 |
| Base integer ISA | RV64I |
| Standard extensions | M, C, Zicsr, Zifencei |
| ISA string | `rv64imc_zicsr_zifencei` |
| Privilege levels | M-mode, U-mode |
| Physical memory protection | PMP, 4 regions |
| Address translation | none (no MMU, no paging) |
| Reset vector | `0x0000_0000_0001_0000` (configurable) |
| Endianness | little-endian |
| Reserved opcode space | `custom-0` (`0001011`), two encodings, semantics unspecified |

Extensions that LV does **not** implement: A (atomics), F/D/Q (floating point), V (vector), H (hypervisor), S-mode, and any form of address translation. Instructions from unimplemented extensions raise an illegal-instruction exception.

### 1.1 Rationale for the ISA profile

Each extension is included for a stated reason, not by copying a default ISA string. The profile supports a small preemptive real-time operating system (RTOS), but no extension is selected solely for one named RTOS.

| Piece | Required for the RTOS target? | Reason for the choice |
|---|---|---|
| **I** (RV64I base) | Yes | The base integer ISA. Nothing executes without it. |
| **64-bit width** | No | A scoping decision, not an RTOS requirement. LV deliberately fills the under-served middle of the open-core field with a simple, deterministic **64-bit** core rather than a small RV32 controller or a large speculative RV64 application core. |
| **M** (multiply/divide) | No | Without M, the toolchain lowers multiply and divide operations to software library calls such as `__muldi3` and `__divdi3`, increasing code size and execution time. Hardware multiply and divide support also simplifies timing analysis for RTOS and application code. |
| **C** (compressed) | No | Compressed encodings reduce code size, which matters directly for the limited tightly coupled instruction memory on the FPGA. C also has broad support in RISC-V compilers and embedded software. |
| **Zicsr** | Yes | Control-and-status-register access is needed for traps, the timer, interrupts, and the `mstatus`/`mtvec`/`mie`/`mip`/`mepc` machinery. A preemptive RTOS scheduler is not possible without it. |
| **Zifencei** | Yes | `FENCE.I` synchronises instruction fetch with prior writes to instruction memory. LV loads firmware into writable instruction memory at boot, so this is required for correctness. |

Extensions deliberately left out, with the reason:

- **A (atomics)** - not required for the single-hart MVP. An RTOS port can protect scheduler and shared-state operations by disabling interrupts around critical sections. Hardware atomics become necessary if the scope expands to symmetric multiprocessing or other concurrent harts.
- **F, D, Q (floating point)** - the MVP workloads are integer. Floating point would add a register file and a large datapath for no benefit at this stage.
- **V (vector), H (hypervisor), S-mode, MMU** - all belong to larger application-class or virtualised systems, which are explicitly out of scope (see RFC-0001 section 4.3).

The resulting `rv64imc_zicsr_zifencei` profile supports the trap, interrupt, timer, privilege, and writable-instruction-memory behaviour needed by the selected single-hart RTOS port without tying the ISA contract to one operating-system implementation.

## 2. Register state

### 2.1 Integer registers

32 integer registers, `x0` to `x31`, each 64 bits. `x0` is hardwired to zero: reads return 0, writes are discarded. The register file has two read ports and one write port.

The ABI names (`ra`, `sp`, `gp`, `tp`, `t0`-`t6`, `s0`-`s11`, `a0`-`a7`) are conventions used by the toolchain and carry no hardware meaning. LV treats all registers except `x0` identically.

### 2.2 Program counter

One 64-bit program counter, `pc`. Instruction addresses are 2-byte aligned because the C extension is implemented (a 4-byte-aligned `pc` is not required). A fetch to a misaligned address (odd byte) raises an instruction-address-misaligned exception.

### 2.3 No floating-point state

There is no floating-point register file and no `fcsr`. The `F`, `D`, and `Q` extensions are absent.

## 3. Instruction set

LV implements the full RV64I base plus the M, C, Zicsr, and Zifencei extensions. This section lists the instruction groups and records LV-specific decisions. It does not reproduce per-instruction semantics from the base specification.

### 3.1 RV64I base integer

- **Computational, register-immediate** - `ADDI`, `SLTI`, `SLTIU`, `XORI`, `ORI`, `ANDI`, `SLLI`, `SRLI`, `SRAI`, and the word forms `ADDIW`, `SLLIW`, `SRLIW`, `SRAIW`.
- **Computational, register-register** - `ADD`, `SUB`, `SLL`, `SLT`, `SLTU`, `XOR`, `SRL`, `SRA`, `OR`, `AND`, and the word forms `ADDW`, `SUBW`, `SLLW`, `SRLW`, `SRAW`.
- **Upper immediate** - `LUI`, `AUIPC`.
- **Control transfer** - `JAL`, `JALR`, `BEQ`, `BNE`, `BLT`, `BGE`, `BLTU`, `BGEU`.
- **Load** - `LB`, `LH`, `LW`, `LD`, `LBU`, `LHU`, `LWU`.
- **Store** - `SB`, `SH`, `SW`, `SD`.
- **Fence** - `FENCE`. Because the core is in-order and single-issue, `FENCE` completes as a no-op with respect to ordering (see section 8), but it is decoded and accepted, not trapped.
- **Environment and breakpoint** - `ECALL`, `EBREAK`.

Shift amounts for the 64-bit shift instructions use the low 6 bits of the shift operand; the word shift instructions use the low 5 bits, as defined by RV64I.

### 3.2 M extension (integer multiply and divide)

- **Multiply** - `MUL`, `MULH`, `MULHSU`, `MULHU`, and the word form `MULW`.
- **Divide and remainder** - `DIV`, `DIVU`, `REM`, `REMU`, and the word forms `DIVW`, `DIVUW`, `REMW`, `REMUW`.

Division by zero and signed overflow follow the RISC-V-defined results (no trap): division by zero returns all-ones for the quotient and the dividend for the remainder; signed overflow (`most-negative / -1`) returns the dividend for the quotient and zero for the remainder.

Multiply and divide are the only base-ISA instructions whose timing is not fixed at one cycle in the single-cycle MVP; their handling is defined in the microarchitecture specification, not here.

### 3.3 C extension (compressed instructions)

All standard RV64C 16-bit encodings are implemented. Each compressed instruction expands to exactly one base instruction and is decoded directly (there is no separate expand-then-decode stage exposed to software). Because C is present, instructions may begin on any 2-byte boundary.

### 3.4 Zicsr extension (CSR access)

`CSRRW`, `CSRRS`, `CSRRC`, `CSRRWI`, `CSRRSI`, `CSRRCI`. Access permissions, read-only CSRs, and the write side effects follow the privileged specification. An attempt to access a CSR that does not exist, or to write a read-only CSR, or to access an M-mode CSR from U-mode, raises an illegal-instruction exception. The set of implemented CSRs is in section 5.

### 3.5 Zifencei extension (instruction-fetch fence)

`FENCE.I`. This synchronises the instruction stream with prior stores to instruction memory. It is required because firmware is written into instruction memory at boot and may be updated at run time; `FENCE.I` guarantees that subsequently fetched instructions observe those writes.

### 3.6 Reserved custom-0 encodings

Two encodings in the `custom-0` opcode space (major opcode `0001011`) are reserved by the LV ISA. Their encodings and semantics are **not** specified in this document. They are the subject of a forthcoming companion paper and a follow-on RFC.

Until that RFC is published, LV decodes the `custom-0` opcode but recognises no encoding within it, so every `custom-0` instruction raises an illegal-instruction exception. Software that does not use these encodings is unaffected. The reservation exists so the opcode space is not later claimed by something else.

## 4. Privilege levels

LV implements two privilege levels:

- **Machine mode (M-mode)** - the highest privilege level, entered on reset and on every trap. All CSRs and all physical memory (subject to PMP) are accessible.
- **User mode (U-mode)** - reduced privilege. U-mode may access only the CSRs the specification permits, and only the physical memory that PMP grants it.

There is no supervisor mode, no MMU, and no virtual memory. All addresses are physical. Traps are never delegated; every trap is taken in M-mode. Because delegation is impossible, the `medeleg` and `mideleg` CSRs are not implemented.

Privilege transitions:

- Reset enters M-mode.
- A trap (exception or interrupt) enters M-mode, sets `mepc`, `mcause`, and `mtval`, and jumps to the address in `mtvec`.
- `MRET` returns from a trap, restoring the privilege level recorded in `mstatus.MPP`.
- U-mode is entered by executing `MRET` with `mstatus.MPP` set to U.

## 5. Control and status registers

This section lists which CSRs exist. The full address allocation and per-field layout is in [`csr-map.md`](csr-map.md).

### 5.1 Machine information (read-only)

| CSR | Notes |
|---|---|
| `misa` | Reports RV64 and the set `IMC`. Writable-but-ignored (WARL); LV holds it fixed. |
| `mvendorid` | 0 (no assigned JEDEC vendor ID for an academic core). |
| `marchid` | Implementation-defined LV architecture ID. |
| `mimpid` | LV implementation version. |
| `mhartid` | 0. LV is single-hart. |

### 5.2 Machine trap setup and handling

| CSR | Notes |
|---|---|
| `mstatus` | Only the fields meaningful for M/U without S-mode are implemented: `MIE`, `MPIE`, `MPP`, `MPRV`. Other fields read as 0. |
| `mtvec` | Trap-vector base address. Direct mode only; vectored mode is not implemented. |
| `mie` | Interrupt-enable bits for the implemented interrupts (see section 7). |
| `mip` | Interrupt-pending bits for the implemented interrupts. |
| `mscratch` | Scratch register for the trap handler. |
| `mepc` | Exception program counter. |
| `mcause` | Trap cause (section 7). |
| `mtval` | Trap value (faulting address or instruction, per cause). |

`medeleg` and `mideleg` are not implemented (no S-mode). `mcounteren` is not implemented in the MVP.

### 5.3 Physical memory protection

`pmpcfg0` and `pmpaddr0` through `pmpaddr3`. See section 6.

### 5.4 Counters

`mcycle` and `minstret` (and their read-only U-mode shadows `cycle` and `instret` if enabled) may be implemented in the MVP for basic measurement. Their presence is confirmed in [`csr-map.md`](csr-map.md). No other hardware performance counters are implemented.

## 6. Physical memory protection

LV implements 4 PMP regions. Each region has an 8-bit configuration field (in `pmpcfg0`) and a 64-bit address field (`pmpaddr0` to `pmpaddr3`).

- Supported address-matching modes: `OFF`, `TOR` (top of range), `NA4` (naturally aligned 4-byte), and `NAPOT` (naturally aligned power of two), as defined by the specification.
- Each region grants some combination of read (`R`), write (`W`), and execute (`X`) permission, and may be locked (`L`).
- PMP is checked on every instruction fetch, load, and store, in both M-mode and U-mode. In M-mode, an access matching a region with the lock bit clear is permitted regardless of `RWX`; a locked region is enforced against M-mode as well.
- An access that violates PMP raises the corresponding access-fault exception (instruction, load, or store/AMO access fault) with the faulting address in `mtval`.

Region priority follows the specification: the lowest-numbered matching region determines the result.

## 7. Traps and exceptions

All traps are taken in M-mode. On a trap, LV sets `mepc` to the address of the interrupted or faulting instruction, `mcause` to the cause below, `mtval` as noted, saves the interrupt-enable state into `mstatus.MPIE`, saves the privilege level into `mstatus.MPP`, clears `mstatus.MIE`, and jumps to `mtvec`.

### 7.1 Synchronous exceptions

| Cause | Exception | `mtval` |
|---|---|---|
| 0 | Instruction address misaligned | faulting address |
| 1 | Instruction access fault (PMP) | faulting address |
| 2 | Illegal instruction | the offending instruction bits |
| 3 | Breakpoint (`EBREAK`) | `pc` |
| 4 | Load address misaligned | faulting address |
| 5 | Load access fault (PMP) | faulting address |
| 6 | Store/AMO address misaligned | faulting address |
| 7 | Store/AMO access fault (PMP) | faulting address |
| 8 | Environment call from U-mode | 0 |
| 11 | Environment call from M-mode | 0 |

Misaligned loads and stores are not emulated in hardware; a misaligned data access raises the misaligned exception rather than being split into aligned accesses.

### 7.2 Interrupts

LV implements the standard machine-level interrupts driven by an external CLINT-style timer and software-interrupt source:

| Cause (interrupt bit set) | Interrupt |
|---|---|
| 3 | Machine software interrupt |
| 7 | Machine timer interrupt |
| 11 | Machine external interrupt |

An interrupt is taken only when globally enabled (`mstatus.MIE`) and individually enabled (`mie`) and pending (`mip`). The interrupt controller and timer are platform components defined in the microarchitecture and memory-map documents, not in the ISA.

## 8. Memory model

LV is in-order and single-issue, so memory operations are performed one at a time in program order. There are no speculative loads, no out-of-order loads, and no store buffer that forwards to later loads. As a result the RISC-V weak memory model (RVWMO) is satisfied trivially, and `FENCE` needs no additional ordering logic.

`FENCE.I` (Zifencei) is the exception that does require action: it ensures instruction fetches after the fence observe stores performed before it. This matters because instruction memory is writable and firmware is loaded at boot.

All accesses are little-endian. Access sizes are byte, halfword, word, and doubleword. Naturally aligned accesses are required; misaligned accesses trap (section 7).

## 9. Reset and boot

On reset:

- `pc` is set to the reset vector, `0x0000_0000_0001_0000` by default. The reset vector is a synthesis-time parameter and may be changed per platform.
- All integer registers except `x0` are set to 0; `x0` is permanently 0.
- The core is in M-mode.
- `mstatus.MIE` is 0 (interrupts globally disabled).
- All CSRs are at their architectural reset values.
- All PMP regions are `OFF` and unlocked.

Reset is synchronous to the core clock. There is no asynchronous reset network.

## 10. Instruction formats reference

For convenience, the RV64 base instruction formats LV decodes:

| Format | Fields | Used by |
|---|---|---|
| R | `funct7 \| rs2 \| rs1 \| funct3 \| rd \| opcode` | register-register arithmetic, M extension |
| I | `imm[11:0] \| rs1 \| funct3 \| rd \| opcode` | register-immediate, loads, `JALR`, CSR, `ECALL`/`EBREAK` |
| S | `imm[11:5] \| rs2 \| rs1 \| funct3 \| imm[4:0] \| opcode` | stores |
| B | `imm[12\|10:5] \| rs2 \| rs1 \| funct3 \| imm[4:1\|11] \| opcode` | branches |
| U | `imm[31:12] \| rd \| opcode` | `LUI`, `AUIPC` |
| J | `imm[20\|10:1\|11\|19:12] \| rd \| opcode` | `JAL` |

Compressed (C) instructions use the RV32C/RV64C 16-bit formats (`CR`, `CI`, `CSS`, `CIW`, `CL`, `CS`, `CA`, `CB`, `CJ`), each expanding to one of the formats above.

## 11. Conformance

LV is conformant if it passes the RISC-V architecture-test suite for `RV64IMC_Zicsr_Zifencei` with the M/U privilege and PMP tests enabled, run through RISCOF against Spike configured to the same profile. The compliance harness lives in [`../../verification/isa-compliance/`](../../verification/isa-compliance/).

The two reserved `custom-0` encodings are outside the conformance scope of this document: while unspecified, they must trap as illegal instructions, and the compliance suite treats them as such.

# Luimneach-V Verification

Per-block unit tests, ISA-compliance suite, reference ISS comparison, and formal properties.

Proposed TODO

| Dir | Purpose | Tool |
|---|---|---|
| `unit/` | One unit testbench per RTL block | Verilator |
| `isa-compliance/` | RV64IMC + Zicsr + Zifencei + M/U + PMP compliance | RISCOF + `riscv-arch-test` |
| `reference-iss/` | Lockstep comparison against an instruction-set simulator | Spike |
| `formal/` | Bounded model checks on decoder, PMP enforcement, CSR consistency | SymbiYosys |

# Luimneach-V Architecture

Authoritative specifications for the Luimneach-V core. To be authored alongside the RTL.

Planned contents:

| File | Purpose |
|---|---|
| `isa-spec.md` | Exact RV64IMC + Zicsr + Zifencei + M/U + PMP profile; the two reserved custom-instruction encodings identified |
| `microarch-spec.md` | Single-cycle datapath, register file, ALU, branch unit, CSR file, PMP enforcement, extension-unit dispatch, memory interface |
| `csr-map.md` | CSR address allocations |
| `custom-instructions.md` | Reserved `custom-0` encodings (semantics deferred to a forthcoming companion paper) |
| `memory-map.md` | Tightly-coupled memory layout |

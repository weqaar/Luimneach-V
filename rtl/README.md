# Luimneach-V RTL

Synthesisable RTL for the core. To be authored TODO

Per-block proposed dir structure:

| Dir | Block | Doctoral MVP? |
|---|---|---|
| `alu/` | 64-bit arithmetic and logic unit | yes |
| `decoder/` | RV64IMC instruction decoder | yes |
| `regfile/` | 32 × 64-bit integer register file, 2R1W | yes |
| `csr/` | CSR file plus trap handling | yes |
| `pipeline/` | Single-cycle datapath integration | yes |
| `pmp/` | Physical memory protection | yes |
| `custom_ext/` | Decode and dispatch path for the reserved `custom-0` encodings | yes |
| `top/` | Top-level `lv_top.sv` | yes |

Verilog style: 2001 syntax, 4-space indent, no tabs, `wire` for combinational nets, `reg` only inside `always` blocks. All modules prefixed `lv_`.

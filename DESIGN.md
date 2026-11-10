# Design Decisions

Running log of architectural decisions for QUADRA, with the reasoning behind each.

## 1. Register File Ports

**Bundle layout:** 2 ALU + 1 MEM + 1 BRANCH. Branch condition is folded into ALU1's compare, so BRANCH adds no independent register reads.

**Worst-case port requirement:** 6 read ports, 3 write ports.
- ALU0: 2R / 1W
- ALU1: 2R / 1W
- MEM: up to 2R (store = base + data), up to 1W (load)

**Typical-case estimate:** using a classic RISC instruction mix (~46% ALU, ~35% load/store, ~17% branch) combined with typical VLIW bundle fill rates (~2.3 of 4 slots used), expected usage is closer to **4 reads / 2 writes** per bundle. The hardware must still be built for the 6R3W worst case, since port count is a static resource, not an average one.

**Options considered:**
| Option | Notes |
|---|---|
| Monolithic multi-ported (flip-flop array) | Simplest to verify; no ISA/packer impact |
| Clustering (VEX-style) | Fixed, always-correct partition; requires explicit cross-cluster copy instructions, adds packer complexity |
| Banking (register-range split) | Cheaper, but risks data-dependent bank conflicts the packer must avoid |
| Replicated RF (N copies, 1 read port each, shared write) | Pure hardware swap, no ISA/packer change — clean area/timing experiment, but write power scales with copy count |

**Decision:** Build a monolithic 6-read / 3-write register file (flip-flop array, combinational read, synchronous write). Enforce `x0` hardwired to zero and add a same-cycle write-conflict assertion. A replicated-RF swap is a candidate later experiment for isolated area/timing comparison.

## 2. Memory Ports

**Baseline:** 1 MEM slot → single-port memory, no port-sharing problem.

**Instruction-mix data used:** load ≈ 26%, store ≈ 9%, other ≈ 65% (classic RISC integer-benchmark mix).

**Coverage of a random 4-instruction group, by memory-port design:**
| Design | Ports | Coverage |
|---|---|---|
| 1 generic port | 1 | ~38% (covers exactly 1 mem op) |
| 1 load port + 1 store port (fixed-function) | 2 | ~68% (fails when a group has 2 loads or 2 stores) |
| 2 generic R/W ports | 2 | ~87% (any mix of ≤2 mem ops) |
| 2 load ports + 1 store port | 3 | ~90% |

A generic R/W port is a standard, available building block (FPGA true dual-port BRAM; ASIC 2-port SRAM macro) — unlike the register file, scaling memory to 2 ports is not a custom-design problem.

**Decision:** 2 generic R/W memory ports, each independently configurable per cycle as a load or a store. Same-address read/write collision behavior (read-first vs. write-first) to be fixed as an explicit spec convention before RTL.

## Final Decision

**Register file: 6 read ports / 3 write ports, monolithic flip-flop array.**
**Data memory: 2 generic read/write ports, standard dual-port SRAM/BRAM style.**

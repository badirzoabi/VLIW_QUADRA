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

## 3. Pipeline Forwarding & Latency Table

**No dynamic hazard detection unit.** QUADRA's hardware never stalls or inserts a bubble on its own — correctness comes entirely from the packer's static schedule, which is guaranteed to never read a register before its producer's result actually exists. A dynamic hazard-detection-and-stall unit would reintroduce exactly the runtime complexity VLIW is designed to move into software, so it is explicitly excluded.

**Forwarding paths (static bypass muxes, not decision logic):**
- `MEM → EX` — forwards an ALU result sitting in the EX/MEM pipeline register to the EX stage of the very next bundle. Covers back-to-back ALU dependencies with zero gap.
- `WB → EX` — forwards a result sitting in the MEM/WB pipeline register to the EX stage of a bundle two cycles behind. Covers dependencies 2 bundles apart.
- `WB → D` — bypasses the register file read in Decode when a producer's write-back lands in the same cycle as a consumer's register read, avoiding a stale read.

These are always-on muxes that pick the freshest available value; they involve no runtime comparison-and-stall decision, so they don't violate the no-dynamic-hazard-detection rule.

**Why the load-use case is different:** an ALU result exists at the end of EX, so `MEM → EX` forwarding delivers it to the very next bundle with no gap. A load's result doesn't exist until the end of its own MEM stage (the data memory read itself takes the full cycle) — which lands at the same time the next bundle is in EX. No forwarding path can deliver a value before it's produced, so a load's result is only available starting 2 cycles after issue, same as a classic scalar RISC-V pipeline's load-use hazard — the difference is that here it's resolved statically by the packer's latency table, not by a runtime-inserted stall.

**Latency table (cycles until a result is safely readable by a dependent instruction):**

| Op | Latency (cycles) | Forwarding path that covers it |
|---|---|---|
| ALU (ADD/SUB/AND/OR/...) | 1 | `MEM → EX` |
| MUL | 1 (fully combinational multiplier in EX) | `MEM → EX` |
| LOAD | 2 | `WB → EX` |
| STORE | 1 (no destination register) | n/a |
| BRANCH | 1 (condition folded into ALU1, no separate latency) | `MEM → EX` |

This table is the single source of truth both the RTL forwarding network and the Python packer's dependency scheduler must agree on exactly — any mismatch here produces silently wrong results rather than a crash.

**Open question:** does the multiplier live in both ALU slots or only one? Replicating a full 32×32 combinational multiplier in both ALUs roughly doubles that block's area; putting it in only one ALU adds a resource constraint the packer must enforce (MUL restricted to a specific ALU slot), similar to the single-MEM-slot rule. Not yet decided.

**Known tradeoff accepted:** a 1-cycle multiplier's delay becomes part of the core's critical path and may set the maximum clock frequency for the whole design, not just MUL instructions. To be confirmed once synthesis timing is available.

## 4. Branch Handling & Control Flow

**Branch resolution: flush, not delay slots.** The branch condition resolves at the end of EX (it rides on ALU1's compare result). Since fetch runs 2 stages ahead of EX, a taken branch unconditionally squashes the 2 bundles already fetched behind it — a fixed, deterministic flush triggered directly by the branch-taken signal, not a dynamic hazard decision. The packer does not need to prove anything safe about what follows a branch; squashed bundles are simply discarded.

**At most one branch per bundle** — already guaranteed structurally (only 1 BR slot exists), enforced additionally as a packer assertion.

**JAL (calls):** scheduled alone in its own bundle (not packed with the destination function's entry — that would require interprocedural scheduling and risks code duplication; deferred as a stretch goal, not baseline). JAL's return-address write reuses **ALU1's existing write port** (the same port already shared with the branch-condition fold) — no new register-file write port needed, so the 6R/3W budget still holds.

## Final Decision

**Register file: 6 read ports / 3 write ports, monolithic flip-flop array.**
**Data memory: 2 generic read/write ports, standard dual-port SRAM/BRAM style.**
**Hazards: no dynamic detection/stall unit; static `MEM→EX`, `WB→EX`, `WB→D` forwarding plus a fixed per-op latency table enforced entirely by the packer.**

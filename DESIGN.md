# Design Decisions

QUADRA is a RISC-V-based, dynamically-scheduled (out-of-order) processor. All instruction scheduling, hazard resolution, and memory disambiguation happen in hardware at runtime; the packer's only job is grouping instructions into fetch groups and isolating control flow.

**Implementation scope (current phase):** the front end — Fetch and Decode — is what this project phase builds and implements. The backend (Rename, ROB, Issue Queue, Execute, LSQ, Retire) described below is a complete architectural design; its RTL implementation is planned for a later phase. No clock-frequency target has been set yet.

## 1. Fetch & Decode

**Fetch/dispatch width: 4 general-purpose instructions per cycle, or exactly 1 branch/jump alone.**

**Memory layout: fixed-width padded groups.** Every group — including a lone branch/jump — occupies a fixed 4-instruction-wide slot in instruction memory, padded with NOPs when fewer than 4 real instructions fill it. This keeps fetch addressing a simple, uniform stride (PC always advances by exactly one group width), with no instruction-alignment or shift hardware needed. The cost is some wasted code space (branches are ≈17% of instructions, Section 7), accepted in exchange for fetch simplicity.

**Decode: 4 parallel decoders**, standard RISC-V field extraction (opcode / rd / rs1 / rs2 / immediate) — ordinary combinational logic, replicated ×4, nothing OoO-specific about decode itself.

**Packer's role:** detect every branch/jump and emit it alone in its own (padded) group; otherwise, greedily group up to 4 general-purpose (ALU/MEM) instructions per group, in program order, with zero dependency analysis. All dependency tracking and hazard resolution happen downstream, in hardware.

## 2. Register Renaming & Physical Register File

**8 read ports / 4 write ports**, monolithic physical register file, 64 physical registers renamed from 32 architectural registers via a Register Alias Table (RAT).

- **Reads:** issue width is capped at 4 total instructions per cycle (general ops and a branch compete for the same 4 slots, not a bonus 5th slot) × 2 operands = 8R.
- **Writes:** only ALU0, ALU1, MEM0-as-load, MEM1-as-load ever write (stores and branches never write) = 4W. JAL's return-address write rides one of these 4 ports.
- **Physical register count:** 32 architectural + 16 (ROB size, max in-flight renames) = 48 minimum, rounded up to 64 for clean 6-bit tags and some slack.

**Dual RAT, for misprediction recovery:**
- **Speculative RAT** — updated immediately at dispatch, for every instruction.
- **Architectural RAT** — updated only when an instruction retires, in program order.

Because retirement is strictly in-order and a branch can never retire before it resolves, the architectural RAT can never have advanced past an unresolved branch — it is always a safe restore point. Recovery on a misprediction is a single bulk copy: architectural RAT → speculative RAT.

**Intra-bundle dependency check.** Up to 4 instructions are renamed in the same cycle, and they can depend on each other — e.g. `I0: ADD r3,r1,r2` followed by `I1: ADD r6,r3,r5` in the same group. A naive per-instruction RAT lookup would give I1 the *stale* mapping for `r3` (from before this cycle), not the one I0 is creating right now. The fix: before falling back to the RAT, each instruction checks its source registers against the destination registers of every instruction *earlier than it in the same group*. If there's a match, it uses that earlier instruction's new physical register directly instead of the RAT. If there are multiple matches (e.g. two writers to the same register within one group), it uses the *nearest* one. Only the RAT itself (updated once, at the end of the cycle) reflects the group's *last* writer to each register — not every intermediate one.

**Physical register reclaim:** each ROB entry records the old physical register mapping it replaced; that register returns to the free list at this entry's own retirement — not at a consumer's retirement, and not earlier (premature reuse is a WAR-hazard bug).

## 3. ROB, Issue Queue & Dynamic Scheduling

**ROB: 16 entries. Issue Queue: 16 entries. Issue width: 4-wide.**

**Selection policy: oldest-ready, resource-compatible.** Each cycle, any issue-queue entry whose operand tag matches a completing instruction's broadcast wakes up; the scheduler selects up to 4 of the ready entries, preferring older instructions and skipping any combination that would need more of a shared resource than exists (e.g. two ready MUL ops, since only ALU0 has a multiplier).

**Broadcast network: 4 parallel CDB lanes**, not a single shared bus. Up to 4 instructions can complete in the same cycle (matching the 4 write ports, Section 2), so the completion/broadcast network needs 4 parallel tag+value lanes — every issue-queue entry's wakeup comparators check its 2 operand tags against all 4 lanes, every cycle. This is the same any-producer-to-any-consumer broadcast each completing instruction uses to wake waiting issue-queue entries and to write the physical register file.

**MUL: lives in ALU0 only, not replicated** — a resource constraint the scheduler enforces directly (never selects two ready MUL ops in the same cycle).

## 4. Branch Handling & Misprediction Recovery

**Every branch/jump is fetched/dispatched alone** in its own (padded) group.

**Full speculation; backward branches predicted taken** — a static, zero-extra-hardware rule (compare target address to branch address at decode).

**Misprediction recovery:** flush every ROB and issue-queue entry younger than the branch, restore the speculative RAT from the architectural RAT in one bulk copy (Section 2), return flushed entries' physical registers to the free list.

**JAL:** dispatched alone; its return-address write rides one of the 4 existing completion ports — no new port needed.

## 5. Memory Ports & Load-Store Queue (LSQ)

**2 generic read/write memory ports.**

**LSQ: 8 entries**, smaller than the ROB/IQ on purpose — LSQ address-comparator cost scales roughly quadratically with entries, far more expensive to grow than the IQ's linear wakeup cost.

**Policy: conservative, no speculative load execution.** A load stalls until every older, unresolved store's address is known — no replay machinery. An address match against an older store with ready data forwards directly.

**Store commit timing: stores write to Data Memory only at retirement**, never speculatively.

## 6. Exception Handling

**Mechanism: reuses the misprediction-recovery path (Section 4).** When an instruction faults during execution, its ROB entry is flagged — nothing else happens yet. Only when that entry becomes the oldest (about to retire) is the flag checked; if set, everything younger is flushed using the same dual-RAT bulk-copy recovery as a branch misprediction, since retirement is already strictly in-order and stores already only commit at retirement (Section 5) — both of which precise-exception handling needs anyway.

**Scope: specified but not exercised.** The target workloads (vector add, dot product, FIR, matrix multiply) are fixed, OS-free numeric kernels not expected to trigger exceptions in normal operation. This mechanism is defined so the design is complete, but it is not a tested or load-bearing part of this project's scope.

## 7. Workload Statistics (basis for Sections 2 &amp; 5)

**Instruction-mix data used:** load ≈ 26%, store ≈ 9%, branch ≈ 17%, other ≈ 48%. Source: [WUSTL CSE560M Problem Set 1](https://classes.engineering.wustl.edu/cse560m/560M.f24/ProbSet1.pdf).

**Coverage of a random 4-instruction group, by memory-port design:**

| Design | Ports | Coverage |
|---|---|---|
| 1 generic port | 1 | ~38% (covers exactly 1 mem op) |
| 1 load port + 1 store port (fixed-function) | 2 | ~68% (fails when a group has 2 loads or 2 stores) |
| 2 generic R/W ports | 2 | ~87% (any mix of ≤2 mem ops) |
| 2 load ports + 1 store port | 3 | ~90% |

**Real-world precedent for favoring load bandwidth:** [Penn CIS 501 (Martin), "Superscalar Memory Access," slide 20](https://acg.cis.upenn.edu/milom/cis501-Fall10/lectures/07_superscalar.pdf) — Core i7 does 1 load + 1 store/cycle, Alpha 21164 does 2 loads *or* 1 store/cycle.

**Register-file port-sizing precedent:** [Penn CIS 501 (Martin), "Superscalar Register Read/Write," slide 23](https://acg.cis.upenn.edu/milom/cis501-Fall10/lectures/07_superscalar.pdf) — stores and branches don't write registers, applied here under the 4-wide issue cap to reach 8R/4W.

**Decision:** 2 generic R/W memory ports, each independently configurable per cycle as a load or a store.

## Final Decision

**Implementation scope: front end (Fetch + Decode) for this project phase. No clock-frequency target set yet.**
**Fetch: 4-wide general-purpose, or 1 branch/jump alone, fixed-width padded groups. Decode: 4 parallel decoders.**
**Register file: 8 read / 4 write ports, monolithic, 64 physical registers renamed from 32 architectural via a dual RAT (speculative + architectural), with an intra-bundle dependency check for same-cycle producer→consumer pairs within a dispatch group. Recovered by a single bulk copy on misprediction.**
**Scheduling: ROB = 16, Issue Queue = 16, 4-wide issue, oldest-ready + resource-compatible selection, 4-lane broadcast (CDB) for wakeup and register-file completion.**
**MUL: single ALU (ALU0) only — a scheduler-enforced resource constraint.**
**Branches: full speculation, backward-taken prediction, dual-RAT bulk-copy recovery on misprediction.**
**Memory: 2 generic read/write ports, LSQ = 8 entries, conservative (no speculative replay), stores commit only at retirement.**
**Exceptions: reuse the misprediction-recovery path, gated on in-order retirement; specified but not exercised by this project's target workloads.**
**Packer: instruction grouping and branch isolation only — no scheduling, no dependency analysis. All hazard resolution is dynamic, in hardware.**

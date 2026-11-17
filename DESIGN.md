# Design Decisions

Running log of architectural decisions for QUADRA, with the reasoning behind each.

**2026-11-18 — Architecture pivot (static VLIW → dynamically-scheduled OoO core).** QUADRA moves from a statically-scheduled VLIW core (the packer does all hazard/scheduling work; hardware never stalls on its own) to a dynamically-scheduled out-of-order core (hardware performs renaming, in-flight scheduling, and memory disambiguation at runtime). Driven by two things: the project's actual priority is a strong, well-verified hardware design, with the software side explicitly secondary; and the supervisor's stated emphasis on out-of-order techniques. Sections below describe the new model. Superseded static-VLIW content (modulo scheduling, fixed-distance forwarding paths) has been removed rather than kept inline — see git history for the prior design.

**Naming note:** the project name, "QUad-issue Architecture with Dependency-aware RISC-V Assembler," was coined for the static design, where the assembler/packer did the dependency analysis. Under this pivot the packer's job shrinks to grouping and branch isolation (Section 1) — the "dependency-aware" half of the name now describes the hardware's renaming/scheduling/LSQ logic, not the assembler. Flagging this rather than silently letting it drift.

## 1. Fetch & Packer — Reduced Scope

**Fetch/dispatch width: 4 general-purpose instructions per cycle, or exactly 1 branch/jump alone.** The packer's only remaining responsibilities:

1. Detect every branch/jump and emit it alone in its own fetch group (generalizes the old JAL-alone rule, Section 4, to all control-flow instructions).
2. Otherwise, greedily group up to 4 general-purpose (ALU/MEM) instructions per group, in program order, with **zero dependency analysis**.

All scheduling intelligence — dependency tracking, hazard avoidance, memory disambiguation — now lives in hardware (Sections 2–5). The packer no longer needs to prove anything about safety; it just groups instructions and flags control flow.

**Superseded:** modulo scheduling / software pipelining (the previous Tier-4 packer algorithm — MII computation, kernel/prologue/epilogue generation, the vector-add worked trace) is no longer needed. The OoO hardware discovers and exploits ILP dynamically every cycle, and adapts to real latencies — including cache misses — that a static schedule could never see. This also resolves, as a side effect, the store→load memory-aliasing question raised against the old packer (a software-pipelined schedule could move a later-iteration load ahead of an earlier-iteration's aliasing store, and the packer had no way to prove that safe): the LSQ (Section 5) disambiguates addresses dynamically and correctly at runtime, so nothing needs to be proven at compile time anymore.

**Branch groups waste dispatch bandwidth on purpose, for simplicity.** When a branch is dispatched, it occupies the entire 4-wide dispatch slot that cycle — the other 3 slots go unused that cycle, rather than having dispatch logic merge in instructions from the next group to fill them. Consistent with every other "accept a small efficiency loss for clean, verifiable hardware" choice in this design.

## 2. Register Renaming & Physical Register File

**Issue width: 4-wide, from the first implementation.** Not staged — the register-file port sizing below already assumes the full 4-wide target, so a narrower interim build buys nothing architecturally, and the real risk of a staged "2-wide now, widen later" plan is that the widening never happens before the deadline.

**Read ports: 8.** Issue width is capped at 4 *total* instructions per cycle — general ops and a branch compete for the same 4 slots, not a bonus 5th slot. 4 issuing instructions × 2 operands = 8R. This settles the static design's open "8R vs. 10R" caveat (which existed because the packer couldn't always guarantee a branch's compare rode free on some other unit): under dynamic issue, the branch simply occupies one of the 4 slots like anything else, so reads never exceed 8.

**Write ports: 4** — unchanged from the static design. ALU0, ALU1, MEM0-as-load, MEM1-as-load are the only possible writers; stores and branches never write. JAL's return-address write still rides one of these 4 existing ports.

**Physical register count: 64.** Sized as architectural count (32) + max in-flight renames (ROB size = 16, Section 3) = 48 minimum, rounded up to 64 for clean 6-bit physical-register tags and some slack.

**Decision: monolithic 8-read / 4-write physical register file, 64 physical registers, renamed from 32 architectural registers via a Register Alias Table (RAT).**

### Renaming mechanics — dual RAT, for misprediction recovery

Two rename tables, not one:

- **Speculative RAT** — updated immediately at dispatch, for every instruction (including ones that later turn out to be on a wrong path).
- **Architectural (retirement) RAT** — updated only when an instruction *retires* from the ROB, in program order, after being confirmed correct.

Because retirement is strictly in-order and a branch can never retire before it resolves, the architectural RAT can never have advanced past an unresolved branch — it is always sitting at a safe, restorable state by construction. **Recovery on a misprediction is a single bulk copy: architectural RAT → speculative RAT.** No per-branch checkpoint storage, no multi-cycle walk-back, and it works correctly regardless of how many branches happen to be in flight at once.

### Physical register reclaim

Each ROB entry records the **old** physical register mapping it replaced when it renamed its destination. That old physical register returns to the free list **at this entry's own retirement** — not when its consumers retire, not at execution, and not earlier. Freeing it earlier risks a WAR-hazard bug (the instruction could still be flushed, and the rename must be undoable); freeing it later than necessary just wastes physical registers.

## 3. ROB, Issue Queue & Dynamic Scheduling

**ROB: 16 entries. Issue Queue: 16 entries.** A lab-project-scale sweet spot — large enough to give the scheduler a real overlap window, small enough to keep the wakeup-broadcast comparators (cost scales roughly linearly with entry count × broadcast width) tractable to design and verify. The LSQ (Section 5) is kept smaller, since its address-comparator cost scales roughly quadratically instead.

**Selection policy: oldest-ready, resource-compatible.** Each cycle, any issue-queue entry whose operand tag matches a completing instruction's broadcast wakes up; the scheduler then selects up to 4 of the ready entries to issue, preferring older instructions (program order) to avoid starving them, and skipping any combination that would need more of a shared resource than exists (see MUL, below).

**Broadcast network (replaces static forwarding).** Every completing instruction broadcasts its destination physical-register tag and value on a common broadcast bus (CDB); any issue-queue entry waiting on that tag wakes up. This is a general any-producer-to-any-consumer network, not the static design's fixed-distance `MEM→EX` / `WB→EX` / `WB→D` paths — those are **superseded**, since there's no longer a fixed bundle-to-bundle distance between a producer and its consumer to build a static path for.

**MUL: lives in ALU0 only, not replicated.** A resource constraint the dynamic scheduler enforces directly — it never selects two ready MUL ops in the same cycle, since only one physical unit can execute one. Chosen for area savings: MUL isn't common enough in the target workloads (vector add, dot product, FIR, matmul) to justify a second multiplier.

## 4. Branch Handling & Misprediction Recovery

**Every branch/jump is fetched/dispatched alone** in its own single-instruction group (Section 1) — generalizes the old JAL-only rule to all control flow.

**Full speculation; backward branches predicted taken** — same zero-extra-hardware static rule as the prior design (compare target address to branch address at decode, no history, no dynamic predictor). This matters specifically because the target workloads are loop-dominated: a loop-closing branch is "taken" on every pass but the last.

**Misprediction recovery:** flush every ROB and issue-queue entry younger than the mispredicted branch, restore the speculative RAT from the architectural RAT in one bulk copy (Section 2), and return any physical registers held by the flushed entries to the free list.

**JAL:** still dispatched alone; its return-address write still rides one of the 4 existing completion/write ports (Section 2) — no new port needed.

## 5. Memory Ports & Load-Store Queue (LSQ)

**2 generic read/write memory ports — unchanged from the static design.** This was never really a static-vs-dynamic-scheduling question; the coverage math and real-world precedent (Section 6) hold regardless of how instructions get scheduled.

**LSQ: 8 entries** — intentionally smaller than the ROB/IQ (16). Every memory op is also a ROB entry, but memory ops are a minority of the mix (~35% combined load+store, Section 6), so 8 covers realistic bursts without over-building the address-comparator hardware, which scales roughly **quadratically** with LSQ size (every store potentially needs to compare its address against every other in-flight load/store) — much more expensive to grow than the IQ's linear wakeup cost.

**Policy: conservative, no speculative load execution.** A load stalls until every older, not-yet-resolved store's address is known — no replay machinery. If an address match is found against an older store whose data is ready, the store forwards its value directly to the load. Simpler, with far less verification surface than speculating a load past an unresolved store address and rolling it back later — matches the project's priority on clean, verifiable hardware over squeezing maximum throughput.

**Store commit timing: a store writes to real data memory only at retirement**, never speculatively — guarantees a flushed (wrong-path) store can never corrupt architectural memory state.

This also resolves the earlier open question about store→load aliasing across software-pipelined iterations (previously raised against the static packer, Section 1): the LSQ disambiguates addresses dynamically and correctly for every case, with no need for the packer to prove disjointness at compile time.

## 6. Workload Statistics (basis for Sections 2 & 5 — unchanged by the pivot)

**Instruction-mix data used:** load ≈ 26%, store ≈ 9%, other ≈ 65% (classic RISC integer-benchmark mix). Source: [WUSTL CSE560M Problem Set 1](https://classes.engineering.wustl.edu/cse560m/560M.f24/ProbSet1.pdf).

**Coverage of a random 4-instruction group, by memory-port design:**

| Design | Ports | Coverage |
|---|---|---|
| 1 generic port | 1 | ~38% (covers exactly 1 mem op) |
| 1 load port + 1 store port (fixed-function) | 2 | ~68% (fails when a group has 2 loads or 2 stores) |
| 2 generic R/W ports | 2 | ~87% (any mix of ≤2 mem ops) |
| 2 load ports + 1 store port | 3 | ~90% |

**Real-world precedent for favoring load bandwidth:** [Penn CIS 501 (Martin), "Superscalar Memory Access," slide 20](https://acg.cis.upenn.edu/milom/cis501-Fall10/lectures/07_superscalar.pdf) — multi-load/store bandwidth is "probably only necessary on processors 4-wide or wider"; Core i7 does 1 load + 1 store/cycle, Alpha 21164 does 2 loads *or* 1 store/cycle.

**Register-file port-sizing precedent:** [Penn CIS 501 (Martin), "Superscalar Register Read/Write," slide 23](https://acg.cis.upenn.edu/milom/cis501-Fall10/lectures/07_superscalar.pdf) — "Nominally, 2N read + N write... in reality, fewer than that: write ports — stores, branches don't write registers." Applied here under the 4-wide issue cap: 8R/4W (Section 2).

**Decision (unchanged):** 2 generic R/W memory ports, each independently configurable per cycle as a load or a store.

## Final Decision

**Fetch/dispatch: 4-wide general-purpose, or 1 branch/jump alone — no scheduling by the packer, only grouping + branch isolation (Section 1).**
**Register file: 8 read / 4 write ports, monolithic, 64 physical registers renamed from 32 architectural via a dual RAT — speculative (updated at dispatch) and architectural (updated at retirement) — recovered by a single bulk copy on misprediction (Section 2).**
**Scheduling: ROB = 16, Issue Queue = 16, 4-wide issue from the first implementation, oldest-ready + resource-compatible selection, general broadcast (CDB) wakeup replacing all static forwarding paths (Section 3).**
**MUL: single ALU (ALU0) only — a scheduler-enforced resource constraint, not a port issue (Section 3).**
**Branches: full speculation, backward-taken prediction, dual-RAT bulk-copy recovery on misprediction (Section 4).**
**Memory: 2 generic read/write ports (unchanged), LSQ = 8 entries, conservative (no speculative replay), stores commit only at retirement (Section 5).**
**Packer: reduced to instruction grouping and branch isolation — no scheduling, no dependency analysis, no memory-alias analysis. All hazard resolution — register and memory — is dynamic, in hardware (Sections 2–5).**

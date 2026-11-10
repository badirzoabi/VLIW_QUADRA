# QUADRA

**QU**ad-issue **A**rchitecture with **D**ependency-aware **R**ISC-V **A**ssembler

A custom VLIW (Very Long Instruction Word) processor built on top of the RISC-V instruction set, paired with a software packer/scheduler that turns ordinary sequential code into wide parallel instruction bundles.

## The idea

Modern superscalar processors spend a huge amount of hardware figuring out, at runtime, which instructions can safely run in parallel. VLIW machines take the opposite approach: the **software** decides what can run together ahead of time, and the **hardware** stays simple — it just executes whatever bundle it's handed, every slot, every cycle, no questions asked.

QUADRA explores that tradeoff hands-on, using RISC-V as the base instruction set instead of inventing one from scratch. The project has two halves that have to agree with each other perfectly:

1. **The processor** — a RISC-V-based core that fetches and executes a *bundle* of multiple instructions per cycle instead of one.
2. **The packer** — a software tool that takes normal, sequential RISC-V-style code, figures out which instructions are independent, and packs them into valid bundles — inserting NOPs or stalls wherever true parallelism isn't possible.

## Architecture overview

Each cycle, the processor fetches one **bundle** made of four fixed slots:

```
| ALU slot 0 | ALU slot 1 | MEM slot (load/store) | BRANCH slot |
```

- Two ALU slots execute independent arithmetic/logic operations in parallel.
- One memory slot handles a single load or store per bundle.
- One branch slot handles control flow.
- A shared, multi-port register file feeds all four slots and is written back by all of them in the same cycle.

The hardware does **no** dynamic hazard detection and **no** dynamic scheduling. It trusts that whatever the packer placed in a bundle is safe to execute simultaneously. All of the intelligence — dependency analysis, hazard avoidance, latency accounting — lives in software.

## The packer/scheduler

The packer is a standalone tool that:

- Parses sequential instructions written in a RISC-V-like syntax.
- Builds a dependency graph between instructions (read-after-write, write-after-read, write-after-write).
- Greedily schedules instructions into bundles, respecting:
  - Which functional unit each instruction needs (ALU / MEM / BRANCH)
  - How many cycles a slow operation (like a multiply or a load) takes before its result is ready
  - That no bundle issues two operations that conflict
- Fills any slot it can't use with a NOP.
- Emits the final bundle stream that the processor's instruction memory loads.

## Stretch goals: making it a real custom ISA, not just RISC-V in groups

The baseline above reuses plain RISC-V instructions arranged into fixed-width bundles. Two further extensions are planned to turn this into a genuinely custom VLIW instruction format:

- **Variable-length bundles ("stop bits").** Instead of always issuing a fixed 4-slot bundle (padded with wasted NOPs when fewer than 4 independent operations exist), each instruction carries a bit marking whether it's the last one in its bundle. This makes bundles only as wide as the available parallelism, shrinking code size and instruction-memory traffic.
- **Cross-slot long immediates.** A single instruction slot doesn't have room for a large constant. This extension lets one slot act as a pure data word that an adjacent instruction's slot uses as its immediate operand, allowing a full-width constant to be built in a single cycle instead of two separate instructions.

## Verification

The project includes a self-checking flow: a software reference model executes the same program the hardware runs, and the final register/memory state from simulation is automatically compared against it. This is paired with a hand-written test suite (vector operations, simple loops, dependency-heavy kernels) designed to stress the scheduler's hazard handling and the processor's bundle execution.

## Goals of the project

- Understand instruction-level parallelism from both the hardware and the compiler/toolchain side.
- Design and verify a non-trivial RTL core from scratch.
- Build real scheduling/dependency-analysis software, not just RTL.
- Practice a full verification methodology: self-checking tests, a golden reference model, and hazard-focused coverage.

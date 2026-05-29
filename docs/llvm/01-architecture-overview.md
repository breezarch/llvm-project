# 01 — Architecture Overview

## The Layered Design

LLVM follows a strict **layered architecture** inspired by the classic three-phase
compiler design (front-end → optimizer → back-end), but decomposed into much finer
granularity.  From the bottom up:

```
┌─────────────────────────────────────────────────────────────┐
│  TOOLS LAYER                                                │
│  opt, llc, lli, llvm-mc, llvm-objdump, llvm-ar, ...       │
│  (80+ command-line executables that compose libraries)      │
├─────────────────────────────────────────────────────────────┤
│  FRONTEND SUPPORT LAYER                                     │
│  OpenMP, OpenACC, HLSL, Offloading, Atomic                 │
│  (IR generation helpers for language frontends)             │
├─────────────────────────────────────────────────────────────┤
│  PASSES & PIPELINES LAYER  ─  LLVMPasses                    │
│  PassBuilder, standard -O[0123sz] pipelines                 │
├─────────────────────────────────────────────────────────────┤
│  OPTIMIZATION LAYER                                         │
│  LLVMScalarOpts, LLVMInstCombine, LLVMVectorize, LLVMIPO,  │
│  LLVMObjCARC, LLVMCoroutines, LLVMCFGuard, LLVMInstrumentation│
│  ─────────── rely on ───────────                            │
│  LLVMAnalysis  (alias analysis, dependence, domtree, etc.)  │
├─────────────────────────────────────────────────────────────┤
│  IR LAYER  ─  LLVMCore                                     │
│  Module, Function, BasicBlock, Instruction, Value, Use,    │
│  Type system, Constants, Metadata, DebugInfo, Attributes   │
│  ─────────── relies on ───────────                          │
│  LLVMSupport, LLVMADT (internal only), LLVMBinaryFormat     │
├─────────────────────────────────────────────────────────────┤
│  CODEGEN & MC LAYER                                         │
│  LLVMCodeGen (SelectionDAG, GlobalISel, regalloc, sched)   │
│  LLVMMC (assembler/disassembler/object emission)            │
│  LLVMMCA (micro-architectural analysis)                     │
├─────────────────────────────────────────────────────────────┤
│  TARGET LAYER  ─  25 backends                               │
│  LLVMX86, LLVMAArch64, LLVMAMDGPU, LLVMRISCV, ...         │
│  Each provides: TargetMachine, InstPrinter, AsmParser,     │
│  Disassembler, .td files → TableGen → C++ code              │
├─────────────────────────────────────────────────────────────┤
│  SUPPORT LAYER                                              │
│  LLVMSupport (OS, I/O, threading, RTTI), LLVMDemangle,     │
│  LLVMObject (ELF/MachO/COFF/XCOFF/GOFF readers),           │
│  LLVMBinaryFormat (shared constants/enums)                  │
└─────────────────────────────────────────────────────────────┘
```

Layers communicate through well-defined APIs; lower layers have **zero dependency**
on higher layers.  This is enforced at the CMake level: `LLVMCore` does not link
against `LLVMAnalysis`; `LLVMSupport` only links against `LLVMDemangle` and
platform libraries.

## The Compilation Data-Flow

A typical LLVM-based compilation proceeds through these IR representations:

```
Source Code (C/C++/Rust/Swift/...)
        │
        ▼
   [Clang / language frontend]
        │
        ▼
   LLVM IR (.ll / in-memory Module)   ← LLVMCore, LLVMAsmParser, LLVMIRReader
        │
        ▼
   [Pass Pipeline (-O0/-O1/-O2/-O3/-Os/-Oz)]  ← LLVMPasses, LLVMAnalysis, LLVMTransform*
        │  │
        │  ├── Inliner, Mem2Reg, GVN, LICM, SLP/Loop vectorize,
        │  │   InstCombine, SimplifyCFG, DCE, CSE, ...
        │  │
        │  ▼
   Optimized LLVM IR
        │
        ▼
   [TargetLowering]  ← LLVMCodeGen (SelectionDAG or GlobalISel)
        │
        ▼
   MachineInstr (MI) / MIR   ← LLVMCodeGen (MachineFunction, MachineBasicBlock)
        │
        ▼
   [Register Allocation, Scheduling, Prolog/Epilog]
        │
        ▼
   MC Layer   ← LLVMMC (MCInst, MCCodeEmitter, MCObjectStreamer)
        │
        ▼
   Object File (.o) / Assembly (.s)  ← ELF, MachO, COFF, Wasm, GOFF, XCOFF
```

This pipeline embodies the classic compiler design from Aho, Lam, Sethi & Ullman
(the "Dragon Book"), with explicit separation of the intermediate representation
(Chapter 6), code optimization (Chapters 9–10), and code generation (Chapter 8).

## Key Architectural Invariants

### 1. SSA Form (Static Single Assignment)

LLVM IR is in #SSA-form: every virtual register is assigned exactly once.
This is grounded in the seminal papers by Cytron et al. (1989, 1991) which proved
that SSA construction is O(n α(n)) for reducible flow graphs.  LLVM enforces SSA
via the `mem2reg` pass (promoting `alloca`/`load`/`store` to φ-nodes / `PHINode`)
and the verifier (`verifyFunction`).

### 2. The Use-Def Chain

Every `Value` in LLVM maintains a doubly-linked list of `Use` objects.
This is the #Use-Def-chain — a fundamental data structure for compiler
optimization.  When an instruction is RAUW'd (replaced-all-uses-with), the
`Use` list is walked and all references are updated.  This design comes from
the SSA literature and is implemented in `llvm/include/llvm/IR/Use.h`.

### 3. Pass Management

LLVM currently supports two pass managers:
- **Legacy PM** (`llvm/IR/LegacyPassManager.h`) — the original, still used by
  some backends and tools.
- **New PM** (`llvm/Passes/PassBuilder.h`) — the modern, analysis-caching design
  with explicit pass nesting and pipeline customization via `PassBuilder`.

The new PM uses a **lazy, caching analysis framework**: analyses are computed on
demand and invalidated when transforms modify the IR they depend on.  This is
inspired by the concept of **attribute grammars** and dependency-directed
computation (attribute evaluation).

### 4. Target Independence vs. Target Specialization

LLVM IR is strictly **target-independent**.  Target-specific knowledge enters
only at the CodeGen layer via:
- `TargetLowering` (legalization of types and operations)
- `TargetRegisterInfo` (register file description)
- `TargetInstrInfo` (instruction set description)
- `TargetFrameLowering` (stack frame layout)
- `.td` (TableGen) files that declaratively describe the ISA

This separation mirrors the #retargetable compiler concept from the 1980s
(e.g., the PQCC project at CMU), but LLVM achieves it with a practical,
production-quality engineering approach.

## Component Dependency Graph (Simplified)

```
LLVMSupport ◄── (almost everything)
LLVMBinaryFormat ◄── LLVMCore, LLVMMC, LLVMObject
LLVMADT (internal-only headers) ◄── all of llvm/
LLVMIR (LLVMCore) ◄── LLVMAnalysis, LLVMTransform*, LLVMCodeGen, LLVMLinker
LLVMAnalysis ◄── LLVMTransform*, LLVMCodeGen, LLVMPasses
LLVMMC ◄── LLVMCodeGen, LLVMTarget (all 25)
LLVMObject ◄── LLVMDebugInfo*, LLVMLTO, many tools
LLVMTargetParser ◄── LLVMTarget, LLVMMC
```

Note that `LLVMSupport` depends only on `LLVMDemangle` (and platform libraries)
— it's essentially the foundation.  `LLVMCore` depends on `LLVMSupport`,
`LLVMBinaryFormat`, `LLVMDemangle`, and `LLVMRemarks` — the IR itself is
self-contained but builds on thin utility libraries.

## Computational Theory Connections

| LLVM Concept | Theoretical Foundation |
|---|---|
| **SSA construction** | Cytron et al. (1989, 1991) — dominance frontiers and φ-node placement |
| **Dominator tree** | Lengauer & Tarjan (1979) — near-linear time algorithm |
| **Alias analysis** | Andersen (1994), Steensgaard (1996) — pointer analysis; Landi & Ryder — flow-sensitive analysis |
| **Data-flow analysis** | Kildall (1973) — monotone data-flow framework (the lattice-theoretic basis for all forward/backward analyses) |
| **Loop detection** | Havlak (1997) — nesting forests from DJ-graphs |
| **Register allocation** | Chaitin (1981 — graph coloring), Poletto & Sarkar (1999 — linear scan), Hack & Goos (2006 — SSA-based) |
| **Instruction scheduling** | List scheduling with critical-path heuristics; software pipelining (Rau, 1994) |
| **Instruction selection** | Tree-pattern matching (Aho & Johnson, 1976 — optimal O(n)), DAG covering (Pelegrí-Llopart & Graham, 1988) |
| **Inlining** | Call-graph analysis; SCC-based bottom-up (Hall, 1991); partial inlining |
| **Vectorization** | Allen & Kennedy (1987 — dependence-based), loop vectorization, SLP (superword-level parallelism by Larsen & Amarasinghe, 2000) |
| **LTO / ThinLTO** | Inter-modular optimization (IPO), whole-program analysis; summary-based cross-module analysis |
| **Automatic differentiation** | Not in LLVM core, but Clad/Enzyme use LLVM for adjoint-mode AD |
| **Fuzzing (libFuzzer)** | Coverage-guided evolutionary fuzzing (Miller et al., 1990; AFL-style feedback loops) |

## Information Flow in a Build

When you run `clang -O2 foo.c -o foo`, here is what happens inside LLVM:

1. **Clang** → generates LLVM IR from the AST
2. **PassBuilder** → builds the `-O2` pipeline:
   - ModuleSimplificationPipeline (inliner, global opt, ...)
   - ModuleOptimizationPipeline (CGSCC passes, loop opts, ...)
   - late LTO pipeline (if applicable)
3. **Optimized IR** → fed to the target backend (e.g., X86)
4. **TargetLowering** → legalizes types/operations for X86
5. **Instruction Selection** → IR instructions → MachineInstrs (via SelectionDAG or GlobalISel)
6. **Register Allocation** → virtual regs → physical regs (greedy/basic)
7. **Prolog/Epilog Insertion** → stack frame setup
8. **MC Layer** → MachineInstr → MCInst → assembly or object file

Each of these stages is itself composed of many individual passes — the modular
design means you can insert, remove, or reorder passes at any point.
